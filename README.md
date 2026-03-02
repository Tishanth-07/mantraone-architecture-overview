<div align="center">

<img src="https://img.shields.io/badge/Status-🚧 In Development-yellow?style=for-the-badge" />
<img src="https://img.shields.io/badge/Type-Multi--Tenant%20SaaS-purple?style=for-the-badge" />
<img src="https://img.shields.io/badge/Architecture-B2B2C%20E--Commerce-blue?style=for-the-badge" />
<img src="https://img.shields.io/badge/Language-TypeScript-3178C6?style=for-the-badge&logo=typescript" />
<img src="https://img.shields.io/badge/Frontend-Next.js%2014-000000?style=for-the-badge&logo=next.js" />
<img src="https://img.shields.io/badge/Backend-Node.js%20%2B%20Express-339933?style=for-the-badge&logo=node.js" />
<img src="https://img.shields.io/badge/Database-PostgreSQL-4169E1?style=for-the-badge&logo=postgresql" />
<img src="https://img.shields.io/badge/Cloud-AWS%20S3-FF9900?style=for-the-badge&logo=amazon-aws" />

<br /><br />

# 🏢 MantraOne

### *One platform. Every store. Zero compromise.*

**A production-grade, multi-tenant B2B2C e-commerce SaaS platform built on the "White-Glove" Agency Model**

*Multi-Tenant · Zero-Trust Security · AWS S3 Direct Upload · Atomic Transactions · CRM Engine*

---

[🏗 Architecture](#-system-architecture) · [✨ Features](#-core-implementations) · [🛠 Tech Stack](#-tech-stack) · [🔐 Security](#-security-model) · [🚀 Getting Started](#-getting-started) · [📊 Status](#-project-status)

</div>

---

> **📌 Note for Recruiters & Engineering Managers:** MantraOne is maintained in a **private repository** as proprietary, commercial SaaS software. This document serves as an **architectural case study** demonstrating system design depth, security engineering, cloud infrastructure decisions, and production problem-solving. Code samples and a live demo are available on request.

---

## 💡 About the Project

MantraOne is a premium, multi-tenant B2B2C e-commerce SaaS platform built on a **"White-Glove" Agency Model**. Rather than allowing open self-registration, a **Super Admin** provisions and manages all tenant accounts — ensuring every shop owner on the platform is vetted and onboarded through a secure, controlled process.

Each provisioned **Tenant (Shop Owner)** receives a fully isolated Admin Dashboard to manage their catalog, customers, and orders. **End Customers** interact with a branded, secure Storefront that is completely scoped to their respective tenant — they never see, or can access, another tenant's data.

This project demonstrates engineering decisions that go beyond "make it work" — every layer addresses a real security or performance concern: price tampering prevention, event-loop protection, atomic database writes, and structurally enforced multi-tenant isolation.

---

## 🏗️ System Architecture

MantraOne follows a strict **4-tier architecture** with three isolated user-access planes, a centralized API gateway with tenant-aware middleware, a relational data layer with ORM-enforced isolation, and a cloud storage layer with a server-bypass upload flow.

<br />

![MantraOne System Architecture](./mantraone-architecture.svg)

<br />

### Architecture Overview

**Three distinct actors** enter through separate frontend zones, each scoped to their role:

| Actor | Entry Point | Scope |
|---|---|---|
| 👑 Super Admin | Super Admin Panel | Platform-wide — provisions tenants, manages accounts |
| 🏪 Tenant (Shop Owner) | Admin Dashboard | Single-tenant — catalog, CRM, orders, analytics |
| 🛒 End Customer | Public Storefront | Read + checkout — isolated to their tenant's store |

All three zones share a common security layer: a custom `<AuthGuard>` component wrapping every protected route, and global **Axios JWT interceptors** that inject tokens into every outgoing request and handle `401 Unauthorized` refresh/redirect flows automatically.

---

## ✨ Core Implementations

### 1. 🔒 Strict Multi-Tenancy & Data Isolation

Logical data isolation is the architectural backbone of the platform. Every core model — `User`, `Category`, `Product`, `Customer`, `Order`, `Transaction` — carries a `tenantId` foreign key. Every Prisma ORM query enforces `where: { tenantId }` at the **database level**, making cross-tenant data leaks structurally impossible — not just a policy, but an engineering guarantee.

```typescript
// Every service method enforces tenantId at the query level
const products = await prisma.product.findMany({
  where: { tenantId: req.tenantId },  // injected from verified JWT
});
```

**Why this matters:** Row-level security enforced at the ORM layer means a misconfigured controller cannot accidentally expose another tenant's data. Isolation is not opt-in — it is the only way the system can query.

---

### 2. 🛡️ Zero-Trust Storefront Checkout

Client-side cart total calculations are a well-known attack surface — any user with browser dev tools can manipulate the price sent to the server. MantraOne's checkout API eliminates this entirely:

- **Ignores all pricing data** sent from the frontend client
- **Re-queries PostgreSQL** for real-time prices on every item at checkout time
- Wraps three critical operations in a single **`prisma.$transaction()`** — guaranteeing atomicity under concurrent load:

```typescript
await prisma.$transaction([
  prisma.order.create({ data: orderData }),           // 1. Create order record
  prisma.transaction.create({ data: financialLog }),  // 2. Log financial transaction
  prisma.product.update({                             // 3. Decrement stock
    where: { id: productId },
    data: { stockQuantity: { decrement: quantity } },
  }),
]);
```

**Why this matters:** If any one of these three writes fails (e.g., insufficient stock, DB timeout), the entire checkout rolls back — no orphaned orders, no phantom stock decrements, no unsynchronized financial logs.

---

### 3. ☁️ AWS S3 Presigned URL Infrastructure

Routing large, multi-image product uploads through the Node.js server would block the **single-threaded event loop** and degrade API response times for all tenants simultaneously. MantraOne solves this by bypassing the server entirely in the upload path:

```
Step 1:  Next.js Client  ──→  Express API          (request presigned URL)
Step 2:  Express API     ──→  Next.js Client        (return time-limited S3 URL)
Step 3:  Next.js Client  ──→  AWS S3 directly       (PUT multipart/form-data)
         ↑ Node.js server is completely out of the upload path at Step 3
```

**Infrastructure resolved:** Configured precise AWS S3 **CORS policies** (allowing `PUT`, `GET`, `POST`, `HEAD`, `DELETE` from the frontend origin) and scoped **IAM inline policies** (`s3:PutObject` mapped strictly to `arn:aws:s3:::mantraone/*`) to overcome browser security restrictions without over-permissioning the IAM role.

---

### 4. 🔑 White-Glove Auth & Route Protection

Public registration is completely disabled by design. The onboarding flow is:

1. Super Admin provisions a new tenant account via the admin panel
2. System generates a **one-time `activationToken`** and dispatches a secure email activation link
3. Tenant follows the link to set their password — **no plain-text passwords are ever stored or transmitted**
4. All subsequent sessions are managed via **JWT access + refresh token** rotation

Every frontend route is protected by a custom Next.js **`<AuthGuard>`** component, and all API calls use global **Axios interceptors** to inject the current access token and silently handle token refresh before expiry.

**Engineering fix documented:** Resolved an auth **infinite-loop race condition** caused by async token validation racing against the React render cycle. Resolution: `try/catch/finally` blocks to explicitly release `isLoading` state regardless of validation outcome, combined with corrected Next.js Route Group layout caching behavior.

---

### 5. 📊 CRM & Data Engineering

**Server-side aggregation:** The CRM engine uses Prisma `_sum` and `_count` aggregations to calculate *Customer Lifetime Value (LTV)*, *revenue metrics*, and *Last Order Date* dynamically on the server — eliminating the need to ship raw order arrays to the client and compute in the browser.

```typescript
const customerMetrics = await prisma.order.aggregate({
  where: { customerId, tenantId },
  _sum: { totalAmount: true },   // LTV
  _count: { id: true },          // Total orders
});
```

**Advanced frontend state:** Data tables feature **debounced search**, status-tab filtering with automatic pagination reset, and **optimistic UI updates** — deleted rows are removed from the UI instantly without a hard page reload, giving a native-app feel in the browser.

---

## 🛠️ Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Frontend** | Next.js 14 (App Router) | SSR + CSR hybrid, file-based routing |
| **UI** | React + Tailwind CSS + Lucide React | Component library + utility-first styling |
| **State / Data** | Axios + JWT Interceptors | Token injection, 401 handling, API client |
| **Backend API** | Node.js + Express.js (REST) | API gateway + business logic |
| **ORM** | Prisma ORM | Type-safe DB queries + migrations |
| **Database** | PostgreSQL | Relational data + transactional integrity |
| **Cloud Storage** | AWS S3 (Presigned URLs) | Direct-upload image storage |
| **Auth** | JWT (access + refresh) + `<AuthGuard>` | Stateless auth + route protection |
| **Language** | TypeScript (full-stack) | End-to-end type safety |

---

## 🔐 Security Model

| Threat | Attack Vector | Mitigation |
|---|---|---|
| **Cross-tenant data leak** | Missing tenantId filter on a query | `tenantId` enforced at ORM level on every model — structurally impossible to bypass |
| **Price tampering at checkout** | Client sends manipulated cart total | Zero-Trust checkout ignores all client pricing; re-queries DB for every price |
| **Partial checkout failure** | DB write fails mid-transaction | `prisma.$transaction()` rolls back all three writes atomically |
| **Account enumeration / spam** | Open public registration | No self-registration; accounts provisioned only by Super Admin |
| **Plain-text credential exposure** | Activation email contains password | One-time `activationToken` link only; user sets password themselves |
| **Auth infinite loop / flash** | Async validation racing React render | `try/catch/finally` pattern explicitly releases loading state; no race condition |
| **Event loop blocking on uploads** | Large file uploads through Node.js | Presigned URL flow; Node.js server is bypassed entirely for upload data |
| **Over-permissioned S3 access** | Broad IAM role on S3 bucket | IAM scoped to `s3:PutObject` on `arn:aws:s3:::mantraone/*` only |

---

## 📊 Project Status

**Phase 1 — ✅ Complete**
```
✅  Multi-tenant database engine & Prisma schema (all core models)
✅  White-Glove auth flow (activation token, JWT, route guards)
✅  Super Admin panel (tenant provisioning & management)
✅  Tenant Admin — full catalog CRUD (products, categories)
✅  AWS S3 presigned URL direct-upload infrastructure
✅  Zero-Trust atomic checkout API
✅  CRM engine (LTV, revenue aggregation, order history)
✅  Optimistic UI updates on data tables
```

**Phase 2 — 🚧 In Progress**
```
🔄  Visual dashboard analytics (revenue charts, order trends)
🔄  Dynamic public Storefront UI for end-customers
⏳  Push notifications for order status updates
⏳  Tenant-level custom domain support
```

---

## 🚀 Getting Started

> **Note:** This is a private commercial repository. The setup guide below is provided for architectural reference and interview/review purposes.

### Prerequisites

- Node.js 18+
- PostgreSQL 14+
- pnpm (`npm install -g pnpm`)
- AWS account with an S3 bucket and scoped IAM credentials

### Environment Variables

**Backend `.env`:**
```env
DATABASE_URL="postgresql://user:password@localhost:5432/mantraone"
JWT_SECRET="your-access-token-secret"
JWT_REFRESH_SECRET="your-refresh-token-secret"
AWS_ACCESS_KEY_ID="your-aws-access-key"
AWS_SECRET_ACCESS_KEY="your-aws-secret-key"
AWS_REGION="ap-south-1"
AWS_S3_BUCKET="mantraone"
PORT=4000
```

**Frontend `.env.local`:**
```env
NEXT_PUBLIC_API_URL="http://localhost:4000"
```

### Database Setup

```bash
# Run migrations
npx prisma migrate dev --name init

# Seed super admin account
npx prisma db seed
```

---

## 📁 Project Structure

```
MantraOne/
│
├── mantraone-backend/                   # Node.js + Express REST API
│   ├── prisma/
│   │   ├── schema.prisma                # Multi-tenant schema (all core models)
│   │   └── migrations/                  # Version-controlled DB migrations
│   ├── src/
│   │   ├── middleware/
│   │   │   ├── authMiddleware.ts        # JWT verification + tenantId extraction
│   │   │   └── tenantMiddleware.ts      # Attaches tenantId to request context
│   │   ├── routes/
│   │   │   ├── auth.routes.ts           # /auth — login, activation
│   │   │   ├── product.routes.ts        # /products — catalog CRUD
│   │   │   ├── category.routes.ts       # /categories
│   │   │   ├── customer.routes.ts       # /customers — CRM
│   │   │   ├── order.routes.ts          # /orders
│   │   │   ├── checkout.routes.ts       # /checkout — Zero-Trust atomic checkout
│   │   │   └── upload.routes.ts         # /upload — S3 presigned URL generator
│   │   ├── services/
│   │   │   ├── checkout.service.ts      # Atomic transaction logic
│   │   │   ├── crm.service.ts           # LTV + revenue aggregation
│   │   │   └── s3.service.ts            # Presigned URL generation
│   │   ├── controllers/                 # Route handlers (thin, delegate to services)
│   │   ├── types/                       # Shared TypeScript interfaces
│   │   └── app.ts                       # Express app factory
│
└── mantraone-frontend/                  # Next.js 14 App Router
    ├── app/
    │   ├── (super-admin)/               # Super Admin protected zone
    │   ├── (admin)/                     # Tenant Admin protected zone
    │   ├── (storefront)/                # Public storefront per tenant
    │   └── (auth)/                      # Activation + login pages
    ├── components/
    │   ├── AuthGuard.tsx                # Route protection wrapper
    │   └── ui/                          # Shared UI components
    ├── lib/
    │   ├── axios.ts                     # Global Axios instance + JWT interceptors
    │   └── auth.ts                      # Token utilities
    └── next.config.js
```

---

## 🧠 Engineering Decisions

**Why Presigned URLs instead of multer?** Multer processes uploads synchronously through the Node.js event loop. On a multi-tenant platform under concurrent load, a single large upload from one tenant degrades response times for all other tenants. Presigned URLs remove the server from the upload path entirely — S3 receives the binary directly from the browser.

**Why `prisma.$transaction()` for checkout?** Three database writes must either all succeed or all fail. Without a transaction, a server crash between creating the order and decrementing stock results in an oversold product and a missing financial log. The transaction guarantee makes this scenario impossible.

**Why disable public registration?** The White-Glove model is a deliberate business decision. Controlled onboarding means every tenant is known, vetted, and properly set up. It also eliminates an entire class of abuse vectors (bot accounts, trial abuse, credential stuffing entry points) at zero engineering cost.

**Why `tenantId` at the ORM layer instead of application logic?** Application-level filtering is fragile — a missed `if` statement in one controller leaks data. ORM-level filtering on every query is a structural guarantee that cannot be accidentally bypassed by a future developer adding a new route.

---

## 🗺️ Roadmap

- [x] Multi-tenant schema + data isolation engine
- [x] White-Glove auth (activation tokens, JWT rotation, route guards)
- [x] Tenant Admin — catalog, CRM, orders
- [x] Zero-Trust atomic checkout
- [x] AWS S3 presigned direct-upload infrastructure
- [ ] Revenue analytics dashboard (charts + KPIs)
- [ ] Dynamic public Storefront with tenant branding
- [ ] Order status push notifications
- [ ] Tenant-level custom domain routing
- [ ] End-to-end test suite (Jest + Supertest + Playwright)

---

## 📬 Contact

<div align="center">

**Tishanth Sivakumar** — Software Engineering Student · Colombo, Sri Lanka 🇱🇰

*Available for Software Engineering internships — open to remote and on-site opportunities*

[![Email](https://img.shields.io/badge/Email-tishanthsivakumar007%40gmail.com-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:tishanthsivakumar007@gmail.com)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-tishanth--t007-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/tishanth-t007/)

</div>

---

<div align="center">

*Architected and developed by Tishanth Sivakumar · Colombo, Sri Lanka 🇱🇰*

*Full-Stack · TypeScript · Next.js · Node.js · PostgreSQL · AWS · Multi-Tenant SaaS*

</div>
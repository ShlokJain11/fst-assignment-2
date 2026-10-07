# Automated Relational Data Seeding, Transactional Event Notification & Secure Full-Stack Endpoints

A unified full-stack architecture demonstrating **Prisma ORM automated relational mock data pipelines**, **Next.js Edge Middleware Role-Based Proxy Gates**, and **Resend Transactional Lifecycle Email Dispatch with Webhook Ingestion**.

---

## 🏗️ Architectural Overview

```
                                    ┌────────────────────────────┐
                                    │    Client Inbound HTTP     │
                                    └─────────────┬──────────────┘
                                                  │
                                                  ▼
                         ┌───────────────────────────────────────────────┐
                         │   Next.js Edge Middleware Proxy Gate Layer    │
                         │             (middleware.ts)                   │
                         │  - Decrypts HTTP-only JWT (jose / WebCrypto)  │
                         │  - RBAC Matrix: Admin vs Member vs Guest      │
                         │  - Attaches x-user-id, x-user-role headers    │
                         └───────┬───────────────┬───────────────┬───────┘
                                 │               │               │
                     [ADMIN ONLY]│       [MEMBER]│        [PUBLIC│GUEST]
                                 ▼               ▼               ▼
                        ┌─────────────────┐ ┌───────────────┐ ┌───────────────┐
                        │ /admin, /api/   │ │ /transactions │ │ /login, /     │
                        │ admin/metrics   │ │ Server Actions│ │ /unauthorized │
                        └────────┬────────┘ └───────┬───────┘ └───────────────┘
                                 │                  │
                                 │                  ▼
                                 │      ┌─────────────────────────────┐
                                 │      │  Prisma Database Mutation   │
                                 │      │  (Transactions / Users)     │
                                 │      └───────────┬─────────────────┘
                                 │                  │
                                 │                  ▼ (Lifecycle Hook)
                                 │      ┌─────────────────────────────┐
                                 │      │ Resend API + React Email    │
                                 │      │ (TransactionAlertEmail.tsx) │
                                 │      └───────────┬─────────────────┘
                                 │                  │
                                 ▼                  ▼
                        ┌─────────────────────────────────────────────┐
                        │  Relational Database (Prisma SQLite/Postgres│
                        │  - Roles (ADMIN, MEMBER, GUEST)             │
                        │  - Users (Faker localized accounts)         │
                        │  - Transactions (FK cascade to User)        │
                        │  - AuditLogs (Immutable security stream)    │
                        │  - EmailLogs (Resend lifecycle & Webhooks)  │
                        └──────────────────────▲──────────────────────┘
                                               │
                                 ┌─────────────┴─────────────┐
                                 │ Inbound Webhook Endpoint  │
                                 │ /api/webhooks/resend      │
                                 │ (delivered, bounced, etc.)│
                                 └───────────────────────────┘
```

---

## 🚀 Quick Start & CLI Pipeline

### 1. Installation
```bash
npm install
```

### 2. Automated Relational Database Reset & Seeding Pipeline (Part A)
Run the automated CLI workflow command to reset the database, push migrations, and execute the localized relational seeding script:
```bash
npm run db:pipeline
```
Or:
```bash
npm run db:reset-seed
```

This single command:
1. Resets the database schema.
2. Synchronizes Prisma models.
3. Populates 3 base roles (`ADMIN`, `MEMBER`, `GUEST`).
4. Generates 3 deterministic anchor personas (`admin@platform.dev`, `member@platform.dev`, `guest@platform.dev`).
5. Generates 15 localized synthetic users via `@faker-js/faker`.
6. Seeds 60+ transactions preserving foreign-key integrity.
7. Populates 50+ relational audit logs and email lifecycle logs.

### 3. Launch Development Server
```bash
npm run dev
```
Open [http://localhost:3000](http://localhost:3000) to view the Mission Control Dashboard.

---

## 📋 Task Breakdown Implementation

### Part A — Relational Schema Modeling & Automated Mock Data Pipeline (CO4)
- **Normalized Prisma Schema (`prisma/schema.prisma`)**:
  - `Role`: Multi-tier access management (`ADMIN`, `MEMBER`, `GUEST`).
  - `User`: Accounts with role foreign-key (`roleId`).
  - `Transaction`: Referential FK to `userId` with cascade deletion, statuses (`PENDING`, `COMPLETED`, `FAILED`, `REFUNDED`), currencies, and reference codes.
  - `AuditLog`: Security and event ledger recording action, actor `userId`, details, and IP address.
  - `EmailLog`: Relational tracking of transactional notifications and Resend delivery statuses.
- **PostgreSQL Compatibility**: A dedicated PostgreSQL schema variant is provided at `prisma/schema.postgresql.prisma`.
- **Seeding Pipeline (`prisma/seed.ts`)**:
  - Uses `@faker-js/faker` with deterministic seeding for reproducibility.
  - Emits real-time execution statistics and guarantees FK constraint integrity.

### Part B — Authenticated Session Enforcement & Middleware Proxy Gates (CO3)
- **Edge Middleware (`middleware.ts`)**:
  - Intercepts requests at the Edge using WebCrypto `jose`.
  - Parses encrypted HTTP-only JWT cookies (`fst2_session_token`).
  - **RBAC Matrix**:
    - `/admin` & `/api/admin/*`: Restricted strictly to `ADMIN`. Non-admins redirected to `/unauthorized` or receive HTTP 403 JSON.
    - `/api/transactions`: Requires `MEMBER` or `ADMIN`. `GUEST` users receive HTTP 403 Forbidden.
  - **Proxy Header Enrichment**: Passes verified session claims (`x-user-id`, `x-user-role`, `x-user-email`) upstream.
- **Protected Route Handlers**:
  - `GET /api/admin/metrics`: Admin-only system telemetry and audit stream.
  - `GET/POST /api/transactions`: Role-gated transaction listing and creation.
- **Server Actions (`app/actions/transaction-actions.ts`)**:
  - `createTransactionAction`: Session-checked mutation creating transaction, audit log, and triggering email dispatch.
  - `switchRoleAction`: Instant session switching between Admin, Member, and Guest for live evaluation.

### Part C — Transactional Lifecycle Dispatch via Resend & React Email (CO3, CO4)
- **Modular React Email Template (`emails/TransactionAlertEmail.tsx`)**:
  - Constructed using `@react-email/components` (`Html`, `Head`, `Body`, `Container`, `Heading`, `Text`, `Hr`, `Button`).
  - Supports dark mode aesthetics and responsive formatting.
- **Resend Dispatch Engine (`lib/email/resend.ts`)**:
  - Dispatches email via Resend API when `RESEND_API_KEY` is present.
  - Includes a zero-config development mock fallback to record simulated delivery into `EmailLog` without network failures.
- **Webhook Ingestion Route Handler (`app/api/webhooks/resend/route.ts`)**:
  - Ingests Resend webhook events (`email.delivered`, `email.bounced`, `email.complained`, `email.opened`).
  - Updates matching `EmailLog` records in the database.
  - Writes an `AuditLog` entry documenting the webhook receipt.

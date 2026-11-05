# NEXIFY — Enterprise Multi-Vendor Marketplace
## Day 1: Project Foundation & Architecture Blueprint

---

## 1. Complete Repository / Folder Structure (Monorepo, Turborepo-managed)

```
nexify/
├── apps/
│   ├── backend/                     # Node.js API (see §5)
│   ├── mobile/                      # Flutter app (see §6)
│   ├── admin-panel/                 # React Admin (see §7)
│   └── seller-panel/                # React Seller (see §7)
├── packages/
│   ├── shared-types/                # Shared TS types/DTOs (backend ↔ web)
│   ├── ui-kit/                      # Shared React component library
│   ├── config/                      # Shared eslint/tsconfig/prettier
│   └── validation-schemas/          # Shared Zod/Yup schemas
├── infra/
│   ├── docker/
│   ├── terraform/                   # Cloud IaC
│   ├── kubernetes/                  # k8s manifests/helm charts
│   └── nginx/
├── .github/
│   └── workflows/                   # CI/CD pipelines
├── docs/
│   ├── architecture/
│   ├── api-contracts/
│   └── runbooks/
├── scripts/                         # DB migration/seed/deploy scripts
├── turbo.json
├── package.json
└── README.md
```

---

## 2 & 3. Clean + Modular Architecture (applies to every service)

Nexify backend follows **Clean Architecture** in 4 layers, wrapped in **feature modules** (Modular Monolith today → extractable to microservices later):

```
Presentation (Controllers/Routes)
        ↓
Application (Use Cases / Services)
        ↓
Domain (Entities / Business Rules — framework agnostic)
        ↓
Infrastructure (DB, Cache, Queue, 3rd-party — implements Domain interfaces)
```

Rule: **dependencies point inward.** Domain layer has zero knowledge of Express, PostgreSQL, or Redis. Infrastructure implements interfaces defined in Domain (Dependency Inversion).

Each business capability (`user`, `product`, `order`, `wallet`, `seller`, `payment`, `notification`, `coupon`, `review`) is a **self-contained module** with its own controller/service/repository/domain — enabling future extraction into independent microservices without rewriting business logic.

---

## 4. Project Directory Tree — Backend Detail

```
apps/backend/
├── src/
│   ├── modules/
│   │   ├── auth/
│   │   │   ├── domain/
│   │   │   │   ├── entities/
│   │   │   │   └── interfaces/
│   │   │   ├── application/
│   │   │   │   ├── use-cases/
│   │   │   │   └── dto/
│   │   │   ├── infrastructure/
│   │   │   │   ├── repositories/
│   │   │   │   └── providers/          # Firebase, JWT
│   │   │   └── presentation/
│   │   │       ├── controllers/
│   │   │       ├── routes/
│   │   │       ├── validators/
│   │   │       └── middlewares/
│   │   ├── user/
│   │   ├── seller/
│   │   ├── product/
│   │   ├── category/
│   │   ├── brand/
│   │   ├── cart/
│   │   ├── wishlist/
│   │   ├── order/
│   │   ├── payment/
│   │   ├── wallet/
│   │   ├── coupon/
│   │   ├── review/
│   │   ├── notification/
│   │   ├── banner/
│   │   ├── delivery/
│   │   ├── admin/
│   │   └── analytics/
│   │       └── (same 4-layer structure in every module)
│   ├── core/
│   │   ├── config/                     # env loader, constants
│   │   ├── database/                   # PG pool, migrations, seeders
│   │   ├── cache/                      # Redis client
│   │   ├── queue/                      # BullMQ / Kafka producers-consumers
│   │   ├── logger/                     # Winston/Pino + transports
│   │   ├── errors/                     # Custom error classes + handler
│   │   ├── security/                   # Helmet, rate-limit, sanitizers
│   │   ├── di/                         # Dependency Injection container
│   │   └── events/                     # Domain event bus
│   ├── shared/
│   │   ├── utils/
│   │   ├── constants/
│   │   └── types/
│   ├── jobs/                            # Cron / background workers
│   ├── app.ts                           # Express app assembly
│   └── server.ts                        # Entry point
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── migrations/
├── seeders/
├── .env.example
├── Dockerfile
└── package.json
```

---

## 5. Backend Architecture

- **Runtime:** Node.js 20 LTS + TypeScript (strict mode)
- **Framework:** Express.js (or Fastify for higher throughput — decision gate in roadmap)
- **Pattern:** Modular Monolith with Clean Architecture, designed for future Microservices split by module boundary
- **API Gateway (future):** Kong / AWS API Gateway once split occurs
- **Communication:** REST (v1) now, gRPC/internal events for inter-service calls post-split
- **Async Processing:** BullMQ (Redis-backed) for emails, OTP, notifications, report generation
- **Event-Driven Core:** internal Domain Event Bus (e.g., `OrderPlaced`, `SellerApproved`) decouples modules — this is what makes the later microservice split painless
- **Process Manager:** PM2 in cluster mode (multi-core utilization) behind Nginx/Load Balancer

---

## 6. Flutter Architecture (Customer App)

**Pattern:** Clean Architecture + Feature-first modularity, **BLoC** for state management (Cubit for simple screens, full BLoC for complex flows like checkout).

```
mobile/lib/
├── core/
│   ├── di/                    # get_it + injectable
│   ├── network/                # dio client, interceptors, refresh-token logic
│   ├── error/                  # Failure classes, exception mapping
│   ├── theme/
│   ├── router/                 # go_router
│   ├── constants/
│   └── utils/
├── features/
│   ├── auth/
│   │   ├── data/
│   │   │   ├── datasources/    # remote (dio) + local (secure storage)
│   │   │   ├── models/         # DTOs (fromJson/toJson)
│   │   │   └── repositories/   # implements domain interface
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   ├── repositories/   # abstract interfaces
│   │   │   └── usecases/
│   │   └── presentation/
│   │       ├── bloc/
│   │       ├── screens/
│   │       └── widgets/
│   ├── product/
│   ├── search/
│   ├── cart/
│   ├── wishlist/
│   ├── checkout/
│   ├── order_tracking/
│   ├── wallet/
│   ├── review/
│   ├── notification/
│   └── profile/
├── shared_widgets/
├── l10n/                       # i18n (Bangla/English)
└── main.dart
```

- **DI:** `get_it` + `injectable` (code-generated, no manual wiring)
- **Networking:** `dio` with interceptors for auth-token injection, refresh-token retry, and centralized error mapping
- **Local persistence:** `flutter_secure_storage` (tokens), `Hive`/`drift` (cache/offline cart)
- **Navigation:** `go_router` (deep-linking ready — needed for promo/notification links)
- **Testing:** `bloc_test`, `mocktail`, golden tests for widgets

---

## 7. React Architecture (Admin Panel & Seller Panel)

Both panels share the same architecture and reuse `packages/ui-kit`.

**Pattern:** Feature-Sliced Design + Clean layering, **React Query** for server state, **Zustand** for lightweight UI/client state (no Redux boilerplate needed at this scale).

```
admin-panel/src/
├── app/
│   ├── providers/              # QueryClientProvider, ThemeProvider, Router
│   ├── routes/                 # route definitions + guarded routes
│   └── store/                  # zustand root store
├── entities/                   # domain entities (Product, Order, Seller...)
│   └── product/
│       ├── api/                # react-query hooks (useProducts, useApproveProduct)
│       ├── model/               # types, zod schemas
│       └── ui/                  # entity-specific display components
├── features/                   # user-actionable features (ApproveSeller, CreateCoupon)
├── widgets/                    # composed UI blocks (Dashboard cards, DataTables)
├── pages/                      # route-level composition ONLY (no logic)
├── shared/
│   ├── api/                    # axios instance, interceptors
│   ├── ui/                     # design-system primitives
│   ├── lib/                    # hooks, formatters
│   └── config/
└── main.tsx
```

- **Server state:** React Query (caching, retries, optimistic updates for approvals)
- **Forms:** React Hook Form + Zod resolver (shared schemas from `packages/validation-schemas`)
- **Auth guard:** route-level HOC checking JWT role claims (`admin` vs `seller`)

---

## 8. Database Architecture (PostgreSQL)

- **Strategy:** Single PostgreSQL cluster (Primary + Read Replicas) at launch → sharding-ready via `tenant_id`/`seller_id` partition keys for future horizontal scale
- **Schema-per-domain convention:** logical schemas (`auth`, `catalog`, `orders`, `wallet`, `marketing`) inside one database for clarity without operational overhead of separate DBs
- **Connection pooling:** PgBouncer in front of the cluster
- **Partitioning:** `orders` and `notifications` tables partitioned by month (range partitioning) — critical for millions-of-rows scale
- **Replication:** Streaming replication, read replicas serve reporting/analytics queries (never hit primary)
- **Migrations:** Versioned, reversible migrations (Knex/Prisma Migrate) — no manual schema edits, ever
- **Backups:** Point-in-time-recovery (WAL archiving) + daily automated snapshots
- **Search offload:** Elasticsearch/OpenSearch synced via CDC (Debezium) from Postgres for product search — Postgres is never used for full-text search at scale
- *(No tables generated per your instruction — this section is structural strategy only)*

---

## 9. Authentication Flow

```
1. Client → Firebase Auth (Google Login / Phone OTP / Email verification)
2. Firebase returns idToken to client
3. Client → Nexify Backend /auth/verify  (sends Firebase idToken)
4. Backend verifies idToken via Firebase Admin SDK
5. Backend issues Nexify-owned tokens:
   - Access Token  (JWT, short-lived, 15 min)
   - Refresh Token (JWT, long-lived, 30 days, stored hashed in DB, rotated on use)
6. Access Token → Authorization: Bearer header on every API call
7. Refresh flow: on 401, client silently calls /auth/refresh with refresh token
   → old refresh token invalidated (rotation) → new pair issued
8. Role-Based Access Control (RBAC): JWT payload carries { userId, role: customer|seller|admin, permissions[] }
9. Middleware chain: authenticate → authorize(role) → validate → controller
```

Why Firebase + own JWT (hybrid): Firebase handles the hard parts (OTP delivery, Google OAuth, email verification infra) while Nexify retains full control of session lifecycle, RBAC, and revocation — required for enterprise-grade auth.

---

## 10. API Folder Structure (versioned, module-based)

```
/api/v1/auth/*
/api/v1/users/*
/api/v1/sellers/*
/api/v1/products/*
/api/v1/categories/*
/api/v1/brands/*
/api/v1/cart/*
/api/v1/wishlist/*
/api/v1/orders/*
/api/v1/payments/*
/api/v1/wallet/*
/api/v1/coupons/*
/api/v1/reviews/*
/api/v1/notifications/*
/api/v1/admin/*
/api/v1/analytics/*
```
Every route file registers: `validator middleware → auth middleware → rbac middleware → controller`. API versioning in the URL path (`/v1/`) allows non-breaking evolution as the marketplace matures.

---

## 11. Repository Pattern

- Domain layer defines **interfaces** (e.g., `IProductRepository`) with methods like `findById`, `findByFilters`, `save`.
- Infrastructure layer provides **concrete implementations** (`PostgresProductRepository`) using the query builder/ORM.
- Use-cases/services depend only on the interface (constructor injection) — enabling unit tests with in-memory/fake repositories and painless swap of persistence tech later.
- No controller or use-case ever writes raw SQL — all persistence access is funneled through repositories.

---

## 12. Service Layer

- **Application Services / Use Cases** orchestrate business workflows (e.g., `PlaceOrderUseCase` coordinates cart validation → payment → inventory decrement → wallet update → notification dispatch → event emission), each a single-responsibility class.
- Services never talk to Express (`req`/`res`) — fully framework-agnostic, testable in isolation.
- Cross-module coordination happens via the **Domain Event Bus**, not direct service-to-service calls, to keep modules decoupled.

---

## 13. Dependency Injection

- **Backend:** custom lightweight DI container (or `tsyringe`) — registers repositories, services, and providers as singletons/scoped instances; controllers receive fully-wired use-cases.
- **Flutter:** `get_it` (service locator) + `injectable` (compile-time code generation of registrations) — zero manual `GetIt.I.registerFactory` boilerplate.
- **React:** Context Providers for cross-cutting concerns (auth, theme); React Query handles data-layer "injection" via hooks.

Benefit: every layer is unit-testable with mocked dependencies, and swapping infrastructure (e.g., SendGrid → SES) touches one binding, not business logic.

---

## 14. State Management Architecture

| Platform | Server State | Client/UI State | Persistent/Offline |
|---|---|---|---|
| Flutter | BLoC/Cubit (per feature) | BLoC (UI-local) | Hive/drift + secure storage |
| React (Admin/Seller) | React Query | Zustand | localStorage (non-sensitive only) |

Rule: server-derived data is **never** duplicated into client state stores — React Query / BLoC repositories are the single source of truth, preventing sync bugs.

---

## 15. Environment Variables (`.env.example` — backend)

```
NODE_ENV=
PORT=
API_BASE_URL=

DATABASE_URL=
DATABASE_READ_REPLICA_URL=
REDIS_URL=

JWT_ACCESS_SECRET=
JWT_REFRESH_SECRET=
JWT_ACCESS_EXPIRY=
JWT_REFRESH_EXPIRY=

FIREBASE_PROJECT_ID=
FIREBASE_CLIENT_EMAIL=
FIREBASE_PRIVATE_KEY=

BKASH_APP_KEY=
BKASH_APP_SECRET=
NAGAD_MERCHANT_ID=
ROCKET_MERCHANT_ID=
STRIPE_SECRET_KEY=          # Visa/MasterCard via Stripe

AWS_S3_BUCKET=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=

SMTP_HOST= / SMTP_USER= / SMTP_PASS=
SENTRY_DSN=
LOG_LEVEL=

RATE_LIMIT_WINDOW_MS=
RATE_LIMIT_MAX=
```
Secrets never committed — `.env` is gitignored; production secrets injected via AWS Secrets Manager / Vault, not flat files.

---

## 16. Configuration Files (root-level)

- `turbo.json` — monorepo task pipeline (build/lint/test caching)
- `tsconfig.base.json` — shared TS compiler options, extended per app
- `.eslintrc.js` + `.prettierrc` — shared via `packages/config`
- `docker-compose.yml` — local dev stack (Postgres, Redis, backend, panels)
- `commitlint.config.js` — enforces Conventional Commits
- `.husky/` — pre-commit (lint-staged) + pre-push (test) hooks

---

## 17. Naming Conventions

- **Files:** `kebab-case` (`product-repository.ts`), Flutter: `snake_case.dart`, React components: `PascalCase.tsx`
- **Classes/Interfaces:** `PascalCase`, interfaces prefixed `I` (`IOrderRepository`)
- **Variables/functions:** `camelCase`
- **DB tables/columns:** `snake_case`, plural table names (`orders`, `order_items`)
- **Constants/env keys:** `SCREAMING_SNAKE_CASE`
- **Branches:** `type/module-short-description` (see §19)
- **API routes:** plural nouns, kebab-case (`/delivery-charges`)

---

## 18. Coding Standards

- Strict TypeScript everywhere backend/React (`noImplicitAny`, `strictNullChecks`)
- Flutter: effective_dart lint set + `very_good_analysis`
- Max function length ~40 lines; max cyclomatic complexity enforced via ESLint rule
- No business logic in controllers/widgets/pages — presentation layer stays thin
- 100% of public functions documented (TSDoc/DartDoc)
- Mandatory PR code review (min. 1 approval) + passing CI before merge

---

## 19. Git Branch Strategy

**Trunk-based with short-lived feature branches (Gitflow-lite):**

```
main                    → production, protected, tag-released
develop                 → integration branch, protected
feature/<module>-<desc> → e.g. feature/wallet-topup
bugfix/<module>-<desc>
hotfix/<desc>           → branches from main, merges to main + develop
release/x.y.z           → stabilization before prod
```
- All merges via PR only, squash-merged, no direct push to `main`/`develop`
- Semantic versioning (`MAJOR.MINOR.PATCH`) tags on `main`

---

## 20. Version Control Structure

- Monorepo (single source of truth, atomic cross-app commits) managed by Turborepo + pnpm workspaces
- Conventional Commits (`feat:`, `fix:`, `chore:`, `refactor:`) → auto-generates changelog via `semantic-release`
- `CODEOWNERS` file routes PR reviews to the right team (backend/mobile/web/devops)

---

## 21. Error Handling Strategy

- Custom error hierarchy: `AppError → (ValidationError, AuthError, NotFoundError, ConflictError, PaymentError...)`, each with HTTP status mapping
- Centralized Express error-handling middleware — single place formatting all error responses: `{ success:false, error:{ code, message, details } }`
- Never leak stack traces or internal messages to clients in production
- Flutter: `Either<Failure, Success>` (fpdart) returned from use-cases — UI never receives raw exceptions
- All errors correlated with a `requestId` (propagated via header) for cross-log tracing

---

## 22. Logging Strategy

- **Backend:** structured JSON logs via Pino/Winston → shipped to centralized log platform (ELK / CloudWatch / Datadog)
- **Log levels:** `error`, `warn`, `info`, `debug` — `debug` disabled in production
- **Audit Log (separate immutable store):** every sensitive action (seller approval, payout, refund, admin login, price change) written to an append-only `audit_logs` table/stream with actor, action, before/after diff, timestamp, IP
- **Request tracing:** correlation ID injected per request, propagated through async jobs and logs
- **Flutter/React:** Sentry/Crashlytics for crash + error reporting

---

## 23. Deployment Strategy

- **Blue-Green deployment** for backend (zero-downtime releases)
- **Environments:** `dev → staging → production`, each isolated infra/DB
- **Mobile:** Fastlane-driven pipeline → Play Store internal track → staged rollout (5%→25%→100%)
- **Web panels:** static build → CDN (CloudFront/Vercel) with cache invalidation on deploy
- **Rollback:** automated rollback on failed health-check post-deploy

---

## 24. Docker Architecture

```
docker-compose.yml (local dev)
├── postgres          (with volume)
├── redis
├── backend           (hot-reload, nodemon)
├── admin-panel        (vite dev server)
├── seller-panel        (vite dev server)
└── nginx-local        (reverse proxy, mirrors prod routing)
```
- Backend production image: multi-stage Dockerfile (`build` stage compiles TS → `runtime` stage: minimal `node:20-alpine`, non-root user, only `dist/` + prod deps copied)
- Images scanned (Trivy) in CI before push to registry (ECR)

---

## 25. CI/CD Planning (GitHub Actions)

```
on: push/PR to develop|main
1. Install & cache deps (pnpm)
2. Lint (eslint/dart analyze)
3. Type-check (tsc --noEmit)
4. Unit tests + coverage gate (≥80%)
5. Integration tests (docker-compose spun up in CI)
6. Build (backend image / React bundles / Flutter APK-AAB via Fastlane)
7. Security scan (Trivy image scan, npm audit, `dependabot`)
8. Push image to ECR / bundles to S3-CDN
9. Deploy to staging (auto) → smoke tests → manual approval gate → deploy to production (blue-green)
```

---

## 26. Scalability Planning

- **Stateless backend** (all session state in Redis/DB, not memory) → horizontal auto-scaling behind ALB
- **Read replicas** absorb read-heavy traffic (product browsing, search)
- **Caching layers** (§27) remove DB pressure for hot paths
- **Async/queue-based** processing for anything non-instant (emails, report generation, image processing)
- **CDN** for all static assets and product images
- **Database partitioning + eventual sharding** by `seller_id`/region as data grows
- **Circuit breakers & bulkheads** around third-party payment gateway calls to prevent cascading failures
- Designed so any single module (e.g., `order`, `product`) can be lifted into an independent microservice + its own DB without touching other modules, once traffic demands it

---

## 27. Caching Strategy

| Data | Cache | TTL / Invalidation |
|---|---|---|
| Product listings/detail | Redis | TTL 5-10min + explicit invalidate on update/approval |
| Categories/Brands (near-static) | Redis, long TTL | Invalidate on admin write |
| Session/refresh-token metadata | Redis | matches token expiry |
| Search results | Elasticsearch (own cache) | re-indexed via CDC |
| Rate-limit counters | Redis | sliding window |
| CDN edge cache | CloudFront | product images, static panel bundles |

Cache-aside pattern throughout: app checks Redis → falls back to DB → populates cache. Write paths explicitly bust relevant keys (no reliance on TTL alone for correctness-critical data like inventory).

---

## 28. Storage Strategy

- **Product/Profile images:** uploaded directly to **S3** (pre-signed URLs from backend — files never pass through app server) → served via CloudFront CDN
- **Image processing:** async Lambda/worker generates thumbnails/WebP variants on upload event
- **Documents (seller KYC, invoices):** private S3 bucket, access via short-lived signed URLs only
- **Database backups/WAL archives:** dedicated S3 bucket with lifecycle policies (Glacier after 90 days)

---

## 29. Cloud Architecture (AWS reference)

```
Route53 → CloudFront (CDN) → ALB → ECS/EKS (backend containers, auto-scaled)
                                        ↓
                    RDS PostgreSQL (Multi-AZ, primary + read replicas)
                    ElastiCache Redis (cluster mode)
                    OpenSearch (product search)
                    S3 (images, docs, backups)
                    SQS/BullMQ-Redis (async jobs)
                    Secrets Manager (credentials)
                    CloudWatch + Sentry (observability)
```
VPC with public subnets (ALB only) and private subnets (compute, DB, cache) — DB never internet-reachable. Infra defined as code via Terraform, environment-parameterized.

---

## 30. Third-Party Services

- **Firebase** — Auth (Google, Phone OTP, Email verify), Push (FCM)
- **Payments** — bKash, Nagad, Rocket (local gateways), Stripe (Visa/MasterCard)
- **AWS** — S3, CloudFront, RDS, ElastiCache, ECS/EKS, SES
- **Sentry** — error/crash monitoring (backend, React, Flutter)
- **Elasticsearch/OpenSearch** — product search
- **Twilio/local SMS gateway** — OTP fallback delivery
- **Datadog/CloudWatch** — metrics & alerting

---

## 31. Complete Development Roadmap (High-Level)

| Phase | Focus | Duration (indicative) |
|---|---|---|
| Phase 0 | Foundation (this doc), tooling, CI/CD skeleton, infra bootstrap | Week 1-2 |
| Phase 1 | Auth module (Firebase + JWT + RBAC) end-to-end | Week 3-4 |
| Phase 2 | Product/Category/Brand modules + Seller onboarding & approval | Week 5-7 |
| Phase 3 | Cart, Wishlist, Checkout, Order lifecycle | Week 8-10 |
| Phase 4 | Payments integration (bKash/Nagad/Rocket/Stripe) + Wallet | Week 11-13 |
| Phase 5 | Admin panel (approvals, coupons, banners, delivery settings) | Week 14-16 |
| Phase 6 | Seller panel (inventory, orders, analytics, reports) | Week 16-18 |
| Phase 7 | Search (Elasticsearch), Notifications, Reviews/Ratings | Week 19-20 |
| Phase 8 | Security hardening audit, load testing, observability | Week 21-22 |
| Phase 9 | Beta rollout, staged production release | Week 23-24 |

---

**Foundation status: complete.** This document governs all subsequent implementation — no code should deviate from this architecture without an explicit revision here first.

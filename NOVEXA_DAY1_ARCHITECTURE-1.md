# NOVEXA — Enterprise Multi-Vendor Marketplace
## Day 1: Project Foundation & Architecture Blueprint

---

## 1. Complete Repository / Folder Structure (Monorepo, Turborepo-managed)

```
novexa/
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

Novexa backend follows **Clean Architecture** in 4 layers, wrapped in **feature modules** (Modular Monolith today → extractable to microservices later):

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
3. Client → Novexa Backend /auth/verify  (sends Firebase idToken)
4. Backend verifies idToken via Firebase Admin SDK
5. Backend issues Novexa-owned tokens:
   - Access Token  (JWT, short-lived, 15 min)
   - Refresh Token (JWT, long-lived, 30 days, stored hashed in DB, rotated on use)
6. Access Token → Authorization: Bearer header on every API call
7. Refresh flow: on 401, client silently calls /auth/refresh with refresh token
   → old refresh token invalidated (rotation) → new pair issued
8. Role-Based Access Control (RBAC): JWT payload carries { userId, role: customer|seller|admin, permissions[] }
9. Middleware chain: authenticate → authorize(role) → validate → controller
```

Why Firebase + own JWT (hybrid): Firebase handles the hard parts (OTP delivery, Google OAuth, email verification infra) while Novexa retains full control of session lifecycle, RBAC, and revocation — required for enterprise-grade auth.

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
-e 

---


# NOVEXA — Architecture Enhancement (Enterprise Upgrade)
## Extension to Day 1 Foundation — New Modules Only

*This document extends the original `NOVEXA_ARCHITECTURE_FOUNDATION.md`. Nothing in the original is modified, removed, or duplicated. Same 4-layer Clean Architecture convention applies to every module below.*

---

## E1. Location Module

### Domain Model (structural, no tables)
Hierarchical geo-entities: `Country → Division → District → Upazila → Union → Area`, each referencing its parent (adjacency, not flattened) so new countries/regions can be onboarded without schema change.

### Backend Module — `modules/location/`
```
location/
├── domain/
│   ├── entities/         # Country, Division, District, Upazila, Union, Area, Address
│   └── interfaces/       # ILocationRepository, IAddressRepository, IGeocodingProvider
├── application/
│   ├── use-cases/
│   │   ├── resolve-address-hierarchy.usecase.ts
│   │   ├── validate-delivery-zone.usecase.ts
│   │   ├── save-address.usecase.ts
│   │   ├── set-default-address.usecase.ts
│   │   └── verify-gps-coordinates.usecase.ts
│   └── dto/
├── infrastructure/
│   ├── repositories/     # PostgresLocationRepository, PostgresAddressRepository
│   └── providers/        # GoogleMapsGeocodingProvider (implements IGeocodingProvider)
└── presentation/
    ├── controllers/       # location.controller.ts, address-book.controller.ts
    ├── routes/
    └── validators/
```

**Key services:**
- `IGeocodingProvider` interface abstracts Google Maps — swappable for Mapbox/OpenStreetMap without touching use-cases.
- `validate-delivery-zone.usecase` cross-checks a customer's Area against the Admin-defined serviceable-zone list before checkout is allowed to proceed.
- `Address Book`: one customer → many `Address` entities, exactly one flagged `isDefault` (enforced at use-case level, not DB constraint alone).

### API additions (`/api/v1/locations/*`, `/api/v1/addresses/*`)
```
/api/v1/locations/countries
/api/v1/locations/divisions?countryId=
/api/v1/locations/districts?divisionId=
/api/v1/locations/upazilas?districtId=
/api/v1/locations/unions?upazilaId=
/api/v1/locations/areas?unionId=
/api/v1/locations/delivery-zones/verify

/api/v1/addresses           (GET list, POST create)
/api/v1/addresses/:id       (PUT update, DELETE)
/api/v1/addresses/:id/default
```

### Flutter Feature — `features/location/`
```
location/
├── data/
│   ├── datasources/   # remote (cascading dropdown APIs) + local (last-used cache)
│   ├── models/
│   └── repositories/
├── domain/
│   ├── entities/
│   ├── repositories/
│   └── usecases/
└── presentation/
    ├── bloc/          # address_book_bloc, location_picker_bloc
    ├── screens/        # address-book, add-address (map picker)
    └── widgets/        # cascading-selector, map-pin-picker
```
- `google_maps_flutter` + `geolocator` packages for GPS capture and pin-drop address selection.
- `location_picker_bloc` emits `DeliveryZoneVerified` / `DeliveryZoneUnavailable` states consumed directly by the checkout flow.

### Admin Management
- CRUD for Country/Division/District/Upazila/Union/Area hierarchy
- Delivery Zone management (mark Areas as serviceable/unserviceable, assign delivery-charge tiers per zone)

### Seller Support
- Sellers select pickup Address from the same Address Book component (reused, not reimplemented)
- Seller "coverage area" preference feeds into future rider-assignment logic (§E3)

---

## E2. Live Chat Module (Architecture Only)

### Backend Module — `modules/chat/`
```
chat/
├── domain/
│   ├── entities/          # Conversation, Message, ParticipantStatus
│   └── interfaces/         # IChatRepository, IPresenceService, IFileUploadProvider
├── application/
│   ├── use-cases/
│   │   ├── start-conversation.usecase.ts     # customer↔seller or customer↔admin
│   │   ├── send-message.usecase.ts
│   │   ├── mark-as-read.usecase.ts
│   │   ├── update-presence.usecase.ts
│   │   └── get-unread-count.usecase.ts
│   └── dto/
├── infrastructure/
│   ├── repositories/       # PostgresChatRepository (history/persistence)
│   ├── realtime/            # SocketIoGateway (implements IPresenceService)
│   └── providers/           # S3ChatAttachmentProvider
└── presentation/
    ├── controllers/          # REST: chat history, conversation list
    ├── gateways/             # socket.io event handlers (namespace: /chat)
    └── routes/
```

### Real-Time Architecture
```
Client (Flutter/React) ⇄ Socket.IO Gateway (sticky-session via Redis adapter)
                                 ↓
                    socket.io-redis-adapter (pub/sub across
                    multiple backend instances — required for
                    horizontal scaling of stateful websockets)
                                 ↓
                    Chat Use-Cases → PostgresChatRepository (message persistence)
                                 → Push Notification dispatch (offline recipient)
```
- **Horizontal scale note:** Socket.IO alone is single-instance-sticky; the Redis adapter is mandatory once backend runs >1 instance, so a message from a user connected to Pod A reaches a recipient connected to Pod B.
- **Presence:** `IPresenceService` tracks online/offline + `lastSeenAt` in Redis (ephemeral, not Postgres) — high write frequency unsuitable for the primary DB.

### Event Contract (socket namespace `/chat`)
```
client→server: message:send, typing:start, typing:stop, message:read
server→client: message:new, message:delivered, message:read,
                presence:update, typing:indicator, unread:count
```

### Message Delivery State Machine
```
SENT → DELIVERED (recipient socket ack) → READ (recipient opened thread)
```
Each transition persisted (`Message.status`) and broadcast to sender for UI ticks.

### Attachments
- Image / PDF / Voice message: client uploads to S3 via pre-signed URL (same pattern as §28 in Foundation doc) → message payload carries the resulting S3 key/URL only, never binary over the socket.
- `IFileUploadProvider` interface keeps chat decoupled from storage vendor.

### Flutter Feature — `features/chat/`
```
chat/
├── data/ (datasources: socket_client + rest_client, models, repositories)
├── domain/ (entities, repositories, usecases)
└── presentation/
    ├── bloc/            # conversation_list_bloc, chat_thread_bloc, presence_bloc
    ├── screens/
    └── widgets/          # message-bubble, typing-indicator, voice-recorder
```
- `socket_io_client` package maintained as a singleton connection via DI, reused across the app session (auth token attached on connect handshake).

---

## E3. Rider Module

### Backend Module — `modules/rider/`
```
rider/
├── domain/
│   ├── entities/        # Rider, RiderWallet, DeliveryAssignment, RiderLocationPing
│   └── interfaces/       # IRiderRepository, IRiderWalletRepository, ILiveTrackingService
├── application/
│   ├── use-cases/
│   │   ├── register-rider.usecase.ts
│   │   ├── approve-rider.usecase.ts            # admin action
│   │   ├── assign-order-to-rider.usecase.ts
│   │   ├── verify-delivery-otp.usecase.ts
│   │   ├── record-rider-location.usecase.ts
│   │   ├── complete-delivery.usecase.ts        # triggers wallet-hold release (§E6)
│   │   └── calculate-rider-earnings.usecase.ts
│   └── dto/
├── infrastructure/
│   ├── repositories/
│   └── realtime/          # RiderLocationSocketGateway (Redis geo-index)
└── presentation/
    ├── controllers/
    ├── routes/
    └── validators/
```

### Delivery OTP Verification Flow
```
Rider arrives → app requests delivery confirmation
→ backend generates one-time OTP, sends to customer (SMS/push)
→ Rider enters OTP in Rider App → verify-delivery-otp.usecase
→ On match: Order.status = DELIVERED → complete-delivery.usecase fires
   → Domain Event: OrderDelivered → consumed by Wallet module (releases held delivery charge, §E6)
                                    → consumed by Rider module (credits rider earnings)
```

### Live GPS Tracking
- Rider app pings location every N seconds → `record-rider-location.usecase` → written to Redis Geo (GEOADD) for low-latency proximity queries, periodically flushed to Postgres for historical `DeliveryHistory`.
- Customer order-tracking screen subscribes via Socket.IO namespace `/tracking/:orderId` for live pin movement — same Redis-adapter pattern as chat (§E2) for multi-instance scale.

### Rider Wallet & Earnings
- `RiderWallet` mirrors the structure of Customer/Seller wallets (§E5 Ledger pattern) — every delivery completion writes a `RiderEarningEntry` (immutable ledger row), never an in-place balance mutation.

### API additions
```
/api/v1/riders/register
/api/v1/riders/:id/approve         (admin)
/api/v1/riders/:id/assignments
/api/v1/riders/:id/wallet
/api/v1/riders/:id/earnings
/api/v1/riders/:id/performance
/api/v1/orders/:id/assign-rider    (admin/system)
/api/v1/orders/:id/delivery-otp/verify
```

### Rider Performance & Delivery History
- Performance derived (not stored redundantly): computed on read from `DeliveryAssignment` outcomes (on-time %, cancellation %, average delivery time) via a read-replica analytics query, cached in Redis (short TTL) — keeps the ledger tables the single source of truth.

*(Rider mobile app is a separate Flutter target — `apps/rider-app/` — sharing `packages/shared-types` and reusing the same feature-first Clean Architecture pattern as the customer app.)*

---

## E4. Return & Refund Modules

Two distinct modules, intentionally decoupled — a Return is a logistics workflow, a Refund is a financial workflow, and either can occur without the other (e.g., refund without physical return for digital/defective-beyond-return cases).

### Backend Module — `modules/returns/`
```
returns/
├── domain/
│   ├── entities/        # ReturnRequest, ReturnItem, ReturnStatus (state machine)
│   └── interfaces/       # IReturnRepository
├── application/
│   ├── use-cases/
│   │   ├── request-return.usecase.ts
│   │   ├── approve-return.usecase.ts        # seller/admin
│   │   ├── reject-return.usecase.ts
│   │   ├── track-return-pickup.usecase.ts
│   │   └── complete-return.usecase.ts        # emits ReturnCompleted event
│   └── dto/
├── infrastructure/repositories/
└── presentation/ (controllers/routes/validators)
```

### Backend Module — `modules/refunds/`
```
refunds/
├── domain/
│   ├── entities/        # RefundRequest, RefundStatus (state machine)
│   └── interfaces/       # IRefundRepository, IPaymentGatewayRefundProvider
├── application/
│   ├── use-cases/
│   │   ├── initiate-refund.usecase.ts        # triggered manually OR by ReturnCompleted event
│   │   ├── approve-refund.usecase.ts          # admin
│   │   ├── process-wallet-refund.usecase.ts   # credits Novexa Wallet (instant)
│   │   ├── process-gateway-refund.usecase.ts  # bKash/Nagad/Rocket/Stripe reversal (async, webhook-confirmed)
│   │   └── get-refund-status.usecase.ts
│   └── dto/
├── infrastructure/
│   ├── repositories/
│   └── providers/         # BkashRefundProvider, NagadRefundProvider, StripeRefundProvider (all implement IPaymentGatewayRefundProvider)
└── presentation/ (controllers/routes/validators)
```

### Return → Refund Workflow
```
ReturnRequest.status:
  REQUESTED → SELLER_REVIEW → APPROVED/REJECTED
     → APPROVED → PICKUP_SCHEDULED → PICKED_UP → RECEIVED_AT_WAREHOUSE
        → INSPECTION_PASSED → ReturnCompleted event emitted
                                    ↓
RefundRequest auto-created, status: PENDING → APPROVED (admin/auto-rule)
   → route by original payment method:
        Wallet-funded order  → process-wallet-refund (instant, ledger entry)
        Gateway-funded order → process-gateway-refund (async; RefundStatus stays
                                PROCESSING until provider webhook confirms → COMPLETED)
```
- Refund routing decision (wallet vs gateway) is a pure domain rule inside `initiate-refund.usecase`, keeping payment-provider specifics fully out of the Returns module (Dependency Inversion, same as §11 Foundation Repository Pattern).

### API additions
```
/api/v1/returns              (POST request, GET list)
/api/v1/returns/:id/approve
/api/v1/returns/:id/reject
/api/v1/returns/:id/tracking

/api/v1/refunds/:id
/api/v1/refunds/:id/approve
/api/v1/refunds/:id/status
```

---

## E5. Wallet Transaction Module

*Extends (does not replace) the existing `wallet` module from the Foundation doc — adds the ledger/transaction sub-system underneath it.*

### Backend Sub-Module — `modules/wallet/transactions/`
```
wallet/transactions/
├── domain/
│   ├── entities/         # WalletLedgerEntry (immutable), TransactionType (enum)
│   └── interfaces/        # ILedgerRepository
├── application/
│   ├── use-cases/
│   │   ├── record-topup.usecase.ts
│   │   ├── hold-delivery-charge.usecase.ts       # see §E6
│   │   ├── release-delivery-hold.usecase.ts       # see §E6
│   │   ├── credit-refund.usecase.ts
│   │   ├── debit-refund-reversal.usecase.ts
│   │   ├── get-transaction-history.usecase.ts
│   │   └── export-ledger.usecase.ts                # CSV/PDF export job
│   └── dto/
├── infrastructure/repositories/
└── presentation/ (controllers/routes)
```

### Core Principle: Append-Only Ledger
`WalletLedgerEntry` rows are **never updated or deleted** — every balance change (top-up, hold, release, refund credit/debit) is a new immutable row with `type`, `amount`, `balanceAfter`, `referenceType` (order/return/refund/topup), `referenceId`. Current wallet balance = either a materialized `balanceAfter` on the latest entry, or derived via `SUM()` — the ledger is always the source of truth, never a bare mutable `balance` column trusted in isolation.

**Transaction types:**
```
TOP_UP | DELIVERY_HOLD | DELIVERY_RELEASE | DELIVERY_DEDUCT
REFUND_CREDIT | REFUND_DEBIT | MANUAL_ADJUSTMENT (admin, always audit-logged)
```

### API additions
```
/api/v1/wallet/transactions           (GET history, filterable by type/date)
/api/v1/wallet/transactions/export    (CSV/PDF)
```

---

## E6. Delivery Wallet Hold Service (Core Business Rule)

### Business Rule
A customer **cannot place an order** unless their Delivery Wallet balance ≥ the order's calculated delivery charge. This is enforced at the domain layer, not merely UI validation.

### Service — `application/services/delivery-hold.service.ts` (used by `order` module)

### Order Placement Flow
```
1. Customer initiates checkout
2. calculate-delivery-charge.usecase (existing, delivery/ module) determines charge
   based on Delivery Zone (§E1) + weight/distance rules
3. hold-delivery-charge.usecase (§E5):
     - Reads current wallet balance from ledger
     - IF balance < deliveryCharge → reject order placement (402-style domain error,
       "InsufficientDeliveryWalletBalance") — order is NEVER created
     - IF sufficient → writes WalletLedgerEntry{ type: DELIVERY_HOLD, amount: -charge }
       → this reduces *available* balance immediately, atomically, within the same
       DB transaction that creates the Order (no partial-hold race condition)
4. Order.status = PLACED, Order.deliveryChargeHoldRef = ledgerEntryId
```

### On Successful Delivery
```
Rider completes delivery (§E3, OTP-verified)
→ OrderDelivered domain event
→ release-delivery-hold.usecase listens:
     writes WalletLedgerEntry{ type: DELIVERY_RELEASE, amount: 0-net }
     writes WalletLedgerEntry{ type: DELIVERY_DEDUCT, amount: -charge, final }
     (held amount converts to a real deduction — final settlement)
```

### On Order Cancellation
```
Order cancelled (customer/seller/admin, before delivery)
→ OrderCancelled domain event
→ release-delivery-hold.usecase listens:
     writes WalletLedgerEntry{ type: DELIVERY_RELEASE, amount: +charge }
     (held amount returned to available balance — automatic, no admin action needed)
```

### State Machine (Delivery Hold)
```
NONE → HELD (on order placement) → DEDUCTED (on delivery)
                                  → RELEASED (on cancellation)
```
Idempotency guaranteed: `hold`/`release` use-cases are keyed by `orderId` — a duplicate event (e.g., retried webhook) cannot double-hold or double-release, checked against existing ledger references before writing.

---

## E7. Enterprise Search Enhancements

*Extends the Foundation's Elasticsearch/OpenSearch search offload (§8 in original doc).*

### New Sub-Module — `modules/search/enhancements/`
```
search/
├── application/use-cases/
│   ├── get-autocomplete-suggestions.usecase.ts
│   ├── get-popular-searches.usecase.ts
│   ├── get-recent-searches.usecase.ts      # per-user, from Redis list (capped, e.g. last 10)
│   ├── get-trending-searches.usecase.ts     # windowed aggregation (last 24h/7d)
│   └── log-search-query.usecase.ts          # fire-and-forget → analytics pipeline
└── infrastructure/
    └── providers/
        └── OpenSearchSuggesterProvider.ts    # completion suggester / edge-ngram index
```

- **Autocomplete:** OpenSearch `completion` suggester (edge-ngram analyzer) — sub-50ms response target, never queries Postgres.
- **Recent Searches:** stored per-user in Redis (`LPUSH` + `LTRIM`), not Postgres — high write volume, low durability requirement.
- **Popular/Trending:** background aggregation job (§ Foundation `jobs/`) periodically rolls up `search_query_log` (append-only, async-written) into a ranked Redis sorted set (`ZINCRBY` per query, decayed windows for "trending" vs raw count for "popular").
- **Search Analytics:** every query + result-count + click-through (if selected) logged asynchronously via the existing queue (BullMQ) — never blocks the search response path.

---

## E8. Fraud Detection Service

### Backend Module — `modules/fraud-detection/`
```
fraud-detection/
├── domain/
│   ├── entities/        # RiskEvent, RiskScore, DeviceFingerprint
│   └── interfaces/       # IRiskScoringEngine, IDeviceFingerprintProvider
├── application/
│   ├── use-cases/
│   │   ├── evaluate-login-risk.usecase.ts
│   │   ├── evaluate-payment-risk.usecase.ts
│   │   ├── detect-duplicate-account.usecase.ts
│   │   └── flag-suspicious-activity.usecase.ts   # writes to Activity Timeline, §E9
│   └── dto/
├── infrastructure/
│   ├── providers/         # DeviceFingerprintProvider (FingerprintJS-based, mobile/web)
│   └── engine/             # RuleBasedRiskScoringEngine (v1) — pluggable for future ML model
└── presentation/ (internal service — invoked by auth/payment modules, not directly public)
```

### Integration Points (cross-cutting, event-driven — no tight coupling)
```
auth module    → on login → emits LoginAttempted → evaluate-login-risk.usecase
                 (checks: new device? impossible-travel velocity? known-bad IP list?)
payment module → on payment failure → emits PaymentFailed → evaluate-payment-risk.usecase
                 (checks: N failures in window → temporary account/payment-method lock)
auth module    → on registration → detect-duplicate-account.usecase
                 (checks: same device fingerprint / phone hash / delivery-address
                 fingerprint reused across multiple accounts)
```

### Risk Score
- Composite score (0–100) computed by `RuleBasedRiskScoringEngine` from weighted signals (new device, VPN/proxy IP, failed-payment count, account age, velocity of orders). Score persisted per event, not just a single mutable "user risk field," preserving history for audit and future ML training data.
- `IRiskScoringEngine` interface designed explicitly so v1 rule-based engine can later be swapped for a trained ML model service without touching calling modules.

### Outcomes
- Score above threshold → soft actions (extra OTP step-up, manual review queue) rather than automatic hard-block, routed to Admin's fraud review dashboard.

---

## E9. Activity Timeline (Audit System Enhancement)

*Extends the Foundation's Audit Log (§22) with a queryable, user-facing/admin-facing timeline view — the Audit Log remains the immutable write path; this adds structured read-side aggregation.*

### Backend Sub-Module — `modules/audit/timeline/`
```
audit/timeline/
├── application/
│   └── use-cases/
│       ├── get-customer-activity-timeline.usecase.ts
│       ├── get-seller-activity-timeline.usecase.ts
│       ├── get-admin-activity-timeline.usecase.ts
│       ├── get-order-timeline.usecase.ts
│       ├── get-wallet-timeline.usecase.ts
│       └── get-login-timeline.usecase.ts
└── infrastructure/
    └── repositories/    # queries partitioned audit_logs table (read replica only)
```

### Design
- All timeline use-cases are **read-only projections** over the existing append-only `audit_logs` stream (§22) plus domain-specific ledgers (`WalletLedgerEntry` for Wallet Timeline, `Order.statusHistory` for Order Timeline) — no new write path, no duplicated source of truth.
- Every domain event already emitted across modules (`OrderPlaced`, `OrderDelivered`, `ReturnCompleted`, `RefundApproved`, `LoginAttempted`, `SellerApproved`, `MessageSent`...) is subscribed to by a single `AuditTimelineListener`, which normalizes them into a common `TimelineEntry{ actorId, actorRole, action, targetType, targetId, metadata, occurredAt }` shape — this is what makes cross-cutting timelines (e.g., "everything this seller did today") possible without per-module custom queries.
- Timeline queries always hit the **read replica**, never the primary — audit/history browsing must never compete with transactional write load.

### API additions
```
/api/v1/admin/timeline/customers/:id
/api/v1/admin/timeline/sellers/:id
/api/v1/admin/timeline/admins/:id
/api/v1/admin/timeline/orders/:id
/api/v1/admin/timeline/wallet/:userId
/api/v1/admin/timeline/logins/:userId
```

---

## E10. Updated Project Structure (Additions Only)

### Repository Structure — new top-level app
```
apps/
├── backend/                # (existing — new modules added inside, listed below)
├── mobile/                 # (existing)
├── rider-app/               # NEW — Flutter, shares packages/shared-types + ui patterns
├── admin-panel/             # (existing — new screens listed below)
└── seller-panel/            # (existing — new screens listed below)
```

### Backend — new modules added to `apps/backend/src/modules/`
```
location/
chat/
rider/
returns/
refunds/
wallet/transactions/        (sub-module of existing wallet/)
search/enhancements/         (sub-module of existing search offload)
fraud-detection/
audit/timeline/              (sub-module of existing audit/logging core)
```

### Flutter (customer app) — new features added to `mobile/lib/features/`
```
location/
chat/
returns_refunds/
```

### Rider App — new Flutter target `apps/rider-app/lib/features/`
```
auth/ (shared pattern) · assignments/ · live_tracking/ · earnings_wallet/ · delivery_otp/
```

### Admin Panel — new sections added to `admin-panel/src/pages/`
```
locations/            (hierarchy + delivery-zone management)
chat-support/           (admin↔customer chat console)
riders/                 (approval, performance, assignments)
returns/
refunds/
wallet-ledger/           (transaction export, manual adjustments)
search-insights/          (trending/popular search analytics)
fraud-review/             (risk-score queue)
activity-timeline/        (cross-entity timeline viewer)
```

### Seller Panel — new sections added to `seller-panel/src/pages/`
```
chat-support/            (seller↔customer chat console)
returns/                  (approve/reject incoming returns)
```

### API Structure — additions
```
/api/v1/locations/*        /api/v1/addresses/*
/api/v1/chat/*              (+ socket namespace /chat)
/api/v1/riders/*             (+ socket namespace /tracking/:orderId)
/api/v1/returns/*
/api/v1/refunds/*
/api/v1/wallet/transactions/*
/api/v1/search/suggestions   /api/v1/search/trending  /api/v1/search/recent
/api/v1/admin/fraud/*
/api/v1/admin/timeline/*
```

### Database Architecture — additions
- New logical schema group: `location`, `chat`, `logistics` (rider/delivery), `finance-extended` (returns/refunds/ledger)
- New partitioned tables (by month, same convention as `orders`): `wallet_ledger_entries`, `messages`, `rider_location_pings` (or Redis-only, flushed periodically), `audit_logs` (already partitioned) — `search_query_log`
- `rider_location_pings`: high-write, short-retention → primary store is Redis Geo; Postgres holds only periodic snapshots for historical analytics, not every ping

### Cloud Architecture — additions
```
ElastiCache Redis  → extended usage: Socket.IO adapter, presence, geo-tracking,
                      recent/trending search, delivery-hold idempotency keys
OpenSearch          → extended usage: completion suggester for autocomplete
S3                   → extended usage: chat attachments (image/PDF/voice)
New service: WebSocket-capable ECS/EKS task group (chat + tracking gateways),
             scaled independently from the stateless REST API tier
```

### Roadmap — additions (new phases inserted after original Phase 9)
| Phase | Focus |
|---|---|
| Phase 10 | Location module + Address Book + Delivery Zone verification |
| Phase 11 | Wallet Ledger + Delivery Wallet Hold Service (core business rule) |
| Phase 12 | Rider module + rider-app + Live GPS tracking |
| Phase 13 | Returns & Refunds workflow (incl. gateway refund integrations) |
| Phase 14 | Live Chat (Socket.IO, multi-instance Redis adapter) |
| Phase 15 | Enterprise Search enhancements (autocomplete, trending) |
| Phase 16 | Fraud Detection service + Admin fraud-review dashboard |
| Phase 17 | Activity Timeline (cross-module audit projection) |

---

**Enhancement status: complete.** All modules above compose additively onto the Day 1 Foundation — same Clean Architecture, same DI/repository conventions, same event-driven decoupling. No existing module, table, or structure was altered.

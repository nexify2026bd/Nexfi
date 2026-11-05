# NEXFIY — Day 3: Enterprise Backend Foundation
## Complete Backend Infrastructure Blueprint (Pre-Module Layer)

*This document extends Day 1 (Architecture) and Day 2 (Database) without modifying either. It defines the **infrastructure layer only** — the plumbing every future business module (Product, Order, Payment, etc.) will be built on top of. No business/domain logic for any specific module is included here, per instruction.*

---

## 1. Enterprise Backend Foundation

### 1.1 Layering Recap (governs every section below)
```
Presentation   → controllers, routes, middlewares, validators
Application    → use-cases, application services, DTOs, ports (interfaces)
Domain         → entities, value objects, domain events, domain services
Infrastructure → repositories (impl), cache, queue, realtime, external providers
```
Infrastructure code **implements** interfaces defined in Application/Domain — never the reverse. This is the Dependency Inversion boundary that everything in this document exists to serve.

### 1.2 Core Bootstrap Folder Structure
```
apps/backend/src/core/
├── config/
│   ├── env.schema.ts           # Zod-validated env schema — app refuses to boot on invalid config
│   ├── env.ts                   # typed, frozen config object built from env.schema
│   └── constants.ts
├── database/
│   ├── pg-pool.ts                # primary + replica pool factories
│   ├── migration-runner.ts
│   └── transaction-manager.ts     # unit-of-work wrapper (see §4.4)
├── cache/
│   ├── redis-client.ts
│   ├── cache.service.ts
│   └── distributed-lock.service.ts
├── queue/
│   ├── queue.registry.ts
│   ├── bullmq-connection.ts
│   └── job-processors/            # registered per-module later, empty at Day 3
├── events/
│   ├── event-bus.ts
│   ├── domain-event.base.ts
│   └── event-registry.ts
├── realtime/
│   ├── socket-server.ts
│   └── redis-adapter.ts
├── security/
│   ├── helmet.config.ts
│   ├── cors.config.ts
│   ├── rate-limiter.ts
│   └── sanitizers.ts
├── auth/
│   ├── firebase-admin.provider.ts
│   ├── jwt.service.ts
│   └── password-hasher.service.ts
├── rbac/
│   ├── permission.registry.ts
│   └── policy.engine.ts
├── errors/
│   ├── app-error.base.ts
│   ├── error-catalog.ts
│   └── global-error-handler.ts
├── logger/
│   ├── logger.ts
│   └── request-context.ts          # AsyncLocalStorage-based correlation ID
├── di/
│   └── container.ts
├── health/
│   └── health-check.service.ts
└── bootstrap.ts                     # composes app.ts: order matters, see §5.6
```

### 1.3 Design Principles Enforced at This Layer
- **SOLID:** every infra service is exposed behind an interface consumed by Application (Dependency Inversion + Interface Segregation — e.g., `ICacheService` is not `IRedisService`, so swapping Redis for another cache store never touches use-cases).
- **DDD:** Domain layer (entities, value objects, domain events) has **zero imports** from `core/` — enforced by an ESLint boundary rule (`eslint-plugin-boundaries`) that fails CI if violated.
- **CQRS-ready:** command (write) and query (read) paths are structurally separated from Day 3 onward — see §4.3 — even though no module yet exercises it, the pipes exist so Product/Order modules built later drop straight in.
- **Horizontal scalability:** nothing in this layer holds in-process state that isn't safe under multiple concurrent instances — sessions, locks, rate-limit counters, presence, and job queues all live in Redis, not memory (detailed per-section below).

---

## 2. Authentication Infrastructure

### 2.1 Hybrid Firebase + Nexfiy-JWT Flow (implementation of Day-1 §9)
```
core/auth/
├── firebase-admin.provider.ts   # wraps Firebase Admin SDK, verifyIdToken()
├── jwt.service.ts                # sign/verify access + refresh tokens (jose or jsonwebtoken)
├── token-pair.factory.ts          # issues { accessToken, refreshToken } given a userId+role claims
├── refresh-token.repository.ts     # impl of IRefreshTokenRepository (hashed storage, rotation)
└── password-hasher.service.ts      # argon2id wrapper (for password-set accounts only)
```

### 2.2 Interfaces (Application layer ports)
```ts
interface IAuthTokenService {
  issueTokenPair(payload: TokenPayload): Promise<{ accessToken: string; refreshToken: string }>;
  verifyAccessToken(token: string): Promise<TokenPayload>;
  rotateRefreshToken(oldToken: string): Promise<{ accessToken: string; refreshToken: string }>;
  revokeRefreshToken(token: string): Promise<void>;
}

interface IIdentityProvider {           // abstracts Firebase — swappable
  verifyIdentityToken(idToken: string): Promise<{ uid: string; email?: string; phone?: string }>;
}
```

### 2.3 Token Design
| Token | Storage | Lifetime | Rotation |
|---|---|---|---|
| Access Token | stateless JWT, client memory/secure storage | 15 min | none — reissued via refresh |
| Refresh Token | JWT + hashed row in `identity.refresh_tokens` (Day 2) | 30 days | rotated on every use; old token immediately invalidated (reuse of a rotated token triggers full session revocation — replay-attack detection) |

### 2.4 Request-Time Flow
```
1. Client → POST /v1/auth/verify { firebaseIdToken }
2. AuthController → VerifyIdentityUseCase
     → IIdentityProvider.verifyIdentityToken()   [Firebase Admin SDK call]
     → find-or-provision identity.users row (repository)
     → IAuthTokenService.issueTokenPair({ userId, roles, permissions })
     → persist hashed refresh token
3. Response: { accessToken, refreshToken, user }
4. Every subsequent request: Authorization: Bearer <accessToken>
   → authenticate middleware (§5.4) verifies + attaches req.identity
5. On 401 (expired access token): client calls /v1/auth/refresh
   → rotate-refresh-token flow → new pair issued, old refresh token burned
```

### 2.5 Firebase Admin Provider — Failure Isolation
`firebase-admin.provider.ts` wraps every SDK call with a circuit breaker (see §10.5) — Firebase outage degrades gracefully (existing sessions keep working off local JWTs; only new logins/registrations are affected) rather than cascading into full API unavailability.

---

## 3. Authorization Infrastructure (RBAC & Permission System)

### 3.1 Model (implements Day 2 `identity.roles/permissions/role_permissions/user_roles`)
```
core/rbac/
├── permission.registry.ts     # static catalog of permission codes, loaded at boot, validated against DB
├── policy.engine.ts            # evaluates "can subject X perform action Y on resource Z"
├── rbac.middleware.ts           # requireRole(), requirePermission() route guards
└── permission-cache.service.ts   # Redis-cached user→permissions resolution (see §6.3)
```

### 3.2 Permission Resolution Flow
```
authenticate middleware attaches req.identity = { userId, roles: [] }  (roles only, lightweight JWT)
        ↓
requirePermission('product.approve') middleware:
   1. permission-cache.service.get(userId)  → Redis GET user:permissions:<userId>
   2. cache miss → policy.engine resolves via DB:
        user_roles → roles → role_permissions → permissions (flattened set)
      → cached in Redis, TTL 10 min, invalidated on role/permission change (event-driven, §8)
   3. IF permission not in set → 403 AppError (ForbiddenError)
```
*Why not embed full permissions in the JWT:* permissions can change (role revoked mid-session) and a stateless JWT can't be invalidated early without a blocklist — access tokens carry only coarse `roles`, fine-grained permission checks always resolve fresh (cached) server-side, giving both performance and correct revocation semantics.

### 3.3 Policy Engine — Extension Point for Resource-Level Rules
```ts
interface IPolicy<TResource> {
  can(actor: AuthenticatedIdentity, action: string, resource?: TResource): boolean | Promise<boolean>;
}
```
Future modules register resource-specific policies (e.g., "a seller may only edit their own product") against this engine — Day 3 ships the engine and the registration mechanism, not any concrete policy (that's module business logic, explicitly excluded).

### 3.4 Middleware Composition
```
router.patch('/v1/admin/sellers/:id/approve',
  authenticate,
  requirePermission('seller.approve'),
  validate(ApproveSellerParamsSchema),
  controller.approve
);
```

---

## 4. Repository Layer Implementation

### 4.1 Base Contracts
```ts
interface IRepository<TEntity, TId> {
  findById(id: TId): Promise<TEntity | null>;
  save(entity: TEntity): Promise<void>;         // insert or update, upsert semantics
  delete(id: TId): Promise<void>;                 // soft-delete where entity supports it
}

interface IPaginatedQuery<TFilter> {
  find(filter: TFilter, pagination: CursorPagination): Promise<PagedResult<TEntity>>;
}
```
Every concrete repository (built per-module, starting Day 4+) implements these against the Day 2 schema — Day 3 delivers only the base contracts, the Postgres query-builder wiring, and the transaction/unit-of-work infrastructure.

### 4.2 Base Postgres Repository (shared query mechanics)
```
core/database/
├── base.repository.ts        # abstract class: connection acquisition, read/write pool routing
├── query-builder.ts            # thin wrapper (Kysely) — typed, no raw string SQL in modules
└── cursor-pagination.util.ts    # keyset pagination helper (never OFFSET at scale)
```
`base.repository.ts` decides **write-pool vs read-pool** per call based on an explicit method parameter (`this.readPool` vs `this.writePool`), implementing Day 2 §17's routing rule at the code level — no implicit fallback to primary.

### 4.3 CQRS-Ready Split
```
application/
├── commands/            # write use-cases — always go through writePool + transaction-manager
│   └── <module>/         (empty at Day 3 — structure only)
└── queries/              # read use-cases — always go through readPool, cacheable
    └── <module>/          (empty at Day 3 — structure only)
```
Commands and Queries are physically separate use-case classes from Day 3 onward, even before any module exists — this is what makes CQRS "ready" rather than retrofitted: a future `GetProductDetailsQuery` never accidentally shares a class with `UpdateProductCommand`.

### 4.4 Unit of Work / Transaction Manager
```ts
interface ITransactionManager {
  runInTransaction<T>(work: (trx: DbTransaction) => Promise<T>): Promise<T>;
}
```
Wraps Postgres `BEGIN/COMMIT/ROLLBACK`. Every multi-repository write (e.g., a future "place order" command touching orders + inventory + wallet ledger) runs inside a single `runInTransaction` call — repositories accept an optional `trx` handle so they can participate in a caller-managed transaction instead of opening their own connection.

### 4.5 Repository Registration (DI)
```ts
container.register<IUserRepository>('IUserRepository', { useClass: PostgresUserRepository });
```
All bindings centralized in `core/di/container.ts` (see §1.2) — controllers/use-cases never `new` a repository directly, always resolve via constructor injection.

---

## 5. API Foundation

### 5.1 REST Standards
- Resource-oriented, plural nouns, kebab-case paths (Day 1 §17 naming convention).
- Response envelope, uniform across all endpoints:
```json
// success
{ "success": true, "data": { }, "meta": { "requestId": "..." } }
// error
{ "success": false, "error": { "code": "VALIDATION_ERROR", "message": "...", "details": [] }, "meta": { "requestId": "..." } }
```
- Cursor-based pagination only (`?cursor=...&limit=...`) — `OFFSET`-based pagination is explicitly disallowed in code review once tables reach Day-2 partition scale (OFFSET degrades linearly with depth).

### 5.2 API Versioning
- URI versioning: `/api/v1/...` (Day 1 §10). A new major version is a parallel route tree (`/api/v2/...`), never an in-place breaking change to `/v1`.
- Version sunset policy: `Deprecation` + `Sunset` HTTP headers added to a version once its successor ships, minimum 6-month overlap window.

### 5.3 Validation Pipeline
```
core/validation/
├── validate.middleware.ts     # generic: validate(schema, 'body'|'query'|'params')
└── schema-registry.ts          # re-exports packages/validation-schemas (shared w/ React panels)
```
- Zod schemas, shared from `packages/validation-schemas` (Day 1 monorepo layout) so the same rules apply client-side (React panels) and server-side — never re-declared per-platform.
- Validation happens **before** the request reaches the controller — a failed validation never touches application/domain code, returns `422 VALIDATION_ERROR` immediately with field-level `details[]`.

### 5.4 Middleware Chain (canonical order — order is a deliberate architectural decision)
```
1. requestContext        — generates/propagates correlation ID (AsyncLocalStorage)
2. httpLogger             — structured access log (method, path, status, latency)
3. helmet                 — security headers
4. cors
5. rateLimiter            — IP + user-scoped (see §6.4)
6. bodyParser / sanitizer  — XSS/NoSQL-injection stripping
7. authenticate           — JWT verify, attaches req.identity (skipped for public routes)
8. requirePermission(...)  — per-route RBAC guard (§3.4)
9. validate(schema)        — per-route Zod validation
10. [controller]
11. globalErrorHandler     — catches everything thrown above, formats per §5.5
```
Ordering rationale: rate-limiting and sanitization happen **before** auth (cheap rejection of abusive/malformed traffic before spending a DB/Redis round-trip on token verification); validation happens **after** auth/authorization (never leak schema details to an unauthenticated caller).

### 5.5 Global Exception Handling
```
core/errors/
├── app-error.base.ts      # abstract AppError { code, httpStatus, message, details? }
├── error-catalog.ts        # ValidationError, AuthError, ForbiddenError, NotFoundError,
│                            # ConflictError, RateLimitError, ExternalServiceError, InternalError
└── global-error-handler.ts  # last Express middleware; maps AppError→HTTP, unknown errors→500
```
- Unknown/unexpected errors are **never** passed through with their raw message/stack to the client in production — logged in full internally (with correlation ID), client receives a generic `INTERNAL_ERROR` envelope.
- Every error response includes the request's correlation ID so a user-reported issue can be grep'd directly out of centralized logs.

### 5.6 Request Lifecycle (end-to-end, text diagram)
```
Client Request
   → Load Balancer / Nginx
   → Express instance (any pod — stateless)
   → Middleware chain (§5.4)
   → Controller (thin: extract DTO, call use-case, map result → HTTP response)
   → Application Use-Case (Command or Query, §4.3)
        → domain validation / business rules (module-specific, NOT Day 3 scope)
        → Repository (via DI) → Postgres primary or replica (§4.2)
        → Cache read/write (§6)
        → Domain Event emission (§8) if state changed
   ← Use-case returns Result/Either
   ← Controller maps to response envelope (§5.1)
   ← globalErrorHandler (if thrown at any layer)
Client Response
```

---

## 6. Redis Infrastructure

### 6.1 Redis Usage Map (single Redis cluster, logically separated by key prefix)
| Concern | Key Pattern | TTL |
|---|---|---|
| Cache (products, categories, etc.) | `cache:<domain>:<id>` | 5-10 min, explicit invalidation on write |
| Session/refresh metadata | `session:<userId>:<deviceId>` | matches refresh token expiry |
| Permission resolution cache | `user:permissions:<userId>` | 10 min, event-invalidated |
| Rate limiter counters | `ratelimit:<scope>:<identifier>` | sliding window |
| Distributed locks | `lock:<resource>` | short (seconds), auto-expire safety net |
| Socket.IO adapter (pub/sub) | internal adapter keys | n/a |
| Rider geo / presence | `geo:riders`, `presence:<userId>` | ephemeral |
| BullMQ job data | `bull:<queueName>:*` | managed by BullMQ |

### 6.2 Cache Service (Cache-Aside, per Day 1 §27)
```ts
interface ICacheService {
  get<T>(key: string): Promise<T | null>;
  set<T>(key: string, value: T, ttlSeconds: number): Promise<void>;
  invalidate(key: string): Promise<void>;
  invalidatePattern(pattern: string): Promise<void>;   // SCAN-based, never KEYS in production
}
```
`invalidatePattern` uses `SCAN` with cursor iteration, never blocking `KEYS *` — required at this data volume to avoid a single-threaded Redis stall.

### 6.3 Distributed Lock Service
```ts
interface IDistributedLockService {
  acquire(resource: string, ttlMs: number): Promise<Lock | null>;
  release(lock: Lock): Promise<void>;
  withLock<T>(resource: string, ttlMs: number, fn: () => Promise<T>): Promise<T>;
}
```
Implemented via Redlock algorithm (single-Redis-cluster variant is acceptable at Nexfiy's initial scale; multi-cluster Redlock reserved for a later HA phase). Used for cross-instance critical sections that a Postgres row lock alone can't cover (e.g., a scheduled job that must run on exactly one pod, coupon-code generation uniqueness under burst load).

### 6.4 Rate Limiter
- Sliding-window counter algorithm (Redis `ZADD`/`ZREMRANGEBYSCORE`), not fixed-window (avoids the burst-at-boundary problem of fixed windows).
- Two tiers applied per request: **IP-based** (pre-auth, protects login/registration endpoints from abuse) and **user-based** (post-auth, protects authenticated endpoints from a single compromised/abusive account) — configured per-route, not a single global limit.
- On limit exceeded: `429 RATE_LIMIT_EXCEEDED` with `Retry-After` header.

### 6.5 Session Infrastructure
Even though auth is JWT-stateless, a lightweight session record is kept in Redis (`session:<userId>:<deviceId>`) for: active-device listing (user-facing "log out other devices"), forced logout (admin action → delete session key → next refresh-token attempt fails even before its DB row is checked, fast-path revocation), and login-timeline correlation with the Day-1 §E9 Activity Timeline.

---

## 7. Queue System (BullMQ)

### 7.1 Architecture
```
core/queue/
├── bullmq-connection.ts     # shared ioredis connection (separate DB index from cache Redis)
├── queue.registry.ts          # central registration: queue name → options
└── job-processors/            # per-module worker registration point (empty at Day 3)
```
- **Dedicated Redis logical DB / connection** for BullMQ, isolated from the cache/session Redis usage (§6) — job-queue traffic (large payloads, frequent polling) must never contend with or evict hot cache keys sharing the same instance's memory pressure.
- Queues registered centrally with explicit concurrency and retry policy per queue (not per job) so operational limits are visible in one file.

### 7.2 Standard Queues Provisioned at Day 3 (infrastructure only — no job logic yet)
```
notification-dispatch     (concurrency: 20)
email                      (concurrency: 10)
sms-otp                     (concurrency: 10)
image-processing             (concurrency: 5)
report-generation             (concurrency: 2)
search-indexing               (concurrency: 10)   -- CDC → OpenSearch sync target
audit-projection               (concurrency: 20)   -- feeds Activity Timeline (Day-1 §E9)
```
Each is a named queue shell with retry policy attached; actual job payload/processor logic is added when its owning module ships.

### 7.3 Retry Strategy
```
attempts: 5
backoff: { type: 'exponential', delay: 2000 }     -- 2s, 4s, 8s, 16s, 32s
removeOnComplete: { age: 3600 }                     -- keep 1hr for debugging, then purge
removeOnFail: false                                   -- failed jobs retained for manual inspection
```
- After exhausting retries, job moves to a **dead-letter queue** (`<queueName>-dlq`) rather than being silently dropped — an alert fires (§11) so a human investigates rather than data loss occurring invisibly.
- Idempotency is a hard requirement for every job processor (queues guarantee at-least-once delivery, never exactly-once) — enforced by convention here, actual idempotency keys are module-specific (e.g., the wallet ledger's `idempotency_key`, Day 2 §11).

### 7.4 Job Monitoring
Bull Board (or equivalent) mounted at an admin-only route for queue depth, failure rate, and DLQ inspection — feeds into the metrics pipeline (§11.2) as well for alerting, not just manual dashboarding.

---

## 8. Event System (Domain Events, Event Bus, Listeners)

### 8.1 Purpose
Implements the Day-1 architectural decision that modules communicate via events, not direct service-to-service calls — this is what keeps the modular monolith cleanly splittable into microservices later.

### 8.2 Structure
```
core/events/
├── domain-event.base.ts      # abstract class: eventName, occurredAt, payload, correlationId
├── event-bus.ts                # publish()/subscribe() — in-process EventEmitter for same-pod
│                                # listeners, backed by a Redis Pub/Sub or BullMQ event-queue
│                                # bridge for cross-pod delivery
├── event-registry.ts            # compile-time map of event name → payload type (type safety)
└── outbox/
    └── transactional-outbox.repository.ts   # see §8.4
```

### 8.3 Publish/Subscribe Contract
```ts
abstract class DomainEvent<TPayload> {
  readonly eventName: string;
  readonly occurredAt: Date;
  readonly correlationId: string;
  readonly payload: TPayload;
}

interface IEventBus {
  publish<T>(event: DomainEvent<T>): Promise<void>;
  subscribe<T>(eventName: string, handler: (event: DomainEvent<T>) => Promise<void>): void;
}
```

### 8.4 Reliable Delivery — Transactional Outbox Pattern
Publishing an event as a side effect of a DB write is unsafe if done as a separate step (a crash between DB commit and event publish loses the event silently). Day 3 provisions the **Outbox pattern**:
```
1. Use-case writes business row(s) AND an `event_outbox` row in the SAME Postgres transaction
   (transaction-manager, §4.4, guarantees atomicity).
2. A background relay worker polls `event_outbox` (or uses logical replication/CDC) and
   publishes to the Event Bus / BullMQ, marking rows as dispatched.
3. Bus delivery failure → row stays undispatched → retried by the relay — no event is ever
   lost to a mid-flight crash.
```
This is the mechanism that makes Day-1's event-driven module boundaries ("OrderPlaced", "OrderDelivered", "ReturnCompleted"...) actually reliable at production scale, rather than a best-effort in-memory emitter.

### 8.5 Event Retry Strategy
Listener failures are retried with the same exponential-backoff policy as queue jobs (§7.3) — internally, cross-pod event delivery IS a BullMQ job under the hood, so it inherits DLQ + retry infrastructure rather than reimplementing it.

### 8.6 Cross-Cutting Listener Provisioned at Day 3
`AuditTimelineListener` (Day-1 §E9) is registered as a **wildcard subscriber** (`subscribe('*', ...)`) at the infrastructure level here — its handler body (normalizing events into `activity_timeline_entries`) is trivial plumbing, not business logic, so it's included as infrastructure: every future module's events are automatically timeline-projected with zero extra wiring per module.

---

## 9. Real-Time Infrastructure (Socket.IO)

### 9.1 Architecture (implements Day-1 §E2/§E3 chat + tracking transport)
```
core/realtime/
├── socket-server.ts          # Socket.IO server attached to the same HTTP server as Express
├── redis-adapter.ts            # @socket.io/redis-adapter — mandatory for multi-pod delivery
├── auth.middleware.ts          # socket handshake JWT verification (reuses jwt.service, §2)
├── namespaces/
│   ├── chat.namespace.ts        (connection handling shell — message logic is module-owned)
│   ├── tracking.namespace.ts     (order/rider live-location channel shell)
│   └── notifications.namespace.ts (server→client push shell)
└── presence.service.ts          # online/offline + lastSeenAt, backed by Redis (§6.5)
```

### 9.2 Why Redis Adapter Is Non-Negotiable Here
Without it, a message published by a client connected to Pod A never reaches a recipient connected to Pod B — Socket.IO's default in-memory adapter only broadcasts within a single process. The Redis adapter turns `io.to(room).emit(...)` into a cluster-wide broadcast via Redis Pub/Sub, which is what makes horizontal scaling of the WebSocket tier possible at all (Day-1 §26).

### 9.3 Connection Lifecycle
```
1. Client connects with `auth: { token }` in handshake
2. auth.middleware verifies JWT → attaches socket.data.identity
3. Server joins socket to identity-scoped room: `user:<userId>`
4. presence.service marks user online, broadcasts presence:update to relevant rooms
5. On disconnect (including ungraceful — detected via heartbeat timeout):
     presence.service marks offline + lastSeenAt, broadcasts presence:update
```

### 9.4 Namespace Separation
- `/chat`, `/tracking`, `/notifications` as **separate Socket.IO namespaces** (not one firehose namespace) — allows independent scaling policy, independent auth rules if ever needed, and prevents a chat-message burst from head-of-line-blocking a live GPS tracking stream sharing the same connection.

### 9.5 Server→Client Push (Notifications)
`notifications.namespace.ts` provides `emitToUser(userId, payload)` — a thin infrastructure function future modules call after writing to `engagement.notifications` (Day 2 §8), falling back to FCM push (§ Day 1 third-party services) when the user has no active socket connection. This fallback decision itself is infra plumbing (check presence, choose channel), not notification *content*, which stays module-owned.

---

## 10. Security Layer

### 10.1 Transport & Headers
- HTTPS enforced at the load balancer; HSTS header set via Helmet.
- Helmet defaults + explicit CSP tuned for the admin/seller panel origins.
- CORS allow-list is environment-driven (`env.ALLOWED_ORIGINS`), never `*` in production.

### 10.2 Input Safety
- All request bodies pass through a sanitizer stripping script tags / known XSS payloads before validation (§5.3) even runs.
- Parameterized queries only — the `query-builder.ts` (§4.2) makes raw string SQL interpolation structurally unavailable to module code (no escape hatch exposed).
- CSRF: N/A for the token-bearer API pattern (mobile + SPA panels use `Authorization` header, not cookies) — if any cookie-based session is ever introduced for a specific flow, double-submit CSRF tokens would be required at that point.

### 10.3 Secrets
- No secret ever read from a committed file — `env.schema.ts` (§1.2) validates presence/shape at boot; production secrets sourced from AWS Secrets Manager (Day 1 §29), injected as environment variables at container start, never baked into the image.

### 10.4 Password Storage
`password-hasher.service.ts` (§2.1) uses **argon2id** (memory-hard, resistant to GPU cracking) — used only for the minority of accounts that set a password; social/OTP-only accounts have `password_hash = NULL` (Day 2 §1).

### 10.5 Resilience — Circuit Breaker for External Calls
```ts
interface ICircuitBreaker {
  execute<T>(fn: () => Promise<T>, fallback?: () => T): Promise<T>;
}
```
Wraps every outbound call to Firebase, payment gateways, SMS providers, Google Maps — a failing dependency trips the breaker (open state) after a threshold, short-circuiting further calls for a cool-down window instead of piling up slow/hanging requests that would otherwise exhaust the connection pool and take down unrelated functionality.

### 10.6 Security Audit Trail
Every authentication event, permission denial, and rate-limit trip is written to `audit.audit_logs` (Day 2 §10) via the event system (§8), not inline in the middleware — keeps middleware fast/synchronous while still guaranteeing a complete security audit trail.

---

## 11. Monitoring & Logging System

### 11.1 Structured Logging
```
core/logger/
├── logger.ts             # Pino, JSON output, log level from env
└── request-context.ts      # AsyncLocalStorage — correlation ID auto-attached to every
                              # log line emitted during a request's lifecycle, no manual passing
```
Every log line: `{ timestamp, level, message, requestId, userId?, module, ...context }` — shipped to centralized aggregation (ELK/CloudWatch/Datadog per Day 1 §22).

### 11.2 Metrics
- `/metrics` endpoint (Prometheus exposition format) exposing: HTTP request duration histograms (per route+status), DB pool utilization, Redis command latency, queue depth per queue, event-bus publish/consume rates, circuit-breaker state.
- RED method (Rate, Errors, Duration) as the baseline dashboard for every endpoint; USE method (Utilization, Saturation, Errors) for infrastructure resources (DB connections, Redis memory, queue workers).

### 11.3 Health Checks
```
GET /healthz    -- liveness: process is up (used by orchestrator restart policy)
GET /readyz     -- readiness: DB pool reachable, Redis reachable, migrations up-to-date
                   (used by load balancer to gate traffic to a pod)
```
`readyz` failing removes a pod from rotation without killing it — distinguishing "temporarily can't serve" from "should be restarted" is what liveness/readiness separation exists for.

### 11.4 Alerting Triggers (wired to the metrics above)
- Error rate > threshold on any route (5 min window)
- Queue DLQ depth > 0
- Replication lag > threshold (Day 2 §17)
- Circuit breaker open for > N minutes
- Wallet-ledger reconciliation drift (Day 2 §11) — routed as a P1, distinct severity from general alerts

### 11.5 Distributed Tracing (forward-looking hook)
`request-context.ts`'s correlation ID is deliberately compatible with OpenTelemetry trace-id format — full distributed tracing (span propagation across services) is not implemented at Day 3 (single modular monolith, limited value yet) but the correlation-ID convention means adopting OTel later requires no request-flow redesign.

---

## 12. Deployment Foundation

### 12.1 Docker (backend image)
```dockerfile
# Multi-stage — build stage compiles TS, runtime stage ships only dist/ + prod deps
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY . .
RUN pnpm build

FROM node:20-alpine AS runtime
RUN addgroup -S nexfiy && adduser -S nexfiy -G nexfiy
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/package.json ./
USER nexfiy
EXPOSE 3000
HEALTHCHECK CMD wget -qO- http://localhost:3000/healthz || exit 1
CMD ["node", "dist/server.js"]
```
Non-root user, minimal base image, health check baked into the image itself — matches Day 1 §24.

### 12.2 Docker Compose (local dev stack — extends Day 1 §24, adds infra services stood up in this document)
```yaml
services:
  postgres:      { image: postgres:16-alpine, volumes: [pgdata:/var/lib/postgresql/data] }
  redis-cache:    { image: redis:7-alpine }
  redis-queue:     { image: redis:7-alpine }     # isolated instance, §7.1 rationale
  pgbouncer:        { image: pgbouncer/pgbouncer, depends_on: [postgres] }
  backend:           { build: ./apps/backend, depends_on: [postgres, redis-cache, redis-queue, pgbouncer] }
  bull-board:         { image: deadly0/bull-board, depends_on: [redis-queue] }
volumes:
  pgdata:
```

### 12.3 CI/CD Readiness (extends Day 1 §25 with Day-3-specific gates)
```
New CI gates introduced at this layer:
- Boundary lint (Domain layer must not import from core/ or infrastructure/)
- Contract test: every registered DI binding resolves without circular dependency
- Migration dry-run against a throwaway Postgres container
- Load test smoke (k6, minimal) against /healthz and a synthetic auth flow
- Outbox-relay integration test: kill relay mid-dispatch, assert no event loss on restart
```

### 12.4 Environment Parity
`dev → staging → production` all run the identical container image; only `env.schema.ts`-validated environment variables differ — eliminates "works on staging, breaks in prod" class of issues caused by divergent infra config rather than code.

---

## 13. Service Interaction Diagram (text)

```
                              ┌────────────────────┐
                              │   Load Balancer     │
                              └──────────┬──────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    ▼                     ▼                     ▼
             ┌────────────┐        ┌────────────┐        ┌────────────┐
             │ Backend Pod │        │ Backend Pod │        │ Backend Pod │   (stateless, N replicas)
             │  Express +  │        │  Express +  │        │  Express +  │
             │  Socket.IO  │        │  Socket.IO  │        │  Socket.IO  │
             └──────┬──────┘        └──────┬──────┘        └──────┬──────┘
                    │  Middleware chain (§5.4) → Use-Case → Repository
                    │
      ┌─────────────┼───────────────┬───────────────┬──────────────────┐
      ▼             ▼               ▼               ▼                  ▼
┌───────────┐ ┌────────────┐ ┌─────────────┐ ┌──────────────┐  ┌───────────────┐
│ PgBouncer │ │ Redis Cache │ │ Redis Queue  │ │ Redis Pub/Sub │  │ Firebase Admin │
│  → PG     │ │ (cache,     │ │ (BullMQ)     │ │ (Socket.IO    │  │ SDK / Payment  │
│  Primary  │ │  session,   │ │              │ │  adapter,      │  │ Gateways /     │
│  + Read   │ │  rate-limit,│ │              │ │  presence)      │  │ Maps (behind   │
│  Replicas │ │  locks)     │ │              │ │                │  │ circuit breaker)│
└───────────┘ └────────────┘ └──────┬───────┘ └────────────────┘  └───────────────┘
                                     │
                                     ▼
                            ┌─────────────────┐
                            │  Worker Process   │  (separate deployment,
                            │  (job processors, │   scales independently
                            │   event relay,     │   from API pods)
                            │   outbox dispatcher)│
                            └─────────────────┘
```
Backend API pods and the Worker process are **separately deployed and independently scaled** — API pods scale on request throughput, Workers scale on queue depth — this split is what prevents a report-generation spike from starving customer-facing API latency, and vice versa.

---

## 14. Dependency Injection — Composition Root

```
core/di/container.ts registers, in dependency order:
  1. Config (env)
  2. Logger
  3. Database pools (write/read) + TransactionManager
  4. Redis clients (cache, queue) + CacheService + DistributedLockService + RateLimiter
  5. EventBus + Outbox relay
  6. Queue registry + BullMQ connections
  7. Firebase Admin provider + JwtService + AuthTokenService
  8. PermissionRegistry + PolicyEngine
  9. Socket.IO server + Redis adapter + Presence service
  10. [future module bindings — repositories, use-cases — registered per module as they ship]

bootstrap.ts resolves the container, wires Express app.use() chain (§5.4) in the exact
documented order, attaches Socket.IO to the same HTTP server, starts health-check
readiness only after step 9 completes successfully.
```

---

**Day 3 status: complete.** This document defines the full backend infrastructure foundation for NEXFIY — authentication, authorization, repository/CQRS scaffolding, API standards, Redis, queues, events (with transactional outbox for reliability), real-time transport, security, observability, and deployment — with zero business/module logic, exactly as scoped. Product, Order, Payment, and all remaining modules build directly on top of this layer starting Day 4.

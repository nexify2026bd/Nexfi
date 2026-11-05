# NEXIFY — Architecture Enhancement (Enterprise Upgrade)
## Extension to Day 1 Foundation — New Modules Only

*This document extends the original `NEXIFY_ARCHITECTURE_FOUNDATION.md`. Nothing in the original is modified, removed, or duplicated. Same 4-layer Clean Architecture convention applies to every module below.*

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
│   │   ├── process-wallet-refund.usecase.ts   # credits Nexify Wallet (instant)
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

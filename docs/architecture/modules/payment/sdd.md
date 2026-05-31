# Software Design Document — Payment Module

> Medusa v2.15.4

## 1. Module Architecture

```
packages/modules/payment/src/
├── index.ts                         # Module entry, DI wiring
├── joiner-config.ts
├── models/
│   ├── payment-collection.ts        # Collection container
│   ├── payment-session.ts           # Provider-specific session
│   ├── payment.ts                   # Committed payment record
│   ├── capture.ts                   # Capture event
│   ├── refund.ts                    # Refund event
│   ├── refund-reason.ts             # Reason catalogue
│   ├── payment-provider.ts          # Provider registration
│   ├── account-holder.ts            # Customer ↔ provider link
│   └── index.ts
├── services/
│   ├── payment-module.ts            # IPaymentModuleService implementation
│   ├── payment-provider.ts          # PaymentProviderService (provider dispatch)
│   └── index.ts
├── loaders/
│   └── providers.ts                 # Dynamic provider registration at startup
├── providers/                       # Built-in test provider
├── migrations/
└── types/
```

### Provider Loading

At startup, `loaders/providers.ts` dynamically resolves every provider package listed in `options.providers`, instantiates it, and registers it in the DI container under `pp_{type}_{id}`. `PaymentProviderService` maintains a `Map<providerId, IPaymentProvider>` and dispatches all operations to the appropriate provider.

---

## 2. Data Model

### Entity-Relationship Overview

```
PaymentCollection (1) ──── (*) PaymentSession
        │                           │
        │                    (1) Payment ──── (*) Capture
        │                           │
        │                           └────── (*) Refund ──── RefundReason
        │
        └── (*) PaymentProvider [M:M pivot: payment_collection_payment_provider]

AccountHolder (1 per customer per provider)
```

### Status Flows

**PaymentCollection.status**:
```
not_paid → awaiting → authorized → partially_captured → captured
                                                      → partially_refunded
                                                      → refunded
                                                      → canceled
```

**PaymentSession.status**:
```
pending → authorized | requires_more | error | canceled
```

### BigNumber Storage

`PaymentCollection.amount`, `authorized_amount`, `captured_amount`, `refunded_amount` and `Payment.amount`, `Capture.amount`, `Refund.amount` all use the dual-column BigNumber pattern (`raw_amount` JSON + computed `amount` NUMERIC).

---

## 3. Service Layer Design

### Class Hierarchy

```
ModulesSdkUtils.MedusaService<EntityMap>
    └── PaymentModuleService (implements IPaymentModuleService)
            ├── Auto-CRUD: list*, retrieve*, create*, update*, delete*
            ├── Custom: createPaymentSession()   – calls provider.initiatePayment()
            ├── Custom: authorizePaymentSession() – calls provider.authorizePayment()
            │           └── on success: creates Payment record
            ├── Custom: capturePayment()          – calls provider.capturePayment()
            │           └── creates Capture; updates amounts
            ├── Custom: refundPayment()            – calls provider.refundPayment()
            │           └── creates Refund; updates amounts
            ├── Custom: cancelPayment()            – calls provider.cancelPayment()
            └── Custom: processEvent()             – webhook dispatch
```

### PaymentProviderService

`PaymentProviderService` is a **provider registry** injected into `PaymentModuleService`. It:
1. Holds a `Map<string, IPaymentProvider>` of loaded providers
2. Exposes typed dispatch methods: `authorizePayment()`, `capturePayment()`, `refundPayment()`, etc.
3. Normalises amounts (adds currency precision from `defaultCurrencies`)
4. Validates provider availability for given currency

### Amount Reconciliation

After every capture or refund:
- `PaymentCollection.captured_amount` is recomputed as `SUM(captures.amount)` on all payments in the collection
- `PaymentCollection.refunded_amount` is recomputed as `SUM(refunds.amount)`
- `PaymentCollection.status` is derived from these sums against `amount`

### Dependency Injection

```ts
type InjectedDependencies = {
  logger?: Logger
  baseRepository: DAL.RepositoryService
  paymentService: IMedusaInternalService<Payment>
  captureService: IMedusaInternalService<Capture>
  refundService: IMedusaInternalService<Refund>
  paymentSessionService: IMedusaInternalService<PaymentSession>
  paymentCollectionService: IMedusaInternalService<PaymentCollection>
  accountHolderService: IMedusaInternalService<AccountHolder>
  paymentProviderService: PaymentProviderService
}
```

---

## 4. Repository Pattern

No custom repository layer — all persistence delegated to `MedusaInternalService` wrappers. `PaymentCollection` cascade-deletes `payment_sessions` and `payments`. `Payment` cascade-deletes `refunds` and `captures`.

---

## 5. Events Emitted

| Event | Trigger |
|---|---|
| `payment-collection.created` | `createPaymentCollections()` |
| `payment-collection.updated` | `updatePaymentCollections()` |
| `payment-collection.deleted` | `deletePaymentCollections()` |
| `payment.captured` | `capturePayment()` success |
| `payment.refunded` | `refundPayment()` success |
| `payment.authorized` | `authorizePaymentSession()` success |
| `payment.payment_collection_status_changed` | Collection status transition |

---

## 6. Error Handling

| Scenario | Error Type | Message |
|---|---|---|
| Provider not found | `NOT_FOUND` | `"Payment provider pp_{id} not found"` |
| Session already authorized | `NOT_ALLOWED` | `"Payment session is already authorized"` |
| Capture exceeds payment amount | `INVALID_DATA` | `"Capture amount exceeds remaining capturable amount"` |
| Refund exceeds captured amount | `INVALID_DATA` | `"Refund amount exceeds captured amount"` |
| Collection already completed | `NOT_ALLOWED` | `"Payment collection is already completed"` |
| Provider error | `INVALID_DATA` | Wraps provider-thrown error message |
| Invalid currency precision | `INVALID_DATA` | Based on `defaultCurrencies` precision map |

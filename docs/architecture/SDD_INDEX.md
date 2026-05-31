# SDD Index — All Custom Modules

> Custom modules live under `apps/backend/src/modules/`. Each follows the
> Port/Adapter pattern with a stable contract, a Medusa-backed adapter, and
> a feature flag for safe rollout.

## Module Map

| Module | Domain | Feature Flag | SDD |
|---|---|---|---|
| [`admin-bff`](#admin-bff) | Admin dashboard BFF | `FEATURE_ADMIN_BFF_V2` | [→](../../apps/backend/src/modules/admin-bff/sdd/00-context/context.md) |
| [`checkout-payment`](#checkout-payment) | Checkout + payment lifecycle | `FEATURE_CHECKOUT_PAYMENT_V2` | [→](../../apps/backend/src/modules/checkout-payment/sdd/00-context/context.md) |
| [`commerce-catalog`](#commerce-catalog) | Product catalog + pricing | `FEATURE_COMMERCE_CATALOG_V2` | [→](../../apps/backend/src/modules/commerce-catalog/sdd/00-context/context.md) |
| [`customer-identity`](#customer-identity) | Auth + customer profile | `FEATURE_CUSTOMER_IDENTITY_V2` | [→](../../apps/backend/src/modules/customer-identity/sdd/00-context/context.md) |
| [`loyalty`](#loyalty) | Points + rewards | `FEATURE_LOYALTY_V2` | [→](../../apps/backend/src/modules/loyalty/sdd/00-context/context.md) |
| [`order-fulfillment`](#order-fulfillment) | Order lifecycle + shipping | `FEATURE_ORDER_FULFILLMENT_V2` | [→](../../apps/backend/src/modules/order-fulfillment/sdd/00-context/context.md) |

---

## admin-bff

**Path:** `apps/backend/src/modules/admin-bff/`  
**Feature flag:** `FEATURE_ADMIN_BFF_V2`  
**Contract:** `ports/admin-bff.contract.ts` → `AdminBffContract`  
**Adapter:** `adapters/medusa/admin-bff.medusa-adapter.ts`

Backend-for-Frontend layer. Aggregates and transforms data from multiple Medusa
modules for the Admin Dashboard so the UI doesn't fan-out to multiple API routes.

| SDD Section | File |
|---|---|
| Context | [00-context/context.md](../../apps/backend/src/modules/admin-bff/sdd/00-context/context.md) |
| Requirements | [01-requirements/requirements.md](../../apps/backend/src/modules/admin-bff/sdd/01-requirements/requirements.md) |
| Contracts | [02-contracts/contracts.md](../../apps/backend/src/modules/admin-bff/sdd/02-contracts/contracts.md) |
| Design | [03-design/design.md](../../apps/backend/src/modules/admin-bff/sdd/03-design/design.md) |
| Delivery | [04-delivery/delivery.md](../../apps/backend/src/modules/admin-bff/sdd/04-delivery/delivery.md) |
| Validation | [05-validation/validation.md](../../apps/backend/src/modules/admin-bff/sdd/05-validation/validation.md) |
| Operations | [06-operations/operations.md](../../apps/backend/src/modules/admin-bff/sdd/06-operations/operations.md) |

---

## checkout-payment

**Path:** `apps/backend/src/modules/checkout-payment/`  
**Feature flag:** `FEATURE_CHECKOUT_PAYMENT_V2`  
**Contract:** `ports/checkout-payment.contract.ts` → `CheckoutPaymentContract`  
**Adapter:** `adapters/medusa/checkout-payment.medusa-adapter.ts`

Orchestrates the cart → checkout → payment transition. Adds business-rule
validation, handles payment provider callbacks, and coordinates rollback via
workflow compensation when payment fails.

| SDD Section | File |
|---|---|
| Context | [00-context/context.md](../../apps/backend/src/modules/checkout-payment/sdd/00-context/context.md) |
| Requirements | [01-requirements/requirements.md](../../apps/backend/src/modules/checkout-payment/sdd/01-requirements/requirements.md) |
| Contracts | [02-contracts/contracts.md](../../apps/backend/src/modules/checkout-payment/sdd/02-contracts/contracts.md) |
| Design | [03-design/design.md](../../apps/backend/src/modules/checkout-payment/sdd/03-design/design.md) |
| Delivery | [04-delivery/delivery.md](../../apps/backend/src/modules/checkout-payment/sdd/04-delivery/delivery.md) |
| Validation | [05-validation/validation.md](../../apps/backend/src/modules/checkout-payment/sdd/05-validation/validation.md) |
| Operations | [06-operations/operations.md](../../apps/backend/src/modules/checkout-payment/sdd/06-operations/operations.md) |

---

## commerce-catalog

**Path:** `apps/backend/src/modules/commerce-catalog/`  
**Feature flag:** `FEATURE_COMMERCE_CATALOG_V2`  
**Contract:** `ports/commerce-catalog.contract.ts` → `CommerceCatalogContract`  
**Adapter:** `adapters/medusa/commerce-catalog.medusa-adapter.ts`

Extends the product and pricing modules with custom catalog attributes, collection
rules, and catalog-level pricing. All queries use `query.graph()` — never direct
module service calls from routes.

| SDD Section | File |
|---|---|
| Context | [00-context/context.md](../../apps/backend/src/modules/commerce-catalog/sdd/00-context/context.md) |
| Requirements | [01-requirements/requirements.md](../../apps/backend/src/modules/commerce-catalog/sdd/01-requirements/requirements.md) |
| Contracts | [02-contracts/contracts.md](../../apps/backend/src/modules/commerce-catalog/sdd/02-contracts/contracts.md) |
| Design | [03-design/design.md](../../apps/backend/src/modules/commerce-catalog/sdd/03-design/design.md) |
| Delivery | [04-delivery/delivery.md](../../apps/backend/src/modules/commerce-catalog/sdd/04-delivery/delivery.md) |
| Validation | [05-validation/validation.md](../../apps/backend/src/modules/commerce-catalog/sdd/05-validation/validation.md) |
| Operations | [06-operations/operations.md](../../apps/backend/src/modules/commerce-catalog/sdd/06-operations/operations.md) |

---

## customer-identity

**Path:** `apps/backend/src/modules/customer-identity/`  
**Feature flag:** `FEATURE_CUSTOMER_IDENTITY_V2`  
**Contract:** `ports/customer-identity.contract.ts` → `CustomerIdentityContract`  
**Adapter:** `adapters/medusa/customer-identity.medusa-adapter.ts`

Customer authentication (email/password, social), profile lifecycle, session and
token management. Auth context is always read via `AuthenticatedMedusaRequest` —
never via manual `req.auth_context` inspection.

| SDD Section | File |
|---|---|
| Context | [00-context/context.md](../../apps/backend/src/modules/customer-identity/sdd/00-context/context.md) |
| Requirements | [01-requirements/requirements.md](../../apps/backend/src/modules/customer-identity/sdd/01-requirements/requirements.md) |
| Contracts | [02-contracts/contracts.md](../../apps/backend/src/modules/customer-identity/sdd/02-contracts/contracts.md) |
| Design | [03-design/design.md](../../apps/backend/src/modules/customer-identity/sdd/03-design/design.md) |
| Delivery | [04-delivery/delivery.md](../../apps/backend/src/modules/customer-identity/sdd/04-delivery/delivery.md) |
| Validation | [05-validation/validation.md](../../apps/backend/src/modules/customer-identity/sdd/05-validation/validation.md) |
| Operations | [06-operations/operations.md](../../apps/backend/src/modules/customer-identity/sdd/06-operations/operations.md) |

---

## loyalty

**Path:** `apps/backend/src/modules/loyalty/`  
**Feature flag:** `FEATURE_LOYALTY_V2`  
**Contract:** `ports/loyalty.contract.ts` → `LoyaltyContract`  
**Adapter:** `adapters/medusa/loyalty.medusa-adapter.ts`

Loyalty points and rewards. Points are integer-only (no floating point). Accrual
is triggered by `order.placed` events. Redemption is validated in a workflow step
before capture to prevent negative balances.

| SDD Section | File |
|---|---|
| Context | [00-context/context.md](../../apps/backend/src/modules/loyalty/sdd/00-context/context.md) |
| Requirements | [01-requirements/requirements.md](../../apps/backend/src/modules/loyalty/sdd/01-requirements/requirements.md) |
| Contracts | [02-contracts/contracts.md](../../apps/backend/src/modules/loyalty/sdd/02-contracts/contracts.md) |
| Design | [03-design/design.md](../../apps/backend/src/modules/loyalty/sdd/03-design/design.md) |
| Delivery | [04-delivery/delivery.md](../../apps/backend/src/modules/loyalty/sdd/04-delivery/delivery.md) |
| Validation | [05-validation/validation.md](../../apps/backend/src/modules/loyalty/sdd/05-validation/validation.md) |
| Operations | [06-operations/operations.md](../../apps/backend/src/modules/loyalty/sdd/06-operations/operations.md) |

---

## order-fulfillment

**Path:** `apps/backend/src/modules/order-fulfillment/`  
**Feature flag:** `FEATURE_ORDER_FULFILLMENT_V2`  
**Contract:** `ports/order-fulfillment.contract.ts` → `OrderFulfillmentContract`  
**Adapter:** `adapters/medusa/order-fulfillment.medusa-adapter.ts`

Order lifecycle from placement to delivery. All mutations go through `core-flows`
workflows — never direct module service calls from routes. Worker must be healthy
for fulfillment workflow steps to execute.

| SDD Section | File |
|---|---|
| Context | [00-context/context.md](../../apps/backend/src/modules/order-fulfillment/sdd/00-context/context.md) |
| Requirements | [01-requirements/requirements.md](../../apps/backend/src/modules/order-fulfillment/sdd/01-requirements/requirements.md) |
| Contracts | [02-contracts/contracts.md](../../apps/backend/src/modules/order-fulfillment/sdd/02-contracts/contracts.md) |
| Design | [03-design/design.md](../../apps/backend/src/modules/order-fulfillment/sdd/03-design/design.md) |
| Delivery | [04-delivery/delivery.md](../../apps/backend/src/modules/order-fulfillment/sdd/04-delivery/delivery.md) |
| Validation | [05-validation/validation.md](../../apps/backend/src/modules/order-fulfillment/sdd/05-validation/validation.md) |
| Operations | [06-operations/operations.md](../../apps/backend/src/modules/order-fulfillment/sdd/06-operations/operations.md) |

---
name: docs-glossary
description: Reference glossary that maps shorthand documentation names to their actual file paths in this repo. Load this skill when a user says "based on the [X] doc", "following the [X] documentation", "use the [X] guide", or references any documentation by a short name or alias. Resolves ambiguous doc references before acting on them.
---

# Documentation Glossary & Reference Book

This is the **single source of truth** for all documentation aliases used in this project. When a user says "based on the medusa-framework doc" or "following the architecture docs", consult this file to resolve the exact file path before reading or acting on it.

---

## How to Use

1. User says something like: *"based on medusa-framework doc, do X"*
2. Look up the alias in the table below
3. Read the resolved file path before proceeding
4. If the alias is ambiguous or not listed, ask the user to clarify

---

## Master Reference Table

| Alias / Shorthand | Resolves To | Description |
|---|---|---|
| `medusa-framework doc` | `docs/llms-full-Medusa-framework-docs.md` | Full official Medusa framework documentation (168k lines). Covers all concepts: modules, workflows, API routes, subscribers, jobs, providers, admin, storefront, deployment, etc. |
| `medusa doc` | `docs/llms-full-Medusa-framework-docs.md` | Same as above |
| `framework doc` | `docs/llms-full-Medusa-framework-docs.md` | Same as above |
| `llms doc` | `docs/llms-full-Medusa-framework-docs.md` | Same as above |
| `official doc` | `docs/llms-full-Medusa-framework-docs.md` | Same as above |
| `medusa docs` | `docs/llms-full-Medusa-framework-docs.md` | Same as above |
| `architecture doc` | `docs/architecture/README.md` | Index of all architecture documentation (arc42, SDD, SpecKit, UML, ArchiMate) |
| `architecture docs` | `docs/architecture/README.md` | Same as above |
| `arc42` | `docs/architecture/overall/arc42.md` | Full arc42 architecture document for the overall system |
| `SDD` | `docs/architecture/overall/sdd.md` | Software Design Document for the overall system |
| `speckit` | `docs/architecture/overall/speckit.md` | SpecKit requirements and specification document |
| `architecture roadmap` | `docs/architecture/ARCHITECTURE_ROADMAP.md` | Architecture evolution and roadmap document |
| `SDD index` | `docs/architecture/SDD_INDEX.md` | Index of all per-module SDD documents |

---

## Skills Reference Table

These are **agent skills** (not docs), but users often reference them by short name:

| Alias / Shorthand | Skill File | When Active |
|---|---|---|
| `building-with-medusa skill` | `.agents/skills/building-with-medusa/SKILL.md` | Any backend Medusa development task |
| `docker skill` | `.agents/skills/docker-setup/SKILL.md` | Docker setup, containerization, compose config |
| `storefront skill` | `.agents/skills/storefront-best-practices/SKILL.md` | Storefront / frontend development |
| `db-generate skill` | `.agents/skills/db-generate/SKILL.md` | Generating database migrations |
| `db-migrate skill` | `.agents/skills/db-migrate/SKILL.md` | Running database migrations |
| `new-user skill` | `.agents/skills/new-user/SKILL.md` | Creating a Medusa admin user |
| `learning skill` | `.agents/skills/learning-medusa/SKILL.md` | Interactive Medusa learning guide |

---

## Per-Module Architecture Docs

Each module has its own documentation subdirectory under `docs/architecture/modules/[module-name]/`.

### Commerce Modules

| Module | Arc42 | SDD | SpecKit |
|---|---|---|---|
| Product | `docs/architecture/modules/product/arc42.md` | `…/product/sdd.md` | `…/product/speckit.md` |
| Order | `docs/architecture/modules/order/arc42.md` | `…/order/sdd.md` | `…/order/speckit.md` |
| Cart | `docs/architecture/modules/cart/arc42.md` | `…/cart/sdd.md` | `…/cart/speckit.md` |
| Payment | `docs/architecture/modules/payment/arc42.md` | `…/payment/sdd.md` | `…/payment/speckit.md` |
| Customer | `docs/architecture/modules/customer/arc42.md` | `…/customer/sdd.md` | `…/customer/speckit.md` |
| Pricing | `docs/architecture/modules/pricing/arc42.md` | `…/pricing/sdd.md` | `…/pricing/speckit.md` |
| Inventory | `docs/architecture/modules/inventory/arc42.md` | `…/inventory/sdd.md` | `…/inventory/speckit.md` |
| Stock Location | `docs/architecture/modules/stock-location/arc42.md` | `…/stock-location/sdd.md` | `…/stock-location/speckit.md` |
| Promotion | `docs/architecture/modules/promotion/arc42.md` | `…/promotion/sdd.md` | `…/promotion/speckit.md` |
| Fulfillment | `docs/architecture/modules/fulfillment/arc42.md` | `…/fulfillment/sdd.md` | `…/fulfillment/speckit.md` |
| Auth | `docs/architecture/modules/auth/arc42.md` | `…/auth/sdd.md` | `…/auth/speckit.md` |
| API Key | `docs/architecture/modules/api-key/arc42.md` | `…/api-key/sdd.md` | `…/api-key/speckit.md` |
| Currency | `docs/architecture/modules/currency/arc42.md` | `…/currency/sdd.md` | `…/currency/speckit.md` |
| Region | `docs/architecture/modules/region/arc42.md` | `…/region/sdd.md` | `…/region/speckit.md` |
| Sales Channel | `docs/architecture/modules/sales-channel/arc42.md` | `…/sales-channel/sdd.md` | `…/sales-channel/speckit.md` |
| Tax | `docs/architecture/modules/tax/arc42.md` | `…/tax/sdd.md` | `…/tax/speckit.md` |
| User | `docs/architecture/modules/user/arc42.md` | `…/user/sdd.md` | `…/user/speckit.md` |
| Notification | `docs/architecture/modules/notification/arc42.md` | `…/notification/sdd.md` | `…/notification/speckit.md` |
| Store | `docs/architecture/modules/store/arc42.md` | `…/store/sdd.md` | `…/store/speckit.md` |
| Index (search) | `docs/architecture/modules/index/arc42.md` | `…/index/sdd.md` | `…/index/speckit.md` |
| Analytics | `docs/architecture/modules/analytics/arc42.md` | `…/analytics/sdd.md` | `…/analytics/speckit.md` |
| RBAC | `docs/architecture/modules/rbac/arc42.md` | `…/rbac/sdd.md` | `…/rbac/speckit.md` |
| Settings | `docs/architecture/modules/settings/arc42.md` | `…/settings/sdd.md` | `…/settings/speckit.md` |

### Infrastructure Modules

| Module | Arc42 | SDD | SpecKit |
|---|---|---|---|
| Event Bus (Local) | `docs/architecture/modules/event-bus-local/arc42.md` | `…/event-bus-local/sdd.md` | `…/event-bus-local/speckit.md` |
| Event Bus (Redis) | `docs/architecture/modules/event-bus-redis/arc42.md` | `…/event-bus-redis/sdd.md` | `…/event-bus-redis/speckit.md` |
| Cache (In-Memory) | `docs/architecture/modules/cache-inmemory/arc42.md` | `…/cache-inmemory/sdd.md` | `…/cache-inmemory/speckit.md` |
| Cache (Redis) | `docs/architecture/modules/cache-redis/arc42.md` | `…/cache-redis/sdd.md` | `…/cache-redis/speckit.md` |
| Workflow Engine (In-Memory) | `docs/architecture/modules/workflow-engine-inmemory/arc42.md` | `…/workflow-engine-inmemory/sdd.md` | `…/workflow-engine-inmemory/speckit.md` |
| Workflow Engine (Redis) | `docs/architecture/modules/workflow-engine-redis/arc42.md` | `…/workflow-engine-redis/sdd.md` | `…/workflow-engine-redis/speckit.md` |
| File | `docs/architecture/modules/file/arc42.md` | `…/file/sdd.md` | `…/file/speckit.md` |
| Locking | `docs/architecture/modules/locking/arc42.md` | `…/locking/sdd.md` | `…/locking/speckit.md` |
| Link Modules | `docs/architecture/modules/link-modules/arc42.md` | `…/link-modules/sdd.md` | `…/link-modules/speckit.md` |

---

## Per-Package Architecture Docs

| Package | Arc42 | SDD | SpecKit |
|---|---|---|---|
| Framework | `docs/architecture/packages/framework/arc42.md` | `…/framework/sdd.md` | `…/framework/speckit.md` |
| Core Flows | `docs/architecture/packages/core-flows/arc42.md` | `…/core-flows/sdd.md` | `…/core-flows/speckit.md` |
| Workflows SDK | `docs/architecture/packages/workflows-sdk/arc42.md` | `…/workflows-sdk/sdd.md` | `…/workflows-sdk/speckit.md` |
| Types | `docs/architecture/packages/types/arc42.md` | `…/types/sdd.md` | `…/types/speckit.md` |
| Utils | `docs/architecture/packages/utils/arc42.md` | `…/utils/sdd.md` | `…/utils/speckit.md` |
| Modules SDK | `docs/architecture/packages/modules-sdk/arc42.md` | `…/modules-sdk/sdd.md` | `…/modules-sdk/speckit.md` |
| JS SDK | `docs/architecture/packages/js-sdk/arc42.md` | `…/js-sdk/sdd.md` | `…/js-sdk/speckit.md` |

---

## Per-App Architecture Docs

| App | Arc42 | SDD | SpecKit |
|---|---|---|---|
| Backend (`apps/backend`) | `docs/architecture/apps/backend/arc42.md` | `…/backend/sdd.md` | `…/backend/speckit.md` |
| Storefront (`apps/storefront`) | `docs/architecture/apps/storefront/arc42.md` | `…/storefront/sdd.md` | `…/storefront/speckit.md` |

---

## Per-Provider Architecture Docs

Located under `docs/architecture/providers/[provider-name]/`.

| Provider | Arc42 | SDD |
|---|---|---|
| Stripe | `docs/architecture/providers/stripe/arc42.md` | `…/stripe/sdd.md` |
| Auth Email+Password | `docs/architecture/providers/auth-emailpass/arc42.md` | `…/auth-emailpass/sdd.md` |
| Auth GitHub | `docs/architecture/providers/auth-github/arc42.md` | `…/auth-github/sdd.md` |
| Auth Google | `docs/architecture/providers/auth-google/arc42.md` | `…/auth-google/sdd.md` |
| File S3 | `docs/architecture/providers/file-s3/arc42.md` | `…/file-s3/sdd.md` |
| File Local | `docs/architecture/providers/file-local/arc42.md` | `…/file-local/sdd.md` |
| Fulfillment Manual | `docs/architecture/providers/fulfillment-manual/arc42.md` | `…/fulfillment-manual/sdd.md` |
| Notification SendGrid | `docs/architecture/providers/notification-sendgrid/arc42.md` | `…/notification-sendgrid/sdd.md` |
| Cache Redis | `docs/architecture/providers/caching-redis/arc42.md` | `…/caching-redis/sdd.md` |

---

## Building-with-Medusa Reference Files

The `building-with-medusa` skill has detailed reference files for each topic:

| Topic | File |
|---|---|
| Custom modules & data models | `.agents/skills/building-with-medusa/reference/custom-modules.md` |
| Workflows & steps | `.agents/skills/building-with-medusa/reference/workflows.md` |
| API routes | `.agents/skills/building-with-medusa/reference/api-routes.md` |
| Module links | `.agents/skills/building-with-medusa/reference/module-links.md` |
| Querying data | `.agents/skills/building-with-medusa/reference/querying-data.md` |
| Authentication | `.agents/skills/building-with-medusa/reference/authentication.md` |
| Data models | `.agents/skills/building-with-medusa/reference/data-models.md` |
| Error handling | `.agents/skills/building-with-medusa/reference/error-handling.md` |
| Frontend integration | `.agents/skills/building-with-medusa/reference/frontend-integration.md` |
| Scheduled jobs | `.agents/skills/building-with-medusa/reference/scheduled-jobs.md` |
| Subscribers & events | `.agents/skills/building-with-medusa/reference/subscribers-and-events.md` |
| Workflow hooks | `.agents/skills/building-with-medusa/reference/workflow-hooks.md` |
| Troubleshooting | `.agents/skills/building-with-medusa/reference/troubleshooting.md` |

---

## Alias Resolution Examples

| User says… | Read this file |
|---|---|
| "based on the medusa-framework doc, dockerize the app" | `docs/llms-full-Medusa-framework-docs.md` → Docker section |
| "following the official medusa docs" | `docs/llms-full-Medusa-framework-docs.md` |
| "based on the arc42 doc" | `docs/architecture/overall/arc42.md` |
| "check the product module SDD" | `docs/architecture/modules/product/sdd.md` |
| "use the storefront skill" | `.agents/skills/storefront-best-practices/SKILL.md` |
| "based on the architecture docs" | `docs/architecture/README.md` (then follow links) |
| "check the order module docs" | `docs/architecture/modules/order/` directory |
| "following the workflows reference" | `.agents/skills/building-with-medusa/reference/workflows.md` |
| "based on the docker skill" | `.agents/skills/docker-setup/SKILL.md` |

---

## Adding New Entries

When new documentation files are added to this project:
1. Open this file
2. Add a row to the appropriate table
3. Include: alias(es) the user might say, the exact relative file path, and a one-line description
4. Commit with: `docs(agents): update docs-glossary with [new doc name]`

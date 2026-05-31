# Documentation Index

This directory contains architecture and design documentation for the Medusa SCS platform.

## Quick Navigation

| Document | Where | Description |
|---|---|---|
| **Architecture Roadmap** | [`architecture/ARCHITECTURE_ROADMAP.md`](architecture/ARCHITECTURE_ROADMAP.md) | Evolutionary path from vanilla Medusa to the current upgrade-safe platform |
| **Service Connections (UML)** | [`architecture/service-connections.puml`](architecture/service-connections.puml) | PlantUML component diagram: all containers and their connections |
| **Module Adapter Pattern (UML)** | [`architecture/module-adapter-pattern.puml`](architecture/module-adapter-pattern.puml) | PlantUML class diagram: Port/Adapter + Feature Flags pattern |
| **SDD Index** | [`architecture/SDD_INDEX.md`](architecture/SDD_INDEX.md) | Index of all 6 module System Design Documents |

## Module SDDs

Each module under `apps/backend/src/modules/` has a `sdd/` directory with:

| Section | Folder | Content |
|---|---|---|
| 00 Context | `sdd/00-context/context.md` | Domain purpose and responsibilities |
| 01 Requirements | `sdd/01-requirements/requirements.md` | Functional and non-functional requirements |
| 02 Contracts | `sdd/02-contracts/contracts.md` | Interface specification and API surface |
| 03 Design | `sdd/03-design/design.md` | Port/Adapter architecture, API routes, upgrade notes |
| 04 Delivery | `sdd/04-delivery/delivery.md` | Build, migration, and deployment notes |
| 05 Validation | `sdd/05-validation/validation.md` | Testing strategy and contract verification |
| 06 Operations | `sdd/06-operations/operations.md` | Running, feature flags, upgrade checklist, troubleshooting |

## Rendering UML Diagrams

PlantUML files (`.puml`) can be rendered:
- Online: [plantuml.com](https://www.plantuml.com/plantuml/uml/)
- VS Code: [PlantUML extension](https://marketplace.visualstudio.com/items?itemName=jebbs.plantuml)
- CLI: `plantuml docs/architecture/service-connections.puml`

## Running the Platform

```bash
# Start full dev stack (API + Worker + Storefront + Postgres + Redis)
yarn docker:up

# Tear down
yarn docker:down

# Upgrade safety checks (run before every Medusa version bump)
yarn verify:upgrade-safety
```

See [`ARCHITECTURE_ROADMAP.md` § Upgrade Runbook](architecture/ARCHITECTURE_ROADMAP.md#8-upgrade-runbook) for the complete upgrade procedure.

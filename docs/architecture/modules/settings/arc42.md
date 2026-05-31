# arc42 Architecture Document — Settings Module

## 1. Introduction and Goals

### 1.1 Requirements Overview
The Settings module must provide a flexible, schema-less key-value store that supports hierarchical scope resolution, serves both core Medusa configuration and third-party plugins, and integrates transparently with the existing store admin API.

### 1.2 Quality Goals

| Priority | Quality Goal   | Scenario                                                         |
|----------|----------------|------------------------------------------------------------------|
| 1        | Flexibility    | A plugin can store custom settings without a schema migration    |
| 2        | Correctness    | User-level setting always overrides store-level override         |
| 3        | Simplicity     | A plugin developer can read/write a setting in two lines of code |
| 4        | Performance    | Setting read adds < 2ms to a request (index-backed query)        |

### 1.3 Stakeholders

| Role              | Expectation                                               |
|-------------------|-----------------------------------------------------------|
| Platform operator | Configure store defaults without editing code             |
| Plugin developer  | Namespace-safe key storage without schema ownership       |
| Core developer    | Central location for cross-module configuration data      |

---

## 2. Architecture Constraints

- No enforced schema on values; any valid JSON must be accepted.
- Must not become a replacement for domain-specific entities (e.g., currency tables).
- Key namespacing must be documented but not technically enforced (convention over constraint).

---

## 3. System Scope and Context

```
┌──────────────────┐     ┌────────────────┐     ┌───────────────┐
│  Admin API       │     │  Plugin/Module  │     │  Core Module  │
│  /admin/stores   │     │  onBoot()       │     │  (e.g., Auth) │
└────────┬─────────┘     └───────┬────────┘     └──────┬────────┘
         │                       │                      │
         └───────────────────────▼──────────────────────┘
                                 │
                  ┌──────────────▼──────────────┐
                  │      SettingsModuleService   │
                  │  get / set / delete / list   │
                  └──────────────┬──────────────┘
                                 │
                  ┌──────────────▼──────────────┐
                  │      setting table (DB)      │
                  │  (scope, key, value JSONB)   │
                  └─────────────────────────────┘
```

---

## 4. Solution Strategy

- **Single generic entity** (`setting`) avoids schema proliferation.
- **Scope hierarchy** (user → store → global) provides override semantics without complex merge logic.
- **JSONB value column** supports all JSON types without serialisation overhead.
- **Unique index on (scope, key)** provides O(log n) lookup per scope level.

---

## 5. Building Block View

### Level 1

```
SettingsModule
  ├── SettingsModuleService     (MedusaService<{Setting}>)
  ├── Setting                   (entity: id, scope, key, value)
  └── ScopeResolver             (hierarchy walker utility)
```

### Level 2 — ScopeResolver

```
ScopeResolver.resolve(key, scopeChain)
  ├── query DB for scope[0] + key
  ├── if found → return value
  ├── else query DB for scope[1] + key
  └── ... until scopeChain exhausted → return undefined
```

---

## 6. Runtime View

### Scenario: Analytics Opt-Out Check

1. `AnalyticsModuleService.track()` called.
2. Settings service called: `get("analytics.opt_out", { scope: "user:usr_01", fallback: ["global"] })`.
3. DB query 1: `SELECT value FROM setting WHERE scope = 'user:usr_01' AND key = 'analytics.opt_out'` — returns `true`.
4. Opt-out confirmed; `track()` returns early without calling provider.

---

## 7. Deployment View

The Settings module runs in-process. The `setting` table is in the main Medusa PostgreSQL database. No external services are required.

---

## 8. Cross-Cutting Concerns

### Namespacing
Key collisions between plugins are prevented by convention: all plugin-owned keys must be prefixed with the plugin's registered slug. No runtime enforcement; enforced by code review.

### Performance
Each `get()` call with a 3-level fallback makes up to 3 sequential DB queries. A composite index on `(key, scope)` keeps each under 1ms. For hot settings (e.g., opt-out on every analytics call), callers should cache the result within the request lifecycle.

### Large Values
Values exceeding 64KB are rejected with `INVALID_DATA`. This prevents accidental misuse of Settings as a document store.

---

## 9. Architecture Decisions

| ID  | Decision                                    | Rationale                                                       |
|-----|---------------------------------------------|-----------------------------------------------------------------|
| AD1 | Schema-less JSONB values                    | Avoids migration overhead for each new setting type             |
| AD2 | Scope hierarchy resolved in application layer | DB-level row-security would couple schema to auth logic        |
| AD3 | No caching in the Settings module itself    | Consumers are better positioned to decide their cache strategy  |
| AD4 | Settings surfaced through Store API endpoint | Keeps the API surface minimal; settings feel like store attributes |

---

## 10. Quality Scenarios

| Quality      | Scenario                                               | Measure                                 |
|--------------|--------------------------------------------------------|-----------------------------------------|
| Flexibility  | New plugin adds a setting key without a PR to Settings | Key accepted; value stored; no migration|
| Correctness  | User sets opt-out; global default is `false`           | User-level value returned; no leakage   |
| Performance  | Setting read with 2-level fallback                     | < 3ms total (2 indexed queries)         |

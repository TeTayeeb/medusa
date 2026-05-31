# SpecKit — Link Modules

## Module Identity

| Attribute      | Value                                      |
|----------------|--------------------------------------------|
| Module Name    | `@medusajs/link-modules` (framework)       |
| Version        | 2.15.4                                     |
| Module Key     | N/A (framework-level concept)              |
| Type           | Infrastructure / Cross-Module Relations    |
| Database Tables| One pivot table per defined link           |
| Event Emitter  | No                                         |
| Event Consumer | Yes (entity deletion events for cascade)   |

---

## Functional Specifications

### SPEC-LINK-001: defineLink API
**Description**: `defineLink(moduleA.linkable.entityA, moduleB.linkable.entityB)` MUST generate: (1) a pivot table DDL, (2) an IoC binding for `RemoteLinkService`, and (3) query graph registration.  
**Acceptance**: After calling `defineLink`, `query.graph({ entity: "entityA", fields: ["entityBs.*"] })` resolves data through the pivot table.

### SPEC-LINK-002: No Cross-Module Foreign Keys
**Description**: Generated pivot tables MUST NOT contain `FOREIGN KEY` constraints referencing tables in other module schemas.  
**Acceptance**: `\d pivot_table_name` in PostgreSQL shows no FK constraints.

### SPEC-LINK-003: Unique Link Constraint
**Description**: A pivot table MUST enforce uniqueness on `(entity_a_id, entity_b_id)`. Duplicate link creation MUST be handled via upsert (no error thrown).  
**Acceptance**: `remoteLink.create(link)` called twice with same IDs → one record in DB; no error.

### SPEC-LINK-004: Link Cascade on Entity Deletion
**Description**: When a parent entity is soft-deleted, all pivot records where that entity appears MUST be soft-deleted within the same transaction or via event subscriber.  
**Acceptance**: `productService.softDelete(["prod_01"])` → `product_sales_channel` rows with `product_id = prod_01` have `deleted_at` set.

### SPEC-LINK-005: Bidirectional Traversal
**Description**: A link defined as A → B MUST be traversable from both A (fields: `bs`) and B (fields: `as`) in `query.graph()`.  
**Acceptance**: `query.graph({ entity: "sales_channel", fields: ["products.*"] })` returns products linked to the sales channel.

### SPEC-LINK-006: Relationship Metadata
**Description**: The `data` JSONB column MUST accept optional metadata on link creation. Metadata MUST be returned when the link is queried.  
**Acceptance**: `remoteLink.create({ ..., data: { rank: 1 } })` → `query.graph` returns `data.rank = 1` on the link.

### SPEC-LINK-007: Hard Delete Cascade
**Description**: Hard-deleting a parent entity MUST hard-delete its pivot records (not just soft-delete).  
**Acceptance**: `productService.delete(["prod_01"])` → all `product_*` pivot rows for `prod_01` are hard-deleted.

### SPEC-LINK-008: dismiss() Operation
**Description**: `RemoteLinkService.dismiss()` MUST remove a specific link record without triggering full cascade logic. Used for intentional relationship removal (e.g., un-assigning a product from a sales channel).  
**Acceptance**: `dismiss({ product_id, sales_channel_id })` → pivot row removed; other product links unaffected.

---

## Non-Functional Specifications

### SPEC-LINK-NFR-001: Cross-Module Query Performance
**Description**: A `query.graph()` traversal through one link MUST complete in < 10ms for up to 1000 parent entities.  
**Target**: Two indexed queries; < 10ms p95.

### SPEC-LINK-NFR-002: Migration Idempotency
**Description**: Running `db:migrate` multiple times MUST NOT create duplicate pivot tables or constraint errors.  
**Target**: `db:migrate` run 3 times → identical schema state each time.

---

## RemoteLinkService Contract

```typescript
interface IRemoteLinkService {
  create(data: LinkDefinition | LinkDefinition[]): Promise<void>
  delete(data: LinkDefinition | LinkDefinition[]): Promise<void>
  softDelete(data: LinkDefinition | LinkDefinition[]): Promise<void>
  restore(data: LinkDefinition | LinkDefinition[]): Promise<void>
  dismiss(data: LinkDefinition | LinkDefinition[]): Promise<void>
}
```

---

## Configuration

Link Modules require no explicit configuration. Links are automatically registered when `defineLink()` is called in a module's link definition file and both linked modules are active.

---

## Test Checklist

- [ ] `defineLink` registers pivot table after migration
- [ ] No FK constraints on pivot table
- [ ] Unique constraint: duplicate create → upsert, no error
- [ ] Soft-delete parent → pivot records soft-deleted
- [ ] Hard-delete parent → pivot records hard-deleted
- [ ] Bidirectional: traverse from both sides of the link
- [ ] `data` metadata persisted and returned in query
- [ ] `dismiss()` removes only specified link
- [ ] `db:migrate` idempotent (no duplicate tables)

---

## Dependencies & Interfaces

| Dependency       | Interface Used                          | Direction |
|------------------|-----------------------------------------|-----------|
| Module A         | `linkable.entityA` descriptor           | Inbound   |
| Module B         | `linkable.entityB` descriptor           | Inbound   |
| Event Bus        | Entity deletion event subscription      | Inbound   |
| Query helper     | ModuleJoinerConfig registration         | Outbound  |
| DB layer         | DDL generation + CRUD on pivot tables   | Outbound  |

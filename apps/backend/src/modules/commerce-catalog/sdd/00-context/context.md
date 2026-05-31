# commerce-catalog — System Design Document

## Context

The `commerce-catalog` module extends Medusa's product and pricing modules
with custom catalog features: extended product metadata, collection rules,
and catalog-level pricing logic.

## Responsibilities

- Extend product data models with custom attributes
- Manage product collections with custom rules
- Provide catalog-level pricing and availability logic
- Expose catalog query APIs for storefront and admin

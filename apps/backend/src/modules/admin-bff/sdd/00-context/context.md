# admin-bff — System Design Document

## Context

The `admin-bff` module is a Backend-for-Frontend (BFF) layer that provides
custom aggregation, transformation, and orchestration logic for the Medusa
Admin Dashboard. It prevents the admin UI from having to fan-out to multiple
Medusa API routes and assembles composite data shapes.

## Responsibilities

- Aggregate data from multiple Medusa modules for admin views
- Transform Medusa data models into admin-optimized DTOs
- Proxy and enrich admin-specific API routes
- Keep admin UI decoupled from raw Medusa module APIs

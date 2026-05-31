# order-fulfillment — System Design Document

## Context

The `order-fulfillment` module manages the order lifecycle from placement
to delivery. It orchestrates Medusa's order, inventory, and fulfillment
modules with custom business rules for picking, packing, and shipping.

## Responsibilities

- Manage order state transitions (placed → fulfilled → delivered)
- Reserve and release inventory across fulfillment steps
- Coordinate shipping provider integrations
- Handle returns and refunds workflow

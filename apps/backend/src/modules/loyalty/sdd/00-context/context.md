# loyalty — System Design Document

## Context

The `loyalty` module implements the loyalty points and rewards system.
It integrates with Medusa's order and customer modules to track earned
points and allow redemption during checkout.

## Responsibilities

- Track loyalty points per customer per order
- Apply redemption rules during checkout
- Expire unused points based on policy
- Provide loyalty balance and history APIs

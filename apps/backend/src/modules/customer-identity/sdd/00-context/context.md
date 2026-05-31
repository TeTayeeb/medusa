# customer-identity — System Design Document

## Context

The `customer-identity` module manages customer authentication, profile
lifecycle, and identity verification. It wraps Medusa's customer and auth
modules with domain-specific identity rules.

## Responsibilities

- Customer registration and profile management
- Authentication flows (email/password, social)
- Identity verification and 2FA orchestration
- Session and token lifecycle management

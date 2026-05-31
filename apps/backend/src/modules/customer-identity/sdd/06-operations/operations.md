# customer-identity — Operations

## Running

```bash
yarn docker:up
```

## Feature Flag

```bash
FEATURE_CUSTOMER_IDENTITY_V2=true
```

## Upgrade Checklist

1. `yarn verify:upgrade-safety`
2. Check `@medusajs/auth` changelog for provider changes
3. Rotate `JWT_SECRET` and `COOKIE_SECRET` in `.env` before deploying

## Troubleshooting

- **401 on authenticated routes**: check `AUTH_CORS` includes your storefront origin
- **Session not persisting**: verify `COOKIE_SECRET` is consistent across all API instances

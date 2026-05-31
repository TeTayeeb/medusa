# @medusajs/notification-sendgrid

SendGrid email notification provider for Medusa v2. Implements `INotificationProvider` via `AbstractNotificationProviderService` and registers under `Modules.NOTIFICATION` with identifier `notification-sendgrid`.

Sends transactional emails via the [SendGrid Web API v3](https://docs.sendgrid.com/api-reference/mail-send) using **dynamic templates** or inline HTML content.

## Installation

```bash
npm install @medusajs/notification-sendgrid
```

## Configuration

```ts
import { Modules } from "@medusajs/framework/utils"

module.exports = defineConfig({
  modules: [
    {
      resolve: "@medusajs/medusa/notification",
      options: {
        providers: [
          {
            resolve: "@medusajs/notification-sendgrid",
            id: "sendgrid",
            options: {
              channels: ["email"],
              api_key: process.env.SENDGRID_API_KEY,
              from: process.env.SENDGRID_FROM,
              // e.g. "Acme Store <orders@acme.com>"
            },
          },
        ],
      },
    },
  ],
})
```

### Options reference

| Option | Type | Required | Description |
|---|---|---|---|
| `api_key` | `string` | ✅ | SendGrid API key (starts with `SG.`) |
| `from` | `string` | ✅ | Default sender address or `"Name <email>"` format |
| `channels` | `string[]` | — | Channels to handle (typically `["email"]`) |

## `send` method

The `send` method accepts a `NotificationDTO` object. Behaviour depends on whether `content` or `template` is specified:

### Dynamic template (recommended)

```ts
await notificationModule.send({
  to: "customer@example.com",
  template: "d-abc123templateid",    // SendGrid dynamic template ID
  data: {                             // Passed as dynamicTemplateData
    order_id: "ord_123",
    customer_name: "Jane",
  },
  channel: "email",
})
```

### Inline HTML content

```ts
await notificationModule.send({
  to: "customer@example.com",
  content: {
    subject: "Your order is confirmed",
    html: "<h1>Thank you!</h1>",
  },
  channel: "email",
})
```

> You cannot mix `content` and `template` in the same notification — SendGrid does not support it. The provider uses `content` if present; otherwise falls back to `template`.

## Attachments

```ts
await notificationModule.send({
  to: "customer@example.com",
  template: "d-xxx",
  attachments: [
    {
      content: "<base64-encoded-pdf>",
      filename: "invoice.pdf",
      content_type: "application/pdf",
      disposition: "attachment",   // or "inline"
      id: "invoice-1",             // optional, for inline
    },
  ],
  channel: "email",
})
```

## Personalizations (advanced)

Use `provider_data.personalizations` for multi-recipient personalised sends:

```ts
{
  provider_data: {
    personalizations: [
      { to: [{ email: "a@example.com" }], dynamic_template_data: { name: "Alice" } },
      { to: [{ email: "b@example.com" }], dynamic_template_data: { name: "Bob" } },
    ]
  }
}
```

When `personalizations` is provided, the `to` field at the top level is ignored.

## Subscriber pattern

Register a subscriber to send notifications on order events:

```ts
// src/subscribers/order-placed.ts
export default async function orderPlacedHandler({ event: { data }, container }) {
  const notification = container.resolve(Modules.NOTIFICATION)
  await notification.createNotifications({
    to: data.email,
    template: "d-order-confirmation-template-id",
    data: { order_id: data.id },
    channel: "email",
    provider_id: "sendgrid",
  })
}
export const config = { event: "order.placed" }
```

## Environment variables

```dotenv
SENDGRID_API_KEY=SG.xxx...
SENDGRID_FROM=orders@acme.com
```

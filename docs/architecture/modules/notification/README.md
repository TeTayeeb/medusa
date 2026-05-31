# Notification Module

The Notification module (`@medusajs/notification`) provides a provider-agnostic notification dispatch system for Medusa. It handles outbound communications — email, SMS, push notifications — through a pluggable provider abstraction. Notifications are triggered by domain events (order placed, invite created, etc.) and dispatched to registered provider plugins such as SendGrid or custom implementations.

## Purpose

Rather than coupling business workflows directly to notification delivery mechanisms, Medusa routes all outbound communication through the Notification module. Workflows emit domain hooks; subscribers listen for those hooks and call the Notification module's `createNotifications()` method with a template and data payload. The module delegates actual sending to the configured provider.

## Key Features

- **Provider Abstraction** — Swap or add notification providers (email, SMS, push) without touching business logic. Each provider declares which channels it handles.
- **Event-Driven Architecture** — Notifications are typically triggered by event subscribers that react to domain events (e.g., `order.placed`, `invite.created`).
- **Template-Based** — Each notification references a `template` identifier and passes a `data` payload; the provider resolves the template and renders it.
- **Delivery Tracking** — Notifications record `status` and `external_id` returned by the provider for delivery tracking and debugging.
- **Retry Support** — Failed notifications can be retried; status tracks attempts.
- **Built-in Providers** — Ships with a `local` provider (logs to console for development) and a `sendgrid` provider for production email.
- **Multi-Channel** — A single module handles all channels; providers declare their supported channels.

## Entities

| Entity                 | Key Fields                                                                                                               |
|------------------------|--------------------------------------------------------------------------------------------------------------------------|
| `Notification`         | `id`, `to`, `channel`, `template`, `data`, `trigger_type`, `resource_id`, `resource_type`, `receiver_id`, `provider_id`, `status`, `external_id` |
| `NotificationProvider` | `id`, `handle`, `name`, `is_enabled`, `channels`                                                                        |

## Admin API

| Method | Endpoint                   | Description                             |
|--------|----------------------------|-----------------------------------------|
| GET    | `/admin/notifications`     | List notifications with filters         |
| POST   | `/admin/notifications/:id/resend` | Resend a failed notification      |

## Module Identifier

```ts
import { Modules } from "@medusajs/framework/utils"
// Modules.NOTIFICATION
```

## Service Usage

```ts
const notificationService = container.resolve(Modules.NOTIFICATION)

// Send a notification
await notificationService.createNotifications({
  to: "customer@example.com",
  channel: "email",
  template: "order-confirmation",
  data: {
    order_id: "ord_01HXXX",
    customer_name: "Jane Doe",
    total: 9999,
  },
  trigger_type: "order.placed",
  resource_id: "ord_01HXXX",
  resource_type: "order",
})

// List notifications for a resource
const notifications = await notificationService.listNotifications({
  resource_id: "ord_01HXXX",
  resource_type: "order",
})
```

## Built-in Providers

| Provider ID        | Channel  | Description                                |
|--------------------|----------|--------------------------------------------|
| `local`            | email    | Logs notifications to console (dev only)   |
| `sendgrid`         | email    | Sends transactional email via SendGrid API |

## Notification Flow

```
Domain Event (order.placed)
  → Event Subscriber
  → notificationService.createNotifications({ template, data, to })
  → NotificationProvider.send()
  → External Service (SendGrid, Twilio, etc.)
  → Update Notification.status + external_id
```

## Related Modules

- **Order Module** — Emits events that trigger order confirmation and status update notifications.
- **User Module** — Invite flow triggers invitation email notifications.
- **Auth Module** — Password reset flows trigger notification dispatch.

## Version

Medusa v2.15.4

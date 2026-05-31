# Software Design Document — Notification Module

## 1. Overview

The Notification module provides a unified, provider-agnostic system for dispatching outbound communications (email, SMS, push notifications) from Medusa. It decouples notification delivery from business logic: workflows and event subscribers call the Notification module with a template identifier and data payload; the module delegates to the appropriate registered provider plugin. Notification records are persisted for delivery tracking and retry.

## 2. Goals and Non-Goals

**Goals:**
- Abstract notification delivery behind a provider interface supporting any channel (email, SMS, push, webhook).
- Persist all notification attempts for auditing, status tracking, and retry.
- Allow multiple providers to coexist (e.g., SendGrid for email, Twilio for SMS).
- Enable event-driven notification dispatch through domain event subscribers.
- Expose an Admin API for listing and resending notifications.

**Non-Goals:**
- Template rendering engine (delegated to providers).
- Scheduling / delayed notifications (a separate scheduling concern).
- In-app notification state (e.g., read/unread UI badges; a separate concern).
- Bulk marketing campaigns (transactional notifications only).

## 3. Data Model

### 3.1 Notification Entity

```ts
@Entity()
export class Notification extends BaseEntity {
  @PrimaryKey()
  id: string  // ULID

  @Property()
  to: string        // Recipient address: email, phone number, device token

  @Property()
  channel: string   // "email" | "sms" | "push" | custom

  @Property()
  template: string  // Template identifier, resolved by provider

  @Property({ nullable: true, type: "jsonb" })
  data?: Record<string, unknown>  // Template variables

  @Property({ nullable: true })
  trigger_type?: string   // e.g. "order.placed", "invite.created"

  @Property({ nullable: true })
  resource_id?: string    // ID of the triggering resource

  @Property({ nullable: true })
  resource_type?: string  // e.g. "order", "invite"

  @Property({ nullable: true })
  receiver_id?: string    // User or customer ID of recipient

  @Property({ nullable: true })
  provider_id?: string    // Which provider handled dispatch

  @Property({ default: "pending" })
  status: string   // "pending" | "sent" | "failed"

  @Property({ nullable: true })
  external_id?: string  // Provider's external reference ID

  @Property({ onCreate: () => new Date() })
  created_at: Date

  @Property({ onUpdate: () => new Date() })
  updated_at: Date

  @Property({ nullable: true })
  deleted_at?: Date
}
```

### 3.2 NotificationProvider Entity

```ts
@Entity()
export class NotificationProvider {
  @PrimaryKey()
  id: string       // e.g. "sendgrid", "local"

  @Property()
  handle: string   // Programmatic handle

  @Property()
  name: string     // Display name

  @Property({ type: "array" })
  channels: string[]  // ["email"], ["sms"], ["email", "push"]

  @Property({ default: true })
  is_enabled: boolean

  @Property({ onCreate: () => new Date() })
  created_at: Date

  @Property({ onUpdate: () => new Date() })
  updated_at: Date

  @Property({ nullable: true })
  deleted_at?: Date
}
```

### 3.3 Database Tables

| Table                   | Primary Key | Notable Indexes                         |
|-------------------------|-------------|-----------------------------------------|
| `notification`          | `id` (ULID) | `resource_id`, `receiver_id`, `status`  |
| `notification_provider` | `id` (string handle) | `is_enabled`, `channels`        |

## 4. Service Interface

```ts
interface INotificationModuleService {
  createNotifications(
    data: CreateNotificationDTO | CreateNotificationDTO[],
    sharedContext?: Context
  ): Promise<NotificationDTO | NotificationDTO[]>

  updateNotifications(
    data: UpdateNotificationDTO[],
    sharedContext?: Context
  ): Promise<NotificationDTO[]>

  listNotifications(
    filters?: FilterableNotificationProps,
    config?: FindConfig<NotificationDTO>,
    sharedContext?: Context
  ): Promise<NotificationDTO[]>

  retrieveNotification(
    id: string,
    config?: FindConfig<NotificationDTO>,
    sharedContext?: Context
  ): Promise<NotificationDTO>
}
```

## 5. Provider Interface

```ts
interface INotificationProvider {
  identifier: string
  channels: string[]

  send(
    notification: NotificationDTO
  ): Promise<{ id: string }>  // Returns provider's external ID
}
```

Provider plugins implement this interface and are registered in `medusa-config.ts` under the Notification module's `providers` array.

## 6. Dispatch Flow

```
createNotifications(dto)
      │
      ├─ Persist Notification record (status: "pending")
      ├─ Resolve provider: match dto.channel → registered INotificationProvider
      ├─ Call provider.send(notification)
      │     ├─ Success: update status = "sent", external_id = result.id
      │     └─ Failure: update status = "failed", log error
      └─ Return NotificationDTO
```

## 7. Event-Driven Integration Pattern

Notifications are typically triggered by workflow hooks rather than direct service calls:

```ts
// In a workflow subscriber
export default async function orderPlacedHandler({ event, container }) {
  const notificationService = container.resolve(Modules.NOTIFICATION)
  await notificationService.createNotifications({
    to: event.data.customer_email,
    channel: "email",
    template: "order-confirmation",
    data: { order_id: event.data.id },
    trigger_type: "order.placed",
    resource_id: event.data.id,
    resource_type: "order",
  })
}
```

## 8. Built-in Providers

| ID          | Package                            | Channels | Use Case      |
|-------------|------------------------------------|----------|---------------|
| `local`     | `@medusajs/notification-local`     | email    | Development   |
| `sendgrid`  | `@medusajs/notification-sendgrid`  | email    | Production    |

## 9. Retry and Status Tracking

Failed notifications can be resent via `POST /admin/notifications/:id/resend`. The resend operation:
1. Retrieves the existing Notification record.
2. Resolves the provider again (allows recovery if provider was temporarily unavailable).
3. Calls `provider.send()` and updates `status` and `external_id`.
4. Creates a new Notification record if a fresh attempt record is desired (configurable).

## 10. Error Handling

| Condition                              | Error Type                        |
|----------------------------------------|-----------------------------------|
| No provider registered for channel    | `MedusaError.Types.NOT_FOUND`     |
| Notification record not found          | `MedusaError.Types.NOT_FOUND`     |
| Provider send failure                  | Status set to "failed"; error logged |
| Invalid notification data              | `MedusaError.Types.INVALID_DATA`  |

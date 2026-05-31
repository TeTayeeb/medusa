# Specification — Notification Module (SpecKit)

## 1. Module Identity

| Attribute       | Value                                              |
|-----------------|----------------------------------------------------|
| Module ID       | `Modules.NOTIFICATION`                             |
| Package         | `@medusajs/notification`                           |
| Medusa Version  | 2.15.4                                             |
| Type            | Communication / Provider Abstraction               |
| Database tables | `notification`, `notification_provider`            |
| API surface     | Admin only                                         |

## 2. Functional Requirements

### FR-NOT-01: Send Notification
- **Given** a valid `to` address, `channel`, `template`, and `data` payload
- **When** `createNotifications(dto)` is called
- **Then** a Notification record is persisted (status: "pending"), the appropriate provider is resolved, `provider.send()` is called, and status is updated to "sent" or "failed"

### FR-NOT-02: Provider Resolution by Channel
- **Given** a notification with `channel: "email"`
- **When** the module resolves a provider
- **Then** the first registered, enabled `INotificationProvider` that declares `"email"` in its `channels` array is selected

### FR-NOT-03: Persist Before Dispatch
- **Given** any `createNotifications()` call
- **When** the provider call fails
- **Then** the Notification record still exists in the database with `status: "failed"` for audit and retry

### FR-NOT-04: Resend Notification
- **Given** an existing Notification record (any status)
- **When** `POST /admin/notifications/:id/resend` is called
- **Then** the provider is resolved again and `provider.send()` is called; status and `external_id` are updated

### FR-NOT-05: List Notifications
- **Given** an authenticated admin request
- **When** `GET /admin/notifications` is called with optional filters
- **Then** return a paginated list of Notification records

### FR-NOT-06: Filter by Resource
- **Given** a `resource_type` and `resource_id` filter
- **When** `listNotifications()` is called
- **Then** return only notifications associated with that resource (e.g., all notifications for order `ord_01HXXX`)

### FR-NOT-07: Provider Registration
- **Given** a module configuration with a `providers` array
- **When** Medusa boots
- **Then** each provider is instantiated, its `channels` are registered, and the provider is available for resolution

### FR-NOT-08: Multi-Provider Coexistence
- **Given** a SendGrid provider (email) and a Twilio provider (sms) both registered
- **When** an email notification is dispatched
- **Then** SendGrid is used; when an SMS notification is dispatched, Twilio is used

### FR-NOT-09: External ID Tracking
- **Given** a provider that returns an external message ID
- **When** `provider.send()` completes successfully
- **Then** `Notification.external_id` is set to the provider's returned ID for correlation

## 3. Non-Functional Requirements

| ID          | Requirement                         | Target                                              |
|-------------|-------------------------------------|-----------------------------------------------------|
| NFR-NOT-01  | Dispatch latency (local provider)   | < 10ms                                              |
| NFR-NOT-02  | Dispatch latency (SendGrid)         | < 2000ms (external HTTP call)                      |
| NFR-NOT-03  | Persistence guarantee               | Notification record exists regardless of send outcome |
| NFR-NOT-04  | Provider failure isolation          | One provider failure must not crash the workflow   |
| NFR-NOT-05  | Notification table growth           | Index on `resource_id`, `status`, `created_at` for query performance |

## 4. Interface Specification

### GET `/admin/notifications`

| Attribute     | Value                                                                              |
|---------------|------------------------------------------------------------------------------------|
| Auth required | Yes (Admin JWT)                                                                    |
| Query params  | `limit`, `offset`, `resource_type`, `resource_id`, `receiver_id`, `channel`, `status` |
| Response 200  | `{ notifications: NotificationDTO[], count, limit, offset }`                      |

### POST `/admin/notifications/:id/resend`

| Attribute     | Value                                            |
|---------------|--------------------------------------------------|
| Auth required | Yes (Admin JWT)                                  |
| Path param    | `id` — Notification ULID                         |
| Response 200  | `{ notification: NotificationDTO }`              |
| Response 404  | Notification not found                           |
| Response 400  | No provider registered for the notification's channel |

## 5. Data Contracts

### NotificationDTO

```ts
type NotificationDTO = {
  id: string
  to: string
  channel: string             // "email" | "sms" | "push"
  template: string
  data: Record<string, unknown> | null
  trigger_type: string | null
  resource_id: string | null
  resource_type: string | null
  receiver_id: string | null
  provider_id: string | null
  status: "pending" | "sent" | "failed"
  external_id: string | null
  created_at: Date
  updated_at: Date
  deleted_at: Date | null
}
```

### CreateNotificationDTO

```ts
type CreateNotificationDTO = {
  to: string
  channel: string
  template: string
  data?: Record<string, unknown>
  trigger_type?: string
  resource_id?: string
  resource_type?: string
  receiver_id?: string
}
```

### INotificationProvider (plugin interface)

```ts
interface INotificationProvider {
  identifier: string
  channels: string[]
  send(notification: NotificationDTO): Promise<{ id: string }>
}
```

## 6. Provider Configuration Example

```ts
// medusa-config.ts
{
  resolve: "@medusajs/notification",
  options: {
    providers: [
      {
        resolve: "@medusajs/notification-sendgrid",
        id: "sendgrid",
        options: {
          api_key: process.env.SENDGRID_API_KEY,
          from: "noreply@store.com",
        },
      },
    ],
  },
}
```

## 7. Edge Cases

| Case                                          | Expected Behaviour                                               |
|-----------------------------------------------|------------------------------------------------------------------|
| No provider registered for `channel: "sms"`  | `createNotifications()` throws `NOT_FOUND`; record not created  |
| Provider returns no external ID               | `external_id` remains null; not an error                        |
| Resend a "sent" notification                  | Allowed; creates new send attempt, updates status               |
| `data` payload contains circular references  | JSON serialization fails; `createNotifications()` throws 400    |
| Provider rate-limited (429 from SendGrid)     | Status set to "failed"; admin can resend manually               |
| Two providers both handle "email"             | First registered provider is used; order matters                |

## 8. Module Boundaries

| In Scope                                   | Out of Scope                                             |
|--------------------------------------------|----------------------------------------------------------|
| Notification dispatch and persistence      | Template rendering (provider responsibility)             |
| Provider abstraction (email, SMS, push)    | Scheduled/delayed notifications                          |
| Delivery status tracking                   | In-app notification UI state (read/unread)               |
| Retry via Admin API resend                 | Bulk marketing campaigns                                 |

## 9. Acceptance Criteria Summary

- [ ] `createNotifications({ to, channel: "email", template, data })` persists record and calls SendGrid
- [ ] Failed send sets `status: "failed"`; record is accessible in Admin list
- [ ] `POST /admin/notifications/:id/resend` retries and updates `status` + `external_id`
- [ ] `GET /admin/notifications?resource_id=ord_01H&resource_type=order` returns all notifications for that order
- [ ] No provider for channel → `NOT_FOUND` error; no orphan Notification record created
- [ ] Local provider logs to console in development mode; no external HTTP call
- [ ] `external_id` is set from provider's return value after successful send

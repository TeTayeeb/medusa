# Architecture Documentation — Notification Module (arc42)

## 1. Introduction and Goals

The Notification module provides a unified, extensible dispatch layer for all outbound communications in Medusa. It decouples notification delivery from business workflows, allowing teams to add or swap notification providers without touching order, user, or auth logic.

**Quality Goals:**

| Priority | Quality Goal   | Description                                                                     |
|----------|----------------|---------------------------------------------------------------------------------|
| 1        | Extensibility  | New channels (SMS, push) and providers added without core changes               |
| 2        | Observability  | All notification attempts persisted with status and external ID                 |
| 3        | Reliability    | Failed notifications are retryable; status transitions are tracked              |
| 4        | Decoupling     | Business workflows emit events; notification logic lives in subscribers         |

## 2. Constraints

- Notification dispatch is asynchronous with respect to the triggering workflow (non-blocking in practice via event subscribers).
- Provider plugins must implement the `INotificationProvider` interface.
- At least one provider must be registered per channel in use.
- The Notification module does not render templates; template rendering is delegated to providers.

## 3. Context and Scope

```
Internal:
  [Order Module] ──event: order.placed──► [Event Bus] ──► [Notification Subscriber]
  [User Module]  ──hook: inviteCreated──► [Event Bus] ──► [Notification Subscriber]
  [Auth Module]  ──hook: passwordReset──► [Event Bus] ──► [Notification Subscriber]

  [Notification Subscriber] ──createNotifications()──► [Notification Module]
  [Notification Module]     ──send()──────────────────► [Provider Plugin]

  [Provider Plugin] ──HTTP call──► [SendGrid / Twilio / custom service]

Admin:
  [Admin Browser] ──GET /admin/notifications, POST .../resend──► [Notification Module]
```

## 4. Solution Strategy

| Challenge                               | Strategy                                                              |
|-----------------------------------------|-----------------------------------------------------------------------|
| Multiple channels (email/SMS/push)      | Provider declares supported channels; module routes by `channel`     |
| Avoiding coupling to workflows          | Event-driven; subscribers are separate from module internals         |
| Failed delivery recovery                | Persist status; Admin API resend endpoint calls provider again       |
| Development/production provider parity  | `local` provider in dev (console log); `sendgrid` in production      |

## 5. Building Block View

```
Notification Module
├── HTTP Layer
│   └── Admin Routes: /admin/notifications (GET, /:id/resend)
│
├── Service Layer
│   └── NotificationModuleService
│       ├── createNotifications()   ← persist + dispatch
│       ├── updateNotifications()
│       ├── listNotifications()
│       └── retrieveNotification()
│
├── Provider Registry
│   └── NotificationProviderService
│       ├── register(provider: INotificationProvider)
│       ├── resolve(channel: string) → INotificationProvider
│       └── send(notification) → { id: string }
│
├── Domain Model
│   ├── Notification
│   └── NotificationProvider
│
└── Built-in Providers
    ├── LocalNotificationProvider  (dev: console.log)
    └── SendGridNotificationProvider (prod: HTTP to SendGrid)
```

## 6. Runtime View

**Scenario A: Order placed → customer email**

```
OrderModule emits event: "order.placed" { data: { id, customer_email, ... } }
  → Event bus delivers to OrderPlacedSubscriber
  → Subscriber calls notificationService.createNotifications({
      to: customer_email,
      channel: "email",
      template: "order-confirmation",
      data: { order_id, items, total },
      trigger_type: "order.placed",
      resource_id: order.id,
      resource_type: "order",
    })
  → Notification record persisted (status: "pending")
  → ProviderRegistry.resolve("email") → SendGridProvider
  → SendGridProvider.send(notification)
      → HTTP POST to SendGrid API with template_id + dynamic_template_data
      → Returns { id: "sendgrid-msg-id-xxx" }
  → Notification.status = "sent", Notification.external_id = "sendgrid-msg-id-xxx"
```

**Scenario B: Admin resends a failed notification**

```
POST /admin/notifications/:id/resend
  → Retrieve Notification record
  → ProviderRegistry.resolve(notification.channel) → provider
  → provider.send(notification)
  → Update status + external_id
  → Response: { notification: NotificationDTO }
```

## 7. Deployment View

Single Medusa process. Notification records stored in PostgreSQL (`notification`, `notification_provider` tables). Provider plugins make outbound HTTP calls to external services. No message queue required — event bus handles async dispatch.

## 8. Cross-Cutting Concerns

| Concern          | Approach                                                                  |
|------------------|---------------------------------------------------------------------------|
| Authentication   | Admin routes: JWT required. No Store API.                                 |
| Error isolation  | Provider failure caught, status set to "failed"; does not crash workflow  |
| Observability    | `external_id` enables correlation with provider dashboards               |
| Sensitive data   | `data` JSON payload should not contain PCI-scope data; provider-side only |
| Retry backoff    | Implemented at subscriber level or via resend Admin API; not automatic    |

## 9. Design Decisions

| ID  | Decision                                  | Rationale                                                              |
|-----|-------------------------------------------|------------------------------------------------------------------------|
| D1  | Provider-side template rendering          | Module stays generic; providers handle platform-specific templates     |
| D2  | Persist before dispatch                   | Notification record exists even if provider call fails (auditable)     |
| D3  | Channel-based provider resolution         | Single provider can handle multiple channels; routing is explicit      |
| D4  | Event-driven (not workflow-embedded)      | Decouples notification logic from business workflows; easier to extend |

## 10. Risks and Technical Debt

| Risk                                         | Mitigation                                                  |
|----------------------------------------------|-------------------------------------------------------------|
| Provider rate limiting (e.g. SendGrid)       | Retry with backoff; monitor `status = "failed"` in admin   |
| Template data containing sensitive info      | Application-level responsibility; audit template payloads  |
| No built-in SMS provider                     | Custom INotificationProvider implementation required        |
| Large `data` payloads in JSONB               | Recommend keeping payload minimal; link to resource ID      |

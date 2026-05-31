# Software Design Document — @medusajs/notification-sendgrid

## 1. Purpose

Deliver transactional emails through the SendGrid API on behalf of Medusa's Notification Module. Supports dynamic templates (data-driven email templates designed in SendGrid's dashboard), inline HTML content, file attachments, and advanced personalizations for multi-recipient sends.

## 2. Architecture

```
Modules.NOTIFICATION
  └── ModuleProvider (notification-sendgrid)
        └── SendgridNotificationService (AbstractNotificationProviderService)
              └── send(notification: NotificationDTO) → {}
```

The provider exposes a single `send` method. The Notification Module invokes it after matching a notification to the `sendgrid` provider based on channel and `provider_id`.

## 3. `send` method internals

```
send(notification):
  1. Validate notification is present.
  2. Map attachments → SendGrid attachment format.
  3. Determine sender: notification.from ?? this.config_.from
  4. Build mailContent:
     a. "content" key present and truthy → { subject, html } (inline mode)
     b. otherwise → { templateId: notification.template } (dynamic template mode)
  5. Check provider_data.personalizations:
     a. If present → use personalizations array (ignores top-level `to`)
     b. If absent → { to: notification.to }
  6. Compose message: { from, dynamicTemplateData, attachments, to/personalizations, ...mailContent }
  7. @sendgrid/mail.send(message)
  8. On success: return {}
  9. On error: throw MedusaError(UNEXPECTED_STATE, "Failed to send email: <code> - <message>")
```

## 4. Content mode selection

The provider cannot mix `html` and `templateId` in the same SendGrid request (API restriction). The logic is:

```ts
if ("content" in notification && !!notification.content) {
  mailContent = { subject: notification.content.subject, html: notification.content.html }
} else {
  mailContent = { templateId: notification.template }
}
```

`"content" in notification` (property existence check) ensures the field is genuinely present, not just falsy.

## 5. Dynamic template data

When using `template` mode, `notification.data` is passed as `dynamicTemplateData`. In the SendGrid template, variables are accessed via Handlebars-style syntax: `{{order_id}}`, `{{customer_name}}`, etc.

## 6. Attachments mapping

```ts
attachments = notification.attachments?.map(a => ({
  content:      a.content,           // Base64-encoded string
  filename:     a.filename,
  content_type: a.content_type,      // MIME type
  disposition:  a.disposition ?? "attachment",
  id:           a.id ?? undefined,   // Inline attachments require `id`
}))
```

## 7. Personalizations

SendGrid [personalizations](https://docs.sendgrid.com/for-developers/sending-email/personalizations) allow per-recipient overrides (subject, dynamic data, CCs, etc.):

```ts
if (personalizations?.length) {
  message = { from, dynamicTemplateData, personalizations, ...mailContent }
} else {
  message = { from, dynamicTemplateData, to: notification.to, ...mailContent }
}
```

When `personalizations` is provided, the `to` field at the message level is omitted — SendGrid routes delivery based on each personalization entry.

## 8. Per-notification sender override

Each notification can override the default `from` address:
```ts
const from = notification.from?.trim() || this.config_.from
```

This allows sending from different addresses per notification type (e.g. `support@acme.com` for support tickets vs. `orders@acme.com` for order confirmations).

## 9. Error handling

```ts
try {
  await mail.send(message)
  return {}
} catch (error) {
  const errorCode = error.code
  const responseError = error.response?.body?.errors?.[0]
  throw new MedusaError(
    MedusaError.Types.UNEXPECTED_STATE,
    `Failed to send email: ${errorCode} - ${responseError?.message ?? "unknown error"}`
  )
}
```

The `@sendgrid/mail` SDK returns a response body with an `errors` array on failure. The provider extracts the first error message for the thrown `MedusaError`. No retry logic is implemented — the Notification Module is responsible for retry strategies.

## 10. SendGrid API key initialisation

```ts
constructor({ logger }, options) {
  super()
  this.config_ = { apiKey: options.api_key, from: options.from }
  mail.setApiKey(this.config_.apiKey)
}
```

`setApiKey` is called once at construction time; the key is stored in the `@sendgrid/mail` singleton.

## 11. Return value

`send` returns `{}` on success. The Notification Module uses this to mark the notification as delivered. No meaningful data is returned from the SendGrid API on success (the SDK discards it).

## 12. Dependencies

| Package | Purpose |
|---|---|
| `@sendgrid/mail` | Official SendGrid Node.js SDK |
| `@medusajs/framework` | `AbstractNotificationProviderService`, `MedusaError` |

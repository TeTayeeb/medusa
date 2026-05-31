# SpecKit — @medusajs/notification-sendgrid

---

## 1. Unit specs — `send` (dynamic template mode)

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U1 | Happy path — template | `{ to: "a@b.com", template: "d-xxx", data: { name: "Jane" } }` | `mail.send` called with `templateId: "d-xxx"`, `dynamicTemplateData: { name: "Jane" }`, `to: "a@b.com"` |
| U2 | Returns empty object on success | Valid notification | Returns `{}` |
| U3 | Default sender used | `notification.from` absent | `from` set to `config_.from` |
| U4 | Per-notification sender override | `notification.from: "support@acme.com"` | `from: "support@acme.com"` |
| U5 | Sender whitespace trimmed | `notification.from: "  orders@acme.com  "` | `from: "orders@acme.com"` |

---

## 2. Unit specs — `send` (inline HTML mode)

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U6 | Happy path — inline content | `{ content: { subject: "Hello", html: "<b>Hi</b>" } }` | `mail.send` called with `subject`, `html`; no `templateId` |
| U7 | Content takes precedence over template | Both `content` and `template` fields present | `content` mode used (no `templateId` in message) |
| U8 | Falsy `content` falls back to template | `{ content: null, template: "d-xxx" }` | Template mode used |

---

## 3. Unit specs — `send` (attachments)

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U9 | Attachment mapped correctly | `attachments: [{ content: "base64", filename: "f.pdf", content_type: "application/pdf" }]` | SendGrid message includes mapped attachment |
| U10 | Default disposition is "attachment" | `attachment.disposition` absent | `disposition: "attachment"` in message |
| U11 | Custom disposition | `disposition: "inline"` | `disposition: "inline"` in message |
| U12 | No attachments | `notification.attachments` absent | `attachments: undefined` in message |
| U13 | Empty attachments array | `attachments: []` | `attachments: undefined` in message (falsy) |

---

## 4. Unit specs — `send` (personalizations)

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U14 | Personalizations override `to` | `provider_data.personalizations: [...]` | Message has `personalizations` set, `to` omitted |
| U15 | No personalizations falls back to `to` | `provider_data` absent | Message has `to: notification.to` |

---

## 5. Unit specs — error handling

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U16 | SendGrid API error | SDK throws `{ code: 403, response: { body: { errors: [{ message: "Forbidden" }] } } }` | Throws `MedusaError(UNEXPECTED_STATE, "Failed to send email: 403 - Forbidden")` |
| U17 | Unknown error shape | SDK throws error without `response.body` | Throws `MedusaError(UNEXPECTED_STATE, "Failed to send email: <code> - unknown error")` |
| U18 | No notification provided | `null` | Throws `MedusaError(INVALID_DATA, "No notification information provided")` |

---

## 6. Unit specs — initialisation

| # | Scenario | Expected outcome |
|---|---|---|
| U19 | API key set at construction | `mail.setApiKey` called with `options.api_key` |
| U20 | Identifier correct | `SendgridNotificationService.identifier === "notification-sendgrid"` |

---

## 7. Integration specs

| # | Scenario | Expected outcome |
|---|---|---|
| I1 | Order placed → email sent | Subscriber fires; `send` called; email delivered via SendGrid |
| I2 | Invalid API key | `send` throws `MedusaError`; notification marked failed |
| I3 | Template with attachments | PDF attached; email received with attachment |
| I4 | Multi-recipient via personalizations | Each recipient receives personalised email |

---

## 8. Acceptance criteria

- `send` always returns `{}` on success (no partial results).
- `content` and `template` modes are mutually exclusive and never mixed in a single request.
- All errors are surfaced as `MedusaError(UNEXPECTED_STATE)` with the SendGrid error code and message.
- Per-notification `from` override trims whitespace and falls back to config default.
- `identifier` is exactly `"notification-sendgrid"`.

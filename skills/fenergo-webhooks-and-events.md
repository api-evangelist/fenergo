---
name: Subscribe to Fenergo events by webhook or polling
description: Register an HMAC-signed webhook or drain the polling feed, and verify the x-fenx-signature on every notification.
api: openapi/fenergo-webhooks-v1-0-openapi.json
operations: [ListWebhooks, AddWebhook, GetWebhook, UpdateWebhook, DeleteWebhook, TestWebhook, ListEventNotificationTypes, GetNext, DiscardAll]
generated: '2026-09-09'
method: generated
source: openapi/ + https://docs.fenergox.com/developer-hub/event-notifications/event-registry
---

# Subscribe to Fenergo events by webhook or polling

Fenergo publishes **93 event types across 15 domains**. There are two delivery patterns and you pick one per
integration: **webhooks** (Fenergo pushes to you) or the **polling API** (you pull).

Scopes: `fenx.webhooks` to manage webhooks, `fenx.eventnotifications` for the polling feed.

## Discover what you can subscribe to

`ListEventNotificationTypes` — `GET /v1/webhooks/event-notification-types` on
`https://api.fenergox.com/webhooks`. The full catalogue is also captured in
`asyncapi/fenergo-event-notifications-webhooks.yml`.

## Option A — webhooks (push)

Base URL: `https://api.fenergox.com/webhooks`

1. `AddWebhook` — `POST /v1/webhooks`. Supply your endpoint, the event types you want, and a **secret**.
   Fenergo stores only a hash of the secret, never the plaintext.
2. `TestWebhook` — `GET /v1/webhooks/{webhookId}/test` to confirm reachability before you rely on it.
3. `ListWebhooks` / `GetWebhook` / `UpdateWebhook` / `DeleteWebhook` manage the registration.
   `UpdateWebhook` is how you rotate the secret.

### Verify every notification

Each delivery carries `x-fenx-signature: sha256=<UPPERCASE HEX>` where the digest is
`HMAC-SHA256(raw request body, shared secret)`.

Recompute it over the **raw body bytes** — from the opening `{` to the closing `}`, before any JSON
re-serialisation — and compare. **If it does not match, reject the message.** Use a constant-time comparison.

## Option B — polling (pull)

Base URL: `https://api.fenergox.com/eventnotifications`

1. `GetNext` — `GET /v2/{feedId}/next` returns the next batch.
2. Process it, then mark it done: `POST /v2/{feedId}/{batchId}/complete`. A batch that is never completed is
   redelivered.
3. `DiscardAll` — `PUT /v2/{feedId}/discardAll` empties the queue for a feed. This is destructive and has no
   undo; use it only to recover a feed you have deliberately abandoned.

## Notification shape

```json
{
  "id": "1a575f9b-...",
  "tenantId": "215d5a67-...",
  "eventType": "screening:searchcompleted",
  "relativeUrl": "screeningquery/api/batch/3a1e8b52-...",
  "correlationId": "11ac5835-...",
  "causationId": "a3fffa4b-...",
  "when": "2021-01-01T00:00:00+00:00"
}
```

- `payload` is empty for many event types — the notification tells you *what* changed, not the new state.
  Follow `relativeUrl` against the matching Query API to read it.
- `correlationId` ties the event back to the `x-correlation-id` you sent on the originating call.
- Non-ASCII characters are escaped to `\uXXXX`.

## Rules

- Events are **at-least-once**. Deduplicate on `id`; the same notification can arrive twice.
- Do not treat an event as the data. Read through `relativeUrl`.
- Event Ingress (pushing events *into* Fenergo) is a different API with a point-based complexity budget —
  see `rate-limits/fenergo-rate-limits.yml`.

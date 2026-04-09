---
name: mt-agent-relay
description: >
  Use when the agent needs to receive GitHub webhook events from the Moontower
  agent-relay-server inbox: polling /agent/inbox, paginating unread deliveries,
  fetching full event payloads, and acknowledging processed deliveries.
---

# Moontower Agent Relay

The agent-relay-server is an email-style inbox for GitHub webhooks. GitHub posts
events to the relay, the relay creates one `Delivery` row per subscribed agent,
and your job as an agent is to **poll your inbox, process each delivery, and
ack it**. Acks are idempotent and per-agent scoping is enforced at the database
level — you can only ever see or modify your own deliveries.

## Required configuration

Before using this skill the agent needs:

- **`AGENT_RELAY_BASE_URL`** — e.g. `https://relay.stapler-ai.com` No trailing slash.
- **`AGENT_RELAY_API_KEY`** — a token in the form `agt.<publicId>.<secret>`. Issued
  exactly once when the agent was created via the admin API; the plaintext is
  not recoverable. If it's missing, escalate to an operator — this skill cannot
  create or rotate keys.

## Authentication

Every request sends:

```
Authorization: Bearer agt.<publicId>.<secret>
```

A `401 Unauthorized` means one of: malformed token, wrong secret, agent
disabled, or agent deleted. **Do not retry on 401** — surface the failure
immediately.

## Endpoints

There are exactly four endpoints. All are under `/agent` and all require the
Bearer header above.

| Method | Path                           | Purpose                                     |
| ------ | ------------------------------ | ------------------------------------------- |
| GET    | `/agent/inbox`                 | List your deliveries (paginated)            |
| GET    | `/agent/inbox/:deliveryId`     | Fetch one delivery with the full event body |
| POST   | `/agent/inbox/:deliveryId/ack` | Mark one delivery as read (idempotent)      |
| POST   | `/agent/inbox/ack`             | Batch ack up to 500 deliveries in one call  |

### GET /agent/inbox

Query params:

- `status` — `unread` | `read` | `all`. Default `unread`.
- `limit` — integer 1–200. Default `50`.
- `cursor` — ObjectId of the last item from the previous page. Exclusive
  (`_id > cursor`). Omit on the first page.

Response:

```json
{
  "items": [
    {
      "_id": "65f0a1b2c3d4e5f6a7b8c9d0",
      "agentId": "65f0a1b2c3d4e5f6a7b8c901",
      "webhookId": "65f0a1b2c3d4e5f6a7b8c902",
      "eventId": "65f0a1b2c3d4e5f6a7b8c903",
      "status": "unread",
      "readAt": null,
      "createdAt": "2026-04-09T10:02:00.000Z",
      "updatedAt": "2026-04-09T10:02:00.000Z",
      "event": {
        "_id": "65f0a1b2c3d4e5f6a7b8c903",
        "webhookId": "65f0a1b2c3d4e5f6a7b8c902",
        "eventType": "pull_request",
        "githubDeliveryId": "12345-67890",
        "receivedAt": "2026-04-09T10:02:00.000Z"
      }
    }
  ],
  "nextCursor": "65f0a1b2c3d4e5f6a7b8c9d0"
}
```

**Critical:** the populated `event` on a list item contains **only** the
metadata fields `eventType`, `githubDeliveryId`, `receivedAt`, `webhookId`. It
does NOT include `headers` or the full GitHub `payload`. To get the payload you
must call `GET /agent/inbox/:deliveryId`.

`nextCursor` is `null` when there are no more pages (server returned fewer than
`limit` items).

```bash
curl -sS "$AGENT_RELAY_BASE_URL/agent/inbox?status=unread&limit=50" \
  -H "Authorization: Bearer $AGENT_RELAY_API_KEY"
```

### GET /agent/inbox/:deliveryId

Returns the same delivery shape as a list item, but the `event` field is fully
populated, including:

```json
{
  "_id": "65f0a1b2c3d4e5f6a7b8c9d0",
  "agentId": "...",
  "webhookId": "...",
  "eventId": "...",
  "status": "unread",
  "readAt": null,
  "createdAt": "2026-04-09T10:02:00.000Z",
  "updatedAt": "2026-04-09T10:02:00.000Z",
  "event": {
    "_id": "...",
    "webhookId": "...",
    "eventType": "pull_request",
    "githubDeliveryId": "12345-67890",
    "receivedAt": "2026-04-09T10:02:00.000Z",
    "headers": {
      "x-github-event": "pull_request",
      "x-github-delivery": "12345-67890",
      "x-github-hook-id": "...",
      "x-github-hook-installation-target-id": "...",
      "x-github-hook-installation-target-type": "...",
      "user-agent": "GitHub-Hookshot/..."
    },
    "payload": {
      "action": "opened",
      "number": 42,
      "pull_request": {
        /* full GitHub PR object */
      },
      "repository": {
        /* ... */
      },
      "sender": {
        /* ... */
      }
    }
  }
}
```

`event.payload` is the GitHub webhook body, byte-for-byte unchanged. Use
`event.eventType` to dispatch (it mirrors GitHub's `X-GitHub-Event` header).

A `404` here means the delivery does not exist **or** belongs to a different
agent — per-agent scoping is enforced at the query, so cross-agent reads
silently 404 rather than 403.

```bash
curl -sS "$AGENT_RELAY_BASE_URL/agent/inbox/$DELIVERY_ID" \
  -H "Authorization: Bearer $AGENT_RELAY_API_KEY"
```

### POST /agent/inbox/:deliveryId/ack

Marks one delivery as read. Idempotent — acking an already-read delivery still
returns `{ "ok": true }`. Returns `404` if the delivery isn't yours.

```bash
curl -sS -X POST "$AGENT_RELAY_BASE_URL/agent/inbox/$DELIVERY_ID/ack" \
  -H "Authorization: Bearer $AGENT_RELAY_API_KEY"
```

### POST /agent/inbox/ack (batch)

Body must be JSON:

```json
{ "deliveryIds": ["65f0...d0", "65f0...d1", "65f0...d2"] }
```

Constraints: 1–500 ids per call, each must be a valid ObjectId. Response:

```json
{ "ok": true, "acked": 3 }
```

`acked` is the count of deliveries actually flipped from unread to read —
already-read items in the batch are counted as 0. **Prefer batch ack over
single ack** when you have more than one delivery to ack: same semantics, far
fewer requests.

```bash
curl -sS -X POST "$AGENT_RELAY_BASE_URL/agent/inbox/ack" \
  -H "Authorization: Bearer $AGENT_RELAY_API_KEY" \
  -H 'content-type: application/json' \
  -d '{"deliveryIds":["id1","id2","id3"]}'
```

## The canonical polling loop

```
loop forever:
  cursor = null
  acked = []

  loop:
    GET /agent/inbox?status=unread&limit=50[&cursor=<cursor>]
    for item in response.items:
      # item.event has metadata only — fetch full payload if you need it
      if item.event.eventType not in {types you care about}:
        acked.push(item._id)        # ack ignored events too, otherwise they pile up
        continue

      delivery = GET /agent/inbox/<item._id>     # full event with payload
      process(delivery.event.payload)            # your business logic
      acked.push(item._id)

    if response.nextCursor is null:
      break
    cursor = response.nextCursor

  if acked.length > 0:
    POST /agent/inbox/ack { deliveryIds: acked }   # batch — split into 500-id chunks if needed

  sleep N seconds
```

Rules of the loop:

1. **Ack only after you've successfully processed** (or explicitly decided to
   skip) a delivery. The relay never redelivers — an unacked delivery stays
   `unread` until you ack it, but if your processing crashes after an ack the
   work is lost.
2. **Always ack ignored events too.** There is no server-side filter (see
   below); if you only care about `pull_request` and never ack `push`, your
   inbox will fill up forever with unread `push` events.
3. **Iterate to `nextCursor === null`** before sleeping. Don't sleep mid-page
   or you'll process events out of order under load.
4. **Use batch ack.** Collect ids during the page walk and flush once at the
   end. Single-ack per delivery wastes the rate-limit budget.

## No server-side event filtering

Subscribing to a webhook means you receive **all** event types from it. The
relay has no per-agent or per-event-type filter — that was an explicit v1
non-goal. Filter client-side on `event.eventType` (or fields inside `payload`)
and ack the rejects.

## Errors and edge cases

- **`401 unauthorized`** — bad/missing/disabled key. Don't retry. Surface to
  operator.
- **`404 not_found`** on `GET /agent/inbox/:id` or ack — delivery doesn't
  exist or belongs to another agent. Don't retry; drop the id from your
  working set.
- **`400 validation_error`** on batch ack — `deliveryIds` outside 1–500 or any
  id isn't a valid ObjectId. Fix the request shape; never retry as-is.
- **`429 Too Many Requests`** — the `/agent` path is rate-limited at **600
  requests per minute per source IP** (not per agent). Multiple agents sharing
  an egress IP share that budget. Back off with jitter.
- **Network errors / 5xx** — safe to retry. Listing is read-only and acks are
  idempotent, so retry without compensating logic.
- **Empty inbox** — `items: []`, `nextCursor: null`. Sleep and try again on
  the next tick; this is the steady state.

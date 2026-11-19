# Extremis REST API and webhooks

For your own code (a server, a CRM, an order system), Extremis has a REST API with API keys and signed webhooks. For AI tools such as Claude or ChatGPT, use the [MCP server](README.md) instead.

**Full reference, with an example request and answer for every endpoint: [extremis.online/developers](https://extremis.online/developers)**

```
https://extremis.online/api/v1
```

- **API keys** (`exm_sk_…`), created by an Owner or Admin in **Settings → Integrations → API & webhooks**.
- **Acts as a teammate.** A key works as the teammate who created it, inside that teammate's current role and chat access.
- **Signed webhooks** when a message arrives or goes out, or a conversation opens, closes or is assigned.
- **Included in every plan.** WhatsApp template messages are charged by Meta as usual.

## API keys

1. Open **Settings → Integrations → API & webhooks** and press **Create key**.
2. Give it a name and choose its access:
   - **Read only**: conversations, messages, contacts, templates, labels, teammates and numbers.
   - **Read and write**: also send replies and templates, start WhatsApp conversations, add notes, assign, change status, label and edit contacts.
3. Copy the key. It is shown **only once**; Extremis keeps only a hash of it.

Send it as a bearer token:

```bash
curl https://extremis.online/api/v1/me \
  -H "Authorization: Bearer $EXTREMIS_API_KEY"
```

- Messages a key sends show its teammate as the sender.
- If the teammate leaves the workspace, their keys stop working.
- A workspace can have up to 25 keys. Revoke a key in the same settings page.

> Keep keys on your server. Never put one in a website, a mobile app or anything people download.

## Endpoints

All paths are under `https://extremis.online/api/v1`. Bodies are JSON with `snake_case` fields; answers use `camelCase`.

| Method and path | Key | What it does |
|---|---|---|
| `GET /me` | any | The workspace, the key's teammate, their role and the key's access |
| `GET /conversations` | any | Conversations the teammate can see. Filters: `status`, `view`, `channel`, `unread_only`, `query`, `sort`, `limit`, `cursor` |
| `GET /conversations/{id}` | any | One conversation with its newest messages and notes (`message_limit`, `before`) |
| `POST /conversations` | write | Start a WhatsApp conversation with an approved template |
| `POST /conversations/{id}/messages` | write | Send a text reply: `{ "text": "..." }` |
| `POST /conversations/{id}/template` | write | Send an approved WhatsApp template |
| `POST /conversations/{id}/notes` | write | Add an internal note |
| `POST /conversations/{id}/assign` | write | `{ "assignee": "<userId>" }`, `"me"` or `null` |
| `POST /conversations/{id}/status` | write | `{ "status": "open" }`, `"pending"` or `"closed"` |
| `POST /conversations/{id}/labels` | write | Replace the labels: `{ "label_ids": [...] }` |
| `GET /contacts`, `GET /contacts/{id}` | any | Contacts the teammate can see |
| `PATCH /contacts/{id}` | write | Change `name`, `email` or `phone` |
| `GET /templates` | any | Approved templates for a `conversation_id` or a `whatsapp_number_id` |
| `GET /labels`, `GET /teammates`, `GET /numbers` | any | The ids the calls above need |

The normal inbox rules apply: on WhatsApp, free text only goes out within 24 hours of the customer's last message; after that, send a template.

**Send only once.** Add an `Idempotency-Key` header (1 to 255 printable characters, such as your order number) to `POST /conversations`, `.../messages` and `.../template`. Sending the same request again with the same key returns the first message and sends nothing new.

**Pages.** Lists return `nextCursor`; pass it back as `cursor`.

## Errors and limits

Errors look like this:

```json
{ "error": { "code": "window_closed", "message": "..." } }
```

| Status | Meaning |
|---|---|
| 400 | The body is not valid JSON |
| 401 | No key, or the key is wrong or revoked |
| 402 | The workspace is on a billing hold: reads work, writes do not |
| 403 | The key is read only, the teammate's role does not allow it, or the workspace is suspended |
| 404 | Not found, or not visible to the key's teammate |
| 409 | A conflict, for example `window_closed` |
| 413 | The body is larger than 64 KB |
| 422 | A parameter or body field is not valid |
| 429 | Too many requests; wait for `Retry-After` seconds |

Rate limits: **120 reads and 30 writes a minute per key**, and 600 requests a minute per client IP address.

## Webhooks

Add up to 10 endpoints in **Settings → Integrations → API & webhooks**: a public `https://` URL, the events you want, and a **signing secret** (`whsec_…`, shown once). **Send test event** sends a `webhook.test` event.

| Event | Sent when |
|---|---|
| `message.received` | A customer sent a message |
| `message.sent` | Your team, an automation, the AI Agent or the API sent a message |
| `conversation.opened` | A new conversation started, or a closed or pending one opened again |
| `conversation.closed` | A conversation was closed |
| `conversation.assigned` | A conversation was assigned, reassigned or unassigned |

Every event is a JSON `POST`. It carries **ids, not message text or contact details**; read the details with an API key.

```json
{
  "id": "msg_b47e1f20-3c9a-4d58-a6e2-0f9d8c7b6a51",
  "type": "message.received",
  "created_at": "2026-11-19T09:20:03.412Z",
  "workspace_id": "q8ZbT2mK4vN7xR1cL5wP9yF3",
  "data": {
    "conversation_id": "6f1c2b9e-4d7a-4e8b-9c3f-1a2b3c4d5e6f",
    "contact_id": "2c8e5a71-9b3d-4f60-8e21-7d4c9a0b6e13",
    "channel": "whatsapp",
    "message_id": "b47e1f20-3c9a-4d58-a6e2-0f9d8c7b6a51",
    "direction": "inbound"
  }
}
```

Headers: `X-Extremis-Event` (the event type), `X-Extremis-Delivery` (the delivery id, the same on every retry) and `X-Extremis-Signature`.

### Check the signature

`X-Extremis-Signature: t=<unix seconds>,v1=<hex>`, where `v1` is the HMAC-SHA256 of `<t>.<raw request body>` keyed with your whole signing secret. Check it on the raw body before parsing, compare in constant time, and refuse old timestamps.

Node.js:

```js
const crypto = require("node:crypto");

function isFromExtremis(rawBody, signatureHeader, secret, toleranceSeconds = 300) {
  if (!signatureHeader) return false;
  const parts = {};
  for (const piece of signatureHeader.split(",")) {
    const at = piece.indexOf("=");
    if (at > 0) parts[piece.slice(0, at).trim()] = piece.slice(at + 1).trim();
  }
  const t = Number(parts.t);
  if (!Number.isInteger(t) || !/^[0-9a-f]{64}$/.test(parts.v1 || "")) return false;
  if (Math.abs(Date.now() / 1000 - t) > toleranceSeconds) return false;
  const expected = crypto
    .createHmac("sha256", secret)
    .update(`${parts.t}.${rawBody}`)
    .digest();
  return crypto.timingSafeEqual(expected, Buffer.from(parts.v1, "hex"));
}
```

Python:

```python
import hashlib
import hmac
import time


def is_from_extremis(raw_body: bytes, signature_header: str, secret: str, tolerance: int = 300) -> bool:
    if not signature_header:
        return False
    parts = {}
    for piece in signature_header.split(","):
        key, sep, value = piece.partition("=")
        if sep:
            parts[key.strip()] = value.strip()
    t, v1 = parts.get("t", ""), parts.get("v1", "")
    if not t.isdigit() or len(v1) != 64:
        return False
    if abs(time.time() - int(t)) > tolerance:
        return False
    signed = t.encode() + b"." + raw_body
    expected = hmac.new(secret.encode(), signed, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, v1)
```

### Answers and retries

- Answer with any `2xx` within 10 seconds, then do the slow work.
- Anything else, including a redirect, is retried after 1 minute, 5 minutes, 30 minutes, 2 hours and 6 hours. After six attempts the delivery is marked failed.
- The same event can arrive more than once: use its `id` to skip duplicates.
- After 20 deliveries in a row fail, the endpoint is turned off. Turn it back on in Settings once your server is fixed; events from while it was off are not sent.
- Each endpoint has a delivery log of its last 50 deliveries, with **Redeliver**.

The URL must use `https://`, port 443 or 1024 to 65535, and a public internet address. Private networks, `localhost` and other internal addresses are refused.

## More

- [Developer reference](https://extremis.online/developers)
- [Use the Extremis API and webhooks](https://extremis.online/help/channels/use-the-extremis-api-and-webhooks) (Help Center)
- Email: support@extremis.online

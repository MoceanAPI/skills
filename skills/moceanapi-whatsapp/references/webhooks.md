# WhatsApp inbound and delivery webhooks

Operations: `receiveWhatsAppDeliveryStatus`, `receiveWhatsAppInboundMessage`. API contracts reviewed **2026-10-05**. Published documentation: [Mocean API](https://moceanapi.com/docs).

These are **routes implemented on the application's server**, not endpoints to call on `rest.moceanapi.com`:

| Operation | Example application route | Body |
| --- | --- | --- |
| `receiveWhatsAppDeliveryStatus` | POST `/webhooks/whatsapp-delivery-status` | JSON |
| `receiveWhatsAppInboundMessage` | POST `/webhooks/whatsapp-inbound-message` | JSON |

Both accept `application/json`. They document HTTP `200`, `202`, or `204` as acknowledgements, with no response body required. Default examples here acknowledge with `200` after durable acceptance. The route names are examples; use the application's conventions. Delivery callbacks are configured through `mocean-event-url` in the send request. The inbound callback configuration mechanism is not specified by these operations; obtain the account's configured route rather than assuming the send callback enables replies.

The webhook contracts specify no authentication scheme. This does not establish that incoming requests are trustworthy. Do not invent HMAC headers, signatures, challenge-verification routes, or IP ranges from the Meta Cloud API. Use the application's HTTPS and request controls; obtain Mocean-specific verification documentation if authenticated webhooks are required. A valid payload schema is not sender authentication.

## Shared envelope

Both require string `message_id` (exactly 36 characters), `event_type`, and object `event_data`. Keep IDs as strings. Delivery `event_type` is `delivery-status`; inbound `event_type` is `message`.

`message_id` identifies the message, **not each status transition**. Received text, replies, and media are under `event_data.content`; they are not outbound `mocean-content` objects.

## Delivery events

The schema requires the three envelope fields, but marks **none** of the inner `event_data` fields required. Handle missing optional fields without pretending a status was supplied.

| `event_data` field | Contract |
| --- | --- |
| `channel_type` | If present, `WHATSAPP`. |
| `from` | Source number string, 0–15 characters. Preserve documented direction; do not reverse `from`/`to` based on assumptions about an outbound send. |
| `to` | Destination identifier string, 4–131. |
| `from_username` | String, 3–35; optional user-set username. |
| `from_bsuid` | String, 4–131, identifier for a user/business pair. |
| `from_parent_bsuid` | String, 0–135, related identifier shared under a parent account. |
| `meta_delivery_status` | `sent`, `delivered`, `read`, `failed`. |
| `timestamp` | Integer UNIX timestamp. |
| `meta_whatsapp_pricing` | Optional object: `pricing_type` (`regular`, `free_entry_point`); `pricing_category` string, 7–28. Do not calculate prices from this alone. |
| `meta_whatsapp_conversation` | Optional object for free-entry-point conversation: `conversation_id` exactly 36 characters, integer `expiration_timestamp`, `pricing_category` (`authentication`, `marketing`, `utility`, `service`, `referral_conversion`). |
| `meta_error_details` | Optional object: integer `error_code`, string `error_title`, string `error_details`. These are Meta diagnostics, distinct from the send API's numeric `status`. |

Representative delivery payload:

```json
{
  "message_id":"12345678-1234-1234-1234-123456789abc",
  "event_type":"delivery-status",
  "event_data":{
    "channel_type":"WHATSAPP",
    "from":"60198765432",
    "to":"60123456789",
    "meta_delivery_status":"delivered",
    "timestamp":1791172800
  }
}
```

Representative failure payload:

```json
{
  "message_id":"12345678-1234-1234-1234-123456789abc",
  "event_type":"delivery-status",
  "event_data":{
    "channel_type":"WHATSAPP",
    "meta_delivery_status":"failed",
    "timestamp":1791172801,
    "meta_error_details":{"error_code":131047,"error_title":"Re-engagement message","error_details":"Customer service window is closed"}
  }
}
```

## Inbound events

Required inner fields: `event_data.channel_type` (`WHATSAPP`), `to` (4–131 characters), integer UNIX `timestamp`, and `content`. Sender phone number is **optional**; do not reject a BSUID-only customer event for lacking it.

Optional fields:

- `from_name` string; `from` string (0–15); `from_username` (3–35); `from_bsuid` (4–131); `from_parent_bsuid` (0–135).
- Booleans `forwarded` and `frequently_forwarded`.
- String `reply_to_msg_id`, correlating a reply to a Mocean message ID.
- `referred_product` object with optional string `catalog_id`, `product_retailer_id`.

`content` requires `type`. Its documented types and conditional fields are:

| Type | Associated fields |
| --- | --- |
| `text` | String `text`. |
| `image`, `video`, `document` | `media_id`, `mime_type`; `text` caption; document can include string `name`. |
| `audio` | `media_id`, `mime_type`, boolean `voice`. |
| `sticker` | `media_id`, `mime_type`, boolean `animated`. |
| `button` | String display `text`, string `payload` for a template quick reply. |
| `reaction` | String `text` representing the reaction; use envelope `reply_to_msg_id` when provided. |
| `interactive_button` | String `id` and `title` (1–24). This is not the outbound `interactive` shape. |
| `interactive_list` | String `id`, `title` (1–24), `description` (0–72). |
| `order` | String `catalog_id`, `product_items` array. Each item can contain string `product_retailer_id`, string `currency`, numeric `item_price`, integer `quantity`. The description specifies two decimal places but no numeric bounds. |
| `location` | Numeric `latitude`, `longitude`; string `address`. The inbound schema does not define a location `name` field. |
| `contacts` | `wa_contacts` array using the contact-card structure in [content-variants.md](content-variants.md#contacts). |

`media_id` and `mime_type` are strings. The inbound schema does not enforce the table's per-type fields as required; preserve incomplete events and avoid dereferencing absent values. Use `media_id` with [media-downloads.md](media-downloads.md); never pass it as a media URL.

Optional `content.referral` object describes an ad/post source: URI `source_url`, `source_type` (`ad`, `post`), string `source_id`, `headline`, `body`, `rich_media_type` (`image`, `video`), URI `rich_media_url`.

Representative BSUID-only interactive reply:

```json
{
  "message_id":"abcdef12-1234-1234-1234-123456789abc",
  "event_type":"message",
  "event_data":{
    "channel_type":"WHATSAPP",
    "from_bsuid":"MY.13491208655302741918",
    "to":"60123456789",
    "timestamp":1791172800,
    "reply_to_msg_id":"12345678-1234-1234-1234-123456789abc",
    "content":{"type":"interactive_button","id":"track_order","title":"Track order"}
  }
}
```

Representative inbound media:

```json
{
  "message_id":"abcdef12-1234-1234-1234-123456789abc",
  "event_type":"message",
  "event_data":{
    "channel_type":"WHATSAPP",
    "from":"60198765432",
    "to":"60123456789",
    "timestamp":1791172800,
    "content":{"type":"document","media_id":"media-example-42","mime_type":"application/pdf","name":"receipt.pdf"}
  }
}
```

## Acceptance, deduplication, and ordering

Persist receipts and downstream work atomically using the target project's database/queue patterns, then acknowledge. A duplicate already durably accepted can be acknowledged again. On persistence failure, return an application failure response instead of reporting acceptance. Retry schedule, delivery ordering, exactly-once behavior, and event retention are unspecified by the provider.

Use a stable fingerprint of the full canonical JSON envelope for duplicate receipt detection, scoped to the configured integration/tenant. This is an application strategy for exact duplicates, not a provider event ID. It preserves different delivery statuses, timestamps, and diagnostic changes for the same message. Enforce uniqueness atomically in storage; an in-memory set cannot ensure durable or concurrent acceptance. Do not key all receipts by `message_id` alone.

Keep the event history. Project local message state with an explicit application policy that handles out-of-order events without replacing `read` with a late `sent`/`delivered` event. Preserve `failed` diagnostics separately; the docs do not specify a universal transition order or conflict-resolution rule. A delivery event missing status can be stored without changing the projection. Do not overwrite the recipient identity from callback direction fields; correlate using the recorded send ID.

## Framework-neutral acceptance example

The handler below receives an already-parsed, schema-validated JSON envelope and configured `integrationId`. Run validation/authentication and body-size controls in the host framework. `store.accept` must atomically persist the receipt and durable work item, enforce key uniqueness, and resolve for duplicates. It must reject on storage failure. The example deliberately leaves status projection to the application's worker.

```javascript
function canonicalJson(value) {
  if (Array.isArray(value)) return `[${value.map(canonicalJson).join(",")}]`;
  if (value !== null && typeof value === "object") {
    return `{${Object.keys(value).sort().map(key =>
      `${JSON.stringify(key)}:${canonicalJson(value[key])}`).join(",")}}`;
  }
  return JSON.stringify(value);
}

async function acceptWhatsAppEvent({ event, integrationId, store, digest }) {
  const fingerprint = await digest(canonicalJson(event)); // e.g. SHA-256
  const key = JSON.stringify([integrationId, fingerprint]);
  await store.accept(key, event); // atomic receipt + durable work; duplicate is a no-op
  return { status: 200, body: "" };
}
```

`digest` is the project's cryptographic hash facility. Do not log canonical payloads or use the hash as authentication. Test that repeated identical envelopes create one receipt, differing statuses for one message create separate receipts, failed persistence is not acknowledged, and state projection tolerates reverse arrival order.

# WhatsApp send content variants

Operation: `sendWhatsAppMessage`. API contract reviewed **2026-10-05**. Published documentation: [Mocean API](https://moceanapi.com/docs). Each JSON example below is a complete `mocean-content` object. Place it inside the [send envelope](sending.md); these objects are not standalone API requests.

Contents: text; rich media; reactions; template substitutions; interactive button/list/product/product-list; location; contacts.

## Text

Required: `type: "text"`, `text` (1–4096 characters). Optional: boolean `preview_url` (default false), `reply_to_msg_id` (0–36 characters). Reply IDs are Mocean `message_id` values from sends or inbound events. The shared text schema also contains `parameter_name`, used for named template substitutions; omit it from ordinary text messages.

```json
{"type":"text","text":"Your order is ready.","preview_url":false,"reply_to_msg_id":"12345678-1234-1234-1234-123456789abc"}
```

## Rich media

Required: `type` (`image`, `audio`, `document`, `video`, `sticker`) and URI `rich_media_url`. Mocean must be able to download the file from that URL. Optional `text` is a caption for image/video/document; `name` is the document filename; `reply_to_msg_id` is 0–36 characters. Do not use captions or document names on unrelated types.

| Type | Supported MIME types | Limit / constraints |
| --- | --- | --- |
| Image | `image/jpeg`, `image/png` | 5 MB |
| Video | `video/mp4`, `video/3gpp` | 16 MB; H.264 video and AAC audio, single audio stream in the API contract. |
| Audio | `audio/aac`, `audio/mp4`, `audio/mpeg`, `audio/amr`, `audio/ogg` with OPUS | 16 MB; ordinary OGG without OPUS is unsupported. |
| Document | `text/plain`, `application/pdf`, `application/vnd.ms-powerpoint`, `application/msword`, `application/vnd.ms-excel`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document`, `application/vnd.openxmlformats-officedocument.presentationml.presentation`, `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` | 100 MB |
| Sticker | `image/webp` | Static 100 KB; animated 500 KB |

These are outbound message limits. Template creation's header-media limit is different; see [templates.md](templates.md). The contract does not specify byte-unit rounding or a media upload endpoint. Verify the actual file, not only its filename extension.

```json
{"type":"image","rich_media_url":"https://media.example/order.jpg","text":"Packed and ready"}
```

```json
{"type":"audio","rich_media_url":"https://media.example/update.mp3"}
```

```json
{"type":"document","rich_media_url":"https://media.example/receipt.pdf","name":"receipt.pdf","text":"Your receipt"}
```

```json
{"type":"video","rich_media_url":"https://media.example/demo.mp4","text":"Assembly instructions"}
```

```json
{"type":"sticker","rich_media_url":"https://media.example/thanks.webp"}
```

## Reactions

Required: `type: "reaction"`, one-emoji `text`, `reply_to_msg_id` exactly 36 characters. Use the Mocean message ID, not an unrelated Meta identifier.

```json
{"type":"reaction","text":"\ud83d\udc4d","reply_to_msg_id":"12345678-1234-1234-1234-123456789abc"}
```

The API contract's prose says one emoji. Published Mocean docs also allow empty `text` to remove a reaction; this is a documented extension of that prose, not a missing-ID removal operation. [Source](https://moceanapi.com/docs#reaction-content).

## Approved template substitutions

Required: `type: "template"`, `wa_template` object with `name` (1–512 characters) and `language` (2–10 characters). Optional:

- `header_params`: array of text or rich-media objects using the corresponding schemas above. Match the approved header format; a permissive union does not authorize arbitrary header types.
- `body_params`: array of text objects, each requiring `type: "text"` and `text` (1–4096). For named placeholders include `parameter_name`. Do not send creation examples as substitutions.
- `wa_buttons`: at most three send-time button objects. Each requires `type` (`url` or `quick_reply`) and integer `index`. A URL button additionally requires `url_parameter` (1–2000 characters); a quick reply requires string `payload`. Indices must correspond to the approved template. The schema gives no numeric index bounds. Omit this array when buttons are absent or all static.

Use the approved template's exact name, language, parameter order/names, header type, and button indices. Positional `body_params` follows placeholder order. Named text parameters use `parameter_name` matching `{{name}}`. The send object does not define a `parameter_format` field; that belongs to creation. The shared text schema's `preview_url` and `reply_to_msg_id` fields do not imply they change template delivery behavior.

Positional example, assuming an approved image-header template with two body variables and the shown buttons:

```json
{
  "type":"template",
  "wa_template":{
    "name":"order_update",
    "language":"en",
    "header_params":[{"type":"image","rich_media_url":"https://media.example/order.jpg"}],
    "body_params":[{"type":"text","text":"Aisha"},{"type":"text","text":"ORDER-42"}],
    "wa_buttons":[
      {"type":"url","index":0,"url_parameter":"ORDER-42"},
      {"type":"quick_reply","index":1,"payload":"order:42:help"}
    ]
  }
}
```

Named example, assuming an approved body-only template containing `{{customer}}` and `{{order_ref}}`:

```json
{
  "type":"template",
  "wa_template":{
    "name":"order_update_named",
    "language":"en",
    "body_params":[
      {"type":"text","parameter_name":"customer","text":"Aisha"},
      {"type":"text","parameter_name":"order_ref","text":"ORDER-42"}
    ]
  }
}
```

Definition-time buttons may exceed three; the documented send-time limit remains three. OTP creation supports `COPY_CODE`, `ONE_TAP`, and `ZERO_TAP`, but this send schema exposes only `url`/`quick_reply` button substitutions. Do not invent an OTP send subtype; obtain the approved template's actual send representation for that feature.

## Interactive messages

Required: `type: "interactive"` and `wa_interactive` with `type` (`button`, `list`, `product`, `product_list`) and `action`. Optional `header` follows the text/media union; optional `body` and `footer` follow the text schema and therefore require `type: "text"` and `text` (1–4096) when included. Some published examples omit this nested `type`; these examples retain it to satisfy the resolved schema.

The action schema lists subtype fields together and does not enforce all subtype-specific requirements. Select fields appropriate to the chosen subtype, as below. Do not claim that validating the generic action object alone proves a usable provider payload. No aggregate row/product count is specified beyond section limits in this source.

| Subtype | Action fields used |
| --- | --- |
| `button` | `wa_buttons` array, at most three. Each requires `type: "reply"`, unique `title` (1–20), unique `id` (1–256); IDs must not start with spaces. |
| `list` | `button_text` (1–20) and `sections` (1–10). Each section has optional `title` (1–24) and `rows`; each row requires unique-across-sections `id` (1–200), `title` (1–24), optional `description` (0–72). |
| `product` | Nonempty `catalog_id` and `product_retailer_id` identifying a product in the sender WABA's connected catalog. |
| `product_list` | Nonempty `catalog_id` and `sections` (1–10); sections have optional `title` (1–24) and `product_items`, each requiring nonempty `product_retailer_id`. |

Button:

```json
{
  "type":"interactive",
  "wa_interactive":{
    "type":"button",
    "body":{"type":"text","text":"Would you like an update?"},
    "action":{"wa_buttons":[
      {"type":"reply","title":"Track order","id":"track_order"},
      {"type":"reply","title":"Contact support","id":"contact_support"}
    ]}
  }
}
```

List:

```json
{
  "type":"interactive",
  "wa_interactive":{
    "type":"list",
    "header":{"type":"text","text":"Order help"},
    "body":{"type":"text","text":"Choose a topic."},
    "action":{
      "button_text":"Show topics",
      "sections":[
        {"title":"Orders","rows":[{"id":"track_order","title":"Track order","description":"See delivery progress"}]},
        {"title":"Support","rows":[{"id":"contact_support","title":"Contact support"}]}
      ]
    }
  }
}
```

Single product:

```json
{
  "type":"interactive",
  "wa_interactive":{
    "type":"product",
    "body":{"type":"text","text":"View this item."},
    "action":{"catalog_id":"catalog-42","product_retailer_id":"sku-100"}
  }
}
```

Product list:

```json
{
  "type":"interactive",
  "wa_interactive":{
    "type":"product_list",
    "header":{"type":"text","text":"Available items"},
    "body":{"type":"text","text":"Browse the collection."},
    "footer":{"type":"text","text":"Ask us about availability."},
    "action":{
      "catalog_id":"catalog-42",
      "sections":[{"title":"Featured","product_items":[{"product_retailer_id":"sku-100"},{"product_retailer_id":"sku-200"}]}]
    }
  }
}
```

## Location

Required: `type: "location"`, `location` with numeric `latitude` and `longitude`. Optional string `name`, `address`. The schema has no coordinate bounds; any geographic range validation is an application rule.

```json
{"type":"location","location":{"latitude":3.047,"longitude":101.691,"name":"Collection desk","address":"Bukit Jalil, Kuala Lumpur"}}
```

## Contacts

Required: `type: "contacts"`, nonempty `wa_contacts` array. Each contact requires `name`, which requires nonempty `formatted_name`. Other contact fields are optional:

| Field | Nested fields and constraints |
| --- | --- |
| `name` | `formatted_name`; optional `first_name`, `last_name`, `middle_name`, `suffix_name`, `prefix` strings. Preserve `suffix_name` as documented. |
| `birthday` | Date string, e.g. `1990-05-20`. |
| `addresses[]` | Optional strings `street`, `city`, `state` (exactly 2 characters), `zip`, `country`, `country_code` (exactly 2), `type` (`HOME`, `WORK`). |
| `emails[]` | Optional `email` (email format), `type` (`HOME`, `WORK`). |
| `org` | Optional strings `company`, `department`, `title`. |
| `phones[]` | Optional `phone` (7–15 characters), `type` (`CELL`, `MAIN`, `IPHONE`, `HOME`, `WORK`), `wa_id` string. Without `wa_id`, the contract says the contact shows an invitation action. |
| `urls[]` | Optional `url` URI, `type` (`HOME`, `WORK`). |

```json
{
  "type":"contacts",
  "wa_contacts":[{
    "name":{"formatted_name":"Aisha Tan","first_name":"Aisha","last_name":"Tan"},
    "birthday":"1990-05-20",
    "addresses":[{"street":"12 Example Road","city":"Kuala Lumpur","state":"KL","zip":"57000","country":"Malaysia","country_code":"MY","type":"WORK"}],
    "emails":[{"email":"support@example.com","type":"WORK"}],
    "org":{"company":"Example Shop","department":"Support","title":"Advisor"},
    "phones":[{"phone":"60123456789","type":"WORK","wa_id":"60123456789"}],
    "urls":[{"url":"https://shop.example","type":"WORK"}]
  }]
}
```

# WhatsApp template management

Operations: `listWhatsAppTemplates`, `getWhatsAppTemplate`, `createWhatsAppTemplate`, `editWhatsAppTemplate`, `deleteWhatsAppTemplate`. API contracts reviewed **2026-10-05**. Published documentation: [Mocean API](https://moceanapi.com/docs). HTTP failures: [errors.md](errors.md).

All operations authenticate with `Authorization: Bearer <MOCEAN_API_TOKEN>` on the server. POST bodies are JSON with `Content-Type: application/json`. GET and DELETE have no body. These operations define templates; sending uses [send-time substitutions](content-variants.md#approved-template-substitutions).

## Endpoints and responses

Base: `https://rest.moceanapi.com/template/whatsapp`.

| Operation | Method and path relative to base | Documented success |
| --- | --- | --- |
| `listWhatsAppTemplates` | GET `/message_templates` | HTTP 200; JSON object with `data` array and optional `paging`. |
| `getWhatsAppTemplate` | GET `/{templateId}` | HTTP 200; JSON template object. |
| `createWhatsAppTemplate` | POST `/message_templates` | HTTP 200; JSON template object. |
| `editWhatsAppTemplate` | POST `/{templateId}` | HTTP 202; JSON template object. This is POST, not PATCH. |
| `deleteWhatsAppTemplate` | DELETE `/message_templates?name=...` | HTTP 200; JSON object with string `success`, example `"true"`. Require that exact string for confirmation; do not test truthiness (`"false"` is truthy). A boolean response is not the documented shape. |

Path `templateId`: required string, 1–100 characters, URL-encode as one path segment. Delete requires query `name` (1–512); optional `hsm_id` (1–100), both strings. The API schema uses `hsm_id`; published docs also spell it `hsm-id`. Use the schema's spelling and verify an actual deployment's accepted spelling if it rejects the selector; do not silently drop it and broaden a deletion.

Template success schemas reuse the definition schema below, requiring `name`, `language`, `category`, and `components`. They do **not** describe template ID, approval status, or quality fields. Retain those values if actually returned, but do not require them based on this contract. Do not interpret create/edit success as approval or `status: 0`. If a feature needs IDs or current approval state and the response lacks them, obtain a verified response contract or user-supplied approved metadata.

Example GET, body omitted:

```http
GET https://rest.moceanapi.com/template/whatsapp/template-42
Authorization: Bearer <MOCEAN_API_TOKEN>
```

Example DELETE, body omitted:

```http
DELETE https://rest.moceanapi.com/template/whatsapp/message_templates?name=order_ready&hsm_id=template-42
Authorization: Bearer <MOCEAN_API_TOKEN>
```

Example delete result:

```json
{"success":"true"}
```

## Listing and pagination

Eight optional query filters are documented:

| Filter | Contract |
| --- | --- |
| `limit` | Integer; maximum results per page. No default or numeric bounds documented. |
| `category` | `AUTHENTICATION`, `MARKETING`, `UTILITY`. |
| `content` | String, 0–1024; the description excludes searching multiple words. |
| `language` | String, 2–10. |
| `name` | String, 0–512. |
| `name_or_content` | String, 0–1024. |
| `quality_score` | `GREEN`, `YELLOW`, `RED`, `UNKNOWN`. |
| `status` | `APPROVED`, `PENDING`, `REJECTED`, `IN_APPEAL`, `PENDING_DELETION`, `DELETED`, `DISABLED`, `PAUSED`, `LIMIT_EXCEEDED`, `ARCHIVED`. |

```http
GET https://rest.moceanapi.com/template/whatsapp/message_templates?limit=2&status=APPROVED&language=en
Authorization: Bearer <MOCEAN_API_TOKEN>
```

Representative list result, constructed from the response schema:

```json
{
  "data":[{"name":"order_ready","language":"en","category":"UTILITY","components":[{"type":"BODY","text":"Your order is ready."}]}],
  "paging":{"cursors":{"before":"cursor-start","after":"cursor-end"}}
}
```

`paging.cursors.before` and `.after` are strings; optional `paging.next` is a URI. The description calls it a Graph API URL. It allows an empty page with a next link and says absence of next ends pagination. **No cursor request parameter is defined.** Do not invent `after`/`before` query inputs or blindly send the Mocean bearer token to a returned URL, especially a different host. Expose returned paging metadata; obtain a verified Mocean pagination procedure before implementing automatic traversal. Disable credential-bearing cross-origin redirects in a client that follows URLs.

## Definition body for create and edit

| Field | Required | Contract |
| --- | --- | --- |
| `name` | Yes | Lowercase letters, digits, underscores only (`^[a-z0-9_]+$`), up to 512 characters. |
| `language` | Yes | String, 2–10; select the intended language variant. |
| `category` | Yes | `AUTHENTICATION`, `MARKETING`, `UTILITY`. |
| `components` | Yes | Array of HEADER/BODY/FOOTER/BUTTONS components below. Every template needs a BODY. |
| `allowCategoryChange` | No | Boolean, default false; permits Meta to recategorize during review. Preserve this camel-case spelling. |
| `message_send_ttl_seconds` | No | Integer, authentication-message TTL in seconds; no range given. |
| `parameter_format` | No | `POSITIONAL` (default) or `NAMED`. Every placeholder needs example data. |

Use this body for `POST /message_templates` or `POST /{templateId}`. Inspect the existing definition before an edit so required components and variables survive. Resolve the explicit target and intended change before a live mutation.

### Components

Components and button variants have `additionalProperties: false` in the contract; do not copy unrelated Meta API fields into them.

| Component | Fields and conditions |
| --- | --- |
| `HEADER` | At most one, optional. Required `type`, `format`, `text` in the resolved schema. `format`: `TEXT`, `IMAGE`, `VIDEO`, `DOCUMENT`, `LOCATION`. `text` up to 60 characters; at most one placeholder; if present, supply `example`. Optional `header_media`: direct HTTP(S) URL, at least 4 characters; limit 16 MB; accepted `application/pdf`, `image/jpeg`, `image/jpg`, `image/png`, `video/mp4`. LOCATION headers only for UTILITY/MARKETING. **Gap:** the schema requires `text` even for nontext formats; it does not resolve the correct nontext/location creation body. Verify that body before implementing it. |
| `BODY` | Required component. Required `type: "BODY"`, `text` up to 1024. Placeholder text requires `example`. Optional `add_security_recommendation` boolean, AUTHENTICATION only. **Gap:** prose calls text required for UTILITY/MARKETING, while the schema requires it for all categories. Retain explicit text in examples; verify any authentication API that omits it. |
| `FOOTER` | At most one, optional. Required `type: "FOOTER"`; optional `text` up to 60 (described as required for UTILITY/MARKETING when this component is used); optional integer `code_expiration_minutes`, AUTHENTICATION only, no range given. |
| `BUTTONS` | At most one, optional. Required `type: "BUTTONS"` and `buttons` array, at most 10 total. Up to 10 QUICK_REPLY, 2 URL, 1 PHONE_NUMBER, 1 OTP buttons individually. Combination/grouping rules are not fully specified. |

### Placeholders and examples

Positional placeholders use `{{1}}`, `{{2}}`, etc. `example.header_text` is an array of strings; `example.body_text` is an array of arrays of strings. Match placeholder order and count. Named placeholders use lowercase letters and underscores; names must be unique.

The API's `ComponentExample` schema declares `header_text_named_params` and `body_text_named_params` as **objects**, without nested properties. Its root also permits string `param_name` and `example`. Mocean's published examples instead use **arrays** of `{param_name, example}`. This conflict is unresolved; do not treat either shape as universally validated. The named definition example below follows the published array example and requires confirmation against the target API before live use. [Evidence](https://moceanapi.com/docs#example-components).

Complete positional create/edit body (also representative of the documented template response shape):

```json
{
  "name":"order_ready",
  "language":"en",
  "category":"UTILITY",
  "parameter_format":"POSITIONAL",
  "components":[
    {"type":"HEADER","format":"TEXT","text":"Order {{1}}","example":{"header_text":["ORDER-42"]}},
    {"type":"BODY","text":"Hello {{1}}, your order is ready.","example":{"body_text":[["Aisha"]]}},
    {"type":"FOOTER","text":"Thank you for shopping with us."},
    {"type":"BUTTONS","buttons":[
      {"type":"URL","text":"View order","url":"https://shop.example/orders/{{1}}","example":["https://shop.example/orders/ORDER-42"]},
      {"type":"QUICK_REPLY","text":"Need help"}
    ]}
  ]
}
```

Complete named body from the published example convention; **known API schema conflict**, not a schema-conformant fixture:

```json
{
  "name":"order_ready_named",
  "language":"en",
  "category":"UTILITY",
  "parameter_format":"NAMED",
  "components":[{
    "type":"BODY",
    "text":"Hello {{customer}}, your order is ready.",
    "example":{"body_text_named_params":[{"param_name":"customer","example":"Aisha"}]}
  }]
}
```

### Button definitions

| Type | Required and optional fields |
| --- | --- |
| `QUICK_REPLY` | Required `type`, `text` (up to 25). |
| `URL` | Required `type`, `text` (up to 25), URI `url`. The URL may contain exactly one `{{1}}` dynamic suffix; if present, provide `example` array of strings, each up to 2000. |
| `PHONE_NUMBER` | Required `type`, `text` (up to 25), `phone_number` string (up to 20). |
| `OTP`, `COPY_CODE` | Required `type: "OTP"`, `otp_type: "COPY_CODE"`. Optional localized `text` up to 25; provider supplies a localized default if absent. |
| `OTP`, `ONE_TAP` | Required `type`, `otp_type: "ONE_TAP"`, `supported_apps`. Optional `text`, `autofill_text`, each up to 25. |
| `OTP`, `ZERO_TAP` | Required `type`, `otp_type: "ZERO_TAP"`, `supported_apps`, `zero_tap_terms_accepted`. That boolean must be true for creation; do not set it without the user's actual acceptance of the terms. Optional `text`, `autofill_text`, each up to 25. |

For ONE_TAP/ZERO_TAP, `supported_apps` is an array of app objects containing `package_name` (up to 224, dotted Android package with at least two segments starting with letters) and `signature_hash` (exactly 11 characters). The description allows alphanumeric/underscore characters. **Schema defect:** `required` is attached to the array, and `additionalProperties: false` is placed inside item `properties`. Treat package/hash as required by prose; do not emit a literal `additionalProperties` field or claim this malformed schema enforces item requirements.

Complete authentication COPY_CODE definition:

```json
{
  "name":"login_code",
  "language":"en",
  "category":"AUTHENTICATION",
  "parameter_format":"POSITIONAL",
  "components":[
    {"type":"BODY","text":"Your verification code is {{1}}.","add_security_recommendation":true,"example":{"body_text":[["123456"]]}},
    {"type":"FOOTER","code_expiration_minutes":5},
    {"type":"BUTTONS","buttons":[{"type":"OTP","otp_type":"COPY_CODE"}]}
  ]
}
```

Complete alternative button components (substitute into an appropriate definition, not into a send payload):

```json
{"type":"BUTTONS","buttons":[{"type":"PHONE_NUMBER","text":"Call support","phone_number":"60123456789"}]}
```

```json
{"type":"BUTTONS","buttons":[{"type":"OTP","otp_type":"ONE_TAP","supported_apps":[{"package_name":"com.example.shop","signature_hash":"AbCdEf01234"}]}]}
```

```json
{"type":"BUTTONS","buttons":[{"type":"OTP","otp_type":"ZERO_TAP","supported_apps":[{"package_name":"com.example.shop","signature_hash":"AbCdEf01234"}],"zero_tap_terms_accepted":true}]}
```

The last JSON illustrates the field shape only; it is not authorization to accept terms or create a template. Definition-time OTP buttons do not establish a new send-time OTP payload variant.

## HTTP client pattern

The following example keeps HTTP/JSON handling separate from operation-specific validation. Callers should validate definition/list shapes above, and inspect delete `success` as shown. A create/edit response lacking approval metadata must not be marked approved.

```javascript
async function requestWhatsAppTemplate({ token, operation, templateId, name, hsmId,
    filters = {}, definition, fetchImpl = fetch, timeoutMs = 10000 }) {
  if (!token) throw new Error("MOCEAN_API_TOKEN is required");
  const methods = { list: "GET", get: "GET", create: "POST", edit: "POST", delete: "DELETE" };
  if (!Object.hasOwn(methods, operation)) throw new Error("Unknown template operation");
  if ((operation === "get" || operation === "edit") && !templateId) {
    throw new Error("templateId is required");
  }
  if (operation === "delete" && !name) throw new Error("Template name is required");
  const path = operation === "get" || operation === "edit"
    ? encodeURIComponent(templateId) : "message_templates";
  const url = new URL(`https://rest.moceanapi.com/template/whatsapp/${path}`);
  if (operation === "list") {
    const allowed = new Set(["limit", "category", "content", "language", "name", "name_or_content", "quality_score", "status"]);
    for (const [key, value] of Object.entries(filters)) {
      if (!allowed.has(key)) throw new Error("Undocumented template filter");
      if (value !== undefined) url.searchParams.set(key, String(value));
    }
  }
  if (operation === "delete") {
    url.searchParams.set("name", name);
    if (hsmId !== undefined) url.searchParams.set("hsm_id", hsmId);
  }
  const headers = { Authorization: `Bearer ${token}` };
  const options = { method: methods[operation], headers, signal: AbortSignal.timeout(timeoutMs) };
  if (operation === "create" || operation === "edit") {
    if (!definition) throw new Error("Template definition is required");
    headers["Content-Type"] = "application/json";
    options.body = JSON.stringify(definition);
  }
  const response = await fetchImpl(url, { ...options, redirect: "error" });
  if (!response.ok) throw new Error(`Mocean template HTTP ${response.status}`);
  const result = await response.json();
  if (!result || typeof result !== "object" || Array.isArray(result)) {
    throw new Error("Invalid template response");
  }
  if (operation === "delete" && result.success !== "true") {
    throw new Error("Template deletion was not confirmed");
  }
  if (operation === "list" && !Array.isArray(result.data)) {
    throw new Error("Missing template data array");
  }
  const templates = operation === "list" ? result.data
    : operation === "delete" ? [] : [result];
  for (const template of templates) {
    if (!template || typeof template.name !== "string" ||
        typeof template.language !== "string" ||
        !["AUTHENTICATION", "MARKETING", "UTILITY"].includes(template.category) ||
        !Array.isArray(template.components)) {
      throw new Error("Invalid template definition response");
    }
  }
  return result;
}
```

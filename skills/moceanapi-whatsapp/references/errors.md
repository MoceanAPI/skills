# WhatsApp errors and uncertain outcomes

Operations: `sendWhatsAppMessage`, `downloadWhatsAppMedia`, `listWhatsAppTemplates`, `getWhatsAppTemplate`, `createWhatsAppTemplate`, `editWhatsAppTemplate`, `deleteWhatsAppTemplate`; delivery diagnostics: `receiveWhatsAppDeliveryStatus`. API contracts reviewed **2026-10-05**. Published documentation: [Mocean API](https://moceanapi.com/docs).

## Send application statuses

The API's send description names `WhatsappStatusCodes` but omits the code definition. This compact table is verified against [Mocean's published WhatsApp status table](https://moceanapi.com/docs#send-whatsapp), checked **2026-10-05**. Do not import SMS/Verify meanings into this table.

| `status` | Meaning |
| --- | --- |
| 0 | Accepted without an application error. |
| 1 | Authentication failure. |
| 2 | Insufficient credit. |
| 6 | Destination missing. |
| 34 | Invalid request/body. |
| 44 | Sender invalid. |
| 72 | Content/interactive type invalid. |
| 78 | Sender bot/WABA configuration invalid or inactive. |
| 92 | Content syntax invalid. |
| 93 | Content object missing. |
| 94 | Content parameters invalid. |

HTTP `202` alone is insufficient. Keep failure `err_msg` and optional `hint` for sanitized diagnostics; unknown nonzero statuses are failures. The `message_id` identifies an accepted message, not delivery. See [sending.md](sending.md) for response validation.

## HTTP failures by operation

| Operation | Documented HTTP failures |
| --- | --- |
| `sendWhatsAppMessage` | Only HTTP 202 is listed; its nonzero application status can indicate rejection. Do not claim an exhaustive HTTP error list. |
| `downloadWhatsAppMedia` | 400, 401, 403, 404, 406, 429, 500. |
| `listWhatsAppTemplates`, `getWhatsAppTemplate`, `deleteWhatsAppTemplate` | 400, 401, 406, 429, 500. |
| `createWhatsAppTemplate` | 400, 401, 406, 413, 415, 429, 500, 502. |
| `editWhatsAppTemplate` | Only 400 is listed as a failure; handle other non-success HTTP results defensively without claiming their bodies are documented. |

Interpretation: 400 invalid/missing input; 401 authentication; media 403 unknown sender and 404 missing media; 406 endpoint/not acceptable; 413 oversized media; 415 media type unsupported; 429 throttling; 500 server failure; 502 media-host fetch failure. Error-body schemas are absent for template/media HTTP errors. A blank body is still an error, not a JSON parser requirement.

**Contract inconsistency:** template list/delete `x-mocean-status-codes` lists 413/415/502 even though their operation responses omit them. This reference uses each operation's explicit `responses` as its documented HTTP list. Handle other errors generically and preserve the actual HTTP result.

## Delivery diagnostics and retry decisions

Delivery webhook `meta_error_details.error_code` is a Meta code. Keep it separate from send application status and HTTP status. Store available title/details with the event after sanitization; do not infer a numeric code mapping missing from the contract.

Fix invalid credentials/configuration/payloads before another explicit attempt. Bound retries for safe reads such as template listing and media downloads according to the application's policy; use rate-limit headers if actually present. No retry schedule, idempotency key, or general deduplicating send mechanism is documented here. Do not automatically repeat a send or template mutation after a timeout, connection loss, malformed success response, or server error that leaves processing uncertain. Record the uncertain outcome and reconcile through stored IDs, callbacks, or verified template state. A provider rejection can be reported without retry.

# Sending WhatsApp messages

Operation: `sendWhatsAppMessage`. API contract reviewed **2026-10-05**. Published documentation: [Mocean API](https://moceanapi.com/docs). Payload variants: [content-variants.md](content-variants.md). Failures: [errors.md](errors.md).

```http
POST https://rest.moceanapi.com/rest/2/send-message/whatsapp
Authorization: Bearer <MOCEAN_API_TOKEN>
Content-Type: application/json
```

| JSON field | Required | Contract |
| --- | --- | --- |
| `mocean-from` | Yes | String, 7–15 characters; registered WhatsApp Business sender number, with country code. |
| `mocean-to` | Yes | String, 4–131 characters; recipient number with country code **or** Business Scoped User ID. One recipient; no batch delimiter contract. |
| `mocean-content` | Yes | Object; select exactly one documented variant with `type`. |
| `mocean-event-url` | No | URI for delivery events on the application's server. This does not document how inbound callbacks are configured. |

Example request:

```json
{
  "mocean-from": "60123456789",
  "mocean-to": "60198765432",
  "mocean-event-url": "https://app.example/webhooks/whatsapp",
  "mocean-content": {
    "type": "text",
    "text": "Your order is ready.",
    "preview_url": false
  }
}
```

The published docs restrict free-form sends to the customer service window: use approved templates for recipients who never messaged the business or whose last message was over 24 hours ago. Do not infer an open window from a prior outbound send. [Source](https://moceanapi.com/docs#send-whatsapp).

## Responses

The API contract documents HTTP `202` and says to inspect `status`, `err_msg`, and `hint`, but leaves the response schema and status-code definition unresolved. The following fields are verified against [Mocean's published send documentation](https://moceanapi.com/docs#send-whatsapp): numeric `status`; `message_id` on success; `err_msg` on failure; optional failure `hint`.

Representative accepted response:

```json
{"status":0,"message_id":"12345678-1234-1234-1234-123456789abc"}
```

Representative rejected response:

```json
{"status":94,"err_msg":"Invalid content parameters","hint":"Required template data is absent"}
```

Accept a send only when the HTTP result succeeds, `status` is numeric zero, and a nonempty string `message_id` is present. Preserve the ID with the local message and recipient. Report nonzero statuses without requiring a success ID. Do not coerce missing or string-valued statuses into success. Unknown nonzero codes remain failures. Acceptance does not establish delivery; receive delivery events through [webhooks.md](webhooks.md).

## Minimal server-side client example

This JavaScript example demonstrates HTTP and response handling; use the target project's equivalent HTTP client, validation, and error model. Payload validation must enforce the selected variant before calling this function. The timeout is an application choice, not a provider requirement. Each call makes one submission.

```javascript
async function sendWhatsApp({ token, payload, fetchImpl = fetch, timeoutMs = 10000 }) {
  if (!token) throw new Error("MOCEAN_API_TOKEN is required");
  const response = await fetchImpl("https://rest.moceanapi.com/rest/2/send-message/whatsapp", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${token}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify(payload),
    signal: AbortSignal.timeout(timeoutMs)
  });
  if (!response.ok) throw new Error(`Mocean HTTP ${response.status}`);
  let result;
  try { result = await response.json(); }
  catch { throw new Error("Invalid Mocean send response; reconcile before resubmitting"); }
  const responseError = message => {
    const error = new Error(message);
    if (typeof result?.message_id === "string" && result.message_id.trim()) {
      error.messageId = result.message_id; // retain for reconciliation
    }
    return error;
  };
  if (!result || typeof result !== "object" || Array.isArray(result) ||
      !Number.isInteger(result.status)) {
    throw responseError("Missing or invalid Mocean send status; reconcile before resubmitting");
  }
  if (result.status !== 0) throw responseError(`Mocean send status ${result.status}`);
  if (typeof result.message_id !== "string" || !result.message_id.trim()) {
    throw responseError("Missing Mocean message_id; reconcile before resubmitting");
  }
  return { messageId: result.message_id, accepted: true };
}
```

Transport errors propagate without retry. In the application, distinguish an explicit provider rejection from an unknown submission outcome. Keep available response identifiers for reconciliation, even if the rest of a response is malformed. Do not forward raw provider diagnostics to users or logs without sanitization.

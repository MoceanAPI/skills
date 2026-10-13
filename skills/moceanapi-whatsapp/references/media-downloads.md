# Downloading inbound WhatsApp media

Operation: `downloadWhatsAppMedia`. API contract reviewed **2026-10-05**. Published documentation: [Mocean API](https://moceanapi.com/docs). HTTP failures: [errors.md](errors.md).

```http
GET https://rest.moceanapi.com/rest/2/media/whatsapp?mocean-from=60123456789&mocean-media-id=media-example-42
Authorization: Bearer <MOCEAN_API_TOKEN>
```

No request body or request `Content-Type` is needed. Required query fields:

| Field | Contract |
| --- | --- |
| `mocean-from` | Registered WABA sender number, string 7–15 characters. Use the application's configured sender; do not substitute the inbound customer's number. |
| `mocean-media-id` | Media identifier from `event_data.content.media_id` in an [inbound event](webhooks.md), string 1–100. |

URL-encode query fields. Keep the bearer token on the server. This is a media-ID download operation, not a general URL fetch or an upload API.

## Response handling

HTTP `200` returns a file, modeled by the API schema as binary `application/octet-stream`. Preserve bytes; do not call a JSON decoder on the success body. Actual responses may identify a specific media MIME type. The provider does not document retention duration or an expiry field here.

HTTP errors: `400` missing parameters, `401` authentication failure, `403` sender not found, `404` media not found, `406` incorrect endpoint/not acceptable, `429` rate limiting, `500` server failure. Their response bodies are not specified and may be empty; do not require a JSON `status` or `err_msg` to report failure.

Illustrative binary response:

```http
HTTP/1.1 200 OK
Content-Type: application/octet-stream

<binary file bytes>
```

Illustrative failure with no body:

```http
HTTP/1.1 404 Not Found
Content-Length: 0
```

Use the target project's streaming facilities for larger files and apply an application-defined download limit. Treat returned filenames and MIME metadata as external input. Do not store arbitrary provider filenames as filesystem paths. These are application safeguards, not extra provider parameters.

## Minimal server-side download example

This bounded in-memory example demonstrates binary handling. Choose the byte limit and timeout for the application; neither is a documented provider default. For production large-file workflows, stream directly into bounded storage.

```javascript
async function downloadWhatsAppMedia({ token, sender, mediaId, maxBytes,
    fetchImpl = fetch, timeoutMs = 10000 }) {
  if (!token) throw new Error("MOCEAN_API_TOKEN is required");
  if (typeof sender !== "string" || sender.length < 7 || sender.length > 15) {
    throw new Error("Invalid WhatsApp sender");
  }
  if (typeof mediaId !== "string" || mediaId.length < 1 || mediaId.length > 100) {
    throw new Error("Invalid WhatsApp media ID");
  }
  if (!Number.isSafeInteger(maxBytes) || maxBytes <= 0) throw new Error("A byte limit is required");
  const url = new URL("https://rest.moceanapi.com/rest/2/media/whatsapp");
  url.searchParams.set("mocean-from", sender);
  url.searchParams.set("mocean-media-id", mediaId);
  const response = await fetchImpl(url, {
    headers: { Authorization: `Bearer ${token}` },
    redirect: "error",
    signal: AbortSignal.timeout(timeoutMs)
  });
  if (response.status !== 200) throw new Error(`Mocean media HTTP ${response.status}`);
  if (!response.body) throw new Error("Missing media response body");
  const reader = response.body.getReader();
  const chunks = [];
  let size = 0;
  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      size += value.byteLength;
      if (size > maxBytes) {
        await reader.cancel();
        throw new Error("Media exceeds application byte limit");
      }
      chunks.push(value);
    }
  } finally { reader.releaseLock(); }
  const bytes = new Uint8Array(size);
  let offset = 0;
  for (const chunk of chunks) { bytes.set(chunk, offset); offset += chunk.byteLength; }
  return { bytes, contentType: response.headers.get("content-type") };
}
```

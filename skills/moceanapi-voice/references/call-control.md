# Hangup and recording retrieval

Use the `call_uuid` from an accepted call, not `session_uuid`. See [errors.md](errors.md) for application status codes. Keep the bearer token server-side.

## Hangup (`hangupCall`)

```http
POST https://rest.moceanapi.com/rest/2/voice/hangup
Authorization: Bearer <MOCEAN_API_TOKEN>
Content-Type: application/x-www-form-urlencoded
```

| Form field | Required | Contract |
| --- | --- | --- |
| `mocean-call-uuid` | Yes | String identifying the call to end. |
| `mocean-resp-format` | No | `JSON` or `XML`; default `XML`. Prefer `JSON`. |

Multipart form data is also listed, but ordinary form encoding suffices. Documented success is HTTP `202` with:

```json
{"status":0,"session_uuid":"session-example","call_uuid":"call-example"}
```

Require HTTP success and integer `status: 0`; nonzero status is failure. Preserve returned identifiers when present. Do not interpret a missing status as success or promise repeated hangup requests are idempotent.

## Recording (`downloadRecording`)

```http
GET https://rest.moceanapi.com/rest/2/voice/rec?mocean-call-uuid=<encoded-call-uuid>
Authorization: Bearer <MOCEAN_API_TOKEN>
```

The only declared query parameter is required string `mocean-call-uuid`. Build the query with the HTTP client's URL encoder. Do not put it in a GET body, despite the source cURL sample doing so. `mocean-resp-format` is mentioned in descriptive text but is not a declared parameter here; omit it in baseline requests.

The flow must have executed `record`. The specification documents HTTP `202` and either binary `audio/mpeg` or a JSON/XML status body. A JSON example is:

```json
{"status":0,"calluuid":"call-example"}
```

Note `calluuid`, without an underscore, in this status response. Even status zero does not contain audio bytes. A missing recording, its readiness timing, retention period, and automatic retry schedule are not defined by these sources.

Check HTTP status and content type before writing a file. Save successful nonempty `audio/mpeg` as binary; do not base64-decode it merely because the schema lists `contentEncoding: base64`. JSON/XML responses must stay status/error results, never `.mp3` files. Parse XML with the project's safe parser when needed; do not enable external entities. Reject unknown types or report them as unsupported rather than claiming success. Do not forward authorization across hosts if handling redirects.

### Response interpretation

| HTTP/content result | Interpretation |
| --- | --- |
| Non-success HTTP status | Request failure, regardless of body type. |
| Successful HTTP status with nonempty `audio/mpeg` | Recording bytes; save or stream as binary with application-appropriate size limits. |
| JSON/XML with integer `status: 0` | Status response only; no recording bytes were returned. |
| JSON/XML with nonzero `status` | Application error; use the voice error table. |
| Missing or malformed status, empty audio, or unsupported content type | Unexpected response; do not report a successful download. |

Inspect the media type independently of optional content-type parameters. Recording status uses `calluuid`; hangup and outbound results use `call_uuid`. Do not invent a replacement response shape in the provider contract; choose application return types to fit the target project.

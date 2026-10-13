# Outbound calls

Contract: `makeOutboundCall`. Error codes: [errors.md](errors.md).

```http
POST https://rest.moceanapi.com/rest/2/voice/dial
Authorization: Bearer <MOCEAN_API_TOKEN>
Content-Type: application/x-www-form-urlencoded
```

| Form field | Required | Contract |
| --- | --- | --- |
| `mocean-to` | Yes | String containing a number with country code, e.g. `60123456789`. Multiple recipients may be separated by commas or whitespace. No numeric recipient limit is specified in these sources. |
| `mocean-command` | Yes | JSON string representing an MC array; encode with the other form fields. See [commands.md](commands.md). |
| `mocean-from` | No | Caller ID string. Do not require it merely because an example includes it. |
| `mocean-resp-format` | No | `JSON` or `XML`; default `XML`. Use `JSON` for the examples here. |
| `mocean-event-url` | No | Public callback URL for call status such as answered/hangup. Its payload and authentication are not specified by the supplied voice contract. |

The documented successful HTTP status is `202 Accepted`. Success has a nonempty `calls` array, with `status: 0`, `receiver`, `session_uuid`, and `call_uuid` per accepted call:

```json
{"calls":[{"status":0,"receiver":"60123456789","session_uuid":"session-example","call_uuid":"call-example"}]}
```

Top-level failures contain `status` and optionally `err_msg`, for example:

```json
{"status":3,"err_msg":"No rate found for destination number."}
```

Inspect each call's status rather than treating an HTTP success or the first accepted call as complete success. Retain successes even when other entries fail. For an entry with nonzero status, do not require success identifiers; expose a sanitized per-recipient failure. Reject missing/noninteger statuses or an empty/missing `calls` array as malformed success, and do not silently coerce unknown shapes.

## Submission handling

Validate required configuration and command fields before submitting. Use a finite request timeout appropriate to the application; the sources do not specify a required client timeout. A transport timeout does not establish whether Mocean accepted the call.

On malformed results after submission, retain available identifiers in the application's reconciliation path; do not resubmit the batch. A partial failure is an outcome to report, not a reason to redial successful recipients. The `mocean-event-url` setting alone is insufficient to implement final-status tracking: obtain a verified event payload contract first.

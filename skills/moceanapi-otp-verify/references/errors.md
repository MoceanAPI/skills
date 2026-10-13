# Verify errors and documentation limits

These application status codes apply to the three Verify operations. Keep HTTP status separate from provider `status`; these codes differ from ordinary SMS and Voice codes.

## Status codes

| Code | Meaning |
| --- | --- |
| `0` | OK. Send/resend accepted, or a code matched for check. |
| `1` | Throttled. |
| `2` | Authorization failed. |
| `3` | System error. |
| `4` | Missing brand. |
| `5` | Missing receiver. |
| `6` | Invalid destination. |
| `7` | Missing request ID. |
| `8` | Missing PIN code. |
| `9` | Brand too long. |
| `10` | Sender ID too long. |
| `11` | Unsupported code length. |
| `13` | Invalid PIN validity. |
| `14` | Invalid next event wait. |
| `15` | Retry limit reached. |
| `16` | Code mismatch. |
| `17` | Request not found. |
| `18` | Insufficient balance. |
| `19` | Concurrent request not allowed. |
| `20` | Cannot route request. |
| `21` | System error. |
| `22` | Unknown request. |
| `23` | Invalid service. |

The catalogue has no code `12`. Preserve unknown nonzero statuses as errors rather than guessing their meanings.

## Supplemental facts

The newer Verify document clarifies:

- `1`: Requests exceed the provider's allowed per-minute rate; no numeric rate is given.
- `2`: No request is sent to the receiver when authorization fails.
- `6`: The receiver is invalid after stripping nonnumeric characters.
- `13`: PIN validity must be 60–3600 seconds.
- `14`: Next event wait must be 60–900 seconds. This field is absent from all three request schemas; do not add it to generated requests.
- `15`: The maximum incorrect-code retry limit is three. This is a provider limit, not a prescribed local resend or check policy.
- `19`: Concurrent requests to the same number and brand are disallowed only for the pay-per-conversion API. Do not impose this as a documented restriction on these three endpoints.
- `23`: The requested service is not enabled on the account.

Brand and sender maximum lengths are not specified in these contracts. Do not invent numeric limits. Status `17` means request not found; it does not establish whether the cause is expiry, a wrong ID, or another provider lifecycle event.

## Application handling

- Inspect HTTP status and the operation's response shape. Missing, null, noninteger, or unknown `status` must never grant verification. HTTP failure is a failure even with a success-shaped body.
- Decode structured nonzero provider status when available, including on HTTP errors. Malformed JSON, HTML error pages, and missing bodies are transport/protocol failures; do not substitute successful values.
- A mismatch (`16`) remains pending within local limits. A retry-limit result (`15`) should stop further checks for that local challenge. A request-not-found result (`17`) should end the unusable local challenge or require a controlled restart. These are application choices, not undocumented provider guarantees.
- Handle throttling through the application's limits and safe retry UI. Credential, balance, routing, and disabled-service failures require configuration or operational handling. Do not report them as a matched OTP or expose raw provider errors to the user.
- Provider wording such as “try again” does not guarantee replay safety. Send/resend may incur charges or duplicate delivery after ambiguous failures. Do not automatically retry them. A local cooldown and counter must not disappear just because a response was lost.
- Keep sanitized provider status and application correlation IDs for diagnosis; omit tokens, OTPs, raw request IDs, full payloads, and unnecessary recipient details. Show safe errors appropriate to the product's account-enumeration policy.

## Sources and gaps

Contract snapshot checked on 2026-10-02 against the Mocean REST API documentation:

- Primary: `openapi.yaml`, the three supported `/rest/2/verify/...` paths, `VerifySmsRequest`, `VerifyResendRequest`, `VerifyCheckRequest`, their response schemas, `RespFormatProperty`, and `VerifyStatusCodes`.
- Supplemental: `new_doc/moceanapi_yaml_20260717/mocean-verify.yaml`, Mocean Verify API Document V3. It uses different operation IDs; the skill keeps the primary OpenAPI operation IDs. Its default, charging, retry, and error details are labelled as supplemental in these references.

The bundled references contain the needed contracts. Source filenames identify provenance, not runtime dependencies. The skill can be used without access to the original source files.

No code-retrieval, cancellation, idempotency key, or provider replay/consumption guarantee is documented. SMS resend does not document expiry refresh, new-code generation, or attempt-budget reset. Do not invent these behaviors. If one is essential to the requested feature, obtain authoritative documentation or a sanitized verified fixture before relying on it.

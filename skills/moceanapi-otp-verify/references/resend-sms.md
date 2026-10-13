# Resend SMS OTP REST contract

Use this reference for the REST operation `resendVerifyCodeSms`.

## Request

```http
POST https://rest.moceanapi.com/rest/2/verify/resend/sms
Authorization: Bearer <MOCEAN_API_TOKEN>
Content-Type: application/x-www-form-urlencoded
```

| Field | Required | Contract |
| --- | --- | --- |
| `mocean-reqid` | Yes | Request ID from the original SMS OTP send. |
| `mocean-resp-format` | No | `xml`, `json`, `XML`, or `JSON`. Defaults to XML; prefer `json`. |

Example form values, before encoding:

```json
{
  "mocean-reqid": "verify-request-example",
  "mocean-resp-format": "json"
}
```

Retrieve `mocean-reqid` from the server's active, owned SMS challenge. This endpoint does not accept a new recipient, brand, code length, or validity. Encode fields using the project's form encoder.

**Supplemental Verify documentation:** This operation resends an already generated OTP to the original number and is charged per attempt.

## Response

The documented result uses HTTP `202`. Example successful JSON:

```json
{
  "reqid": "verify-request-example",
  "status": 0,
  "to": "60123456789",
  "is_number_reachable": "reachable",
  "resend_number": "2"
}
```

| Field | Type | Meaning |
| --- | --- | --- |
| `reqid` | string | Provider verification request identifier. |
| `status` | integer | `0` is success; nonzero is a provider error. |
| `to` | string | Original recipient number. |
| `is_number_reachable` | string | Reachability metadata. |
| `resend_number` | string | Number of times the code has been resent; specific to this operation. |
| `err_msg` | string | Present only for nonzero status. |

Keep `resend_number` as a string at the API boundary; validate before numeric conversion if needed. It is not the application's check-attempt counter. HTTP acceptance does not complete verification.

**Application safeguards:** Check ownership, channel, lifecycle, expiry, cooldown, and resend limits before the request. Preserve the original challenge binding and validate returned identifiers when present. Do not silently replace it with an unrelated request or recipient. Maintain local counters and cooldowns even when a response omits `resend_number`.

The contract does not specify a new code, refreshed expiry, reset retry budget, or idempotency key. Do not extend the original local expiry or reset attempts on resend. Use [errors.md](errors.md) for failures; an ambiguous resend must not trigger an automatic repeat.

Source: Mocean REST API `openapi.yaml`, `VerifyResendRequest`, `VerifyRequestResponse`, and `RespFormatProperty`; supplemental `mocean-verify.yaml` (2026-07-17 documentation), Resend Code over SMS.

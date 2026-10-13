# Send SMS OTP REST contract

Use this reference for the REST operation `sendVerifyCodeSms`.

## Request

```http
POST https://rest.moceanapi.com/rest/2/verify/req/sms
Authorization: Bearer <MOCEAN_API_TOKEN>
Content-Type: application/x-www-form-urlencoded
```

Encode the fields with the project's form-encoding facility, not manual string concatenation. Request `mocean-resp-format=json`; the API defaults to XML.

| Field | Required | Contract |
| --- | --- | --- |
| `mocean-to` | Yes | Recipient phone number including country code, for example `60123456789`. Keep it as a string. |
| `mocean-brand` | Yes | Company or application name shown in the OTP message. |
| `mocean-from` | No | SMS sender ID. |
| `mocean-code-length` | No | Integer `4` or `6`. |
| `mocean-pin-validity` | No | Integer expiry duration in seconds, from `60` to `3600`. |
| `mocean-resp-format` | No | `xml`, `json`, `XML`, or `JSON`. Prefer `json`. |

Example form values, before encoding:

```json
{
  "mocean-to": "60123456789",
  "mocean-brand": "MyApp",
  "mocean-code-length": 6,
  "mocean-pin-validity": 300,
  "mocean-resp-format": "json"
}
```

**Supplemental Verify documentation:** When omitted, code length defaults to four or the account setting, and validity defaults to 300 seconds or the account setting. Requests are charged per attempt. These details supplement the OpenAPI schema; they are not additional fields.

**Application recommendation:** Reuse configured settings. For a new flow, explicitly request six digits and 300 seconds so the application does not depend on account defaults.

## Response

The documented result uses HTTP `202`. Example successful JSON:

```json
{
  "reqid": "verify-request-example",
  "status": 0,
  "to": "60123456789",
  "is_number_reachable": "reachable"
}
```

| Field | Type | Meaning |
| --- | --- | --- |
| `reqid` | string | Provider verification request identifier. |
| `status` | integer | `0` is success; nonzero is a provider error. |
| `to` | string | Recipient number. |
| `is_number_reachable` | string | Reachability metadata; the OpenAPI example is `unknown`. |
| `err_msg` | string | Present only for nonzero status. |

The OpenAPI specification shares this response schema with SMS resend; `resend_number` applies only to resend and is not returned by initial send. No OTP is returned. The schema does not mark response properties as required; the application still needs valid `status` and a nonempty `reqid` to establish a usable pending challenge.

HTTP acceptance and reachability do not establish delivery or verification. Store the successful `reqid` with the local challenge, then use [check-code.md](check-code.md) to check user input. If `to` is returned, validate it against the intended recipient rather than silently rebinding the challenge.

Use [errors.md](errors.md) for nonzero statuses, HTTP failures, and ambiguous outcomes. Do not automatically repeat an uncertain send.

Source: Mocean REST API `openapi.yaml`, `VerifySmsRequest`, `VerifyRequestResponse`, and `RespFormatProperty`; supplemental `mocean-verify.yaml` (2026-07-17 documentation), Send Code over SMS.

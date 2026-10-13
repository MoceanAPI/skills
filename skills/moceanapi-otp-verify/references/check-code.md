# Check OTP REST contract

Use this reference for the REST operation `checkVerifyCode`. It checks an OTP issued through SMS.

## Request

```http
POST https://rest.moceanapi.com/rest/2/verify/check
Authorization: Bearer <MOCEAN_API_TOKEN>
Content-Type: application/x-www-form-urlencoded
```

| Field | Required | Contract |
| --- | --- | --- |
| `mocean-reqid` | Yes | Provider request ID obtained from the original send. |
| `mocean-code` | Yes | User-submitted OTP as a string. Preserve leading zeros. |
| `mocean-resp-format` | No | `xml`, `json`, `XML`, or `JSON`. Defaults to XML; prefer `json`. |

Example form values, before encoding:

```json
{
  "mocean-reqid": "verify-request-example",
  "mocean-code": "012345",
  "mocean-resp-format": "json"
}
```

Resolve the request ID from the caller's owned, active local challenge rather than passing through an arbitrary client value. Enforce the challenge's expiry, purpose, and local check limit before the provider call. Form-encode the code as text; converting it to an integer changes `012345` to `12345`.

## Response

The documented result uses HTTP `202`. Example successful JSON:

```json
{
  "reqid": "verify-request-example",
  "status": 0
}
```

| Field | Type | Meaning |
| --- | --- | --- |
| `reqid` | string | Provider verification request identifier. |
| `status` | integer | `0` indicates a matched code; nonzero indicates an error. |
| `err_msg` | string | Present only for nonzero status. |

Response properties are not marked required in the OpenAPI schema. Require a valid integer `status`; do not substitute missing status with zero. Validate a returned `reqid` against the stored challenge when present. Do not require send-only recipient or reachability fields.

Example mismatch body:

```json
{
  "status": 16,
  "err_msg": "The code inserted does not match the expected value."
}
```

The newer Verify document also shows HTTP `400` for mismatch; the primary OpenAPI specification lists `202` response bodies. Parse a usable provider error body on HTTP failures when possible, but never interpret a non-success HTTP result as verification, even if its body says `status: 0`.

**Application safeguards:** Grant verification only after HTTP success, a valid response with `status: 0`, and an atomic local completion of the still-active challenge. Bind the resulting authorization to the stored purpose and recipient. A replay, expired challenge, different session, or concurrent losing completion must not repeat the protected action. These local safeguards do not imply that Mocean documents request consumption or replay rejection.

Use [errors.md](errors.md) for mismatch (`16`), retry limit (`15`), request not found (`17`), and other errors. On a timeout, leave the outcome uncertain and do not grant verification. Automatic retries can consume provider attempts or encounter undocumented consumption behavior; do not add them without a verified contract and application policy.

Source: Mocean REST API `openapi.yaml`, `VerifyCheckRequest`, `VerifyCheckResponse`, and `RespFormatProperty`; supplemental `mocean-verify.yaml` (2026-07-17 documentation), Verify Code.

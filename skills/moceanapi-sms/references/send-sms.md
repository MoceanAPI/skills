# Send SMS REST contract

Use this reference for the MCP operation `sendSms`.

## Request

```http
POST https://rest.moceanapi.com/rest/2/sms
Authorization: Bearer <MOCEAN_API_TOKEN>
Content-Type: application/x-www-form-urlencoded
```

Prefer `mocean-resp-format=json`. Encode the form with the target project's normal form-encoding facility; do not concatenate an unescaped body manually.

### Form fields

| Field | Required | Contract |
|---|---:|---|
| `mocean-from` | Yes | SMS sender ID displayed to the recipient. It must be approved where destination rules require approval. |
| `mocean-to` | Yes | Receiver number including country code. Separate multiple numbers with a comma or whitespace; maximum 500 receivers. |
| `mocean-text` | Yes | Message content. Every receiver in one request gets the same content. |
| `mocean-udh` | No | User Data Header for a binary message. |
| `mocean-coding` | No | `1` = 7-bit, `2` = 8-bit, `3` = UCS-2. If omitted, Mocean normally uses 7-bit, switches to 8-bit when UDH is present, and can select UCS-2 for non-GSM characters. |
| `mocean-dlr-mask` | No | `1` requests a delivery report; `0` disables it. Default is `0`. |
| `mocean-dlr-url` | Conditional | Public callback URL. Required when `mocean-dlr-mask=1`. |
| `mocean-schedule` | No | Future send time in `YYYY-MM-DD hh:mm:ss`, using Malaysia time (GMT+8). Leading zeros are required. |
| `mocean-mclass` | No | Set to `1` together with `mocean-alt-dcs=1` for Flash SMS. |
| `mocean-alt-dcs` | No | Set to `1` together with `mocean-mclass=1` for Flash SMS. |
| `mocean-charset` | No | `ISO-8859-1`, `ISO-8859-7`, `UTF-8`, or `Windows-1252`. |
| `mocean-validity` | No | Validity period in seconds. |
| `mocean-resp-format` | No | `json`/`JSON` or `xml`/`XML`; the API default is XML. Prefer `json`. |

Do not use a single multi-recipient request when each recipient needs different text. Create separate submissions or use the application's existing batch/outbox mechanism.

## Response

A successful submission is documented as HTTP `202 Accepted`. Acceptance is not final delivery.

```json
{
  "messages": [
    {
      "status": 0,
      "receiver": "60123456789",
      "msgid": "cust20013050311050614001"
    }
  ]
}
```

An error item has a nonzero `status` and can contain `receiver` and `err_msg`. Always inspect every `messages[]` item, even when the HTTP response is parseable and other recipients succeeded.

```json
{
  "messages": [
    {
      "status": 0,
      "receiver": "60123456789",
      "msgid": "cust20013050311050614001"
    },
    {
      "status": 28,
      "receiver": "invalid_number",
      "err_msg": "Invalid destination number"
    }
  ]
}
```

Return or persist a per-recipient outcome. Do not collapse a partial failure into a single success flag.

## Submission status codes

| Code | Meaning |
|---:|---|
| 0 | OK; accepted for this receiver. |
| 1 | Authorization failed. No SMS is sent to any receiver. |
| 2 | Insufficient balance. |
| 4 | Destination is not whitelisted. |
| 5 | Destination is blacklisted. |
| 6 | No destination specified. |
| 8 | Sender ID not found. |
| 9 | Invalid UDH. |
| 10 | Invalid message class. |
| 17 | Invalid validity value. |
| 19 | Invalid character set or message body. |
| 20 | Insufficient headers. |
| 23 | Empty message text. |
| 24 | Unknown error. |
| 26 | Invalid schedule format. |
| 27 | More than 500 receivers. |
| 28 | Invalid destination number. |
| 29 | Message body too long. |
| 32 | Message throttled. |
| 34 | Unknown request. |
| 37 | Invalid sender length. |
| 40 | System under maintenance. |
| 43 | SMS flooding detected. |
| 44 | Invalid sender ID. |
| 45 | System error. |
| 48 | Sender is blacklisted. |
| 49 | Sender is not whitelisted. |
| 50 | Inappropriate content detected. |
| 65 | Destination not found. |
| 69 | Unsupported character. |
| 70 | Invalid service. |
| 73 | Destination and sender are blacklisted. |

## Implementation invariants

- Validate required configuration and input before submitting.
- Store `msgid` only for an item whose `status` is `0`.
- Do not mark an SMS delivered from this response; use a delivery report or message-status query.
- Retry explicit throttling or temporary service errors only when the surrounding application has a duplicate-prevention strategy. Never blindly retry an ambiguous timeout.
- Do not log the bearer token or full message body. Mask recipient numbers when full values are not operationally necessary.

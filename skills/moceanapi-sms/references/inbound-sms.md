# Inbound SMS contract

Use this reference for `receiveInboundSms`.

Implement a public route on the user's server. The URL and path should use the current system URL and current path. If either is unclear, prompt the user for clarification.

```http
POST https://your-server.example/webhooks/inbound-sms
Content-Type: application/x-www-form-urlencoded
```


| Field            | Type    | Meaning                                                         |
| ---------------- | ------- | --------------------------------------------------------------- |
| `mocean-from`    | string  | Sender phone number.                                            |
| `mocean-to`      | string  | Receiving number or short code.                                 |
| `mocean-time`    | string  | Gateway timestamp, for example `2018-12-30 23:59:59`.           |
| `mocean-text`    | string  | Received message text.                                          |
| `mocean-keyword` | string  | Matched inbound keyword.                                        |
| `mocean-coding`  | integer | `1` for text/mobile-telephony character set or `3` for Unicode. |


Validate and normalize the payload without destroying original values needed for auditing. Persist the event using the best stable identity available in the application. Treat the message text and phone numbers as untrusted input and sensitive data.
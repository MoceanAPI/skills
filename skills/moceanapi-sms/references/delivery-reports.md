# SMS delivery report contract

Use this reference for `receiveSmsDeliveryReport`.

Implement a public route on the user's server. The path is configurable; this is the example used by the MCP contract:

```http
PUT https://your-server.example/webhooks/delivery-report
Content-Type: application/x-www-form-urlencoded
```

Set this URL as `mocean-dlr-url` and send `mocean-dlr-mask=1` in the original SMS request.

| Field | Type | Meaning |
|---|---|---|
| `mocean-from` | string | Phone number of the original SMS recipient. |
| `mocean-to` | string | Phone number of the original SMS sender. |
| `mocean-dlr-status` | integer | `1` success, `2` failed, `3` expired. |
| `mocean-msgid` | string | Message ID returned by the original submission. |
| `mocean-error-code` | string | Delivery error code. Keep it as a string so leading zeros are preserved. |

Find the local record by `mocean-msgid`. Store the event idempotently and do not let an older or duplicate callback corrupt a terminal state.

## Delivery error codes

| Code | Meaning | Code | Meaning |
|---|---|---|---|
| `000` | No error | `001` | Absent subscriber |
| `002` | Handset memory capacity exceeded | `003` | Equipment protocol error |
| `004` | Equipment lacks SMS capability | `005` | Unknown subscriber |
| `006` | Illegal subscriber | `007` | SMS teleservice not provisioned |
| `008` | Illegal equipment | `009` | Call barred |
| `010` | Facility not supported | `011` | Subscriber busy for mobile-terminated SMS |
| `012` | System failure | `013` | Required data missing |
| `014` | Unexpected data value | `015` | Unidentified subscriber |
| `016` | Absent subscriber; no paging response | `017` | Absent subscriber; IMSI detached |
| `018` | Absent subscriber; roaming restriction | `019` | Subscriber busy for USSD |
| `047` | SMSC SS7 error | `049` | SMS discarded |
| `050` | SMS rejected | `051` | Malformed SMS |
| `052` | SMS expired | `053` | Insufficient credit or destination not allowed |
| `054` | Invalid destination | `055` | No outbound route |
| `056` | SMS buffered | `057` | Peer response timeout |
| `058` | Throttled | `059` | Suspected spam |
| `060` | Invalid service keyword | `061` | Subscriber blacklisted |
| `062` | Subscriber not whitelisted | `063` | Invalid sender length |
| `065` | System under maintenance | `066` | Invalid data coding |
| `067` | Invalid USSD session ID | `068` | SMS flooding detected |
| `069` | Invalid sender ID | `070` | Invalid credentials |
| `071` | Subscriber opted out | `072` | Invalid MNC/MCC |
| `073` | Invalid mobile-originated ID | `074` | Rejected by peer SMSC |
| `075` | Inappropriate content | `076` | Sender ID blacklisted |
| `077` | Sender ID not whitelisted | `078` | Pricing information not found |
| `079` | Multiple charging packages | `080` | Invalid message ID |
| `081` | Charging failed | `091` | Invalid schedule date |
| `094` | URL blacklisted | `095` | URL not whitelisted |
| `096` | Expired without final carrier status | `100` | Source account blacklisted |
| `101` | Destination account blacklisted | `102` | Source account not whitelisted |
| `103` | Destination account not whitelisted | `121` | Invalid service |
| `134` | Invalid type | `137` | Subscriber and sender blacklisted |
| `155` | Invalid receiver | `159` | Invalid URL |
| `160` | Traffic volume limit reached | `255` | Unknown error |

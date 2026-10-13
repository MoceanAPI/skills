# SMS pricing contract

Use this reference for `getAccountPricing`.

```http
GET https://rest.moceanapi.com/rest/2/account/pricing?mocean-type=sms&mocean-resp-format=json
```

| Query parameter | Required | Contract |
|---|---:|---|
| `mocean-type` | No | Set explicitly to `sms` for this skill. SMS is also the documented default. |
| `mocean-mcc` | No | Mobile Country Code. If used, `mocean-mnc` is also required. |
| `mocean-mnc` | No | Mobile Network Code. If used, `mocean-mcc` is also required. |
| `mocean-resp-format` | No | Prefer `json`; XML is also supported. |
| `mocean-delimiter` | CSV only | `;`, `:`, or `|` for CSV output. |

A successful JSON result contains `destinations[]` entries with `country`, `operator`, `mcc`, `mnc`, `price`, and `currency`.

Pricing status codes are `0` OK, `1` authorization failed, `3` no rate found, `24` unknown error, `32` throttled, `40` system down, `45` system error, `46` invalid MCC/MNC, `51` invalid delimiter, `52` multiple charging packages, and `72` invalid type.

Treat prices as strings or decimals rather than binary floating-point values when calculating. Do not assume cached pricing remains current indefinitely.

# SMS message status contract

Use this reference for `getMessageStatus`.

```http
GET https://rest.moceanapi.com/rest/2/report/message?mocean-msgid=<MSG_ID>&mocean-resp-format=json
```

`mocean-msgid` is required. The response contains:

- `status`: API operation status; `0` means the query succeeded.
- `message_status`: `1` success, `2` failed, `3` expired, `4` pending, `5` not found.
- `msgid`: queried message ID.
- `credit_deducted`: credit information returned by Mocean.
- `err_msg`: present for a nonzero operation status.

Operation status codes are `0` OK, `1` authorization failed, `24` unknown error, `25` invalid message ID, `32` throttled, `40` system down, and `45` system error. Do not read `message_status` as valid when `status` is nonzero.

Build the query string with the project's URL facility. Keep the returned `msgid` linked to the relevant local SMS record.

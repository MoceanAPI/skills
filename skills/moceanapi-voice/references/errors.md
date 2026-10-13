# Voice API errors

These codes apply to dial, hangup, and recording operations. Keep HTTP status separate from application status. Unknown nonzero codes are errors, too.

Any non-success HTTP response is a failure regardless of its body. Some documented 401 examples use a `messages` array instead of the voice schema’s top-level `status`; do not treat `messages` as a voice success response.


| Code | Meaning                                                |
| ---- | ------------------------------------------------------ |
| 0    | OK. For dial, accepted rather than answered.           |
| 1    | Authorization failed; no call made to any recipient.   |
| 2    | Insufficient balance to call recipients.               |
| 3    | No rate for destination.                               |
| 4    | At least one destination is not whitelisted.           |
| 5    | At least one destination is blacklisted.               |
| 6    | No destination specified.                              |
| 7    | Invalid call UUID.                                     |
| 8    | Too many recipients. No numeric maximum is given here. |
| 9    | Invalid request.                                       |
| 10   | System error.                                          |
| 11   | Destination not found.                                 |
| 12   | Missing MC input or parameter.                         |
| 13   | MC syntax error.                                       |
| 14   | Unknown error.                                         |
| 15   | Maintenance.                                           |


The wording for code 10 suggests retrying, but does not establish that replay cannot duplicate a call. Do not implement unconditional dial retries, including on ambiguous transport or server failures.

# Mocean Command (MC)

MC is an ordered JSON array of actions. The call flow runs when the call is answered; the call ends when its actions finish. Submit the array as a JSON string inside the dial form. Webhooks return the array directly as JSON, without a wrapper or form encoding.

The table below lists the required fields for each action. The generic OpenAPI schema requires only `action`, but the documented per-action parameters are also required.

| Action | Required fields besides `action` | Optional fields |
| --- | --- | --- |
| `say` | `language`: string; `text`: string | `barge-in`: boolean; `clear-digit-cache`: boolean |
| `play` | `file`: audio URL string or array of URL strings | `barge-in`: boolean; `clear-digit-cache`: boolean |
| `sleep` | `duration`: integer milliseconds | None documented |
| `collect` | `event-url`: callback URL string; `min`: number; `max`: number; `timeout`: number of milliseconds to wait for each digit | `terminators`: string |
| `dial` | `to`: string containing a single phone number | `from`: caller ID string; `dial-sequentially`: boolean (default false) |
| `record` | None | No settings documented |

For `say` and `play`, `barge-in` lets a digit interrupt the audio and enter the digit cache; `clear-digit-cache` clears entered digits. `terminators` ends input when fewer than the minimum digits have been entered, according to the source wording. Do not assert further terminator behavior or undocumented defaults.

Documented `say.language` values: `en-US`, `en-GB`, `de-DE`, `de-AT`, `de-CH`, `cmn-CN`, `ja-JP`, `ko-KR`, `ms-MY`, `id-ID`, `th-TH`, `vi-VN`, `ru-RU`, `zh-CN`, `zh-TW`, `zh-HK`. Do not invent voice names or speech model settings.

Use positive integer digit counts with `min <= max` and a positive timeout in examples as application validation choices. The sources do not specify numeric bounds or defaults. Use publicly reachable audio and callback URLs. Supported audio formats, file-size limits, and URL-fetch authentication are not specified here.

## Speech, pause, and audio

```json
[
  {"action":"say","language":"en-GB","text":"Your appointment is confirmed."},
  {"action":"sleep","duration":1000},
  {"action":"play","file":"https://example.com/audio/goodbye.mp3"}
]
```

Replace example URLs with actual hosted assets before live use.

## Keypad menu

```json
[
  {"action":"say","language":"en-GB","text":"For support, press 1.","barge-in":true,"clear-digit-cache":true},
  {"action":"collect","event-url":"https://example.com/webhooks/mocean/collect","min":1,"max":1,"timeout":10000}
]
```

Make collect the last action in this example so the callback determines the next actions without assuming how a returned array interacts with trailing actions. See [webhooks.md](webhooks.md) for digit branching and timeout handling.

## Transfer to one number

```json
[{"action":"dial","to":"60123456789","dial-sequentially":false}]
```

The command `dial` connects a phone number within the current flow; the REST `/voice/dial` operation initiates outbound calls. For the MC `dial` action, accept exactly one phone number as a string in `to`; reject arrays and comma- or whitespace-separated lists. Allow the optional `dial-sequentially` field to be set to `true` or `false`; its default is `false`. Preserve an explicitly supplied value. The flag does not expand the supported destination count.

## Record subsequent call audio

```json
[
  {"action":"say","language":"en-GB","text":"This call will be recorded."},
  {"action":"record"},
  {"action":"say","language":"en-GB","text":"Please tell us how we can help."}
]
```

`record` starts recording after that action; it is not a speech-recognition action. Do not add an invented recording duration, format, or transcription field. This example does not promise a timed voicemail capture: no wait-for-speech or recording-stop behavior is specified. Retrieve the recording using the call UUID and [call-control.md](call-control.md).

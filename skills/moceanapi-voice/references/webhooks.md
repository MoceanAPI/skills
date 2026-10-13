# Inbound and keypad callbacks

These are routes hosted by the user's application, not endpoints on `rest.moceanapi.com`. Inbound calling requires a rented Mocean virtual number and configured answer URL. A collect action supplies its own `event-url`. Use public HTTPS URLs for deployed handlers.

| Operation | Request fields |
| --- | --- |
| `receiveInboundCall` | `mocean-session-uuid`, `mocean-call-uuid`, `mocean-to` (called virtual number), `mocean-from` (caller) |
| `receiveCollectAction` | `mocean-session-uuid`, `mocean-call-uuid`, `mocean-digits` |

Preserve these values as strings, including leading zeros in phone numbers and digits. Use the documented `mocean-*` field names; some examples use unprefixed aliases, but they are not established by the parameter contract. An empty `mocean-digits` string means no input before timeout. Treat a missing or malformed field as malformed input, not automatically as a timeout.

## Transport and response defaults

The OpenAPI contract describes POST form data; prose examples also use GET query parameters. Support POST `application/x-www-form-urlencoded` and GET query input for answer and collect routes unless a verified deployment contract narrows the method. Parse only the selected method's input and reject duplicate/non-string values for fields used by the handler; do not merge query and body values. JSON request bodies are not the baseline contract.

Return HTTP `200`, `Content-Type: application/json`, and a raw MC array. The prose and wire examples use `200`; the OpenAPI response table says `202`. This is an explicit example-based default, not a verified exclusive provider requirement. Both sources agree that the body must contain MC.

Do not install bearer-token middleware on these callbacks merely because the OpenAPI paths declare `ApiTokenAuth`: the sources do not establish that Mocean sends your API token to your server. No signature header or validation algorithm is documented. Apply existing HTTPS, request-size, and access controls without claiming they authenticate Mocean. Verify a provider-supported mechanism before relying on callback authenticity for privileged actions.

## Call-flow response behavior

For inbound calls, return the application's initial MC array. For collect callbacks, select the next MC array from the exact `mocean-digits` string and the application's current menu state.

| Input condition | Application behavior |
| --- | --- |
| Recognized selection | Return the configured next actions, such as `dial` to a configured support number. |
| Empty string | Handle no input before timeout, for example with a short `say` message or a bounded menu retry. |
| Unrecognized selection | Return the application's invalid-selection flow. |
| Missing or malformed input | Use the application's validation/fallback behavior; do not interpret it as a documented timeout. |

These branches are application choices, not provider-defined menu rules. Transfer destinations come from trusted application configuration, not arbitrary callback values. Return a raw MC array with no application-specific response wrapper. See [commands.md](commands.md) for action fields and JSON payload examples.

For malformed input or a failed downstream dependency, a handler can return a configured short `say` message that ends the flow. This fallback is application policy, not a provider error contract. Keep callbacks fast and do not depend on slow work to produce the next commands.

If a callback updates business state, make those effects idempotent. Repeated legitimate collect steps can share a call UUID and digits, so those values alone are not a unique event key. Use the application's menu-step state to distinguish retries from new interactions, and make replay responses consistent. Retry timing and provider event IDs are unspecified; do not fabricate them.

The outbound `mocean-event-url` status callback is distinct from collect `event-url`. Its payload, method, and response requirements are absent from these voice sources. Do not reuse these payload definitions or borrow SMS/WhatsApp event fields for it.

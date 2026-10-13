---
name: moceanapi-sms 
description: Implement, modify, debug, and test Mocean SMS REST API integrations, including outbound messaging, inbound SMS, delivery reports, message status tracking, SMS pricing, and webhooks. Use for SMS-related coding in Mocean projects, even without explicit provider mention.
---

# MoceanAPI SMS

Build SMS integrations from the REST contracts in this skill. Let the target project's language, framework, and existing conventions determine the code structure.

## Route the request

Read only the reference for the requested operation:

- Outbound SMS (`sendSms`): [references/send-sms.md](references/send-sms.md)
- Delivery reports (`receiveSmsDeliveryReport`): [references/delivery-reports.md](references/delivery-reports.md)
- Inbound SMS (`receiveInboundSms`): [references/inbound-sms.md](references/inbound-sms.md)
- Message status (`getMessageStatus`): [references/message-status.md](references/message-status.md)
- SMS pricing (`getAccountPricing`): [references/sms-pricing.md](references/sms-pricing.md)

For a feature spanning several operations, read each applicable reference.

## Build the integration

1. Inspect the target repository before choosing files, abstractions, HTTP facilities, routes, persistence, configuration, and tests.
2. Reuse the project's existing HTTP client and application patterns. Generate direct REST calls; do not suggest or add a Mocean SDK.
3. Keep `MOCEAN_API_TOKEN` on a trusted server. Never embed it in browser or mobile code, commit it, print it, or return it to clients.
4. For Mocean API calls, send `Authorization: Bearer <token>` and set `mocean-resp-format=json` when that operation documents the parameter, unless the user requires another documented format.
5. Treat webhook operations as routes hosted by the user's application. Do not send requests to the example webhook URLs as though they were Mocean API endpoints.
6. Parse webhook payloads as form data. The MCP contract documents no webhook authentication scheme, so do not invent or claim signature verification. Use the application's HTTPS, request-size, rate-limit, and network controls.
7. Persist webhook events idempotently, return HTTP `200` after safe acceptance, and queue slow downstream work.
8. Validate both the HTTP result and the documented application-level `status`. HTTP `202` means accepted, not delivered.
9. Preserve each successful SMS `msgid` and associate it with the local message and recipient so later status and delivery-report operations can use it.
10. For multi-recipient sends, process every item in `messages`; the request can contain both successful and failed recipients.
11. Do not automatically retry an ambiguous send failure such as a timeout after the request may have reached Mocean. A retry can send a duplicate SMS. Use the application's outbox or deduplication design when reliable retry behavior is required.
12. Log operational context without tokens, full message bodies, or unnecessary full phone numbers.



## Authorization boundary

Creating or editing integration code does not authorize a live SMS, status query, or pricing query. Make a live request only when the user explicitly asks for it and has supplied the required runtime configuration.

## Verify the result

- Add or update tests using the project's normal test tools.
- Cover success, documented nonzero status, malformed response, and transport failure for API clients.
- Cover form parsing, idempotent persistence, and HTTP `200` acknowledgement for webhooks.
- When multiple recipients are supported, test a response containing both success and failure entries.



## Boundaries

This skill covers only the five operations listed above. Exclude the Conversion API and account-balance operation. Exclude Verify/OTP operations even when their delivery channel is SMS. Exclude WhatsApp, voice, number lookup, and all SDK usage.
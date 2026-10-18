# Contract C — fan-out to named subscribers (`webhook_*`)

The one place a third event shape is justified, because neither `http_outbound_*` nor `import_*`
carries *who we sent it to*.

| event | extra keys | severity |
|---|---|---|
| `webhook_published` | `webhook_event`, `subscriptions` (fan-out size), `payload_bytes` | info |
| `webhook_delivery_sent` | `worker`, `subscription_id`, `owner`, `webhook_event`, `target_host` | info |
| `webhook_delivery_failed` | the above + `consecutive_failures`, `error_class` | error |
| `webhook_subscription_disabled` | same keys as `webhook_delivery_sent` | warn |

The load-bearing key is **`owner`**: a stable subscriber name, not the URL. A panel grouped on the
address breaks the moment a subscriber is re-registered elsewhere.

Never log the subscriber's full `target_url` (it often carries a token or signature in the query
string) nor the `payload` content (it carries customer data): log `target_host` and
`payload_bytes`. Rules: `/observability`, section 2.

The same delivery already emits `http_outbound_*` events (contract A) for the transport view;
don't duplicate status/duration here. If you add a fan-out of your own, reuse these names.

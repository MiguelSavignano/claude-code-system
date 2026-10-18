# Contract C — fan-out to named subscribers (`webhook_*`)

The one place a third event shape is justified, because neither the `http_outbound_*` nor
the `import_*` contract carries *who we sent it to*.

Reference implementation: e.g. a `Webhooks::DeliveryWorker` / `Webhooks::Publish` pair in your
reference repo, documented in its external-services doc.

| event | extra keys | severity |
|---|---|---|
| `webhook_published` | `webhook_event`, `subscriptions` (fan-out size), `payload` | info |
| `webhook_delivery_sent` | `worker`, `subscription_id`, `owner`, `webhook_event`, `target_url` | info |
| `webhook_delivery_failed` | the above + `consecutive_failures`, `error_class` | error |
| `webhook_subscription_disabled` | same keys as `webhook_delivery_sent` | warn |

The load-bearing key is **`owner`** — a stable subscriber name, not the URL. A panel grouped
on `target_url` breaks the moment a subscriber is re-registered at a new address.

The same delivery already emits `http_outbound_*` events (contract A) if you want the
transport view — don't duplicate status/duration here.

If you add a fan-out of your own, reuse these names.

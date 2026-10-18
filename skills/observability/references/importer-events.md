# Contract B — batch jobs / importers / cronjobs (`import_run_*` + `import_item_*`)

Applies to **any** loop-over-resources job (imports, scheduled fan-outs, report generation), not
just literal importers; that's why the grouping key is the neutral `pipeline`.

## The two levels

1. **Run level** (the job entrypoint): fetch the work, then log started / finished / failed.
2. **Item level** (inside the loop, one DB transaction per item): commit → `import_item_finished`
   with a `status`; rollback → `import_item_failed` with the error.

Log the item event **after** the transaction resolves, so the log matches DB truth. Never log
success before commit.

| event | extra keys | when |
|---|---|---|
| `import_run_started` | `worker`, `fetched` (rows about to be looped), options | after fetch, before the loop |
| `import_run_finished` | `fetched`, `duration_ms` | after the loop |
| `import_run_failed` | `error_class`, `error_message`, `duration_ms` | run-level crash (e.g. fetch failed) |
| `import_item_finished` | resource ids, `status` (`processed`/`skipped_x`/…), created/updated counts | one row committed |
| `import_item_failed` | resource ids, `error_class`, `error_message` | one row rolled back |

All events share `event`, `pipeline` (stable per job type, e.g. `"orders"`) and `run_id` (one UUID
per run).

- **`run_id` is not `trace_id`.** `trace_id`/`span_id` come from the active span, like on every
  other line (`/observability`, section 4). To find a run's spans in Tempo, also set `run_id` as a
  span attribute on the run's root span.
- **Resource ids, not resource content.** No names, emails or row payloads (`/observability`,
  section 2). `error_message` only if the error cannot echo row data (validation and DB errors
  often do); otherwise `error_class` alone.
- `status` on `import_item_finished` is for **expected, non-exceptional** outcomes (not found,
  wrong type…); `import_item_failed` is exceptions only. Mixing them turns the error panel into noise.

## Wiring

Write the helper once per stack (a Ruby concern, a Python context manager…) that opens the run,
generates `run_id`, emits the run events and exposes `log_item_finished` / `log_item_failed`. Keep
the current `run_id` in request/context-local storage so code that doesn't own the run (a sync
step called from another pipeline) can log items under it without threading it through
signatures. Each job then only declares its `pipeline` and wraps its loop.

No helper in a stack yet: emit the same names and keys by hand through the app's JSON logger.
Identical names across languages let one panel group by `pipeline` across apps.

## The gap you may be asked to fix, not work around

Jobs outside this contract often log free text (`'[STATUS] Starting …'`) that LogQL can't filter
by field. If you're touching one, prefer migrating it onto this contract over adding another
free-text tag, but ask first if the migration is bigger than the task at hand.

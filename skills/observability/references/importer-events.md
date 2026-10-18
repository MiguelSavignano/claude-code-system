# Contract B — batch jobs / importers / cronjobs (`import_run_*` + `import_item_*`)

Reference implementation: e.g. `app/workers/importers/run_observability.rb` +
`docs/importer-observability.md` in your reference repo.

Applies to **any** loop-over-resources batch job — imports, scheduled fan-outs, report
generation — not just literal "importers". That's why the label key is the neutral
`pipeline`.

## The two levels

1. **Run level** (the worker/job entrypoint): fetch the work, then log started / finished / failed.
2. **Item level** (inside the loop, one DB transaction per item): commit → `import_item_finished`
   with a `status`; rollback → `import_item_failed` with the error.

Log the item event **after** the transaction resolves, so the log matches DB truth. Never
log success before commit.

| event | extra keys | when |
|---|---|---|
| `import_run_started` | `worker`, `fetched` (rows about to be looped), options | after fetch, before the loop |
| `import_run_finished` | `fetched`, `duration_ms` | after the loop |
| `import_run_failed` | `error_class`, `error_message`, `duration_ms` | run-level crash (e.g. fetch failed) |
| `import_item_finished` | resource identifiers, `status` (`processed`/`skipped_x`/…), created/updated counts | one row committed |
| `import_item_failed` | resource identifiers, `error_class`, `error_message` | one row rolled back |

All events share `event`, `pipeline` (stable per job type, e.g. `"orders"`) and `run_id`
(one UUID per run — reuse it as the request `trace_id` so outbound HTTP calls made during
the run correlate).

`status` on `import_item_finished` is for **expected, non-exceptional** outcomes only (not
found, wrong type, …). `import_item_failed` is exceptions only. Mixing the two turns the
error panel into noise.

## Ruby (Sidekiq)

Write the concern once (e.g. `Importers::RunObservability`), then in each worker include it,
define `pipeline_name`, and wrap the run in `with_import_run`:

```ruby
class MyImportWorker
  include Sidekiq::Worker
  include Importers::RunObservability

  def perform
    with_import_run do |run|
      rows = fetch_rows
      run[:fetched] = rows.size
      rows.each { |r| import_one(r) }
    end
  end

  private

  def pipeline_name = 'my_pipeline'

  def import_one(row)
    status = ActiveRecord::Base.transaction(requires_new: true) { do_import(row) }
    Rails.logger.info(item_fields(row).merge(event: 'import_item_finished', status: status).to_json)
  rescue StandardError => e
    Rails.logger.error(item_fields(row).merge(event: 'import_item_failed',
                                               error_class: e.class.name, error_message: e.message).to_json)
  end

  def item_fields(row) = { pipeline: pipeline_name, importer: self.class.name, order_id: row.id }
end
```

## Python (Django / Celery)

Port the same helper (e.g. `core/run_observability.py`). Same event names and shape,
contextvar-based `run_id` propagation instead of `RequestStore`:

```python
from core.run_observability import ImportRun

class MyImportWorker:
    def run(self):
        with ImportRun("my_pipeline", worker=self.__class__.__name__) as run:
            rows = fetch_rows()
            run.set(fetched=len(rows))
            for row in rows:
                self._import_one(run, row)

    def _import_one(self, run, row):
        try:
            status = do_import(row)
        except Exception as exc:
            run.log_item_failed(exc, order_id=row.id)
        else:
            run.log_item_finished(status=status, order_id=row.id)
```

Code that doesn't own the run but still logs items under it (a sync step called from inside
another pipeline's run, e.g. a `DocumentSync` called from an `AccountProvisioner`) uses
module-level `log_item_finished` / `log_item_failed` functions — they pick up whichever `run_id`
is active via `current_run_id()` without threading it through call signatures. Document the full
contract in that repo (e.g. `docs/observability/run-observability.md`).

## Other stacks

No port yet — emit the same event names and keys by hand through the app's JSON logger.
Identical event names across languages is what lets one Grafana panel group by `pipeline`
across apps later.

## The gap you may be asked to fix, not work around

Workers outside this contract often have **no consistent job/worker status label** — e.g. ad-hoc
`Rails.logger.tagged(['SIDEKIQ', WorkerName]) { '[STATUS] Starting …' }` free text, which LogQL
can't structure-filter. If you're touching one of those workers, prefer migrating it onto this
contract over adding another free-text tag — but ask first if the migration is bigger than the
task at hand.

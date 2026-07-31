# Model-Scoped Quota Routing and Spark Meter

## Problem

The TUI treats the backward-compatible `limit_id = "codex"` rate-limit
snapshot as one account-global bucket. In a multi-agent session, different
models can emit that same default identifier for different quota pools. A
background Spark child can therefore replace the foreground Sol snapshot in
the shared `codex` slot, even though the foreground model never changed.

The status line also has no independent, compact way to display the Spark
pool. Discarding background Spark updates would hide the contamination but
would also throw away useful quota information.

## User-Visible Contract

- `quota-summary` continues to display the quota associated with the
  foreground model.
- A rolling update from a different model must not replace the foreground
  model's default `codex` snapshot.
- A rolling update from a child using the same model may update the foreground
  meter because it consumes the same pool.
- A new default-off status-line item, `spark-quota-summary`, displays the
  latest Spark quota as `sp:<remaining>%(<countdown>)` when a single window is
  available. For the observed weekly-only pool this renders, for example,
  `sp:82%(5d12h)`.
- The Spark item is toggleable through the existing `/statusline` picker and
  through the ordered `tui.status_line` configuration list.
- When no Spark snapshot is available, the item is omitted rather than
  displaying guessed or stale placeholder data.
- Explicitly named rate-limit buckets and existing spend-control or hard-stop
  handling retain their current behavior.

## Design

### Carry update provenance

Add an optional source-model slug to the internal token-count event. Extend the
v2 `account/rateLimits/updated` notification with optional source-thread and
source-model fields.

The core turn context supplies the model slug. App-server copies that slug and
adds the conversation ID that it already receives while translating
token-count events. The fields remain optional so older persisted events and
clients retain their existing behavior. The generated app-server schemas and
app-server API documentation must be updated with the additive fields.

### Preserve model-scoped default snapshots

Keep the existing per-`limit_id` map for account and named-bucket surfaces.
Add model-scoped storage for default `codex` rolling snapshots. When a default
snapshot has source-model metadata:

1. cache it under the normalized source model;
2. replace the foreground `codex` display only when the source thread is the
   active thread or the source model matches the foreground model;
3. otherwise retain the model-scoped snapshot without replacing the
   foreground display.

Updates without source metadata follow the existing merge path for backward
compatibility. Non-default `limit_id` snapshots continue to merge globally.

This rule lets same-model children contribute current usage while preventing
a different-model child from contaminating the foreground meter.

### Resolve the Spark snapshot

The Spark meter prefers a model-scoped rolling snapshot whose normalized
source-model slug contains the `spark` model token. A full account read may
also seed model-scoped storage when a named bucket's user-facing `limit_name`
normalizes to a Spark model label.

The implementation must not depend on the opaque `codex_bengalfox` identifier.
Track the most recently updated Spark model key alongside the model-scoped map;
if more than one Spark-labelled snapshot exists, that key selects the meter.

### Status-line integration

Add `SparkQuotaSummary` to the existing status-line item enum, preview data,
interactive picker, theme-color classification, and value dispatch. It is not
added to the default item list, so existing configurations are unchanged.

The display reuses the existing countdown and remaining-percentage rules. The
single-window compact form intentionally uses the approved `sp:` prefix
instead of repeating the window label. If a future Spark response supplies
multiple windows, prefix the existing duration-aware summary as
`sp:5h:87%(2h1m) wk:50%(5d22h)` so the values remain distinguishable.

## Compatibility and Failure Handling

- App-server fields are additive and nullable on the wire.
- Missing provenance preserves current behavior instead of dropping updates.
- Unknown or non-Spark named buckets remain available to `/status` and do not
  appear in the Spark meter.
- Different-model updates still participate in existing account hard-stop and
  spend-control handling; only their foreground display projection is
  isolated.
- No model routing, agent-role configuration, quota arithmetic, or duration
  classification changes are part of this work.

## Test Strategy

Use RED-first tests for:

1. Sol foreground at 58% used remains `wk:42%` after a Spark child reports 18%
   used under default `codex`.
2. A same-model child may update the foreground meter.
3. A different-model child remains available through
   `spark-quota-summary` as `sp:82%(5d12h)`.
4. The Spark item is omitted before Spark telemetry exists.
5. A named Spark bucket from a full account read can seed the Spark meter
   without relying on its opaque limit ID.
6. Missing source metadata preserves the legacy merge behavior.
7. Protocol serialization carries the optional source thread and source model.
8. The `/statusline` picker and preview include the default-off Spark item,
   with snapshot coverage for the new user-visible output.

## Verification

Regenerate app-server schemas, update the app-server rate-limit documentation,
run formatting and scoped tests for protocol, app-server, core, and TUI, review
and accept only intentional TUI snapshots, and run the complete `just test`
suite because shared protocol/core code changes. Build and publish the patched
binary only after the tree, schemas, tests, and branch ancestry are verified.

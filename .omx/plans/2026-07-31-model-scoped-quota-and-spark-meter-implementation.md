# Model-Scoped Quota and Spark Meter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `executing-plans` to implement this plan task by task.

**Goal:** Prevent background Spark quota updates from replacing the foreground Codex quota display and add a separately toggleable compact Spark quota meter named `spark-quota-summary`.

**Architecture:** Carry optional source model and source thread provenance with sparse rolling rate-limit events. Preserve the existing global `limit_id` snapshot map for account-wide and named limits, add a model-scoped snapshot map for rolling `codex` snapshots, and project a rolling snapshot into the foreground `codex` entry only when its source thread or source model matches the foreground session. Render the optional Spark meter from the most recently observed model-scoped Spark snapshot.

**Tech Stack:** Rust, Serde/JSON Schema/TypeScript schema generation, Ratatui TUI, Insta snapshots, Cargo/Just.

---

## Task 1: Lock the wire-provenance contract with failing tests

**Files:**
- Modify: `codex-rs/app-server/src/outgoing_message.rs`
- Modify: `codex-rs/app-server/src/bespoke_event_handling.rs`
- Modify: `codex-rs/app-server/tests/common/rollout.rs`
- Modify: `codex-rs/app-server/tests/suite/v2/thread_resume.rs`
- Modify: `codex-rs/app-server/src/request_processors/token_usage_replay.rs`
- Modify: `codex-rs/core/src/session/tests.rs`

- [ ] Extend the outgoing-notification serialization test to expect:

```json
{
  "method": "account/rateLimits/updated",
  "params": {
    "rateLimits": { "...": "..." },
    "sourceThreadId": "thread-123",
    "sourceModel": "gpt-5.3-codex-spark"
  }
}
```

- [ ] Extend the bespoke token-count forwarding test so a `TokenCountEvent` carrying `source_model: Some("gpt-5.3-codex-spark")` must produce a notification with the same model and the active conversation ID.
- [ ] Add `source_model: None` to unrelated `TokenCountEvent` fixtures so their intended legacy behavior remains explicit.
- [ ] Run:

```bash
CARGO_TARGET_DIR=/home/xthe/.cache/codex-target just test -p codex-app-server outgoing_message
CARGO_TARGET_DIR=/home/xthe/.cache/codex-target just test -p codex-app-server token_count
```

Expected: RED because `TokenCountEvent` and `AccountRateLimitsUpdatedNotification` do not yet expose provenance.

## Task 2: Carry provenance through core and app-server protocol

**Files:**
- Modify: `codex-rs/protocol/src/protocol.rs`
- Modify: `codex-rs/core/src/session/mod.rs`
- Modify: `codex-rs/app-server-protocol/src/protocol/v2/account.rs`
- Modify: `codex-rs/app-server/src/bespoke_event_handling.rs`
- Modify: `codex-rs/app-server/src/outgoing_message.rs`
- Modify: `codex-rs/app-server/README.md`

- [ ] Add an optional source model to the internal event:

```rust
pub struct TokenCountEvent {
    pub info: Option<TokenUsageInfo>,
    pub rate_limits: Option<RateLimitSnapshot>,
    pub source_model: Option<String>,
}
```

- [ ] Populate `source_model` from `turn_context.model_info.slug` when core emits a token-count event.
- [ ] Add additive nullable fields to the v2 notification:

```rust
pub struct AccountRateLimitsUpdatedNotification {
    pub rate_limits: RateLimitSnapshot,
    pub source_thread_id: Option<String>,
    pub source_model: Option<String>,
}
```

- [ ] In bespoke event handling, attach the active `conversation_id` and forward the event’s source model.
- [ ] Update the app-server README to document both fields and the fact that older producers may omit provenance.
- [ ] Re-run the two Task 1 tests. Expected: GREEN.
- [ ] Commit this protocol slice with a Lore-format message.

## Task 3: Lock foreground isolation and Spark caching with failing TUI tests

**Files:**
- Modify: `codex-rs/tui/src/chatwidget/tests/status_and_layout.rs`
- Modify: `codex-rs/tui/src/app/tests/rate_limits.rs`

- [ ] Add a ChatWidget regression test with a foreground Sol model and known thread:
  - deliver Sol `codex` telemetry and assert the foreground quota is stored;
  - deliver Spark telemetry from a different thread and assert the global foreground `codex` display is unchanged;
  - assert the Spark snapshot is retained in the model-scoped cache.
- [ ] Add coverage proving same-thread or same-model rolling telemetry can update the foreground display.
- [ ] Preserve backwards compatibility with a test proving missing provenance still updates the existing global `codex` entry.
- [ ] Extend the app event helper to construct notifications with optional thread/model provenance and add an event-level contamination regression.
- [ ] Run:

```bash
CARGO_TARGET_DIR=/home/xthe/.cache/codex-target just test -p codex-tui model_scoped
CARGO_TARGET_DIR=/home/xthe/.cache/codex-target just test -p codex-tui rolling_rate_limit
```

Expected: RED because the TUI has no provenance-aware ingestion or model-scoped cache.

## Task 4: Implement provenance-aware TUI ingestion

**Files:**
- Modify: `codex-rs/tui/src/chatwidget.rs`
- Modify: `codex-rs/tui/src/chatwidget/constructor.rs`
- Modify: `codex-rs/tui/src/chatwidget/rate_limits.rs`
- Modify: `codex-rs/tui/src/app/app_server_events.rs`
- Modify: `codex-rs/tui/src/app/tests/rate_limits.rs`

- [ ] Add:

```rust
rate_limit_snapshots_by_model: BTreeMap<String, RateLimitSnapshotDisplay>,
latest_spark_rate_limit_model: Option<String>,
```

- [ ] Introduce a small `RollingRateLimitSnapshotOrigin` value object holding optional source thread and source model; retain the legacy no-origin wrapper for existing callers.
- [ ] Normalize model keys with trimmed ASCII lowercase and detect Spark by a `spark` token rather than by an opaque rate-limit ID.
- [ ] For rolling default `codex` snapshots:
  - always preserve model-scoped telemetry when a source model exists;
  - mark the latest Spark model key when applicable;
  - project into the global foreground `codex` entry only when both provenance fields are available and either the thread matches `self.thread_id` or the model matches `self.current_model()`;
  - preserve legacy projection whenever either provenance field is unavailable.
- [ ] Seed the model-scoped Spark cache from a full account snapshot only when a named bucket’s `limit_name` contains a Spark token.
- [ ] Keep hard-stop generation in `App` account-wide. Apply ChatWidget quota nudges and warnings only to snapshots that project into the foreground entry.
- [ ] Clear the model-scoped cache and latest Spark key when account rate-limit state is cleared.
- [ ] Pass notification provenance from `app_server_events.rs`.
- [ ] Re-run Task 3 tests. Expected: GREEN.
- [ ] Commit this behavioral slice with a Lore-format message.

## Task 5: Lock compact Spark formatting and picker behavior with failing tests

**Files:**
- Modify: `codex-rs/tui/src/status/rate_limits.rs`
- Modify: `codex-rs/tui/src/bottom_pane/status_line_setup.rs`
- Modify: `codex-rs/tui/src/chatwidget/tests/status_surface_previews.rs`

- [ ] Add fixed-clock formatter tests:

```text
sp:82%(5d12h)
sp:5h:87%(2h1m) wk:50%(5d22h)
```

The first is a single-window Spark snapshot; the second is a dual-window snapshot.

- [ ] Add picker parsing/preview coverage for the exact config ID `spark-quota-summary`.
- [ ] Add status-surface coverage proving the item is omitted without Spark telemetry and rendered from model-scoped Spark telemetry when present.
- [ ] Run:

```bash
CARGO_TARGET_DIR=/home/xthe/.cache/codex-target just test -p codex-tui spark_quota
CARGO_TARGET_DIR=/home/xthe/.cache/codex-target just test -p codex-tui status_surface_preview
```

Expected: RED because the formatter and status-line item do not exist.

## Task 6: Add the toggleable `sp` status item

**Files:**
- Modify: `codex-rs/tui/src/status/rate_limits.rs`
- Modify: `codex-rs/tui/src/bottom_pane/status_line_setup.rs`
- Modify: `codex-rs/tui/src/bottom_pane/status_surface_preview.rs`
- Modify: `codex-rs/tui/src/bottom_pane/status_line_style.rs`
- Modify: `codex-rs/tui/src/chatwidget/status_controls.rs`
- Modify: `codex-rs/tui/src/chatwidget/status_surfaces.rs`
- Modify: `codex-rs/tui/src/chatwidget/tests/status_surface_previews.rs`

- [ ] Add `StatusLineItem::SparkQuotaSummary`, serialized as `spark-quota-summary`, with a default-off picker entry and description.
- [ ] Add the `StatusSurfacePreviewItem` mapping and placeholder `sp:82%(5d12h)`.
- [ ] Style it with the existing quota/limit accent.
- [ ] Add `format_spark_quota_summary`:
  - if only one window is displayable, omit its duration label and return `sp:<remaining>%(<countdown>)`;
  - if two windows are displayable, prefix the existing labeled compact summary with `sp:`.
- [ ] Resolve the live value from `latest_spark_rate_limit_model` and `rate_limit_snapshots_by_model`; return `None` when no Spark telemetry exists.
- [ ] Re-run Task 5 tests and inspect any `.snap.new` files. Accept only snapshots whose changes are the intentional new picker/status item. Expected: GREEN.
- [ ] Commit this UI slice with a Lore-format message.

## Task 7: Regenerate contracts and verify the integrated change

**Files:**
- Regenerate: `codex-rs/app-server-protocol/schema/json/**`
- Regenerate: `codex-rs/app-server-protocol/schema/typescript/**`

- [ ] Run schema generation:

```bash
just write-app-server-schema
```

- [ ] Verify generated notification schemas expose nullable `sourceThreadId` and `sourceModel`.
- [ ] Run focused protocol/server/TUI verification:

```bash
CARGO_TARGET_DIR=/home/xthe/.cache/codex-target just test -p codex-app-server-protocol
CARGO_TARGET_DIR=/home/xthe/.cache/codex-target just test -p codex-app-server
CARGO_TARGET_DIR=/home/xthe/.cache/codex-target just test -p codex-tui
```

- [ ] Run clippy fixes and formatting in the repository-prescribed order:

```bash
CARGO_TARGET_DIR=/home/xthe/.cache/codex-target just fix -p codex-core
CARGO_TARGET_DIR=/home/xthe/.cache/codex-target just fix -p codex-app-server
CARGO_TARGET_DIR=/home/xthe/.cache/codex-target just fix -p codex-tui
just fmt
```

- [ ] Inspect `git diff --check`, `git status --short`, generated-schema diffs, and the final source diff. Do not disturb unrelated user changes.
- [ ] Ask before running the repository-wide `just test`, as required by the checkout’s root `AGENTS.md`; if authorized, run it with the Linux-native target directory.
- [ ] Commit any remaining generated/documentation changes using the Lore protocol.

## Task 8: Build and hand off the patched CLI

**Files:**
- Inspect only: current patched-build helper/configuration under `/home/xthe/bin` and `~/.codex`

- [ ] Confirm the existing source-built override path and use the checkout’s established non-destructive build/install path; do not rebase or run the global npm updater because this task is not an upstream refresh.
- [ ] Build the patched `codex` binary with `CARGO_TARGET_DIR=/home/xthe/.cache/codex-target`.
- [ ] Verify the installed executable identity/version and that the working tree is clean apart from any explicitly preserved user-owned files.
- [ ] Report the config ID (`spark-quota-summary`), concise rendering (`sp:…`), test evidence, commit IDs, and any unrun repository-wide verification.

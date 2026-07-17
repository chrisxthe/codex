# Quota Summary Window Labels Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Label weekly-only quota summaries as weekly while preserving correct dual-window rendering if the 5-hour quota returns.

**Architecture:** Keep `format_quota_summary` as the display boundary and derive compact labels from each window's duration metadata. Reuse the existing TUI duration classifier, map its `weekly` label to compact `wk`, and preserve positional fallbacks only when duration metadata is unavailable or unrecognized.

**Tech Stack:** Rust, chrono, Codex TUI status surfaces, cargo-nextest via `just test`.

---

### Task 1: Restore the rebased integration fixture

**Files:**
- Modify: `codex-rs/tui/src/chatwidget/tests/status_and_layout.rs:3137`

- [x] **Step 1: Initialize the newly required protocol field**

Add this field to the existing `RateLimitSnapshot` fixture in
`status_line_quota_summary_includes_reset_countdowns`:

```rust
spend_control_reached: None,
```

- [x] **Step 2: Verify the fixture compiles far enough to run its existing test**

Run:

```bash
just test -p codex-tui status_line_quota_summary_includes_reset_countdowns
```

Expected: the test runs and passes instead of failing with missing field
`spend_control_reached`.

### Task 2: Lock weekly-only labeling with formatter tests

**Files:**
- Modify: `codex-rs/tui/src/status/rate_limits.rs:582`

- [x] **Step 1: Add a failing weekly-only formatter regression test**

Add this test beside `quota_summary_formats_remaining_percent_with_countdowns`:

```rust
#[test]
fn quota_summary_labels_weekly_window_in_primary_slot() {
    let now = Local
        .with_ymd_and_hms(2026, 7, 17, 12, 0, 0)
        .single()
        .expect("timestamp");
    let weekly = RateLimitWindowDisplay {
        used_percent: 25.0,
        resets_at: Some("12:00 on 24 Jul".to_string()),
        resets_at_unix: Some(now.timestamp() + (7 * 24 * 60 * 60)),
        window_minutes: Some(10_080),
    };

    assert_eq!(
        format_quota_summary(Some(&weekly), None, now),
        Some("wk:75%(7d)".to_string())
    );
}
```

- [x] **Step 2: Run the focused formatter test and verify the bug**

Run:

```bash
just test -p codex-tui quota_summary_labels_weekly_window_in_primary_slot
```

Expected before implementation: FAIL because actual output starts with `5h:`.

- [x] **Step 3: Derive compact labels from window duration**

Use the existing public classifier wrapper and its fallback label helper:

```rust
use crate::chatwidget::fallback_limit_label;
use crate::chatwidget::limit_label_for_window;
```

Update `format_quota_summary` so each slot supplies only its compatibility fallback:

```rust
if let Some(part) = format_quota_summary_window(primary, "5h", now) {
    parts.push(part);
}
if let Some(part) = format_quota_summary_window(secondary, "wk", now) {
    parts.push(part);
}
```

Replace `format_quota_summary_window` with the duration-aware version:

```rust
fn format_quota_summary_window(
    window: Option<&RateLimitWindowDisplay>,
    fallback_label: &str,
    now: DateTime<Local>,
) -> Option<String> {
    let window = window?;
    let is_secondary = fallback_label == "wk";
    let label = match limit_label_for_window(window.window_minutes, is_secondary).as_str() {
        "weekly" => "wk".to_string(),
        label if label == fallback_limit_label(is_secondary) => fallback_label.to_string(),
        label => label.to_string(),
    };
    let remaining = (100.0f64 - window.used_percent).clamp(0.0f64, 100.0f64);
    let countdown = window
        .resets_at_unix
        .and_then(|seconds| format_reset_countdown(seconds, now))?;
    Some(format!("{label}:{remaining:.0}%({countdown})"))
}
```

- [x] **Step 4: Run both formatter cases**

Run:

```bash
just test -p codex-tui quota_summary_
```

Expected: the weekly-only case renders `wk:75%(7d)` and the existing dual-window case remains
`5h:87%(2h1m) wk:50%(5d22h)`.

### Task 3: Lock the end-to-end status-line behavior

**Files:**
- Modify: `codex-rs/tui/src/chatwidget/tests/status_and_layout.rs:3133`

- [x] **Step 1: Add a weekly-only status-line test**

Add this second async test beside the existing quota-summary integration test:

```rust
#[tokio::test]
async fn status_line_quota_summary_labels_weekly_only_primary_window() {
    let (mut chat, _rx, _op_rx) = make_chatwidget_manual(/*model_override*/ None).await;
    chat.config.tui_status_line = Some(vec!["quota-summary".to_string()]);
    let now = chrono::Utc::now().timestamp();
    chat.on_rate_limit_snapshot(Some(RateLimitSnapshot {
        limit_id: None,
        limit_name: None,
        primary: Some(RateLimitWindow {
            used_percent: 25,
            window_duration_mins: Some(10_080),
            resets_at: Some(now + (7 * 24 * 60 * 60)),
        }),
        secondary: None,
        credits: None,
        individual_limit: None,
        spend_control_reached: None,
        plan_type: Some(PlanType::Plus),
        rate_limit_reached_type: None,
    }));

    assert_eq!(
        status_line_text(&chat),
        Some("wk:75%(7d)".to_string())
    );
}
```

- [x] **Step 2: Run the focused status-line tests**

Run:

```bash
just test -p codex-tui status_line_quota_summary
```

Expected: both the weekly-only and dual-window status-line tests pass.

### Task 4: Verify, commit, rebuild, and publish

**Files:**
- Verify: `codex-rs/tui/src/status/rate_limits.rs`
- Verify: `codex-rs/tui/src/chatwidget/tests/status_and_layout.rs`
- Update artifact: `/home/xthe/bin/codex-patched-0.144.5-patched`
- Update symlink: `/home/xthe/bin/codex-patched`
- Update patch: `/home/xthe/.cache/codex-patched-patches/quota-summary-status-line.patch`

- [x] **Step 1: Format and run the scoped repository gate**

Run from `codex-rs`:

```bash
just fmt
CARGO_TARGET_DIR=/home/xthe/.cache/codex-target just test -p codex-tui
```

Expected: formatter exits zero and the full `codex-tui` suite passes.

- [x] **Step 2: Confirm snapshots and tree contents**

Run:

```bash
find . -name '*.snap.new' -print
git status --short
```

Expected: no pending snapshots; only the intended implementation-plan/code/test changes are
present before commit.

- [ ] **Step 3: Commit with the Lore protocol**

Stage only the implementation plan and intended source/test files, then commit with an intent-led
message and `Constraint`, `Rejected`, `Confidence`, `Scope-risk`, `Directive`, `Tested`, and
`Not-tested` trailers.

- [ ] **Step 4: Rebuild and export the clean committed HEAD**

Run:

```bash
/home/xthe/bin/codex-patched-refresh
/home/xthe/bin/codex-patched-export-patch
```

Expected: `/home/xthe/bin/codex-patched-0.144.5-patched` is rebuilt, the stable symlink targets it,
and the exported patch is refreshed from the final clean HEAD.

- [ ] **Step 5: Prove publish preconditions**

Run:

```bash
git status --short
git merge-base --is-ancestor origin/main HEAD
git branch --show-current
```

Expected: clean tree, ancestry exit zero, branch `xthe/quota-summary-status-line`.

- [ ] **Step 6: Publish with a refreshed lease and verify the remote head**

Run:

```bash
branch="$(git branch --show-current)"
git fetch chrisxthe "${branch}:refs/remotes/chrisxthe/${branch}"
git push --force-with-lease chrisxthe "${branch}"
git ls-remote chrisxthe "refs/heads/${branch}"
```

Expected: push succeeds and `git ls-remote` reports the same commit as local `HEAD`.

# Quota Summary Window Labels

## Problem

The carried `quota-summary` status-line formatter assumes the protocol's primary rate-limit
window is always the 5-hour limit and the secondary window is always the weekly limit. Codex now
returns a weekly-only quota in the primary slot, causing the status line to label weekly usage as
`5h`.

## Design

Derive each compact quota label from `RateLimitWindowDisplay::window_minutes` instead of its
primary or secondary position. Reuse the existing duration classification used by other TUI
rate-limit surfaces:

- approximately 300 minutes renders as `5h`;
- approximately 10,080 minutes renders as `wk` in the compact quota summary;
- other recognized durations retain their existing human-readable duration label;
- when duration metadata is missing or unrecognized, preserve the current positional fallback
  (`primary` as `5h`, `secondary` as `wk`) for compatibility with older payloads.

The formatter continues to preserve protocol order and omits windows that lack a reset timestamp,
matching its existing behavior.

## Scope

Limit the implementation to the carried quota-summary formatter and its tests. Do not change the
protocol model, the standalone `five-hour-limit` or `weekly-limit` status-line items, or general
`/status` rate-limit rendering.

## Verification

Add a formatter regression test for a weekly-only window delivered in the primary slot and an
integration-level status-line test for the same payload. Preserve the existing dual-window test to
prove that a future return of the 5-hour quota still renders `5h` alongside `wk`. Run `just fmt`
and `just test -p codex-tui`, confirm no pending snapshots, rebuild/export the patched binary, and
publish only from a clean tree with `--force-with-lease`.

---
title: Sliding Panes
description: Two deliberate Right presses (Press, Release, Press) slide the file list out of view, gated behind a config flag
type: adr
status: proposed
created: 2026-05-21
---

# FEAT-0014 Sliding Panes

## Context and Problem Statement

FEAT-0012 made Right in the file list move focus to the diff. After focus moves, the file list is still visible -- useful for "select then peek back" workflows, wasteful for "select then dive in" workflows. The user asked for a second Right (after focus already moved to the diff) to collapse the file list, completing the gesture in one keystroke direction.

Two reasons this is harder than it looks:

1. Terminal auto-repeat fires repeated `Press` events when a key is held. Without further intervention, holding Right would interpret as Press, Press, Press, ... and trigger the hide on what the user reads as a single gesture.
2. Not every user wants this behavior. The previous iteration (single Right hides) was experienced as accidental dismiss, especially on trackpad.

## Decision Drivers

- A held arrow should never disappear or reappear the panel.
- The hide must be a deliberate second gesture, not a continuation of the first.
- Users who want the legacy behavior (single Right focuses without hiding) should not have hides forced on them.
- The kitty progressive keyboard enhancement (`REPORT_EVENT_TYPES`) is the only reliable way to distinguish Press from auto-repeat Repeat across terminals.

## Considered Options

### 1. Timing-based: require >N ms between Right #1 and Right #2

Track timestamp of first Right; second Right within N ms is treated as held. Works on every terminal.

Rejected: terminal auto-repeat initial delay is 300-500ms typical, with subsequent repeats every 30-50ms. No safe threshold exists that distinguishes the first auto-repeat from a deliberate second press.

### 2. Press + Release + Press via kitty `REPORT_EVENT_TYPES`

Push `KeyboardEnhancementFlags::REPORT_EVENT_TYPES` alongside the existing `DISAMBIGUATE_ESCAPE_CODES`. Crossterm emits `KeyEventKind::Press` / `Repeat` / `Release` events. Pending-hide arms on the file-list Press; consumes only on a Press in the diff that follows a Release of the previous Right. Held key sequence (Press, Repeat, Repeat, ..., Release) never consumes.

Falls back gracefully on terminals that don't support the flag: no Repeat or Release events are emitted, and the gesture degrades to the iteration-1 behavior (one Press hides, including via auto-repeat). The config flag gates whether the hide gesture is enabled at all.

### 3. Drop the hide-on-Right gesture; use `<leader>e` to hide

Right in file list only focuses diff. Panel stays visible. To hide, the user invokes `<leader>e` (the existing toggle). Simpler, terminal-agnostic, loses the symmetric Right-Right gesture.

## Decision Outcome

Chosen: option 2, gated by a config flag.

Config option `arrow_hide_file_list: bool` defaults to `false`. When false, Right in the file list focuses the diff and stops there (FEAT-0012 behavior, no hide). When true, the second deliberate Right hides the file list using the kitty REPORT_EVENT_TYPES detection.

The `App::pending_hide_file_list` flag is set when the user releases Right after pressing it in the file list. It is consumed by the next Press of Right while focus is on the diff. Held key (no Release between the two Right presses) never satisfies the arm-then-consume sequence, so the panel stays visible.

`REPORT_EVENT_TYPES` is pushed alongside `DISAMBIGUATE_ESCAPE_CODES` at terminal init, but only when `supports_keyboard_enhancement()` is true. On terminals that don't support kitty, the flag is silently ignored and `KeyEventKind::Repeat` / `Release` events are never produced; the hide gesture degrades to "one Press hides" if the config flag is on. Users on non-kitty terminals are encouraged to keep the flag off and use `<leader>e` instead.

The existing main-loop filter `if key.kind == KeyEventKind::Press` is widened to also accept `KeyEventKind::Repeat`, so `j` and `k` continue to auto-repeat correctly. The hide logic specifically checks for `Press` (not Repeat) when consuming the pending arm.

## Consequences

- [+] Held Right never disappears the panel, even on terminals with auto-repeat.
- [+] Config flag means users who don't want the gesture aren't surprised by it.
- [+] On kitty terminals (Ghostty, kitty, WezTerm, Alacritty) the gesture is fully working.
- [-] Behavior is terminal-dependent. Documentation must call out that the gesture relies on REPORT_EVENT_TYPES.
- [-] The main-loop event filter is widening from "Press only" to "Press + Repeat", which means a future code path that relies on Press-only semantics needs to filter explicitly.

## Future Considerations

- A help-popup line documenting the gesture and the config flag.
- If the kitty flag is unavailable AND the config gates is on, surface a startup warning so users know the gesture won't work as advertised.
- Consider a similar press-release-press gate for any future "destructive on hold" gesture (e.g., a hypothetical `dd` to delete that gets repeated). The pattern is general.

[FEAT-0012]: FEAT-0012 Arrow-Key Pane Focus.md
[FEAT-0013]: FEAT-0013 Mouse Horizontal Scroll Routing.md

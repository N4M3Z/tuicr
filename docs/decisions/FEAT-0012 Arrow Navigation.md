---
title: Arrow Navigation
description: Left/right arrow keys drive tree navigation in the file list and slide focus to the diff at the boundary
type: adr
status: proposed
created: 2026-05-21
---

# FEAT-0012 Arrow Navigation

## Context

`<leader>h` / `<leader>l` and `Tab` switch focus between the file list and the diff. Two keystrokes for what is conceptually a single direction-of-attention gesture, and the file tree itself can only be expanded with `Enter` / `Space`.

## Decision

`h` / `l` / Left / Right scroll the focused panel horizontally while there's hidden content to scroll, then fall through to a secondary action at the scroll boundary. Same pattern on both sides; only the secondary action differs.

In the diff:
- `h` while `scroll_x > 0` scrolls left.
- `h` at `scroll_x == 0` slides focus to the file list (revealing it if hidden).
- `l` scrolls right.

In the file list:
- `l` while `scroll_x < max_scroll_x` scrolls right.
- `l` at `scroll_x == max_scroll_x` falls through to gitui-style tree nav: expand a collapsed folder, descend into an expanded one, or slide focus to the diff when the selection is a file.
- `h` while `scroll_x > 0` scrolls left.
- `h` at `scroll_x == 0` falls through to gitui-style tree nav: collapse an expanded folder, otherwise ascend to the parent. No-op at the root.

Mouse horizontal scroll bypasses this layer; see [FEAT-0013][FEAT-0013].

## Consequences

- [+] One-keystroke pane switching at the natural scroll boundary.
- [+] The file tree expands and collapses with the same keys used to navigate it, matching the gitui mental model.
- [+] Horizontal scroll keeps working in both panels for long content (deeply nested paths, long lines).
- [-] On a wide file-list panel where everything fits, `scroll_x == max_scroll_x == 0`, so the very first `l` press already triggers tree nav. Users with narrow panels get the layered behavior; users with wide panels effectively see the gitui-only behavior. The boundary state is invisible until the user crosses it.

[FEAT-0013]: FEAT-0013 Mouse Horizontal Scroll Routing.md

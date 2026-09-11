# Windows 11 Monitor Manager — Design (Phase 1)

**Date:** 2026-09-11
**Status:** Approved design, pre-implementation
**Author:** Taha + Claude

## 1. Purpose

A Windows 11 desktop tool that shows which open windows are on which
monitor and lets you rearrange them. Built as a full window manager
delivered in phases; this spec covers **Phase 1** (the live viewer +
mover). Phases 2–3 are scoped at the end so Phase 1 doesn't paint us
into a corner.

## 2. Stack

- **Backend:** Rust, using the official `windows` crate (Microsoft's
  metadata-generated Win32 projection) for all Win32 calls.
- **Shell:** Tauri v2 (single window, system tray).
- **Frontend:** Vue 3 + TypeScript (Taha's expertise), Vite.

Rationale: most "native-adjacent" option that stays productive — the
`windows` crate is as close to the real Win32 API as you get without
C++, compiles to a small native binary (good for later Gumroad-style
distribution), and Tauri lets the entire UI be Vue. Rust is the
deliberate learning target.

## 3. Scope of Phase 1

**In:**
- System-tray-backed single window. Clicking the window's close (X)
  **hides to tray**; the window stays alive. Tray icon left-click (or
  double-click) shows/focuses the window; tray right-click → **Exit**
  fully quits the process.
- A proportional visual map of the physical monitor layout, with each
  manageable window drawn as a tile in its real position.
- **Live, event-driven updates** as windows open, close, move,
  minimize, or change focus.
- **Drag a window tile** from one monitor to another to actually move
  the real window. A per-tile "send to →" menu is the keyboard/fallback
  path.
- **Self-move number hotkeys:** while the app window is focused,
  pressing `1`, `2`, `3`, … moves the **app's own window** to the
  monitor with that number. In-app only (no global hotkey / OS
  registration); does nothing when the app is not focused.

**Out (deferred, see §12):** FancyZones zone overlay + snap-to-zone,
saved/named layouts, auto-place rules, global hotkeys, virtual-desktop
awareness.

## 4. What counts as a "manageable window"

The single trickiest correctness surface. A top-level window is
manageable when **all** hold:

- It is top-level and **un-owned** (`GetWindow(GA_ROOTOWNER)` resolves
  to itself), or is an owned window that sets `WS_EX_APPWINDOW`.
- It is `IsWindowVisible` **or** minimized (`IsIconic`).
- It has a non-empty title.
- It is **not** `WS_EX_TOOLWINDOW`, **unless** it also sets
  `WS_EX_APPWINDOW`.
- It is **not** DWM-cloaked: `DwmGetWindowAttribute(DWMWA_CLOAKED)`
  returns 0. (This is what filters ghost UWP shells /
  `ApplicationFrameHost` phantom windows that pass `IsWindowVisible`.)

This heuristic lives in `win/filter.rs` as a pure function over a
`WindowProbe` struct (title, styles, ex-styles, owner, cloaked,
iconic), so it is **unit-tested against a fixture set** independently
of any live enumeration.

## 5. Module layout (backend)

```
src-tauri/src/
  main.rs          # Tauri setup, tray, window-close→hide wiring
  commands.rs      # Tauri commands invoked from Vue
  state.rs         # AppState: current window+monitor model, diffing
  win/
    mod.rs
    monitors.rs    # enumerate monitors + work areas (virtual-screen px)
    enum.rs        # enumerate top-level windows, apply filter
    filter.rs      # pure is_manageable() heuristic (§4) + tests
    move.rs        # move/restore a window to a target monitor (§7) + rect-math tests
    events.rs      # WinEvent hook thread (§6)
    dpi.rs         # Per-Monitor-V2 setup + coordinate helpers
```

Frontend:

```
src/
  App.vue
  components/MonitorMap.vue   # the hero canvas: monitors + window tiles + drag
  components/WindowTile.vue
  stores/desktop.ts           # subscribes to backend events, holds model
  lib/ipc.ts                  # typed wrappers over Tauri invoke/listen
  lib/scale.ts                # virtual-screen → canvas scaling (pure, tested)
  lib/hotkeys.ts              # digit-key self-move handler (focused only)
```

## 6. Live updates — the WinEvent hook thread

`SetWinEventHook` requires a thread that runs a Windows message pump,
which Tauri's async runtime does not provide. Therefore:

- The backend spawns **one dedicated OS thread** (`win/events.rs`) that
  installs out-of-context hooks for: `EVENT_OBJECT_LOCATIONCHANGE`,
  `EVENT_OBJECT_CREATE`, `EVENT_OBJECT_DESTROY`, `EVENT_OBJECT_HIDE`,
  `EVENT_OBJECT_SHOW`, `EVENT_SYSTEM_FOREGROUND`,
  `EVENT_SYSTEM_MINIMIZESTART`, and `EVENT_SYSTEM_MINIMIZEEND`. It then
  runs a `GetMessage` loop.
- The hook callback runs **on that thread**. It filters to top-level
  `OBJID_WINDOW` events and pushes raw events onto a channel. It does
  **not** call back into Win32 enumeration from inside the callback.
- A coalescing consumer on the main side **debounces**
  `LOCATIONCHANGE` (these flood continuously during a drag — coalesce
  to ~60ms trailing), recomputes the affected window(s), diffs against
  `AppState`, and emits a Tauri event (`desktop:changed`) with the
  delta to Vue.
- **Shutdown:** `UnhookWinEvent` for each hook, then
  `PostThreadMessage(tid, WM_QUIT)` to break the message loop; join the
  thread on app exit.
- **Supervision:** if the hook thread panics/exits unexpectedly, the
  backend falls back to a **1s polling** re-scan and emits a
  `desktop:degraded` status the UI surfaces as a small banner. The app
  keeps working; it just isn't real-time.

## 7. Moving a window

`move_window(hwnd, target_monitor_id)`:

1. Read placement (`GetWindowPlacement`) — note if maximized.
2. If maximized, `ShowWindow(SW_RESTORE)` first.
3. Translate the window rect from its current monitor's work area to
   the target monitor's work area, **preserving relative position and
   size**; clamp so the window fits inside the target work area.
4. `SetWindowPos` (no activate, no z-order change) to the new rect.
5. If it was maximized, `ShowWindow(SW_MAXIMIZE)` on the new monitor.

The **rect translation is a pure function** in `win/move.rs`
(`translate_rect(rect, from_workarea, to_workarea) -> rect`) and is
unit-tested with mixed-size/mixed-DPI monitor fixtures, no Win32
needed.

**Self-move hotkey** reuses this: `move_self_to_monitor(n)` resolves
the app's own HWND (Tauri window handle) and the n-th monitor (by the
stable ordering in §8), then calls the same move path.

## 8. Coordinates, DPI, monitor numbering

- The process is manifested **Per-Monitor-V2 DPI aware**
  (`SetProcessDpiAwarenessContext` at startup as a belt-and-braces
  fallback to the manifest).
- All geometry is in **physical pixels** on the **virtual-screen
  coordinate space**. The Vue map computes the bounding box of all
  monitors and scales it to fit the canvas (`lib/scale.ts`, pure +
  tested), so the 5120×1440 ultrawide and other monitors stay
  proportionally correct.
- **Monitor numbering** for the map labels and the self-move hotkeys is
  a **stable ordering**: sort by (x, then y) of each monitor's origin,
  1-indexed. This is deterministic and independent of Windows' own
  device numbering (which is unstable across replug/virtual desktops).
  The number shown on each monitor in the map is exactly the digit key
  that sends the app there.

## 9. Self-move number hotkeys (detail)

- Handled entirely in the **focused frontend** (`lib/hotkeys.ts`): a
  `keydown` listener on `window`, active only while the document has
  focus. No `SetWindowsHookEx`, no Tauri global-shortcut plugin — so it
  is impossible to trigger when the app isn't up and focused, exactly
  as specified.
- Keys `1`–`9` (and `0`→10 if ever needed) map to the 1-indexed
  monitor ordering from §8. Pressing a digit for a monitor that
  doesn't exist is a no-op (optional brief toast).
- Ignored when focus is in a text input (future-proofing for search).
- Calls the `move_self_to_monitor(n)` command (§7).

## 10. IPC surface (Tauri commands + events)

Commands (Vue → Rust):
- `get_snapshot() -> Desktop` — full current model (monitors +
  windows) for initial render / manual refresh.
- `move_window(hwnd: u64, monitor_id: u32)` — §7.
- `move_self_to_monitor(n: u32)` — §7/§9.
- `focus_window(hwnd: u64)` — bring a window forward (used by tile
  double-click; cheap win, low cost).

Events (Rust → Vue):
- `desktop:changed` — delta (added/removed/moved windows, monitor
  changes).
- `desktop:degraded` — hook thread fell back to polling.

Types are shared via a single TS definition mirrored from the Rust
structs (`Desktop`, `MonitorInfo`, `WindowInfo`).

## 11. Error handling

- Every Win32 FFI call returns `Result`; helpers convert
  `windows::core::Error` into a local `WinError`.
- A **vanished HWND** mid-operation (moved/destroyed between event and
  action) is a **normal outcome**, not an error: drop it from state and
  continue. Never panic on a dead window.
- The hook thread is supervised (§6); its death degrades to polling
  rather than crashing.
- Tauri commands return typed errors the UI can show inline; a failed
  move surfaces a small non-blocking toast, never a modal.

## 12. Testing strategy

- **Rust unit tests (pure, fixture-driven):**
  - `filter::is_manageable` across a table of window probes (tool
    windows, cloaked shells, owned dialogs, minimized apps, titleless
    windows).
  - `move::translate_rect` across mixed-size/mixed-DPI monitor pairs,
    including maximized and clamp-to-fit cases.
  - monitor ordering (§8) determinism.
- **Vue component tests (Vitest):** `lib/scale.ts` scaling math;
  `MonitorMap` tile placement + drag-to-move against a mocked IPC;
  `lib/hotkeys.ts` digit mapping + focused-only + input-field guard.
- **Manual dev harness:** live WinEvent behavior (open/move/minimize a
  few real windows) — the OS-hook path can't be meaningfully unit
  tested; verified by hand on the real multi-monitor rig.
- Lint/format gates: `cargo fmt` + `cargo clippy` clean;
  `eslint`/`vue-tsc` clean.

## 13. Deferred phases (context only, not built here)

- **Phase 2 — FancyZones-aware map + saved layouts.** Read
  `%LOCALAPPDATA%/Microsoft/PowerToys/FancyZones/applied-layouts.json`
  + `custom-layouts.json`, reproduce each zone's rect from the stored
  grid percentages × monitor work area (minus spacing), overlay zones
  under the window tiles, and add **snap-to-zone** by `SetWindowPos`-ing
  to a computed zone rect. Known boundary: no public FancyZones API to
  drive its own snapping / update `app-zone-history.json` reliably — we
  reproduce placement, we don't drive FancyZones. Known gotcha: matching
  our enumerated monitors to FancyZones' device identity (EDID serial +
  instance + virtual desktop) via the display-config API. Plus
  save/restore named layouts (window→monitor+rect sets, e.g.
  "work"/"gaming").
- **Phase 3 — auto-place rules.** "App X → monitor/zone N on open,"
  driven by the same `EVENT_OBJECT_CREATE` hook.

## 14. Documentation (as-we-go)

Documentation is a first-class deliverable, kept current with the code
rather than bolted on at the end.

- **README.md** — overview, feature list, roadmap, dev quickstart,
  links into the docs site. Maintained continuously.
- **VitePress docs site** under `docs/` (`docs:dev` / `docs:build`
  scripts). Vue-native, matches the stack. Sections:
  - *Architecture* — the module map (§5), the WinEvent hook thread
    (§6), coordinate/DPI model (§8).
  - *Win32 internals* — the manageable-window heuristic (§4) and the
    move/translate semantics (§7), written as reference for the
    learning-Rust-and-Win32 goal.
  - *IPC reference* — the command/event surface (§10) with the shared
    types.
  - *Contributing / dev setup*.
- The existing `docs/superpowers/specs/` design docs stay in-tree; the
  VitePress site links to them rather than duplicating them.
- **Rule:** each implementation task that adds or changes a public
  behavior updates the relevant doc page in the same change. Rust
  public items carry `///` doc comments; exported TS has TSDoc.

## 15. Open items for implementation planning

- Confirm Tauri v2 tray + `windows` crate versions at plan time
  (pin exact versions).
- Decide the delta shape for `desktop:changed` (full-replace vs
  granular) — start with full-replace of the changed monitor's windows
  for simplicity; optimize only if the UI flickers.

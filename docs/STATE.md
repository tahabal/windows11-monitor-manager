# Project State — Windows 11 Monitor Manager

_Last updated: 2026-09-12_

## Where things stand

**Phase:** Phase 1, pre-implementation. No app code written yet — design and
plan are complete and approved.

- **Spec (approved):** `docs/superpowers/specs/2026-09-11-monitor-manager-design.md`
- **Implementation plan:** `docs/superpowers/plans/2026-09-11-monitor-manager-phase1.md`
  — 11 TDD tasks, each stamped with its Linear ID.
- **Repo:** public — https://github.com/tahabal/windows11-monitor-manager
- **Linear project:** https://linear.app/raccoonsoft/project/windows-11-monitor-manager-2eda90adb709
  (team RAC/Raccoon), issues **RAC-282 → RAC-292**, priority-ordered.

## Decisions locked

- **Stack:** Rust + Tauri v2 + Vue 3/TS. Win32 only via the official `windows` crate.
- **Phase 1 scope:** tray-backed single window (close → hide to tray; tray
  right-click → Exit); live event-driven monitor map via `SetWinEventHook`;
  drag-to-move windows across monitors + a "send to →" menu; focused-only
  number-key self-move (digit → stable monitor ordering).
- **Window scope:** normal top-level app windows (visible + minimized);
  exclude tool/tray/cloaked/system windows.
- **Deferred:** Phase 2 = FancyZones-aware overlay + snap-to-zone + saved
  layouts; Phase 3 = auto-place rules. Global hotkeys and virtual-desktop
  awareness are also out of Phase 1.

## Next step

Execute the plan starting at **RAC-282** (scaffold) via superpowers
subagent-driven-development (recommended) or executing-plans. Work the tickets
in priority order; keep the plan checkboxes and Linear status in sync, and log
meaningful progress with the idea-hub `add_history_entry` tool.

## Ticket map

| Ticket | Priority | Task |
|--------|----------|------|
| RAC-282 | Urgent | Scaffold Tauri v2 + Vue 3 + TS + Vitest + VitePress |
| RAC-283 | High | Monitor model + stable ordering + EnumDisplayMonitors |
| RAC-284 | High | Manageable-window filter + EnumWindows probe |
| RAC-285 | High | translate_rect + move_to_monitor |
| RAC-286 | High | App state + IPC commands |
| RAC-287 | High | WinEvent hook thread + live events + polling fallback |
| RAC-288 | Medium | Desktop store + scale-to-fit canvas + live MonitorMap |
| RAC-289 | Medium | Drag-to-move + send-to-monitor menu |
| RAC-290 | Medium | Tray + close-to-tray + Exit + Per-Monitor-V2 DPI |
| RAC-291 | Low | Focused-only number-key self-move |
| RAC-292 | Low | Documentation pass + README + CI |

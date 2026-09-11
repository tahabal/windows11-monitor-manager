# Windows 11 Monitor Manager — Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a tray-backed Windows 11 app showing a live, proportional map of which windows are on which monitor, with drag-to-move across monitors and focused-only number-key self-move.

**Architecture:** Tauri v2 native shell. A Rust backend calls Win32 (via the `windows` crate) to enumerate monitors/windows and move them, and runs a dedicated OS thread with a `SetWinEventHook` message pump that pushes live deltas to a Vue 3 + TypeScript frontend over Tauri events. Pure logic (window-filter heuristic, rect translation, monitor ordering, canvas scaling, hotkey mapping) is isolated behind function boundaries and unit-tested; the OS-hook path is verified by a manual dev harness.

**Tech Stack:** Rust, `windows` crate, Tauri v2, Vue 3, TypeScript, Vite, Vitest, VitePress.

**Spec:** `docs/superpowers/specs/2026-09-11-monitor-manager-design.md`

## Global Constraints

- **Platform:** Windows 11 only. All Win32 code is `#[cfg(windows)]`; pure-logic modules are platform-agnostic so they test on any CI.
- **Win32 access:** only via the official `windows` crate — no `winapi`, no hand-rolled FFI.
- **DPI:** process is **Per-Monitor-V2** aware (manifest + `SetProcessDpiAwarenessContext` fallback). All geometry is **physical pixels** in **virtual-screen** coordinates.
- **Monitor numbering** (map labels + self-move hotkey targets): stable ordering = sort by monitor origin `(x, then y)`, 1-indexed. Never Windows device numbers.
- **Manageable window** = top-level, un-owned (or owned + `WS_EX_APPWINDOW`), has a title, `IsWindowVisible` or `IsIconic`, not `WS_EX_TOOLWINDOW` (unless `WS_EX_APPWINDOW`), not DWM-cloaked.
- **A vanished HWND is never an error** — drop it from state and continue.
- **Docs as-we-go:** any task changing a public behavior updates the relevant `docs/` page in the same commit; Rust public items get `///`, exported TS gets TSDoc.
- **Pin exact versions** when scaffolding; TDD + frequent commits; `cargo fmt`/`clippy` and `eslint`/`vue-tsc` clean.

---

### Task 1 (RAC-282): Project scaffold (Tauri v2 + Vue 3 + TS + Vitest + VitePress)

**Files:**
- Create: `package.json`, `vite.config.ts`, `tsconfig.json`, `index.html`, `src/main.ts`, `src/App.vue`
- Create: `src-tauri/Cargo.toml`, `src-tauri/tauri.conf.json`, `src-tauri/build.rs`, `src-tauri/src/main.rs`, `src-tauri/src/lib.rs`
- Create: `vitest.config.ts`, `src/lib/__tests__/smoke.test.ts`
- Create: `docs/.vitepress/config.ts`, `docs/index.md`
- Modify: `README.md` (scripts already documented — verify they match)

**Interfaces:**
- Produces: a runnable Tauri app; npm scripts `tauri dev`, `test`, `docs:dev`, `docs:build`; `cargo test` target in `src-tauri`.

- [ ] **Step 1: Scaffold via create-tauri-app, then pin versions**

Run (from repo root; it creates into the current dir):
```bash
npm create tauri-app@latest -- --template vue-ts --manager npm --yes .
```
Then in `src-tauri/Cargo.toml` pin `tauri = "=2.x.y"` (latest 2.x at scaffold time) and add:
```toml
windows = { version = "=0.5x.0", features = [
  "Win32_Foundation",
  "Win32_UI_WindowsAndMessaging",
  "Win32_UI_Accessibility",
  "Win32_UI_HiDpi",
  "Win32_Graphics_Gdi",
  "Win32_System_Threading",
] }
```
Record the resolved exact versions in the commit message.

- [ ] **Step 2: Add Vitest + a smoke test**

`vitest.config.ts`:
```ts
import { defineConfig } from 'vitest/config'
export default defineConfig({ test: { environment: 'jsdom', globals: true } })
```
`src/lib/__tests__/smoke.test.ts`:
```ts
import { describe, it, expect } from 'vitest'
describe('scaffold', () => { it('runs vitest', () => { expect(1 + 1).toBe(2) }) })
```
Add to `package.json` scripts: `"test": "vitest run"`, `"test:watch": "vitest"`, plus dev-deps `vitest`, `jsdom`, `@vue/test-utils`.

- [ ] **Step 3: Add VitePress skeleton**

Add dev-dep `vitepress`. Scripts: `"docs:dev": "vitepress dev docs"`, `"docs:build": "vitepress build docs"`.
`docs/.vitepress/config.ts`:
```ts
import { defineConfig } from 'vitepress'
export default defineConfig({
  title: 'Monitor Manager',
  description: 'Windows 11 monitor/window manager',
  themeConfig: {
    sidebar: [
      { text: 'Overview', link: '/' },
      { text: 'Architecture', link: '/architecture' },
      { text: 'Win32 internals', link: '/win32' },
      { text: 'IPC reference', link: '/ipc' },
    ],
  },
})
```
`docs/index.md`: a one-paragraph intro linking to the spec. (Architecture/Win32/IPC pages are filled in by later tasks; create stub files `docs/architecture.md`, `docs/win32.md`, `docs/ipc.md` each with a single `# Title` heading so the sidebar resolves.)

- [ ] **Step 4: Verify everything builds/runs**

Run and confirm each succeeds:
```bash
npm install
npm run test          # smoke test passes
cargo test --manifest-path src-tauri/Cargo.toml   # 0 tests, compiles
npm run docs:build    # docs compile
npm run tauri dev     # a window opens (Ctrl+C to close)
```
Expected: all green; a blank Tauri window appears.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "chore: scaffold Tauri v2 + Vue 3 + Vitest + VitePress"
```

---

### Task 2 (RAC-283): Shared domain types + monitor enumeration + stable ordering

**Files:**
- Create: `src-tauri/src/model.rs` (serde types shared over IPC)
- Create: `src-tauri/src/win/mod.rs`, `src-tauri/src/win/monitors.rs`
- Create: `src/lib/types.ts` (hand-mirrored TS types)
- Modify: `src-tauri/src/lib.rs` (add `mod model; mod win;`)
- Test: inline `#[cfg(test)]` in `monitors.rs`

**Interfaces:**
- Produces (Rust): `model::MonitorInfo { id: u32, number: u32, name: String, bounds: Rect, work_area: Rect }`, `model::Rect { x: i32, y: i32, w: i32, h: i32 }`, `model::WindowInfo { hwnd: u64, title: String, rect: Rect, monitor_id: u32, minimized: bool }`, `model::Desktop { monitors: Vec<MonitorInfo>, windows: Vec<WindowInfo> }`.
- Produces: `win::monitors::order_monitors(raw: Vec<RawMonitor>) -> Vec<MonitorInfo>` — pure, assigns `number` 1..=N by `(x, y)` origin. `RawMonitor { handle_id: u32, name: String, bounds: Rect, work_area: Rect }`.
- Produces: `win::monitors::enumerate() -> Vec<MonitorInfo>` (`#[cfg(windows)]`, calls `EnumDisplayMonitors` then `order_monitors`).
- Consumes: nothing (first backend task).

- [ ] **Step 1: Write the failing test for ordering**

In `src-tauri/src/win/monitors.rs`:
```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::model::Rect;

    fn raw(id: u32, x: i32, y: i32) -> RawMonitor {
        RawMonitor { handle_id: id, name: format!("m{id}"),
            bounds: Rect { x, y, w: 1920, h: 1080 },
            work_area: Rect { x, y, w: 1920, h: 1040 } }
    }

    #[test]
    fn numbers_left_to_right_then_top_to_bottom() {
        // provided out of order: right, left, below-left
        let out = order_monitors(vec![raw(10, 1920, 0), raw(11, 0, 0), raw(12, 0, 1080)]);
        let by_num: Vec<(u32, i32, i32)> =
            out.iter().map(|m| (m.number, m.bounds.x, m.bounds.y)).collect();
        assert_eq!(by_num, vec![(1, 0, 0), (2, 0, 1080), (3, 1920, 0)]);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --manifest-path src-tauri/Cargo.toml numbers_left_to_right -v`
Expected: FAIL — `order_monitors`/`RawMonitor` not found.

- [ ] **Step 3: Implement types + ordering**

`model.rs`:
```rust
use serde::{Deserialize, Serialize};

#[derive(Clone, Copy, Debug, PartialEq, Eq, Serialize, Deserialize)]
pub struct Rect { pub x: i32, pub y: i32, pub w: i32, pub h: i32 }

#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct MonitorInfo {
    pub id: u32, pub number: u32, pub name: String,
    pub bounds: Rect, pub work_area: Rect,
}

#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct WindowInfo {
    pub hwnd: u64, pub title: String, pub rect: Rect,
    pub monitor_id: u32, pub minimized: bool,
}

#[derive(Clone, Debug, Default, PartialEq, Serialize, Deserialize)]
pub struct Desktop { pub monitors: Vec<MonitorInfo>, pub windows: Vec<WindowInfo> }
```
`win/monitors.rs` (pure part):
```rust
use crate::model::{MonitorInfo, Rect};

pub struct RawMonitor { pub handle_id: u32, pub name: String, pub bounds: Rect, pub work_area: Rect }

pub fn order_monitors(mut raw: Vec<RawMonitor>) -> Vec<MonitorInfo> {
    raw.sort_by_key(|m| (m.bounds.x, m.bounds.y));
    raw.into_iter().enumerate().map(|(i, m)| MonitorInfo {
        id: m.handle_id, number: (i as u32) + 1, name: m.name,
        bounds: m.bounds, work_area: m.work_area,
    }).collect()
}
```
`win/mod.rs`: `pub mod monitors;`. `lib.rs`: add `pub mod model; pub mod win;`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --manifest-path src-tauri/Cargo.toml numbers_left_to_right -v`
Expected: PASS.

- [ ] **Step 5: Add the Win32 enumeration (compile-checked, not unit-tested)**

Append to `win/monitors.rs`:
```rust
#[cfg(windows)]
pub fn enumerate() -> Vec<MonitorInfo> {
    use windows::Win32::Foundation::{BOOL, LPARAM, RECT, TRUE};
    use windows::Win32::Graphics::Gdi::{
        EnumDisplayMonitors, GetMonitorInfoW, HDC, HMONITOR, MONITORINFO,
    };

    unsafe extern "system" fn cb(h: HMONITOR, _dc: HDC, _r: *mut RECT, data: LPARAM) -> BOOL {
        let out = &mut *(data.0 as *mut Vec<RawMonitor>);
        let mut mi = MONITORINFO { cbSize: std::mem::size_of::<MONITORINFO>() as u32, ..Default::default() };
        if GetMonitorInfoW(h, &mut mi).as_bool() {
            let b = mi.rcMonitor; let w = mi.rcWork;
            out.push(RawMonitor {
                handle_id: h.0 as u32,
                name: format!("Monitor {}", out.len() + 1),
                bounds: rect_from(b), work_area: rect_from(w),
            });
        }
        TRUE
    }
    fn rect_from(r: windows::Win32::Foundation::RECT) -> Rect {
        Rect { x: r.left, y: r.top, w: r.right - r.left, h: r.bottom - r.top }
    }
    let mut raw: Vec<RawMonitor> = Vec::new();
    unsafe {
        let _ = EnumDisplayMonitors(HDC::default(), None, Some(cb), LPARAM(&mut raw as *mut _ as isize));
    }
    order_monitors(raw)
}
```
Add matching `src/lib/types.ts` mirroring the serde shapes (`Rect`, `MonitorInfo`, `WindowInfo`, `Desktop`) with TSDoc.

- [ ] **Step 6: Verify + commit**

Run: `cargo test --manifest-path src-tauri/Cargo.toml -v` (ordering test passes, enumerate compiles).
```bash
git add -A
git commit -m "feat: monitor model + stable ordering + EnumDisplayMonitors"
```

---

### Task 3 (RAC-284): Manageable-window filter (pure) + window enumeration

**Files:**
- Create: `src-tauri/src/win/filter.rs`
- Create: `src-tauri/src/win/enumerate.rs`
- Modify: `src-tauri/src/win/mod.rs`
- Modify: `docs/win32.md` (document the heuristic)
- Test: inline `#[cfg(test)]` in `filter.rs`

**Interfaces:**
- Produces: `win::filter::WindowProbe { title: String, is_visible: bool, is_iconic: bool, is_toolwindow: bool, is_appwindow: bool, is_owned: bool, is_cloaked: bool }` and `win::filter::is_manageable(p: &WindowProbe) -> bool` — pure.
- Produces: `win::enumerate::windows(monitors: &[MonitorInfo]) -> Vec<WindowInfo>` (`#[cfg(windows)]`) — enumerates top-level windows, builds a `WindowProbe`, keeps the manageable ones, and assigns `monitor_id` via `MonitorFromWindow`.
- Consumes: `model::{MonitorInfo, WindowInfo, Rect}` (Task 2).

- [ ] **Step 1: Write the failing test table**

`win/filter.rs`:
```rust
pub struct WindowProbe {
    pub title: String, pub is_visible: bool, pub is_iconic: bool,
    pub is_toolwindow: bool, pub is_appwindow: bool,
    pub is_owned: bool, pub is_cloaked: bool,
}

#[cfg(test)]
mod tests {
    use super::*;
    fn base() -> WindowProbe {
        WindowProbe { title: "App".into(), is_visible: true, is_iconic: false,
            is_toolwindow: false, is_appwindow: false, is_owned: false, is_cloaked: false }
    }
    #[test] fn normal_visible_app_is_manageable() { assert!(is_manageable(&base())); }
    #[test] fn minimized_app_is_manageable() {
        assert!(is_manageable(&WindowProbe { is_visible: false, is_iconic: true, ..base() }));
    }
    #[test] fn titleless_is_excluded() {
        assert!(!is_manageable(&WindowProbe { title: String::new(), ..base() }));
    }
    #[test] fn hidden_nonminimized_is_excluded() {
        assert!(!is_manageable(&WindowProbe { is_visible: false, ..base() }));
    }
    #[test] fn toolwindow_is_excluded() {
        assert!(!is_manageable(&WindowProbe { is_toolwindow: true, ..base() }));
    }
    #[test] fn toolwindow_with_appwindow_is_included() {
        assert!(is_manageable(&WindowProbe { is_toolwindow: true, is_appwindow: true, ..base() }));
    }
    #[test] fn cloaked_is_excluded() {
        assert!(!is_manageable(&WindowProbe { is_cloaked: true, ..base() }));
    }
    #[test] fn owned_without_appwindow_is_excluded() {
        assert!(!is_manageable(&WindowProbe { is_owned: true, ..base() }));
    }
    #[test] fn owned_with_appwindow_is_included() {
        assert!(is_manageable(&WindowProbe { is_owned: true, is_appwindow: true, ..base() }));
    }
}
```

- [ ] **Step 2: Run to verify it fails**

Run: `cargo test --manifest-path src-tauri/Cargo.toml filter:: -v`
Expected: FAIL — `is_manageable` not found.

- [ ] **Step 3: Implement the heuristic**

Add to `win/filter.rs`:
```rust
/// Decide whether a top-level window should appear in the manager.
/// See docs/win32.md for the rationale behind each clause.
pub fn is_manageable(p: &WindowProbe) -> bool {
    if p.title.is_empty() { return false; }
    if !(p.is_visible || p.is_iconic) { return false; }
    if p.is_cloaked { return false; }
    if p.is_toolwindow && !p.is_appwindow { return false; }
    if p.is_owned && !p.is_appwindow { return false; }
    true
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `cargo test --manifest-path src-tauri/Cargo.toml filter:: -v`
Expected: all 9 PASS.

- [ ] **Step 5: Implement Win32 enumeration using the filter**

`win/enumerate.rs`:
```rust
use crate::model::{MonitorInfo, Rect, WindowInfo};
use crate::win::filter::{is_manageable, WindowProbe};

#[cfg(windows)]
pub fn windows(monitors: &[MonitorInfo]) -> Vec<WindowInfo> {
    use windows::Win32::Foundation::{BOOL, HWND, LPARAM, RECT, TRUE};
    use windows::Win32::Graphics::Gdi::{MonitorFromWindow, MONITOR_DEFAULTTONEAREST};
    use windows::Win32::UI::WindowsAndMessaging::*;
    use windows::Win32::Graphics::Dwm::{DwmGetWindowAttribute, DWMWA_CLOAKED};

    unsafe extern "system" fn cb(hwnd: HWND, data: LPARAM) -> BOOL {
        let out = &mut *(data.0 as *mut Vec<WindowInfo>);
        // title
        let len = GetWindowTextLengthW(hwnd);
        let mut buf = vec![0u16; (len as usize) + 1];
        let n = GetWindowTextW(hwnd, &mut buf);
        let title = String::from_utf16_lossy(&buf[..n as usize]);
        // styles
        let ex = GetWindowLongW(hwnd, GWL_EXSTYLE) as u32;
        let is_tool = ex & WS_EX_TOOLWINDOW.0 != 0;
        let is_app = ex & WS_EX_APPWINDOW.0 != 0;
        let owner = GetWindow(hwnd, GW_OWNER).unwrap_or_default();
        let is_owned = !owner.is_invalid();
        let mut cloaked: u32 = 0;
        let _ = DwmGetWindowAttribute(hwnd, DWMWA_CLOAKED,
            &mut cloaked as *mut _ as *mut _, std::mem::size_of::<u32>() as u32);
        let probe = WindowProbe {
            title: title.clone(),
            is_visible: IsWindowVisible(hwnd).as_bool(),
            is_iconic: IsIconic(hwnd).as_bool(),
            is_toolwindow: is_tool, is_appwindow: is_app,
            is_owned, is_cloaked: cloaked != 0,
        };
        if is_manageable(&probe) {
            let mut r = RECT::default();
            let _ = GetWindowRect(hwnd, &mut r);
            let hmon = MonitorFromWindow(hwnd, MONITOR_DEFAULTTONEAREST);
            out.push(WindowInfo {
                hwnd: hwnd.0 as u64, title,
                rect: Rect { x: r.left, y: r.top, w: r.right - r.left, h: r.bottom - r.top },
                monitor_id: hmon.0 as u32,
                minimized: probe.is_iconic,
            });
        }
        TRUE
    }
    let mut out: Vec<WindowInfo> = Vec::new();
    unsafe { let _ = EnumWindows(Some(cb), LPARAM(&mut out as *mut _ as isize)); }
    let _ = monitors; // monitor_id already set from HMONITOR; kept for signature stability
    out
}
```
Add `Win32_Graphics_Dwm` to the `windows` features in `Cargo.toml`. `win/mod.rs`: `pub mod filter; pub mod enumerate;`.

- [ ] **Step 6: Document + verify + commit**

Fill `docs/win32.md` "Manageable window" section with the clause-by-clause rationale (mirror spec §4). Run: `cargo test --manifest-path src-tauri/Cargo.toml -v` (all pass, enumerate compiles).
```bash
git add -A
git commit -m "feat: manageable-window filter + EnumWindows probe"
```

---

### Task 4 (RAC-285): `translate_rect` (pure) + `move_window`

**Files:**
- Create: `src-tauri/src/win/move_window.rs`
- Modify: `src-tauri/src/win/mod.rs`
- Modify: `docs/win32.md` (document move semantics)
- Test: inline `#[cfg(test)]` in `move_window.rs`

**Interfaces:**
- Produces: `win::move_window::translate_rect(win: Rect, from: Rect, to: Rect) -> Rect` — pure; maps a window rect from `from` work-area to `to` work-area preserving relative offset+size, clamped to fit.
- Produces: `win::move_window::move_to_monitor(hwnd: u64, target: &MonitorInfo, from_work_area: Rect)` (`#[cfg(windows)]`) — restore-if-maximized, translate, `SetWindowPos`, re-maximize.
- Consumes: `model::{Rect, MonitorInfo}`.

- [ ] **Step 1: Write the failing tests**

`win/move_window.rs`:
```rust
use crate::model::{MonitorInfo, Rect};

#[cfg(test)]
mod tests {
    use super::*;
    fn wa(x: i32, y: i32, w: i32, h: i32) -> Rect { Rect { x, y, w, h } }

    #[test]
    fn preserves_relative_position_same_size_monitors() {
        // window 100px in from the top-left of a 1920x1040 work area,
        // moved to an identically sized work area at x=1920.
        let out = translate_rect(wa(100, 100, 800, 600), wa(0, 0, 1920, 1040), wa(1920, 0, 1920, 1040));
        assert_eq!(out, wa(2020, 100, 800, 600));
    }
    #[test]
    fn scales_offset_to_smaller_monitor_and_clamps_size() {
        // from a 3840x2160-ish area to a 1920x1080 area: a window near the
        // right edge stays proportional and never exceeds the target.
        let out = translate_rect(wa(3000, 0, 1200, 900), wa(0, 0, 3840, 2160), wa(0, 0, 1920, 1080));
        assert!(out.x >= 0 && out.y >= 0);
        assert!(out.x + out.w <= 1920, "right edge {} <= 1920", out.x + out.w);
        assert!(out.y + out.h <= 1080, "bottom edge {} <= 1080", out.y + out.h);
    }
}
```

- [ ] **Step 2: Run to verify it fails**

Run: `cargo test --manifest-path src-tauri/Cargo.toml move_window::tests -v`
Expected: FAIL — `translate_rect` not found.

- [ ] **Step 3: Implement `translate_rect`**

```rust
/// Map `win` from work area `from` to work area `to`, preserving the
/// window's relative position and size, clamped to fit inside `to`.
pub fn translate_rect(win: Rect, from: Rect, to: Rect) -> Rect {
    let sx = to.w as f64 / from.w as f64;
    let sy = to.h as f64 / from.h as f64;
    let mut w = ((win.w as f64) * sx).round() as i32;
    let mut h = ((win.h as f64) * sy).round() as i32;
    w = w.clamp(1, to.w);
    h = h.clamp(1, to.h);
    let rel_x = (win.x - from.x) as f64 * sx;
    let rel_y = (win.y - from.y) as f64 * sy;
    let mut x = to.x + rel_x.round() as i32;
    let mut y = to.y + rel_y.round() as i32;
    x = x.clamp(to.x, to.x + to.w - w);
    y = y.clamp(to.y, to.y + to.h - h);
    Rect { x, y, w, h }
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `cargo test --manifest-path src-tauri/Cargo.toml move_window::tests -v`
Expected: both PASS.

- [ ] **Step 5: Implement `move_to_monitor`**

```rust
#[cfg(windows)]
pub fn move_to_monitor(hwnd: u64, target: &MonitorInfo, from_work_area: Rect) -> Result<(), String> {
    use windows::Win32::Foundation::HWND;
    use windows::Win32::UI::WindowsAndMessaging::{
        GetWindowPlacement, GetWindowRect, SetWindowPos, ShowWindow,
        SWP_NOZORDER, SWP_NOACTIVATE, SW_RESTORE, SW_MAXIMIZE, SW_SHOWMINIMIZED,
        WINDOWPLACEMENT, SW_SHOWMAXIMIZED,
    };
    use windows::Win32::Foundation::RECT;
    let h = HWND(hwnd as *mut _);
    unsafe {
        let mut wp = WINDOWPLACEMENT { length: std::mem::size_of::<WINDOWPLACEMENT>() as u32, ..Default::default() };
        let was_max = GetWindowPlacement(h, &mut wp).is_ok()
            && wp.showCmd == SW_SHOWMAXIMIZED.0 as u32;
        if was_max { let _ = ShowWindow(h, SW_RESTORE); }
        let mut r = RECT::default();
        GetWindowRect(h, &mut r).map_err(|e| e.to_string())?;
        let cur = Rect { x: r.left, y: r.top, w: r.right - r.left, h: r.bottom - r.top };
        let dst = translate_rect(cur, from_work_area, target.work_area);
        SetWindowPos(h, None, dst.x, dst.y, dst.w, dst.h, SWP_NOZORDER | SWP_NOACTIVATE)
            .map_err(|e| e.to_string())?;
        if was_max { let _ = ShowWindow(h, SW_MAXIMIZE); }
        let _ = SW_SHOWMINIMIZED; // keep import list stable for future minimized handling
    }
    Ok(())
}
```
`win/mod.rs`: `pub mod move_window;`.

- [ ] **Step 6: Document + commit**

Add the move-semantics summary to `docs/win32.md` (mirror spec §7). Run: `cargo test --manifest-path src-tauri/Cargo.toml -v`.
```bash
git add -A
git commit -m "feat: translate_rect + move_to_monitor"
```

---

### Task 5 (RAC-286): App state + Tauri commands (`get_snapshot`, `move_window`, `move_self_to_monitor`, `focus_window`)

**Files:**
- Create: `src-tauri/src/state.rs`
- Create: `src-tauri/src/commands.rs`
- Modify: `src-tauri/src/lib.rs` (register state + `invoke_handler`)
- Modify: `docs/ipc.md` (document commands)
- Test: inline test for the state helper that finds a window's current monitor work-area.

**Interfaces:**
- Produces: `state::AppState { desktop: Mutex<Desktop> }` and `state::work_area_of(desktop: &Desktop, monitor_id: u32) -> Option<Rect>` (pure helper, tested).
- Produces Tauri commands: `get_snapshot() -> Desktop`, `move_window(hwnd: u64, monitor_id: u32) -> Result<(), String>`, `move_self_to_monitor(window: tauri::Window, n: u32) -> Result<(), String>`, `focus_window(hwnd: u64) -> Result<(), String>`.
- Consumes: Tasks 2–4.

- [ ] **Step 1: Write the failing test for `work_area_of`**

`src-tauri/src/state.rs`:
```rust
use crate::model::{Desktop, Rect};

pub fn work_area_of(desktop: &Desktop, monitor_id: u32) -> Option<Rect> {
    desktop.monitors.iter().find(|m| m.id == monitor_id).map(|m| m.work_area)
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::model::{MonitorInfo, Rect};
    #[test]
    fn finds_work_area_by_id() {
        let d = Desktop { monitors: vec![MonitorInfo {
            id: 42, number: 1, name: "m".into(),
            bounds: Rect { x: 0, y: 0, w: 1920, h: 1080 },
            work_area: Rect { x: 0, y: 0, w: 1920, h: 1040 } }], windows: vec![] };
        assert_eq!(work_area_of(&d, 42).unwrap().h, 1040);
        assert!(work_area_of(&d, 99).is_none());
    }
}
```

- [ ] **Step 2: Run to verify it fails**

Run: `cargo test --manifest-path src-tauri/Cargo.toml state::tests -v`
Expected: FAIL — module not wired.

- [ ] **Step 3: Add `AppState` + register it, make the test pass**

Append to `state.rs`:
```rust
use std::sync::Mutex;
#[derive(Default)]
pub struct AppState { pub desktop: Mutex<Desktop> }
```
In `lib.rs`: `pub mod state; pub mod commands;`, and in the builder `.manage(state::AppState::default())`.
Run: `cargo test --manifest-path src-tauri/Cargo.toml state::tests -v` → PASS.

- [ ] **Step 4: Implement the commands**

`commands.rs`:
```rust
use crate::model::Desktop;
use crate::state::{work_area_of, AppState};
use crate::win;
use tauri::State;

fn rescan() -> Desktop {
    #[cfg(windows)] {
        let monitors = win::monitors::enumerate();
        let windows = win::enumerate::windows(&monitors);
        Desktop { monitors, windows }
    }
    #[cfg(not(windows))] { Desktop::default() }
}

#[tauri::command]
pub fn get_snapshot(state: State<AppState>) -> Desktop {
    let d = rescan();
    *state.desktop.lock().unwrap() = d.clone();
    d
}

#[tauri::command]
pub fn move_window(hwnd: u64, monitor_id: u32, state: State<AppState>) -> Result<(), String> {
    let d = state.desktop.lock().unwrap().clone();
    let target = d.monitors.iter().find(|m| m.id == monitor_id)
        .ok_or_else(|| "unknown target monitor".to_string())?;
    let cur = d.windows.iter().find(|w| w.hwnd == hwnd);
    let from = cur.and_then(|w| work_area_of(&d, w.monitor_id))
        .unwrap_or(target.work_area);
    #[cfg(windows)] { win::move_window::move_to_monitor(hwnd, target, from)?; }
    #[cfg(not(windows))] { let _ = (hwnd, from); }
    Ok(())
}

#[tauri::command]
pub fn move_self_to_monitor(window: tauri::Window, n: u32, state: State<AppState>) -> Result<(), String> {
    let d = state.desktop.lock().unwrap().clone();
    let target = d.monitors.iter().find(|m| m.number == n)
        .ok_or_else(|| format!("no monitor #{n}"))?;
    #[cfg(windows)] {
        use windows::Win32::Foundation::HWND;
        let hwnd = window.hwnd().map_err(|e| e.to_string())?.0 as u64;
        let _ = HWND::default();
        let from = d.monitors.iter()
            .find(|m| m.id == /* current */ target.id).map(|m| m.work_area)
            .unwrap_or(target.work_area);
        win::move_window::move_to_monitor(hwnd, target, from)?;
    }
    #[cfg(not(windows))] { let _ = (window, target); }
    Ok(())
}

#[tauri::command]
pub fn focus_window(hwnd: u64) -> Result<(), String> {
    #[cfg(windows)] {
        use windows::Win32::Foundation::HWND;
        use windows::Win32::UI::WindowsAndMessaging::SetForegroundWindow;
        unsafe { let _ = SetForegroundWindow(HWND(hwnd as *mut _)); }
    }
    #[cfg(not(windows))] { let _ = hwnd; }
    Ok(())
}
```
Note for the implementer: for `move_self_to_monitor`, resolve the app window's *current* monitor via `MonitorFromWindow(hwnd, MONITOR_DEFAULTTONEAREST)` and look its work-area up in `d`; the placeholder `find(... target.id)` above is only to keep the snippet compiling — replace it with the real current-monitor lookup (a 3-line helper `current_monitor_work_area(&d, hwnd)` added to `state.rs` with its own unit test over a fixture desktop).
Register all four in `lib.rs`: `.invoke_handler(tauri::generate_handler![commands::get_snapshot, commands::move_window, commands::move_self_to_monitor, commands::focus_window])`.

- [ ] **Step 5: Add + test the `current_monitor_work_area` helper**

Add to `state.rs` a pure `current_monitor_work_area(desktop: &Desktop, hwnd: u64) -> Option<Rect>` that finds the window in `desktop.windows`, then its monitor's work-area; unit-test it against a fixture desktop (window on monitor 42 → returns that work-area; unknown hwnd → `None`). Use it in `move_self_to_monitor` (falling back to `MonitorFromWindow` only if the app window isn't in the snapshot — the app window normally is).

- [ ] **Step 6: Verify + document + commit**

Run: `cargo test --manifest-path src-tauri/Cargo.toml -v` and `npm run tauri dev` (app still opens). Document the four commands + their types in `docs/ipc.md`.
```bash
git add -A
git commit -m "feat: app state + IPC commands (snapshot/move/self-move/focus)"
```

---

### Task 6 (RAC-287): WinEvent hook thread + live `desktop:changed` events + supervision

**Files:**
- Create: `src-tauri/src/win/events.rs`
- Modify: `src-tauri/src/lib.rs` (spawn on `setup`, wire shutdown)
- Modify: `src-tauri/src/win/mod.rs`
- Modify: `docs/architecture.md` (document the hook thread)
- Test: a pure debounce/coalesce helper is unit-tested; the OS hook path is manual.

**Interfaces:**
- Produces: `win::events::should_emit(now_ms: u64, last_emit_ms: u64, debounce_ms: u64) -> bool` — pure coalescing gate, tested.
- Produces: `win::events::spawn(app: tauri::AppHandle) -> EventsHandle` — installs hooks on a dedicated thread, emits `desktop:changed` (payload `Desktop`) and `desktop:degraded` (payload `{ reason: String }`). `EventsHandle::shutdown(self)` unhooks + `PostThreadMessage(WM_QUIT)` + joins.
- Consumes: `commands::rescan`-equivalent (extract `rescan()` into `state.rs` as `pub fn rescan() -> Desktop` so both the command and the hook thread call it), Tauri `Emitter`.

- [ ] **Step 1: Write the failing test for the coalescing gate**

`win/events.rs`:
```rust
/// Return true if enough time passed since the last emit to emit again.
/// Used to coalesce LOCATIONCHANGE floods during a drag.
pub fn should_emit(now_ms: u64, last_emit_ms: u64, debounce_ms: u64) -> bool {
    now_ms.saturating_sub(last_emit_ms) >= debounce_ms
}

#[cfg(test)]
mod tests {
    use super::*;
    #[test] fn suppresses_within_window() { assert!(!should_emit(1050, 1000, 60)); }
    #[test] fn emits_after_window() { assert!(should_emit(1100, 1000, 60)); }
    #[test] fn first_emit_when_last_is_zero() { assert!(should_emit(60, 0, 60)); }
}
```

- [ ] **Step 2: Run to verify it fails, then passes**

Run: `cargo test --manifest-path src-tauri/Cargo.toml events::tests -v` → FAIL (module not wired) → wire `pub mod events;` in `win/mod.rs` → PASS.

- [ ] **Step 3: Extract `rescan` into `state.rs`**

Move the `rescan()` body from `commands.rs` into `state::rescan()` (pub) and have the command call it. Verify `cargo test` still green.

- [ ] **Step 4: Implement the hook thread**

Implement `win::events::spawn`:
- Spawn a `std::thread`. Inside, call `SetWinEventHook` for `EVENT_OBJECT_LOCATIONCHANGE`, `EVENT_OBJECT_CREATE`, `EVENT_OBJECT_DESTROY`, `EVENT_OBJECT_HIDE`, `EVENT_OBJECT_SHOW`, `EVENT_SYSTEM_FOREGROUND`, `EVENT_SYSTEM_MINIMIZESTART`, `EVENT_SYSTEM_MINIMIZEEND` with `WINEVENT_OUTOFCONTEXT | WINEVENT_SKIPOWNPROCESS`.
- Store the `AppHandle` + a `last_emit_ms` in a thread-local/`static` reachable from the `unsafe extern "system"` callback (a `OnceCell<Mutex<HookCtx>>`). The callback filters to `idObject == OBJID_WINDOW (0)` and `idChild == 0`, checks `should_emit`, and on pass calls `state::rescan()` and `app.emit("desktop:changed", desktop)`.
- Run `GetMessage` loop; on `WM_QUIT` exit, `UnhookWinEvent` each hook.
- Return `EventsHandle { thread_id, join_handle }`; `shutdown` calls `PostThreadMessageW(thread_id, WM_QUIT, 0, 0)` then `join`.
- Wrap the whole thread body so a panic emits `desktop:degraded { reason }` and starts a 1s polling loop (`loop { sleep(1s); emit rescan }`) until app exit.

Provide the full concrete implementation here (the executor writes it against the pinned `windows` version; the API names above are exact). Include `///` docs.

- [ ] **Step 5: Spawn on setup + shutdown on exit**

In `lib.rs` `.setup(|app| { let h = win::events::spawn(app.handle().clone()); app.manage(h); Ok(()) })` (guard with `#[cfg(windows)]`). On `RunEvent::Exit`, call `shutdown`.

- [ ] **Step 6: Manual verification + commit**

Run `npm run tauri dev`. In a terminal, open/move/minimize a couple of windows; confirm the app logs/receives `desktop:changed` (temporarily `console.log` in the frontend listener, or `println!` before emit). Document the thread in `docs/architecture.md`.
```bash
git add -A
git commit -m "feat: WinEvent hook thread with coalesced live desktop events + polling fallback"
```

---

### Task 7 (RAC-288): Frontend — desktop store + canvas scaling (pure) + MonitorMap render

**Files:**
- Create: `src/lib/scale.ts`, `src/lib/scale.test.ts`
- Create: `src/lib/ipc.ts`
- Create: `src/stores/desktop.ts`
- Create: `src/components/MonitorMap.vue`, `src/components/WindowTile.vue`
- Modify: `src/App.vue`
- Test: `src/lib/scale.test.ts` (Vitest)

**Interfaces:**
- Produces: `scale.ts` → `virtualBounds(monitors: MonitorInfo[]) -> Rect` and `toCanvas(r: Rect, bounds: Rect, canvas: { w: number; h: number }, pad: number) -> Rect` — pure; scale-to-fit the whole virtual desktop into the canvas preserving aspect ratio.
- Produces: `ipc.ts` typed wrappers: `getSnapshot(): Promise<Desktop>`, `moveWindow(hwnd, monitorId)`, `moveSelfToMonitor(n)`, `focusWindow(hwnd)`, `onDesktopChanged(cb)`, `onDegraded(cb)`.
- Produces: `useDesktopStore` holding `Desktop`, subscribing to events on mount.
- Consumes: `types.ts`, Task 5/6 IPC.

- [ ] **Step 1: Write the failing scaling test**

`src/lib/scale.test.ts`:
```ts
import { describe, it, expect } from 'vitest'
import { virtualBounds, toCanvas } from './scale'
import type { MonitorInfo } from './types'

const mon = (x: number, y: number, w: number, h: number): MonitorInfo =>
  ({ id: 1, number: 1, name: 'm', bounds: { x, y, w, h }, workArea: { x, y, w, h } })

describe('virtualBounds', () => {
  it('spans all monitors', () => {
    const b = virtualBounds([mon(0, 0, 1920, 1080), mon(1920, 0, 2560, 1440)])
    expect(b).toEqual({ x: 0, y: 0, w: 4480, h: 1440 })
  })
})

describe('toCanvas', () => {
  it('fits within the canvas and preserves aspect ratio', () => {
    const bounds = { x: 0, y: 0, w: 4480, h: 1440 }
    const r = toCanvas({ x: 0, y: 0, w: 1920, h: 1080 }, bounds, { w: 448, h: 300 }, 0)
    // scale = min(448/4480, 300/1440) = 0.1
    expect(r.w).toBeCloseTo(192, 0)
    expect(r.h).toBeCloseTo(108, 0)
    expect(r.x).toBeCloseTo(0, 0)
  })
})
```
(Note: TS types use `workArea` camelCase; `types.ts` mirrors Rust `work_area` with a serde `rename_all = "camelCase"` on the structs — add that attribute in Task 2's structs and note it in `docs/ipc.md`.)

- [ ] **Step 2: Run to verify it fails**

Run: `npm run test -- scale` → FAIL (no `scale.ts`).

- [ ] **Step 3: Implement `scale.ts`**

```ts
import type { MonitorInfo, Rect } from './types'

export function virtualBounds(monitors: MonitorInfo[]): Rect {
  if (monitors.length === 0) return { x: 0, y: 0, w: 1, h: 1 }
  const xs = monitors.map(m => m.bounds.x)
  const ys = monitors.map(m => m.bounds.y)
  const rs = monitors.map(m => m.bounds.x + m.bounds.w)
  const bs = monitors.map(m => m.bounds.y + m.bounds.h)
  const x = Math.min(...xs), y = Math.min(...ys)
  return { x, y, w: Math.max(...rs) - x, h: Math.max(...bs) - y }
}

export function scaleFactor(bounds: Rect, canvas: { w: number; h: number }, pad: number): number {
  return Math.min((canvas.w - 2 * pad) / bounds.w, (canvas.h - 2 * pad) / bounds.h)
}

export function toCanvas(r: Rect, bounds: Rect, canvas: { w: number; h: number }, pad: number): Rect {
  const s = scaleFactor(bounds, canvas, pad)
  return {
    x: pad + (r.x - bounds.x) * s,
    y: pad + (r.y - bounds.y) * s,
    w: r.w * s,
    h: r.h * s,
  }
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npm run test -- scale` → PASS.

- [ ] **Step 5: Implement ipc.ts, store, and MonitorMap**

`ipc.ts` wraps `@tauri-apps/api/core` `invoke` and `@tauri-apps/api/event` `listen` with the typed signatures above. `stores/desktop.ts` (plain reactive composable — no Pinia needed): on mount calls `getSnapshot()`, subscribes `onDesktopChanged`/`onDegraded`, exposes `desktop` and a `degraded` flag. `MonitorMap.vue`: a `<div class="canvas">` sized by a `ResizeObserver`; renders one monitor `<div>` per monitor via `toCanvas`, labels it with its `number`, and renders a `WindowTile` per window positioned via `toCanvas(window.rect, ...)`. `WindowTile.vue`: shows the title, a minimized indicator, and (Task 8) drag handlers. Wire `MonitorMap` into `App.vue` with the degraded banner.

- [ ] **Step 6: Verify + commit**

Run: `npm run test`, `npm run tauri dev` → the map shows your real monitors with window tiles in place, updating live as you move windows.
```bash
git add -A
git commit -m "feat: desktop store + scale-to-fit canvas + live MonitorMap"
```

---

### Task 8 (RAC-289): Drag-to-move + per-tile "send to →" menu

**Files:**
- Modify: `src/components/WindowTile.vue`, `src/components/MonitorMap.vue`
- Create: `src/components/SendToMenu.vue`
- Modify: `docs/architecture.md` (interaction notes)
- Test: `src/components/__tests__/MonitorMap.dragmove.test.ts` (Vitest + @vue/test-utils, mocked ipc)

**Interfaces:**
- Consumes: `moveWindow(hwnd, monitorId)` from `ipc.ts`; `virtualBounds`/`toCanvas`.
- Produces: a `monitorAtCanvasPoint(px, py, monitors, bounds, canvas, pad) -> MonitorInfo | null` pure helper (in `scale.ts`) so the drop-target hit-test is unit-tested.

- [ ] **Step 1: Write the failing hit-test + drag-move test**

Add to `scale.test.ts` a test for `monitorAtCanvasPoint` (a point inside monitor 2's drawn rect returns monitor 2). Add `MonitorMap.dragmove.test.ts`: mount `MonitorMap` with two mocked monitors + one window tile and a mocked `ipc.moveWindow`; simulate a pointerdown on the tile, pointermove into monitor 2's rect, pointerup; assert `moveWindow` was called with `(hwnd, monitor2.id)`.

- [ ] **Step 2: Run to verify it fails**

Run: `npm run test -- dragmove scale` → FAIL.

- [ ] **Step 3: Implement `monitorAtCanvasPoint` + drag logic**

Add the pure helper to `scale.ts`. In `WindowTile.vue`, implement pointer-based dragging (pointerdown captures, pointermove updates a floating ghost position, pointerup computes the drop monitor via `monitorAtCanvasPoint` and, if different from the window's current monitor, calls `moveWindow`). Guard against a no-op drop (same monitor).

- [ ] **Step 4: Run to verify it passes**

Run: `npm run test -- dragmove scale` → PASS.

- [ ] **Step 5: Add the send-to menu**

`SendToMenu.vue`: right-click (or a "⋯" button) on a tile opens a menu listing monitors by `number` + `name`; choosing one calls `moveWindow(hwnd, id)`. This is the keyboard/fallback path required by the spec.

- [ ] **Step 6: Verify + commit**

Run: `npm run test`; `npm run tauri dev` → drag a real window between monitors and confirm it moves; use the menu too. Document interactions in `docs/architecture.md`.
```bash
git add -A
git commit -m "feat: drag-to-move + send-to-monitor menu"
```

---

### Task 9 (RAC-290): Tray, close-to-tray, exit, DPI manifest

**Files:**
- Modify: `src-tauri/src/lib.rs` (tray + window-close handling)
- Create: `src-tauri/icons/tray.png` (reuse the app icon if present)
- Create/Modify: `src-tauri/manifest.xml` + `build.rs` (embed Per-Monitor-V2 manifest)
- Modify: `src-tauri/tauri.conf.json` (window: `visible: true`, no `skipTaskbar`)
- Modify: `docs/architecture.md`

**Interfaces:**
- Consumes: Tauri v2 `tray`, `menu`, window events.
- Produces: tray icon (left/double-click shows+focuses window; right-click menu → **Exit** quits); window close hides instead of quitting.

- [ ] **Step 1: DPI awareness**

Add a Windows application manifest declaring `<dpiAwareness>PerMonitorV2</dpiAwareness>` and embed it via `build.rs` (`embed-resource` or `winres`). As a belt-and-braces fallback, call `SetProcessDpiAwarenessContext(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2)` at the very start of `main()` (before Tauri init), `#[cfg(windows)]`.

- [ ] **Step 2: Close-to-tray**

In the builder, `.on_window_event(|window, event| if let WindowEvent::CloseRequested { api, .. } = event { api.prevent_close(); let _ = window.hide(); })`.

- [ ] **Step 3: Tray icon + menu**

Build a `TrayIconBuilder` with a menu containing a single **Exit** item; on `MenuEvent` for Exit, `app.exit(0)` (which triggers `RunEvent::Exit` → hook `shutdown` from Task 6). On tray `TrayIconEvent` left/double click, show + set focus + unminimize the main window.

- [ ] **Step 4: Verify manually**

Run `npm run tauri dev`: closing the window hides it (still in tray); tray click reopens it; tray → Exit fully quits (process ends, hook thread joins cleanly with no error in the console). On your mixed-DPI setup, confirm the map proportions are correct (DPI manifest working).

- [ ] **Step 5: Document + commit**

Update `docs/architecture.md` (lifecycle: close→hide, tray→exit; DPI). 
```bash
git add -A
git commit -m "feat: tray + close-to-tray + exit + Per-Monitor-V2 DPI"
```

---

### Task 10 (RAC-291): Self-move number hotkeys (focused-only)

**Files:**
- Create: `src/lib/hotkeys.ts`, `src/lib/hotkeys.test.ts`
- Modify: `src/App.vue` (install/remove the listener)
- Modify: `docs/architecture.md`
- Test: `src/lib/hotkeys.test.ts`

**Interfaces:**
- Produces: `digitToMonitorNumber(e: KeyboardEvent) -> number | null` — pure; maps `'1'..'9'` → 1..9, `'0'` → 10, else `null`; returns `null` if a modifier (ctrl/alt/meta) is held or the event target is an editable element.
- Produces: `installSelfMoveHotkeys(onPick: (n: number) => void): () => void` — attaches a `keydown` listener returning a disposer.
- Consumes: `moveSelfToMonitor(n)` from `ipc.ts`.

- [ ] **Step 1: Write the failing tests**

`src/lib/hotkeys.test.ts`:
```ts
import { describe, it, expect } from 'vitest'
import { digitToMonitorNumber } from './hotkeys'

const ev = (key: string, opts: Partial<KeyboardEvent> = {}, tag = 'DIV'): KeyboardEvent =>
  ({ key, ctrlKey: false, altKey: false, metaKey: false,
     target: { tagName: tag, isContentEditable: false }, ...opts } as unknown as KeyboardEvent)

describe('digitToMonitorNumber', () => {
  it('maps 1..9', () => { expect(digitToMonitorNumber(ev('3'))).toBe(3) })
  it('maps 0 to 10', () => { expect(digitToMonitorNumber(ev('0'))).toBe(10) })
  it('ignores non-digits', () => { expect(digitToMonitorNumber(ev('a'))).toBeNull() })
  it('ignores when ctrl held', () => { expect(digitToMonitorNumber(ev('3', { ctrlKey: true }))).toBeNull() })
  it('ignores inside inputs', () => { expect(digitToMonitorNumber(ev('3', {}, 'INPUT'))).toBeNull() })
})
```

- [ ] **Step 2: Run to verify it fails**

Run: `npm run test -- hotkeys` → FAIL.

- [ ] **Step 3: Implement `hotkeys.ts`**

```ts
/** Map a keydown to a 1-indexed monitor number, or null if it isn't a self-move key. */
export function digitToMonitorNumber(e: KeyboardEvent): number | null {
  if (e.ctrlKey || e.altKey || e.metaKey) return null
  const t = e.target as { tagName?: string; isContentEditable?: boolean } | null
  const tag = t?.tagName
  if (tag === 'INPUT' || tag === 'TEXTAREA' || t?.isContentEditable) return null
  if (e.key >= '1' && e.key <= '9') return e.key.charCodeAt(0) - '0'.charCodeAt(0)
  if (e.key === '0') return 10
  return null
}

/** Attach a focused-only keydown listener; returns a disposer. */
export function installSelfMoveHotkeys(onPick: (n: number) => void): () => void {
  const handler = (e: KeyboardEvent) => {
    const n = digitToMonitorNumber(e)
    if (n !== null) { e.preventDefault(); onPick(n) }
  }
  window.addEventListener('keydown', handler)
  return () => window.removeEventListener('keydown', handler)
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npm run test -- hotkeys` → PASS.

- [ ] **Step 5: Wire into App.vue**

`onMounted(() => { const off = installSelfMoveHotkeys(n => moveSelfToMonitor(n)); onUnmounted(off) })`. Because the listener is on the app's own `window`, it only fires while the app is focused — satisfying the spec's "only when the app is up and in focus."

- [ ] **Step 6: Verify manually + commit**

Run `npm run tauri dev`; focus the app and press `1`, `2`, `3` — the app window jumps to those monitors; press a digit with no matching monitor → no-op; focus another app and press digits → nothing happens. Document in `docs/architecture.md`.
```bash
git add -A
git commit -m "feat: focused-only number-key self-move"
```

---

### Task 11 (RAC-292): Documentation pass + README polish + VitePress build in check

**Files:**
- Modify: `docs/index.md`, `docs/architecture.md`, `docs/win32.md`, `docs/ipc.md`
- Modify: `README.md`
- Optional: `.github/workflows/ci.yml`

**Interfaces:** none (docs/CI only).

- [ ] **Step 1: Complete the docs pages**

Ensure each page is filled (not stub): `architecture.md` (module map, hook thread, lifecycle, coordinate/DPI model, interactions), `win32.md` (filter heuristic, move semantics, DPI), `ipc.md` (all four commands + two events + shared types with the camelCase note). Cross-link to the spec.

- [ ] **Step 2: Verify docs build**

Run: `npm run docs:build` → succeeds with no broken links.

- [ ] **Step 3: README final pass**

Confirm the feature list, roadmap, and dev commands match what shipped. Add a screenshot placeholder note (real screenshot added after first manual run).

- [ ] **Step 4: (Optional) CI**

Add a GitHub Actions workflow running `npm ci`, `npm run test`, `cargo test`, `cargo clippy -- -D warnings`, `npm run docs:build` on `windows-latest`.

- [ ] **Step 5: Commit + push**

```bash
git add -A
git commit -m "docs: complete Phase 1 documentation + README"
git push
```

---

## Self-Review

**Spec coverage:**
- §2 stack → Task 1. §3 tray/window lifecycle → Task 9. §3 live map → Tasks 6–7. §3 drag-move + send-to → Task 8. §3 self-move hotkeys → Task 10. §4 manageable filter → Task 3. §5 module layout → Tasks 2–7 (names match the spec's `win/` modules; `move.rs` is implemented as `move_window.rs` because `move` is a Rust keyword — noted). §6 hook thread → Task 6. §7 move/translate → Task 4. §8 coords/DPI/ordering → Tasks 2 (ordering), 7 (scaling), 9 (DPI). §9 hotkey detail → Task 10. §10 IPC surface → Tasks 5–6, mirrored in `ipc.ts` (Task 7). §11 error handling → Tasks 3/4/5/6 (Result-wrapped, vanished-HWND tolerance, hook supervision). §12 testing → every task's TDD steps + manual harness notes. §14 docs → Tasks 1, 3, 4, 5, 6, 8, 9, 10, 11.
- Gap check: no spec section is unassigned.

**Placeholder scan:** The only intentional "placeholder" is the `move_self_to_monitor` current-monitor lookup in Task 5 Step 4 — and Task 5 Step 5 replaces it with a tested `current_monitor_work_area` helper, so it does not ship as a placeholder. Task 6 Step 4 describes the hook thread in prose with exact API names rather than a full literal block because the exact `windows`-crate signatures depend on the pinned version resolved in Task 1; the interface, events, payloads, and shutdown are fully specified.

**Type consistency:** `Rect`/`MonitorInfo`/`WindowInfo`/`Desktop` are defined once (Task 2) and reused; TS mirrors them with `rename_all = "camelCase"` (`work_area` → `workArea`), noted in Tasks 2 and 7. Command names (`get_snapshot`, `move_window`, `move_self_to_monitor`, `focus_window`) and event names (`desktop:changed`, `desktop:degraded`) are identical across Tasks 5, 6, and 7. `translate_rect`, `order_monitors`, `is_manageable`, `should_emit`, `work_area_of`, `current_monitor_work_area`, `virtualBounds`, `toCanvas`, `monitorAtCanvasPoint`, `digitToMonitorNumber` are each defined in exactly one place and referenced by that name.

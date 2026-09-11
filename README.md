# Windows 11 Monitor Manager

A Windows 11 desktop tool that shows which open windows are on which
monitor — and lets you rearrange them.

Built as a full window manager, delivered in phases. **Phase 1** (in
progress) is a live, drag-to-rearrange monitor map.

> Status: early development. Phase 1 not yet shipped.

## Features (Phase 1)

- **Live monitor map** — a proportional, to-scale view of your physical
  monitor layout with every open window drawn as a tile in its real
  position, updating in real time as windows open, move, and minimize.
- **Drag to move** — drag a window from one monitor to another to
  actually move it. A per-window "send to →" menu is the keyboard path.
- **Number-key self-move** — while the app is focused, press `1`/`2`/`3`/…
  to send the app's own window to that monitor.
- **Lives in the tray** — closing the window hides it to the system
  tray; right-click the tray icon → **Exit** to quit fully.

## Roadmap

| Phase | Scope |
|-------|-------|
| **1** | Live monitor map, drag-to-move, self-move hotkeys, tray. |
| **2** | FancyZones-aware overlay (draw your PowerToys zones + snap-to-zone) and saved/named layouts. |
| **3** | Auto-place rules — send an app to a monitor/zone automatically when it opens. |

## Tech stack

- **[Tauri v2](https://tauri.app/)** — native shell, small binary.
- **Rust** + the official [`windows`](https://crates.io/crates/windows)
  crate — all Win32 window/monitor calls.
- **Vue 3 + TypeScript** + Vite — the UI.

## Development

> Prerequisites: [Rust](https://rustup.rs/), Node.js LTS, and the
> [Tauri prerequisites](https://tauri.app/start/prerequisites/) for
> Windows (WebView2, MSVC build tools).

```bash
npm install
npm run tauri dev      # run the app in dev mode
cargo test --manifest-path src-tauri/Cargo.toml   # Rust unit tests
npm run test           # frontend (Vitest) tests
npm run docs:dev       # docs site (VitePress) locally
```

## Documentation

Project documentation is a [VitePress](https://vitepress.dev/) site
under [`docs/`](./docs/), covering architecture, the Win32 internals,
and the IPC surface. The approved Phase 1 design lives at
[`docs/superpowers/specs/2026-09-11-monitor-manager-design.md`](./docs/superpowers/specs/2026-09-11-monitor-manager-design.md).

## License

MIT — see [LICENSE](./LICENSE).

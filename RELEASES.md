# natyv release notes

Release notes for every natyv release, newest first. Each release covers the whole toolchain: the
[CLI](https://github.com/natyv-io/cli), [core](https://github.com/natyv-io/core),
[shared](https://github.com/natyv-io/shared), the [SDKs](https://github.com/natyv-io/sdks), and
[ntx-lsp](https://github.com/natyv-io/ntx-lsp), which are all tagged together.

**Releases:** [v0.2.0](#v020)

---

## v0.2.0

Released 2026-09-28.
```bash
brew update && brew upgrade natyv
```

Go SDK: `go get github.com/natyv-io/sdks/go@v0.2.0`

### Highlights

#### Drawing with `<Canvas>`

A new `<Canvas>` widget for charts, diagrams, graphs, and any visual no widget covers. You describe
what to draw with a builder, and the host keeps that display list and redraws from it:

```go
func handleResize(w, h float32) error {
    return chart.Draw(func(d *widgets.Drawing) {
        d.Line(0, h/2, w, h/2, 1, axisColor)
        d.Circle(w/2, h/2, 6, widgets.ShapeStyle{Fill: &pointColor})
        d.Text(8, 8, "Throughput", labelColor, "left")
    })
}
```

- The builder supports lines, polylines, rects, rounded rects, circles, polygons, arcs and wedges, and
  text, all antialiased.
- In `.ntx`: `<Canvas onResize={handleResize} onClick={handleClick}/>`. Click handlers receive the `x, y`
  of the click, and resize handlers receive the canvas's new size.
- Drawings survive a memory recycle, so nothing needs redrawing.
- The host bounds every display list and rejects oversized ones instead of letting a guest grow host
  memory without limit.

#### System tray, and apps that outlive their window

- An app can now put an icon in the macOS menu bar or the Windows/Linux notification area.
- Tray menus support buttons, checkboxes, separators, and submenus (`widgets.CreateTray`).
- New window lifecycle controls let an app hide its window and keep running: `SetWindowVisible`,
  `SetQuitOnLastWindowClose`, and `OnStartupWindowClose`.
- On macOS this needed a fix to SDL's own Zig packaging. We've submitted it upstream, and natyv
  builds against our fork until it lands in a tagged release.

#### Text editing

`TextField` and `TextArea` now have a cursor and a selection model:
- click to place the caret;
- click-drag or Shift+arrow to select;
- Cmd/Ctrl+C/X/V copy, cut and paste through the system clipboard.

#### Raw HID devices

Talk to macropads, custom controllers, and other HID devices from the guest with the new `hid`
package in the Go SDK (`hid.Enumerate`, `hid.Open`, then write output reports).
- HID is off by default.
- You allow devices by vendor/product id in `conf.natyv.json`:
  ```json
  { "hid": { "enabled": true, "allowed_devices": [{ "vendor_id": "0x1234", "product_id": "0x5678" }] } }
  ```
- Input reports and disconnects arrive as events.
- Open devices stay open across a memory recycle.
- On macOS, the built app needs Input Monitoring permission. On Linux it needs a udev rule. natyv
  reports a missing permission as its own error.

#### Window and font customization

Set the window's background, its starting size, and an app-wide font, either in your `.ntss`
stylesheet:

```
window { backgroundColor: "#101014", width: 1280, height: 800 }
font { file: "Inter.ttf", size: 15 }
```

or in `conf.natyv.json` under `ui` (`background_color`, `width`, `height`, `font`, `font_size`).
- If both set the same value, the `.ntss` value wins.
- Font files live in your app's `assets/` directory.
- Per-widget fonts and font sizes are coming in v0.3.0.

### Smaller installs, measured startup

- **`brew install natyv` used to download 378 MB of platform dependencies that the install never
  used.** It now downloads none on the Homebrew path, and a developer build of the toolchain fetches
  31 MB.
- **`NATYV_STARTUP_TRACE=1`** prints how long each phase of startup took. For reference, natyv
  reaches a window in about 115–135 ms. If your app starts slower than that, the time is going to
  your own startup work (for the demo mail client, its IMAP connection).

### Security and robustness

We audited every host function a guest can call and fixed everything we found before this release:

- `max_widgets` is enforced again. Without that limit, a guest could trigger an out-of-bounds
  write in the host.
- Guest SQL can no longer `ATTACH` or otherwise reach files outside the app's own database.
- Persisted state and open windows are capped, and windows are no longer orphaned.
- Numbers from the guest are validated at the boundary. Non-finite and out-of-range values are
  rejected, where before they could reach unchecked arithmetic in release builds.
- File dialogs no longer drop large selections.
- Text truncation respects UTF-8 character boundaries.
- A recycle threshold set lower than the app's own post-recycle memory no longer recycles in a loop.
  natyv recycles once, raises the effective threshold, and logs a warning.

### Upgrading from v0.1.0

- **Labels and Buttons now size to their content by default.** v0.1.0 gave them fixed sizes (24 px
  tall Labels, 120×32 px Buttons). If your layout relied on those sizes, set `height`/`width` in
  the widget's style token.
- **Your first `natyv build` after upgrading recompiles your app.** The build cache is now keyed on
  the `natyv` binary itself, so a stale wasm built by an older toolchain is never reused. The same
  applies to locally built CLIs. `--force` is only needed as an escape hatch now.
- **Handlers generated from `.ntx` survive a recycle together.** A tag with several handlers (for
  example a Canvas with `onClick` and `onResize`) gets a single recycle binding that reattaches all
  of them. If you call `natyv.RegisterBinding` by hand, a widget keeps only one binding, so register
  one kind that reattaches every handler.

### Also new

- [`ntx-lsp`](https://github.com/natyv-io/ntx-lsp) has its first tagged release, and it understands
  `<Canvas>`.

**Full changelogs:**
[core](https://github.com/natyv-io/core/compare/v0.1.0...v0.2.0) ·
[cli](https://github.com/natyv-io/cli/compare/v0.1.0...v0.2.0) ·
[shared](https://github.com/natyv-io/shared/compare/v0.1.0...v0.2.0) ·
[sdks](https://github.com/natyv-io/sdks/compare/v0.1.0...v0.2.0)

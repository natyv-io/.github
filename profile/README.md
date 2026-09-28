<p align="center">
  <img src="https://github.com/natyv-io/.github/blob/main/profile/logo.png" alt="natyv" height="200" />
</p>

<h1 align="center">natyv</h1>
<p align="center"><strong>A native desktop app runtime with no bundled browser.</strong></p>

<p align="center">
  <!-- A static badge, not the shields.io live-member-count one (that variant needs
       Discord's numeric server/guild ID, not an invite code, plus the server widget
       enabled under Server Settings -> Widget -- swap to that later if wanted. -->
  <a href="https://discord.gg/xGu2N5e5dq"><img src="https://img.shields.io/badge/Discord-Join-1F6F5C?logo=discord&logoColor=white" /></a>
  <a href="https://github.com/natyv-io/core"><img src="https://img.shields.io/badge/tests-595%2B%20passing-1F6F5C" /></a>
  <a href="https://pkg.go.dev/github.com/natyv-io/sdks/go"><img src="https://img.shields.io/badge/pkg.go.dev-reference-1F6F5C?logo=go&logoColor=white" /></a>
  <a href="https://github.com/natyv-io/homebrew-natyv"><img src="https://img.shields.io/badge/install-brew-1F6F5C?logo=homebrew" /></a>
  <a href="https://github.com/natyv-io/.github/blob/main/RELEASES.md"><img src="https://img.shields.io/badge/release%20notes-v0.2.0-1F6F5C" /></a>
</p>

---

## What is natyv?

natyv is a native desktop app runtime. Your app's logic runs as a WebAssembly guest module (via
[Extism](https://extism.org)) inside a native host that natyv provides. The host owns the window and
render loop; your guest code declares the UI and business logic and talks to the host through a
small, capability-scoped set of host functions.

No bundled Chromium. No DOM. No JavaScript runtime shipped inside your app unless you actually write
your logic in JavaScript.

## Why natyv?

- **Leaner than Electron, with the numbers to show it.** We built the same mail client three ways —
  natyv, Electron, and Tauri — and benchmarked all three back to back, same machine, same workload:

  | | natyv | Electron | Tauri |
  |---|---|---|---|
  | Disk size | 27 MB | 244 MB | **11 MB** |
  | Idle memory (RSS) | **58 MB** | 331 MB | 69 MB |
  | Idle CPU | **0.0%** | 0.0% | 0.0% |

  natyv's idle-CPU number wasn't a given — the first real measurement caught it sustaining 70-93% CPU
  continuously at idle, a genuine bug found by benchmarking against real alternatives instead of
  testing in isolation. It's fixed now, and the fix (and the regression it caught along the way) is a
  real commit history, not a claim.
- **Handles WebAssembly's own memory ceiling instead of ignoring it.** WASM linear memory can only
  grow — there's no `memory.shrink` in the spec, on any shipping engine — so a long-running WASM-hosted
  app leaks by construction unless something recycles it. natyv's host checkpoints your app's state,
  recycles the guest instance, and resumes automatically once memory crosses a threshold you set,
  measured at ~30ms end to end, with nothing on screen even rebuilding. See below for what makes this
  safe to turn on without hand-rolling your own checkpoint logic.
- **More language-agnostic than most "native shell" frameworks.** The interop boundary is
  WebAssembly via Extism, not one specific native language bolted onto a webview. Your app logic
  runs in whichever guest language has (or gets) an Extism PDK, not just whatever the framework's
  own backend happens to be written in.
- **Ships to more than the OS you developed on.** `natyv build` cross-compiles the very same guest
  code to Windows and Linux, not just macOS — both verified in real VMs (a real window, real event
  dispatch, clean shutdown), not just "should work."
- **Honest about the real cost.** `libextism` (Wasmtime + Cranelift under the hood) is a genuine,
  non-trivial dependency, so natyv isn't "basically free." What it buys you is a runtime with no
  browser engine anywhere in the picture, which is the specific tax this architecture is built to
  avoid regardless of which language your app logic is written in.

## Installation

**macOS** (via [Homebrew](https://brew.sh)):

```bash
brew tap natyv-io/natyv
brew install natyv
```

**Linux and Windows**: build the toolchain from source for now (brew formulas are macOS-first); apps
you build with it can already target Windows and Linux via cross-compilation — see above.

## Prerequisites

- [Zig](https://ziglang.org/download/) 0.16.0. `natyv` checks your installed version itself and
  reports a clear error if it doesn't match, rather than failing confusingly from inside natyv-core's
  own build.

## Creating an app

Once natyv is installed, scaffolding and building a real app looks like this:

```bash
natyv init
```

```
What language is your guest code in? (go)
> go
```

This scaffolds a real, working starter app in the current directory: `conf.natyv.json`, a `guest/`
Go module wired up against the Go SDK, and a starter `guest/app.go.ntx`:

```go
package main

import "github.com/natyv-io/sdks/go/widgets"

expose App

func App(parent widgets.Container) error {
    <Container>
        <Label>Hello from natyv!</Label>
    </Container>
}
```

Then build and run it:

```bash
natyv build
./dist/<your-app-name>
```

`natyv build` transpiles `.ntx` into real Go, compiles your guest code to WebAssembly, and bundles
everything into one self-contained native binary. No separate runtime install is needed on the
machine you hand it to. That's it: a real window, running your own code, with zero browser engine
involved.

### A real interaction, not just a label

`.ntx` isn't just markup — event handlers are real guest functions, and styling is real named
stylesheet tokens resolved at build time, not inline CSS:

```go
var count int
var countLabel widgets.Label

func onClick() error {
    count++
    return countLabel.SetText(fmt.Sprintf("Clicked %d times", count))
}

func App(parent widgets.Container) error {
    <Container styles={root}>
        <Label ref={&countLabel} styles={field}>Clicked 0 times</Label>
        <Button styles={primaryBtn} onClick={onClick}>Click me</Button>
    </Container>
}
```

(`root`, `field`, and `primaryBtn` are named tokens resolved from your app's own stylesheet — this is
the exact pattern [`mail-natyv`](https://github.com/natyv-io/mail-natyv) uses throughout, not a
simplified version for this README.)

### See a full app, not a toy

[`mail-natyv`](https://github.com/natyv-io/mail-natyv) is a real IMAP/SMTP mail client built on
natyv — send, read, delete, paginate — and the natyv leg of the benchmark above. It's also where a
real, hand-rolled TCP/TLS + IMAP/SMTP stack for WASM guests lives, since existing Go mail libraries
assume `crypto/tls`, which doesn't exist under this target.

## Memory reclamation, handled for you

Recycling a guest instance only works if everything your app actually cares about survives the swap —
otherwise you've just traded a memory leak for a broken app. natyv's toolchain does that bookkeeping
for you, in two pieces:

Wrap any state you want to survive a recycle in `natyv.Persisted[T]` — a plain generic value, not a
framework-owned type you have to restructure your app around:

```go
var clickCount = natyv.Persisted[int]("clickCount", 0)

func onClick() error {
    v, err := clickCount.Get()
    if err != nil {
        return err
    }
    return clickCount.Set(v + 1)
}
```

Event handlers need no annotation at all: the `.ntx` transpiler automatically generates the code that
reattaches every `onClick=`/`onChange=`/etc. handler in your app to its live widget after a recycle —
for every widget kind the SDK ships, not just the simple ones. You write `onClick={handler}` once,
same as always; there's no separate registry to hand-maintain.

Turn recycling on with one field in `conf.natyv.json`:

```json
{ "memory": { "recycle_threshold_mb": 150 } }
```

## Repos

| Repo | What it is |
|---|---|
| [core](https://github.com/natyv-io/core) | The runtime: what a natyv app actually ships as and runs on. |
| [cli](https://github.com/natyv-io/cli) | The `natyv` CLI itself: `natyv init`/`build`/`prepare`/`get`/`bind`. |
| [sdks](https://github.com/natyv-io/sdks) | Guest-side SDKs, one subdirectory per language (Go today, including hand-rolled TCP/IMAP/SMTP). |
| [shared](https://github.com/natyv-io/shared) | Code shared between `core` and `cli` (the config schema and `.ntx` transpiler). |
| [mail-natyv](https://github.com/natyv-io/mail-natyv) | A real demo app — see above. |

Editor support (real diagnostics, semantic highlighting, hover/go-to-definition — not just syntax
coloring) lives in its own set of repos per editor: [`vscode-ntx`](https://github.com/natyv-io/vscode-ntx),
[`zed-ntx`](https://github.com/natyv-io/zed-ntx), and [`jetbrains-ntx`](https://github.com/natyv-io/jetbrains-ntx),
all built on the shared [`ntx-lsp`](https://github.com/natyv-io/ntx-lsp) language server.

## What's next

natyv is under active development. What's coming, roughly in the order it's being built:

**Per-widget fonts and font sizes.** Custom fonts today replace natyv's built-in font app-wide. In
v0.3.0, each widget can pick its own font and size.

**Accessibility.** The honest state of things: because natyv renders its own widgets instead of
hosting a browser engine, it inherits none of a webview's accessibility for free. Keyboard navigation
works today; screen reader support does not yet. It's the biggest item on this list and it's being
built on [AccessKit](https://accesskit.dev), which bridges to VoiceOver, NVDA/Narrator, and Orca
through each platform's native accessibility API.

## App signing, coming as a paid service

Building for Windows and Linux is free and open. Getting a macOS binary through Gatekeeper and a
Windows binary past SmartScreen is its own tax — one most solo devs and small teams pay in
notarization credentials, signing certs, and toolchain plumbing rather than in code. natyv's paid
signing service is designed around a narrow, honest boundary: **you send us an already-built binary,
we sign and notarize it with credentials you own — we never see your source or your build
artifacts.**

<p align="center">
  <a href="https://tally.so/r/68BbRB"><img src="https://img.shields.io/badge/Join%20the%20waitlist-App%20signing%20service-1F6F5C" /></a>
</p>

## Community

💬 Discord: [discord.gg/xGu2N5e5dq](https://discord.gg/xGu2N5e5dq)

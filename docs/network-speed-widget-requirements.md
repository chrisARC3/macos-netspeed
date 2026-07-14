# Network Speed Monitor — Requirements

**Version:** 0.9
**Date:** July 13, 2026
**Owner:** Chris
**Status:** Requirements gathering complete. Every value below is an explicit decision — no silent defaults. Ready to scope v1 into build increments.
**Changelog:** v0.9 (Jul 13, 2026) — **Removed FR-15 (launch at login).** The app no longer manages launch-at-login (dropped the planned `SMAppService`/ServiceManagement login-item registration); anyone who wants NetSpeed to start at login can add it in **System Settings → General → Login Items**. Chris's call, to keep v1 minimal and SwiftUI-only and avoid the login-item registration and its log-out/log-in testing overhead. Synced the rest of the doc: §2 drops "starts on its own," §3 gains a matching non-goal, §6's launch-at-login note and the `SMAppService` API mention are updated, and FR-16's "reopen" note no longer points at launch-at-login. **Increment 4.2 is removed** from the build plan (no code had been written). v0.8 (Jul 12, 2026) — Reverted the v0.7 centered-title approach (hidden native title bar + custom centered title): the extra vertical space raised the minimum window height too much. Back to the **standard native title bar**; **FR-18** now specifies the title text is the app name + **major.minor version** (e.g. "NetSpeed 1.0"), read from the bundle's marketing version. v0.7 (Jul 12, 2026) — Added **FR-18**: the main window title (**"NetSpeed"**) is **horizontally centered** — because macOS doesn't expose native title-bar text alignment to SwiftUI, the native title-bar text is hidden (SwiftUI `.hiddenTitleBar`) and a centered title is drawn in the window content, keeping the app SwiftUI-only. Also synced §6 (Technical Notes) to the SwiftUI-only reality — the window is a SwiftUI `Window` scene and frame persistence uses automatic window restoration (no `NSWindow`/`setFrameAutosaveName`). v0.6 (Jul 12, 2026) — Build **targets Apple Silicon (`arm64`) only** — no Intel (`x86_64`) slice, no universal binary — recorded in §6; consistent with the single-machine, personal-use scope (§9). v0.5 (Jul 11, 2026) — FR-6 readout format changed to **`X.X Mbps (Y.Y MBps)`** (Mbps primary, MBps a parenthetical secondary; was `X.X/Y.Y Mbps/MBps`), rendered in fixed-width tabular-digit fields; FR-17 now specifies the numeric readout is **horizontally centered** in the window. Graph axis labels settled: x-axis shows the window endpoints (`60s`…`now`), y-axis is auto-scaled Mbps with 0 pinned at the bottom. Color palette finalized to match Activity Monitor — blue for download (in), red for upload (out) — on both the graph lines and the readout arrows; direction labels right-aligned. v0.4 (Jul 10, 2026) — FR-10 changed from primary-interface-only to **summing all physical interfaces (`en*`)**, after an SMB transfer over a secondary Wi-Fi interface proved invisible while Activity Monitor showed it; now matches Activity Monitor's total. v0.3 — dual-unit `X.X/Y.Y Mbps/MBps` readout (FR-5, FR-6). v0.2 — initial locked requirements.

---

## 1. Overview

A lightweight macOS app that displays real-time network throughput (download and upload) in a small floating desktop window. It measures actual traffic passively — it never generates its own network load — and shows a short rolling graph so recent spikes and dips are visible at a glance.

This is a personal-use utility, built in Swift with Xcode.

## 2. Goals

- See current upload/download speed at a glance without opening Activity Monitor or a browser.
- Zero added bandwidth — the tool observes, it does not test.
- Stay out of the way: small, unobtrusive, remembers where I put it.

## 3. Non-Goals (v1)

- No speed tests / max-capacity measurement.
- No latency/ping, total data usage, or per-app breakdown.
- No long-term logging or historical storage to disk.
- No menu bar presence, Notification Center widget, or multi-window support.
- No built-in launch-at-login; a user who wants NetSpeed to start at login can add it via System Settings → General → Login Items. *(v0.9 — was FR-15.)*

## 4. Functional Requirements

### 4.1 Form factor
- FR-1: The app presents a single small **floating desktop window** the user can position anywhere on screen.
- FR-2: The window uses a **normal window level** (not forced always-on-top) in v1. *(Deferred — see §7.)*
- FR-3: The window is **opaque** in v1 (no adjustable transparency). *(Deferred — see §7.)*

### 4.2 What it measures and shows
- FR-4: Display **live download throughput** and **live upload throughput**, updated each sample.
- FR-5: Speeds are shown in **both Mbps (megabits per second)** — byte counters × 8 — **and MBps (megabytes per second)** — bytes ÷ 1,000,000 (equivalently, Mbps ÷ 8). Mbps compares directly to advertised ISP plans; MBps reflects real file-transfer rates. Both use the decimal 1,000,000 basis, consistent with §8 item 1.
- FR-6: Each direction's readout shows both figures to **one decimal place** in the format **`X.X Mbps (Y.Y MBps)`** (e.g. `45.3 Mbps (5.7 MBps)`), where `X.X Mbps` is the primary figure and `Y.Y MBps` a secondary parenthetical. Rendered with monospaced (tabular) digits in fixed-width fields so the readout doesn't shift as values change magnitude (NFR-3). *(v0.5 — was `X.X/Y.Y Mbps/MBps` in v0.3–v0.4.)*
- FR-7: Display a **rolling graph** (sparkline) of recent throughput, held **in memory only** — nothing is written to disk.
  - FR-7a: Graph window length: **60 seconds** of history.
  - FR-7b: Download and upload are drawn as **two separate lines**, each a distinct color, with labels/units.

### 4.3 How it measures
- FR-8: Throughput is measured **passively** by reading the OS network interface byte counters each interval and computing the delta over elapsed time. The app generates no traffic of its own.
- FR-9: **Sampling interval is user-adjustable from 1 to 5 seconds.**
  - FR-9a: Default on first launch: **1 second**.
  - FR-9b: The chosen rate stays **fixed until the user changes it** (no auto-variation).
  - FR-9c: The rate is changed via an in-app control (slider or menu); the choice is remembered across launches.

### 4.4 Which interface
- FR-10: Measure throughput **summed across all active physical network interfaces** (Ethernet, Wi-Fi, and wired adapters — BSD `en*`), so the reading reflects **total** network activity regardless of which interface carries it (matching Activity Monitor's total). This covers multiple interfaces being active at once — e.g. an SMB mount bound to Wi-Fi while the internet default route is on Ethernet. **Loopback and virtual interfaces are excluded** (VPN `utun*`, AirDrop `awdl*`/`llw*`, bridges): VPN traffic is already counted once on the physical interface it rides, so counting the tunnel too would double-count. *(v0.4 — was "monitor the single primary interface" through v0.3; see changelog.)*

### 4.5 Window behavior
- FR-11: **Remember window position** across launches and restarts.
- FR-12: **Remember window size** across launches and restarts.
- FR-13: On the **very first launch**, the window appears in the **top-left** of the screen; thereafter it reopens where the user last left it.
- FR-14: **Default window size is medium** (sized to show the numbers plus a legible 60-second two-line graph) and the window is **resizable**.
- FR-15: *(Removed in v0.9.)* **Launch at login is not managed by the app.** NetSpeed does not register itself as a login item; a user who wants it to start at login can add NetSpeed in **System Settings → General → Login Items**. Dropped to keep v1 minimal and avoid the `SMAppService`/ServiceManagement machinery. *(Was: "the app starts automatically when the user logs in.")*
- FR-16: **Closing the window quits the app** (confirmed). Because there is no menu bar item in v1, the window is the entire app; to reopen, relaunch it. *(See §8.)*

### 4.6 Layout
- FR-17: Content layout is **numbers on top, graph below**: current download/upload figures at the top, the two-line 60-second rolling graph beneath them with labels/units. The numeric readout is **horizontally centered** in the window and stays centered as the window is resized. *(v0.5)*
- FR-18: The main window uses the **standard native title bar**, titled with the app name followed by its **major.minor version** — e.g. **"NetSpeed 1.0"** — read from the bundle's marketing version (`CFBundleShortVersionString`) so it tracks the project version automatically. The native title bar's default alignment (left-aligned on macOS 26 Tahoe) is accepted. *(v0.8 — supersedes the v0.7 attempt at a horizontally-centered custom title via a hidden title bar, which was reverted because the extra vertical space raised the minimum window height too much.)*

## 5. Non-Functional Requirements

- NFR-1: **Low footprint** — minimal CPU and memory; polling interface counters at 1–5 s is cheap.
- NFR-2: **No elevated privileges** — reading interface counters and detecting the active interface must work without admin rights or a helper daemon.
- NFR-3: **Readable at a glance** — clear numbers, legible at the default medium size and when resized smaller.
- NFR-4: **Resilient** — if no interface is active or counters are briefly unavailable, show a sensible empty/zero state rather than crashing.
- NFR-5: **Respects system appearance** — the UI follows the system **light/dark** setting.

## 6. Technical Notes (candidate approaches to validate in design)

- **Language / UI:** Swift + SwiftUI throughout — **no AppKit/`NSWindow` hosting**. The floating window is a SwiftUI `Window` scene: `.defaultSize` sets the medium default (FR-14) and `.windowResizability(.contentMinSize)` allows resizing down to the content minimum (FR-14, NFR-3). It uses the standard native title bar, titled with the app name plus its major.minor version (e.g. "NetSpeed 1.0") from the bundle version (FR-18). *(v0.7 — the original plan hosted the window in an `NSWindow` for positioning control; the SwiftUI-only `Window` scene proved sufficient through Increments 3.1–3.2. v0.8 reverted a brief `.hiddenTitleBar` custom-title experiment.)*
- **Project identity:** product name `NetSpeed`; organization identifier `com.arc3solutions`; bundle identifier `com.arc3solutions.NetSpeed`.
- **Minimum OS:** **macOS 26 Tahoe** or later. All modern APIs assumed available (SwiftUI Charts).
- **Architecture:** **Apple Silicon (`arm64`) only.** The Xcode target builds `arm64` exclusively (`ARCHS = arm64`) — no Intel (`x86_64`) slice and no universal binary — matching the single-machine, personal-use scope (§9). *(v0.6)*
- **Throughput source:** interface byte counters via `getifaddrs()` (`ifi_ibytes` / `ifi_obytes`) or `sysctl` (`NET_RT_IFLIST2`), polled each interval; throughput = byte delta ÷ elapsed time. No special entitlements required.
- **Interface scope:** sum byte counters across all physical interfaces (BSD `en*`) from `getifaddrs()` each interval; loopback and virtual/VPN interfaces excluded. *(v0.4 change — an earlier design used the SystemConfiguration framework to follow a single primary interface; see FR-10.)*
- **Remember position & size:** macOS **automatic window restoration** for the SwiftUI `Window` scene saves and restores the frame (position and size) across launches — no explicit persistence code. *(v0.7 — supersedes the original `NSWindow` `setFrameAutosaveName`/`UserDefaults` approach; SwiftUI-only, verified in Increment 3.2. Rides the system's window-restoration behavior, honoring the "Close windows when quitting an application" setting.)*
- **Launch at login:** *(Removed in v0.9 — see FR-15.)* Delegated to the user via **System Settings → General → Login Items**; the app does not use `SMAppService`.
- **Rolling graph:** in-memory ring buffer of the last 60 s of samples, rendered with SwiftUI Charts (two line series).
- **Sampling control:** timer whose interval is bound to a user setting (1–5 s), persisted in `UserDefaults`.

## 7. Deferred / Future Considerations

Explicitly out of v1, noted so the design doesn't paint us into a corner:

- Always-on-top window level.
- Adjustable window transparency.
- Latency/ping, total data usage, and scheduled speed tests.
- Manual interface selection (pick a specific interface).
- Additional units (MB/s, auto-scaling).
- Auto-adaptive sampling (faster when busy, slower when idle).
- Persistent/long-term logging and longer history windows.
- A menu bar item (would allow reopening the window without relaunching).

## 8. Decisions Made Without a Direct Question (please veto if any are wrong)

These small items weren't asked as separate questions; I set sensible values so nothing is left implicit:

1. **Mbps means bits per second** (byte counters × 8), matching ISP advertising — not megabytes.
2. **Numeric format:** one decimal place, shown as `X.X Mbps (Y.Y MBps)` (e.g. `45.3 Mbps (5.7 MBps)`) — per FR-6 (updated in v0.5; was `X.X/Y.Y Mbps/MBps` in v0.3–v0.4).
3. **Closing the window quits the app** (FR-16) — **confirmed by Chris**. v1 has no menu bar item to reopen it from.
4. **UI follows system light/dark appearance** (NFR-5).
5. **Download and upload use two distinct colors, matching Activity Monitor:** blue for download (data in) and red for upload (data out) — applied to both the graph lines and the readout arrows. (Palette finalized in v0.5; supersedes the earlier "finalized in design" placeholder.)

## 9. Assumptions

- Single user, single machine, personal use — no distribution, App Store, or sandboxing constraints assumed for v1.
- The rolling graph is transient; quitting the app discards history by design.

---

*Requirements gathering is complete. Next step (per the incremental approach): break v1 into small, verifiable build increments before opening Xcode.*

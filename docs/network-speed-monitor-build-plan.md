# Network Speed Monitor — Incremental Build Plan

**Version:** 0.1
**Date:** July 10, 2026
**Companion to:** `network-speed-widget-requirements.md` (v0.5)
**Approach:** Small, verifiable increments. Each one has a single goal, a concrete build step, and a way to confirm it works before moving on. We do not start an increment until the previous one is verified.

---

## How to read this

Each increment lists:

- **Goal** — the one thing this step proves.
- **Build** — what gets added.
- **Verify** — the observable check that it works.
- **Done when** — the pass condition.

Requirement IDs (FR-#, NFR-#) reference the requirements doc so we can trace coverage.

A quick way to generate traffic for testing throughout: download a large file (e.g. a big OS update or a public test file) or run a video stream, and watch the numbers respond.

---

## Phase 0 — Project skeleton

### Increment 0.1 — Create and run an empty app  ✅ COMPLETE (Jul 10, 2026)
- **Goal:** A buildable macOS SwiftUI app that launches and shows a window.
- **Build:** New Xcode project → macOS App, SwiftUI lifecycle, Swift language. Product name `NetSpeed`; organization identifier `com.arc3solutions`; bundle identifier `com.arc3solutions.NetSpeed`. Deployment target macOS 26 Tahoe. Project saved inside `network-speed-monitor/NetSpeed/`.
- **Verify:** Build & run. An empty window appears; the app quits cleanly.
- **Done when:** Clean build, window shows, no console errors.
- **Result:** Built, ran, and quit successfully. macOS 26.0 confirmed available and selected as the minimum deployment target.

---

## Phase 1 — Core measurement (console only, no real UI yet)

### Increment 1.1 — Read interface byte counters once  ✅ COMPLETE (Jul 10, 2026)
- **Goal:** Prove we can read cumulative bytes in/out from the OS. (FR-8)
- **Build:** A small function using `getifaddrs()` that returns total `ifi_ibytes` / `ifi_obytes` for the active interfaces. Call it once at launch and `print()` the raw totals.
- **Verify:** Console prints two large, plausible byte counts.
- **Done when:** Numbers print and are non-zero on a machine that's been online.
- **Result:** Verified Jul 10, 2026. Compiled and ran the actual `NetworkCounters.swift` logic on this machine (real source + a small driver): it printed large, non-zero counters for the active interface **`en0`** — in ≈ 2,713,996,288 B (~2.7 GB), out ≈ 2,570,272,768 B (~2.5 GB) — alongside all other interfaces (unused ones read 0, as expected). The full `NetSpeed` app target also builds, links, and code-signs cleanly (`** BUILD SUCCEEDED **`) with the `.onAppear` snapshot call wired in, and launches. The app's own `print()` is best viewed in the Xcode console; it emits the same snapshot. Confirmed in Xcode by Chris under the App Sandbox (`ENABLE_APP_SANDBOX = YES`): the running app printed non-zero `en0` (in 3,347,009,536 / out 1,106,627,584), which also validates **NFR-2** (reads counters without elevated privileges). By then `en0`'s *out* counter had wrapped back past 4 GiB since the standalone run — the exact 32-bit `getifaddrs` wrap behavior we documented, now handled explicitly in Increment 1.2's delta math.

### Increment 1.2 — Poll on a timer and compute Mbps  ✅ COMPLETE (Jul 10, 2026)
- **Goal:** Turn raw counters into live download/upload throughput. (FR-4, FR-5, FR-8)
- **Build:** A 1-second timer that reads counters, computes `(bytesNow − bytesLast) × 8 ÷ seconds ÷ 1_000_000` for down and up, and prints them per direction in the dual-unit format `↓ X.X/Y.Y Mbps/MBps  ↑ X.X/Y.Y Mbps/MBps` (FR-6, per requirements v0.3).
- **Verify:** Start a large download; the download figure climbs and settles; it drops when the download stops.
- **Done when:** Console figures track real activity and read ~0 when idle.
- **Result:** Verified Jul 10, 2026. Chris ran it in Xcode — download/upload update every second in the console and match Apple's Activity Monitor; idle reads ~0. Headless pre-check confirmed the delta→Mbps math (0.0 Mbps idle vs 149.0 Mbps during a 50 MB test download; byte-delta arithmetic exact) and a warning-free build. The per-interface wrap guard handles the 32-bit `getifaddrs` counter wrap. Interface scope is a temporary non-loopback aggregate; Increment 1.3 narrows it to the primary interface. Readout extended to dual-unit `Mbps/MBps` per the v0.3 requirements change.

### Increment 1.3 — Follow the active (primary) interface  ✅ COMPLETE (Jul 10, 2026)
- **Goal:** Automatically read whichever interface is actually carrying traffic. (FR-10)
- **Build:** Use SystemConfiguration to find the primary network service / interface name, and restrict the counter read to that interface. Re-check when it changes.
- **Verify:** On Wi-Fi, numbers reflect Wi-Fi. Plug in Ethernet (or switch networks); readings follow the new interface without restarting.
- **Done when:** Switching networks moves the reading to the correct interface.
- **Result:** Verified Jul 10, 2026. Chris turned Ethernet off with the app running; the console followed the primary interface live — the label moved `[en0]` → `[en1]` (Ethernet → Wi-Fi) with no restart, no crash, and no spurious spike. The first `[en1]` line showed a small real blip (~0.5/0.3 Mbps, Wi-Fi association/DHCP) then settled to ~0, confirming the seamless baseline handoff (we already held last tick's counters for the newly-primary interface). Δt held ~1.00 s; the ~7 s of `[en0] 0.0` before the switch was macOS's own failover latency. Headless pre-check confirmed `PrimaryInterface.current()` resolves the real default-route interface (en0) and that restricting the read to it still tracks a 50 MB download (148.9 Mbps) and reads ~0 idle; SystemConfiguration read works under the App Sandbox (NFR-2). **Completes Phase 1.**
- **⚠️ Superseded (Jul 10, 2026, requirements v0.4):** per Chris's decision, throughput now **sums all physical interfaces (`en*`)** rather than following a single primary interface — an SMB transfer bound to a secondary Wi-Fi interface (`en1`) was invisible under primary-only monitoring while Activity Monitor showed it (~111 MB/s). `PrimaryInterface.swift` (the SystemConfiguration detection built in this increment) was removed; the aggregate now lives in `SpeedSampler`. Verified headlessly: a download forced over `en1` is captured by the `en*` sum (~70–108 Mbps) where the old `en0`-only read showed ~0. The interface-following work verified above remains valid history.

---

## Phase 2 — Minimal on-screen UI

### Increment 2.1 — Live numbers on screen  ✅ COMPLETE (Jul 10, 2026)
- **Goal:** Show the live figures in the window instead of the console. (FR-4, FR-17)
- **Build:** An observable model publishing down/up values; a SwiftUI view showing them (numbers-on-top layout stub). Timer updates the model.
- **Verify:** The two numbers update every second on screen and match what the console showed.
- **Done when:** On-screen numbers update live.
- **Result:** Verified Jul 10, 2026. Chris confirmed the on-screen Down/Up numbers update live every second and match the console and Activity Monitor. `SpeedSampler` is now an `@Observable` model; `ContentView` shows a numbers-on-top layout (`.monospacedDigit()`) with a caption naming the active interface(s). This increment also absorbed the v0.4 FR-10 change: an SMB transfer on `en1` and a browser/curl download on `en0` both display correctly, and the caption identifies whichever interface is most active. (The interim "numbers never change" report was diagnosed to an idle `en0` while the real traffic was on `en1` — not a UI bug; the `@Observable` wiring is sound.)

### Increment 2.2 — 60-second rolling graph, two lines  ✅ COMPLETE (Jul 11, 2026)
- **Goal:** Add the in-memory history graph. (FR-7, FR-7a, FR-7b)
- **Build:** A fixed-length ring buffer holding the last 60 samples; a SwiftUI Chart with two line series (download, upload) below the numbers. Oldest samples fall off as new ones arrive.
- **Verify:** Graph scrolls left over ~60 s; two distinct lines respond to traffic; closing/reopening starts fresh (no persistence).
- **Done when:** Graph shows a live 60-second, two-line view.
- **Result:** Verified Jul 11, 2026. Chris ran it in Xcode and confirmed the two-line 60-second graph renders below the numbers: two distinct colored lines (download/upload) with a legend, the plot scrolls left over ~60 s with the oldest samples falling off the left edge, and closing/reopening starts fresh (in-memory only — FR-7). Headless pre-check: a `RingBuffer` unit test compiled against the real `SpeedHistory.swift` confirmed the fixed-capacity FIFO clamps to 60, evicts oldest-first, preserves order, and resets via `removeAll()` (12/12 checks); the full app target builds warning-free with the new SwiftUI `Charts` view. New files: `SpeedHistory.swift` (`SpeedSample` + value-type `RingBuffer`) and `SpeedChart.swift` (two `LineMark` series, x-axis pinned to the last 60 s via `chartXScale`, y-axis `includesZero`). `SpeedSampler` appends one sample per tick and clears history on `start()`; `ContentView` hosts the chart under the numbers. Colors are Charts' defaults via `foregroundStyle(by:)`; exact palette, axis labels/units, and light/dark polish are deferred to 2.3.
- **Post-completion fix (Jul 12, 2026):** corrected a 60-second-window off-by-one Chris spotted while verifying Increment 3.1 — the download/upload lines stopped one tick short of the left "60s" axis. Cause: the chart's x-axis domain is a fixed 60 s wide (`[−60 s, now]`), but the ring buffer held **60** samples, which at the 1 s interval span only 59 s, so the oldest point sat one tick inside the left edge (the right "now" edge always touched, since the newest sample *is* `now` — the tell-tale asymmetry). Fix: buffer capacity 60 → **61** in `SpeedSampler` (a full 60 s span needs 61 fencepost points, one per second from −60 s to 0 s), so the oldest sample now lands exactly on the left axis. Headless check against the real `SpeedHistory.swift` confirmed `gap == 0` at capacity 61 and *reproduced* the 1 s gap at capacity 60 (4/4); warning-free build. The Increment 4.1 note in `SpeedSampler` now tracks `60 / interval + 1`. Confirmed fixed in Xcode by Chris.

### Increment 2.3 — Formatting, labels, appearance  ✅ COMPLETE (Jul 11, 2026)
- **Goal:** Make it readable and native-looking. (FR-6, NFR-3, NFR-5)
- **Build:** One-decimal formatting in the dual-unit format `X.X Mbps (Y.Y MBps)` (FR-6, format B per v0.5), rendered with a **monospaced-digit font** and fixed-width fields so the readout doesn't shift as values change width (NFR-3) — no zero-padding. Direction labels, distinct colors for down/up, and support for system light/dark appearance.
- **Verify:** Reads `45.3 Mbps (5.7 MBps)` style; legible in both light and dark mode; colors clearly distinguish the two lines.
- **Done when:** Layout matches FR-17 and reads cleanly in both appearances.
- **Result:** Verified Jul 11, 2026, over several rounds of visual polish confirmed by Chris in Xcode. Final state: dual-unit readout in format B `X.X Mbps (Y.Y MBps)` — Mbps primary, MBps a de-emphasized secondary parenthetical — with tabular digits over fixed-width (figure-space-padded) fields so nothing shifts as magnitudes change (NFR-3); the readout is horizontally centered and stays centered on resize (FR-17, v0.5). Direction labels are right-aligned and ordered word-then-arrow so the words line up and the tinted arrows align vertically beside the numbers. Graph gained axis labels — x-axis endpoints `60s`…`now`, y-axis auto-scaled Mbps with 0 pinned at the bottom and ≤3 gridlines — plus a legend. Colors follow Activity Monitor via a shared `SpeedPalette`: **blue for download (in), red for upload (out)**, on both the graph lines and the readout arrows; all system-dynamic, so the UI adapts to light/dark (NFR-5). New files: `ReadoutFormat.swift` (Foundation-only fixed-width formatter, headless-tested 12/12), `SpeedReadout.swift`, `SpeedPalette.swift`; `SpeedChart.swift` gained the axes/colors; `SpeedSampler`'s old-format string properties were removed. Phase-2 cleanup: removed all console `print()` — the per-tick/banner prints in `SpeedSampler` and the unused `NetworkCounters.printSnapshot()` — leaving zero `print()` in the source. Builds warning-free. **Completes Phase 2.**

---

## Phase 3 — Window behavior

### Increment 3.1 — Floating, resizable, medium default  ✅ COMPLETE (Jul 12, 2026)
- **Goal:** The right window shell. (FR-1, FR-2, FR-14)
- **Build:** Host the view in an `NSWindow` at normal window level, medium default size, resizable.
- **Verify:** Window floats like a normal window, resizes, and the graph/numbers stay legible when smaller.
- **Done when:** Resizes cleanly at a sensible default size.
- **Result:** Verified Jul 12, 2026. Chris confirmed in Xcode: the window floats like a normal window, resizes, and keeps the numbers + two-line graph legible when smaller, at a sensible medium default. Hosted as a **pure-SwiftUI `Window` scene**, not the `NSWindow` the Build line above anticipated — a design choice settled with Chris to keep 3.1 minimal and defer AppKit until it's actually needed: `WindowGroup` → a single unique `Window("NetSpeed", id: "main")` (floating at the normal window level — FR-1/FR-2; single-window per §3, so Cmd+N spawns no duplicate — which also sets up FR-16/3.4), `.defaultSize(width: 420, height: 320)` for the medium default (FR-14), and `.windowResizability(.contentMinSize)` so it resizes freely down to the content's minimum and stays legible (FR-14, NFR-3). NSWindow hosting is deferred to Increment 3.2, where frame autosave and first-launch top-left placement need the AppKit control (requirements §6). Headless: warning-free `xcodebuild`. **Two changes were folded in the same day (Jul 12):** the Increment 2.2 graph off-by-one fix (see its post-completion note above), and an **Apple-Silicon-only build** — `ARCHS = arm64` across all configs, verified `arm64`-only via `lipo`, recorded in requirements v0.6 (§6).

### Increment 3.2 — Remember position and size  ✅ COMPLETE (Jul 12, 2026)
- **Goal:** Persist where and how big the window is. (FR-11, FR-12)
- **Build:** Frame autosave (`setFrameAutosaveName`) or `UserDefaults`-backed frame save/restore.
- **Verify:** Move and resize the window, quit, relaunch — it returns to the same place and size.
- **Done when:** Position and size survive a relaunch.
- **Result:** Verified Jul 12, 2026. Chris moved and resized the window, quit with ⌘Q, and relaunched — it returned to the same position and size. Implemented **SwiftUI-only** per Chris's strong preference for a light, simple app, which meant *no persistence code*: a SwiftUI `Window` scene saves and restores its frame via macOS's standard window restoration, so neither the `setFrameAutosaveName` nor the `UserDefaults` approach the Build line anticipated was needed. Accepted trade-offs (Chris's explicit choice): it rides system window restoration — honoring System Settings → Desktop & Dock → "Close windows when quitting an application" — and the frame is saved on a normal quit (⌘Q), not an Xcode Stop (SIGKILL). **Watch-item for Increment 3.4:** restoration is built around windows open at quit, so when closing the window is made to quit the app (FR-16), we must confirm the close→quit path still restores the frame. Headless: warning-free `xcodebuild` (nothing to unit-test — restoration is OS behavior). Requirements §6 was updated to this SwiftUI-only reality (no `NSWindow`).
- **Alongside 3.2 (separate change — FR-18, requirements v0.8):** the window title now reads **"NetSpeed 1.0"** — app name + major.minor version, read from the bundle's marketing version (`CFBundleShortVersionString`) so it tracks the project version. A brief v0.7 experiment (hidden native title bar + a horizontally-centered custom title) was **reverted** because its extra vertical space raised the minimum window height too much; the standard native title bar (left-aligned on macOS 26) is used instead. Verified by Chris in Xcode (title reads "NetSpeed 1.0"; vertical space reclaimed); headless checks confirmed the built app's version key is `1.0` and the title helper yields "NetSpeed 1.0" (trimming to major.minor for longer versions). Recorded here as it landed during 3.2, but it is not part of the FR-11/FR-12 persistence scope.

### Increment 3.3 — First-launch position (top-left)  ✅ COMPLETE (Jul 12, 2026)
- **Goal:** Sensible fresh-install placement. (FR-13)
- **Build:** When no saved frame exists, position the window top-left; otherwise use the saved frame.
- **Verify:** Clear saved state (or run a fresh build) → window opens top-left. With saved state → opens where left.
- **Done when:** First run is top-left; later runs honor the saved frame.
- **Result:** Verified Jul 12, 2026. Chris confirmed both cases in Xcode. **(1) Fresh install:** with the app quit and its saved state cleared (`rm -rf ~/Library/Saved Application State/com.arc3solutions.NetSpeed.savedState`), relaunching opened the window at the **top-left** at the medium 420×320 default, sitting acceptably clear of the menu bar — which was the one open design question, resolved as good. **(2) Restoration still wins:** after moving/resizing the window away from the corner and quitting with ⌘Q, relaunching reopened it **where last left**, not back at top-left. Implemented **SwiftUI-only** per Chris's standing preference, with the mechanism settled with him before coding (chosen over a slight-inset `UnitPoint` variant or AppKit `visibleFrame` placement): a single `.defaultPosition(.topLeading)` on the `Window` scene, beside `.defaultSize`. Like `.defaultSize`, `.defaultPosition` takes effect only when there's no restored frame, so it sets the fresh-install placement with no first-launch-detection code and composes cleanly with 3.2's automatic restoration — no AppKit. Headless: warning-free `xcodebuild` (only the benign `appintentsmetadataprocessor` line); nothing to unit-test, as placement is OS behavior like 3.2's restoration.

### Increment 3.4 — Closing the window quits the app  ✅ COMPLETE (Jul 12, 2026)
- **Goal:** Confirmed close behavior. (FR-16)
- **Build:** Set the app to terminate after the last window closes.
- **Verify:** Close the window → the app fully quits (gone from the app switcher).
- **Done when:** Closing the window ends the process.
- **Result:** Verified Jul 12, 2026. Chris confirmed both halves in Xcode via the close→quit path. **FR-16 is satisfied with zero code** — it is the macOS 26 SwiftUI framework default for a single unique `Window` scene: closing the window (red button / ⌘W) terminates the app (gone from the ⌘-Tab app switcher). Confirmed by inspection that nothing in the project forces it — there is no `Info.plist` file (`GENERATE_INFOPLIST_FILE = YES`, only an empty copyright key), no `LSUIElement`/agent key, no `@NSApplicationDelegateAdaptor` / `applicationShouldTerminateAfterLastWindowClosed`, and `ContentView`'s `.onDisappear` only calls `sampler.stop()`. The mechanism we pre-settled with Chris (delegate adaptor vs. pure-SwiftUI `onDisappear`) was thus **obviated** — the platform already does it, so neither was added. **Watch-item resolved (carried since 3.2):** the decisive test used the *close→quit* path rather than ⌘Q — moved and resized the window to a distinctive spot, quit by **closing** it (not Xcode Stop, which SIGKILLs), relaunched, and it **restored to the exact previous size and position**. So macOS automatic window restoration survives the close→quit path; FR-11/FR-12/FR-13 hold on the app's primary quit path, and no explicit frame-persistence fallback is needed. Headless: no code change to build — the finding came from inspecting the project config. **Completes Phase 3.**

---

## Phase 4 — Settings and lifecycle

### Increment 4.1 — Adjustable sampling interval (1–5 s)  ✅ COMPLETE (Jul 13, 2026)
- **Goal:** User-controlled refresh rate. (FR-9, FR-9a, FR-9b, FR-9c)
- **Build:** A control (slider or menu, 1–5 s) bound to a persisted setting; the timer rebinds to the chosen interval. Default 1 s on first launch.
- **Verify:** Change the rate → update cadence visibly changes; quit and relaunch → the chosen rate is remembered.
- **Done when:** Rate is adjustable, effective immediately, and persistent.
- **Result:** Verified Jul 13, 2026 (built Jul 12–13 over three UI rounds). Chris confirmed in Xcode: the sampling rate is adjustable, takes effect immediately, and persists across launches (FR-9, FR-9a–c). **Implementation** — `@AppStorage("samplingIntervalSeconds")` (default 1 s, FR-9a) is the SwiftUI-idiomatic UserDefaults binding (requirements §6), so the choice is remembered with no persistence code; `SpeedSampler.start(intervalSeconds:)` sizes the history buffer from the new `nonisolated static historyCapacity(forIntervalSeconds:)` = `60 / interval + 1` (61/31/21/16/13 for 1–5 s) so the graph stays a full 60-second window at any rate; `setInterval(seconds:)` re-arms the timer live by reusing `start` — which re-primes the byte-counter baseline (so the first post-change sample is a real delta, not a spike) and rebuilds the buffer, so changing the rate restarts the 60-second graph from empty (accepted behavior). The throughput math was already interval-agnostic (bytes ÷ elapsed seconds), so it needed no change. **Control — three placement/appearance rounds settled with Chris:** (1) a menu `Picker` first placed in the title-bar **toolbar** (zero content height); (2) that surfaced a macOS toolbar **auto-overflow** — at the minimum window width the item collapsed into the toolbar's `»` overflow on a fresh launch (SwiftUI exposes no way to pin a toolbar item; that is AppKit `NSToolbarItem.visibilityPriority`), so the control moved **inline** into the window body, sharing the interfaces-caption row as a compact labeled `Menu` (`⏱ Sample: …`) that cannot overflow and reads more clearly; (3) the unit is spelled out with correct singular/plural via a `secondsLabel(_:)` helper — "1 second", "2 seconds" … "5 seconds" — on both the button and the menu items. All SwiftUI-only, no AppKit. **Headless:** `historyCapacity` unit-tested against the real source (1→61 … 5→13, with the real `RingBuffer` clamping to each) and warning-free `xcodebuild` across all three rounds. **Starts Phase 4.**

### Increment 4.2 — Launch at login  ❌ REMOVED (Jul 13, 2026)
- **Goal:** Start automatically on login. (FR-15 — removed in requirements v0.9)
- **Build:** A toggle using `SMAppService` to register/unregister the app as a login item.
- **Verify:** Enable the toggle, log out and back in → the app is running. Disable → it isn't.
- **Done when:** Login-item state matches the toggle.
- **Result:** Removed Jul 13, 2026 by Chris's decision, before any code was written. FR-15 (launch at login) is dropped from v1 (requirements v0.9): NetSpeed will not manage login items via `SMAppService`; anyone who wants launch-at-login can add NetSpeed in System Settings → General → Login Items. Rationale — keep v1 minimal and SwiftUI-only, avoiding the ServiceManagement machinery and the login-item registration/testing overhead (which also sidesteps the app-path and log-out/log-in verification burden). Nothing to remove from the codebase. **With 4.1 complete and 4.2 removed, Phase 4 is done.**

---

## Phase 5 — Robustness and final check

### Increment 5.1 — Empty / zero state  ✅ COMPLETE (Jul 13, 2026)
- **Goal:** Graceful behavior when there's no active interface. (NFR-4)
- **Build:** Detect no primary interface or unreadable counters; show a zero/empty state instead of erroring.
- **Verify:** Turn off Wi-Fi / unplug Ethernet → no crash; UI shows zero or an empty state; recovers when connectivity returns.
- **Done when:** Loss and return of connectivity are handled without a crash.
- **Result:** Verified Jul 13, 2026 — **verify-only, no code** (Chris's call): NFR-4 was already met by construction, so 5.1 was a verification pass, not a build. Chris confirmed all three cases in Xcode: **(1) loss** — turning off Wi-Fi / unplugging Ethernet dropped the readout to `0.0 Mbps (0.0 MBps)`, flattened the graph to the 0 baseline, kept the "All interfaces (idle)" caption, and the app kept running (no crash/hang); **(2) return** — reconnecting recovered the readings with no spurious spike; **(3) extreme** — all interfaces off still showed `0.0` + a flat graph, no crash. The assistant-side check was a code-path inspection (nothing changed, no build): `SpeedChart` uses `samples.last?.time ?? Date()` so an empty history is safe (an empty `Chart` just draws axes), `NetworkCounters.readAll()` guards `getifaddrs` and returns `[]` on failure, and `SpeedSampler.tick()` publishes `0.0` / idle / flat when no `en*` traffic is seen, with the `guard let prev` baseline-skip absorbing a reconnect so no false spike appears. A distinct "no active network interface" caption was considered and deliberately skipped: macOS keeps interfaces listed-but-idle when connectivity drops, so "All interfaces (idle)" is accurate and a true-empty message would rarely trigger.

### Increment 5.2 — Footprint sanity check  ✅ COMPLETE (Jul 13, 2026)
- **Goal:** Confirm it's lightweight. (NFR-1)
- **Build:** No new feature — measure with Activity Monitor / Instruments while idle and under load.
- **Verify:** Idle CPU is negligible; memory is small and stable (no growth from the ring buffer).
- **Done when:** Resource use is low and stable over time.
- **Result:** Verified Jul 13, 2026. The initial Release-build measurement flagged two concerns: CPU **1–2%** (2.4% peak) at the 1 s rate on an M4 Mac mini, and Real Memory ~**85–100 MB** creeping ~**0.1 MB / 2 min**. A `sample` of the process traced ~**97%** of the app's CPU to the once-per-second **SwiftUI Charts** re-render (~24 ms/tick of NSHostingView layout + SwiftUI view-graph updates + Charts internals); the app's own logic (getifaddrs, tick math) was negligible, and Charts' per-render allocation/metadata churn matched the slow memory growth. That spawned **Increment 5.2a** (Charts → `Canvas`). After it, Chris re-measured over **3+ hours**: CPU now cycles **0.5–1.0%** (occasional 1.1–1.4% blips) and memory is **stable with no creep** — Real Memory actually *declined* 95 MB → 86 MB over the run. **NFR-1 satisfied.**

### Increment 5.2a — Graph rendering optimization (Canvas)  ✅ COMPLETE (Jul 13, 2026)
- **Goal:** Cut the graph's per-tick render cost and the memory growth surfaced in 5.2, with no change to the graph's appearance. (NFR-1)
- **Build:** Reimplement `SpeedChart` on SwiftUI `Canvas` instead of SwiftUI `Charts` — two `Path` strokes for the download/upload series, hand-drawn axis pieces (60s/now, Mbps, ≤3 auto-scaled gridlines), and a static legend. Same `SpeedChart(samples:)` interface; no other file changes.
- **Verify:** Graph looks the same; re-measure CPU and memory (Release) against the 5.2 baseline.
- **Done when:** Render cost and memory growth drop with no visual regression.
- **Result:** Verified Jul 13, 2026. Settled with Chris (chosen over confirming the leak in Instruments first, or leaving it as-is). `SpeedChart` now paints directly with `Canvas` — no view-graph diffing, no GeometryReader layout passes, no per-render Charts metadata/hashable churn; `import Charts` is gone, so the framework is no longer linked. Y-axis auto-scaling is a small pure helper (`yGridValues` → a "nice" 0-based ceiling with ~3 gridlines; `axisLabel` formatting), **headless-tested** against the real source across a magnitude sweep (idle→[0,1], 45→[0,25,50], 950→[0,500,1000]; each starts at 0, covers the data, and never over-scales past 2×) plus a warning-free `xcodebuild`. GUI + footprint confirmed by Chris (see 5.2): CPU ~1–2% → **0.5–1.0%**, memory creep **eliminated** (stable, 95→86 MB over 3+ h), graph visually per spec. Pure SwiftUI throughout — consistent with the standing SwiftUI-only preference; the earlier Charts choice (2.2/2.3) served through development, and this trades Charts' free axes/legend for ~40 lines of drawing to reclaim the footprint.

### Increment 5.3 — Requirements traceability pass  ✅ COMPLETE (Jul 13, 2026)
- **Goal:** Confirm every requirement is met. (all)
- **Build:** Walk the requirements doc FR/NFR list against the running app; note any gaps.
- **Verify:** Each requirement checks out or is explicitly deferred.
- **Done when:** The checklist is fully accounted for.
- **Result:** Verified Jul 13, 2026. Walked the full v0.9 FR/NFR list against the built app with Chris, who agreed **every v1 requirement is met and verified, or explicitly out of scope — no gaps.** FR-1–FR-18 and NFR-1–NFR-5 each trace to a completed, verified increment (see the Coverage map and the per-increment results above): form factor 3.1; live dual-unit readout 1.2/2.1/2.3; 60 s two-line graph 2.2/2.3/5.2a; passive `en*`-summed measurement 1.1/1.2/SpeedSampler; adjustable 1–5 s sampling 4.1; window position/size/first-launch/close-quits 3.1–3.4; title bar FR-18; footprint NFR-1 5.2/5.2a; no-privilege reads NFR-2 1.1/1.3; legibility NFR-3 2.3/5.2a; resilient zero-state NFR-4 5.1; light/dark NFR-5 2.3/5.2a. Out of scope, not counted as gaps: the §7 deferred items (always-on-top, transparency, latency/usage/speed-tests, manual interface pick, extra units, auto-adaptive sampling, persistent logging, menu-bar item) and FR-15 launch-at-login (removed in v0.9, delegated to System Settings). **Completes Phase 5 — NetSpeed v1 is feature-complete.**

---

## Coverage map (requirement → increment)

- Passive measurement / counters (FR-8): 1.1, 1.2
- Live down/up in Mbps + MBps (FR-4, FR-5, FR-6): 1.2, 2.1, 2.3
- Throughput scope — sum all physical `en*` interfaces (FR-10, v0.4): `SpeedSampler` (revised from the single-primary-interface approach originally built in 1.3)
- 60-second two-line graph, in-memory (FR-7, 7a, 7b): 2.2
- Numbers-on-top layout (FR-17): 2.1, 2.3
- Light/dark (NFR-5): 2.3
- Floating / normal level / resizable / medium (FR-1, FR-2, FR-14): 3.1
- Remember position & size (FR-11, FR-12): 3.2
- Window title "NetSpeed X.Y" (FR-18, v0.8): SwiftUI `Window` scene title added during Phase 3, separate from the 3.2 persistence increment
- First-launch top-left (FR-13): 3.3
- Close quits app (FR-16): 3.4
- Adjustable sampling 1–5 s (FR-9): 4.1
- Launch at login (FR-15): **removed in v0.9** — delegated to System Settings (was Increment 4.2)
- Resilient empty state (NFR-4): 5.1
- Low footprint (NFR-1): 5.2, 5.2a (Canvas graph optimization)
- Opaque window (FR-3): inherent (no transparency added)

---

## Prerequisites before Increment 0.1

- Xcode installed and able to build/run a macOS app on this machine (macOS 26 Tahoe).
- A folder decision: keep the Xcode project inside `network-speed-monitor/`.

---

*Progress: **NetSpeed v1 is feature-complete (Jul 13, 2026).** Phases 1 (0.1, 1.1, 1.2, 1.3), 2 (2.1, 2.2, 2.3), 3 (3.1, 3.2, 3.3, 3.4), 4 (4.1 done; 4.2 launch-at-login removed in v0.9), and 5 (5.1, 5.2, 5.2a, 5.3) are all complete. Every v1 FR/NFR is met and verified, or explicitly out of scope (5.3 traceability pass) — no gaps. Any further work would be v2 / the §7 deferred items.*

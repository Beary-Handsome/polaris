# Planning: Concurrent Off-Screen Sessions + Per-App Audio Isolation ("Family Mode")

**Status:** Planning / design. No code yet.
**Date:** 2026-06-28
**Target platform:** Linux (openSUSE Tumbleweed dev host, AMD/VAAPI). KDE Plasma 6 / Wayland desktop.
**Related:** [headless-source-selection](2026-06-28-headless-source-selection.md)

## Goal

Let a Moonlight client launch a game into an **isolated, off-screen session** that runs
*concurrently* with the host user's live desktop — so e.g. a kid can play a streamed game
while the parent keeps working on the same PC — with a **separate audio path** (the game's
sound goes only to the stream; the desktop user hears only their own audio).

## Key finding: ~80–90% already exists

A read of the Linux platform code shows Polaris was largely architected for this. Status
of each required capability:

| Capability | Status | Evidence (file:line) |
|---|---|---|
| Session runs alongside live desktop (no seat/GPU takeover) | **EXISTS** | `session_manager.h:10-14` (explicitly: no display switching, no KWin routing, no focus-steal prevention); `session_manager.cpp:252-262` (saves no display state) |
| Off-screen nested compositor w/ own `WAYLAND_DISPLAY` | **EXISTS** | labwc headless: `cage_display_router.cpp:141-176` (`WLR_BACKENDS=headless`), `:744-767` (`HEADLESS-1`, unsets host DISPLAY/WAYLAND_DISPLAY, `setsid`), child env `:851-868` |
| Headless mode user-selectable | **EXISTS** | `stream_display_policy.cpp:30-80`; config `headless_mode`, `use_cage_compositor`, `prefer_gpu_native_capture` (`config.h:163-165`) |
| Game audio → dedicated null sink, captured from monitor | **EXISTS** | `platform/linux/audio.cpp:865-890` (null sinks), `:1010-1108` (`route_process_audio_to_sink` moves only env-tagged processes) |
| Desktop user keeps own default sink | **EXISTS** | `audio.cpp:290-296` (`should_route_session_sink_without_default`); `platform/linux/audio.cpp:900-913` (restores default) |
| Per-session sink env injection | **EXISTS** | `process.cpp:2683-2684` sets `PULSE_SINK` + `POLARIS_SESSION_AUDIO_SINK` |
| Per-session `WAYLAND_DISPLAY` injection | **EXISTS** | `process.cpp:3151-3165` |
| Input does not leak to host desktop | **EXISTS** | `inputtino_wayland_virtual_input.cpp:320-327,554` (blocks host-uinput fallback in headless); gamepad bwrap isolation `inputtino_gamepad_isolation.*`, wired `process.cpp:3019-3046` |
| **Per-app "isolate me" opt-in flag** | **PARTIAL** | per-app `env` (`process.cpp:4578`) and `virtual-display` (`process.cpp:4561`) hooks exist, but no dedicated isolation flag |
| **One-click "play while I work" preset** | **MISSING** | only separate global toggles today |
| **Per-session unique audio sinks (multi-session)** | **MISSING** | 3 global singleton sinks `sink-sunshine-*` (`platform/linux/audio.cpp:808-811`) |

### How the audio isolation works today (important detail)

`route_process_audio_to_sink()` enumerates PulseAudio sink-inputs and moves **only**
processes whose `/proc/<pid>/environ` contains `POLARIS_SESSION_AUDIO_SINK=<sink>` or
`PULSE_SINK=<sink>` (`process_env_has_session_audio_sink()`, `platform/linux/audio.cpp:641-673`).
Everything belonging to the desktop user is left on the default sink. Capture reads the
sink's `.monitor`. This is exactly the "two independent audio paths" we want.

## What to actually build

This is **refinement of existing switches**, not new subsystems:

1. **Per-app `isolated_session` flag.** Add to the app parser next to
   `ctx.virtual_display = app_node.value("virtual-display", false)` (`process.cpp:4561`).
   Surface in the web UI app editor (`src_assets/.../ConfigView.vue`, near the existing
   display/audio options ~`:364-387`).

2. **Let the per-app flag override display policy at launch.** Policy is resolved in the
   launch flow via `stream_display_policy::resolve*()` (read in `process.cpp:~2426`), which
   today reads only the *global* `headless_mode`/`use_cage_compositor`. Insertion: allow the
   per-app flag to force headless+cage for that app even when the global default is DESKTOP.

3. **Guarantee audio isolation for an isolated app.** The sink env block already exists
   (`process.cpp:2675-2692`). Ensure the session's `host_audio` is false when the per-app
   isolation flag is set (force it just before `select_sink_name`, ~`process.cpp:2681`), so
   the game's audio is always routed to the dedicated sink and never the desktop default.

4. **(Multi-session) Per-session unique null sinks.** Today the three `sink-sunshine-*`
   sinks are process-global singletons (`platform/linux/audio.cpp:865-890`); two concurrent
   isolated games would collide on the same sink + env marker. For >1 simultaneous isolated
   session, generate a per-session uniquely-named null sink and tag the child with it.

5. **One-click preset.** A web-UI preset that sets `headless_mode + use_cage_compositor +
   virtual_sink + host_audio off` and the per-app flag, labeled for the family use case.

## Validate-with-config-first (no code)

Before writing code, the behavior can be exercised with existing toggles:
`headless_mode = on`, `linux_use_cage_compositor = on`, a `virtual_sink`, host audio off.
Launch a game from Moonlight and confirm: (a) nothing appears on the physical monitors,
(b) the desktop stays usable, (c) game audio is on the stream only. This validates the
whole path on real hardware before any feature work.

## Top risks

1. **Audio env inheritance through Steam/Proton.** Isolation depends on the game inheriting
   `POLARIS_SESSION_AUDIO_SINK`/`PULSE_SINK` in its environment. Proton/Steam re-exec
   through helper processes; a child that drops/rewrites env won't be matched by
   `route_process_audio_to_sink`, and that stream could leak to the desktop's default sink
   (audible to the host). **Hardening target #1.**
2. **Off-screen vs. GPU-native capture tension.** Headless labwc can fall back to software
   (SHM) capture, and the code may force the compositor *windowed* (visible) to keep capture
   GPU-native (`cage_display_router.cpp:744-755`; fallback machinery `process.cpp:2819-3018`).
   The documented crash is **NVIDIA-specific** (Vulkan renderer on wlroots 0.19) — the AMD
   dev host is on the favorable side, but VAAPI headless-capture performance should be
   measured.
3. **Single shared sink ⇒ one isolated session at a time** until per-session sinks land (#4).

## Bottom line

The end-to-end mechanism exists on Linux. Remaining work = (1) per-app opt-in instead of
global toggles, (2) per-session unique sinks for multi-session correctness, (3) hardening
the audio env tagging against Proton/Steam re-exec.

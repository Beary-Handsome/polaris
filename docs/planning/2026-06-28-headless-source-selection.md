# Planning: Selectable Headless Source (Virtual / EVDI / Physical Dongle)

**Status:** Planning / design. No code yet.
**Date:** 2026-06-28
**Related:** [offscreen-isolated-sessions](2026-06-28-offscreen-isolated-sessions.md)

## Goal

Give the user an explicit choice of *where* the off-screen session renders, instead of
today's auto-detect:

1. **Virtual labwc-headless** — pure software headless output (no DRM connector).
2. **EVDI virtual DRM connector** — a "software dummy plug": real `cardN`/connector,
   KMS/DMA-BUF capturable, no hardware.
3. **Physical dummy dongle / EDID emulator** — a real connector on a real port, leased to
   the isolated session so it stays off the desktop. **(This is the path we want.)**

### Why bother — benefits of a real DRM connector

A real connector (EVDI or physical dongle) yields a standard EDID, normal mode list,
hardware cursor, and the GPU's *normal* code path → reliable GPU-native KMS/DMA-BUF capture
+ hardware encode, avoiding the headless-wlroots software-copy fallback. The physical
dongle additionally guarantees the most driver-compatible path (the historical reason
dummy plugs exist) and is the best fallback for driver stacks where virtual capture is flaky.

### Trade-offs

| Source | Isolated from desktop | Capture quality | Hardware | Notes |
|---|---|---|---|---|
| labwc-headless (virtual) | clean | sometimes software (SHM) copy | none | current default; most isolated |
| EVDI virtual connector | clean | near-native KMS | none (kernel mod) | likely the AMD sweet spot |
| Physical dongle | needs DRM lease | native, most compatible | dongle + port | most compatible; resolution pinned to EDID |

Virtual also wins on **arbitrary client resolution** (create the output at the client's
exact resolution per session); a fixed-EDID dongle needs custom modelines / EDID override.

## Backend facts (verified, read-only)

`virtual_display.h:22-27` enumerates backends: `EVDI`, `WAYLAND_WLR`, `KSCREEN_DOCTOR`,
`NONE`. **Auto-detected by priority, not user-selectable** (`detect_backend()`,
`virtual_display.cpp:1066-1095`; no config key forces a backend).

- **EVDI** (`virtual_display.cpp:217-785`): loads `evdi` kmod, dlopens `libevdi`, generates a
  128-byte EDID, `fn_connect()`, then discovers the new `cardN` + connector name
  (`find_evdi_output()` `:519-549`). **Real DRM connector — KMS-capturable.** Gated only by
  module/lib availability (`is_available()` `:509-511`); attaches to `config::video.adapter_name`
  GPU if set. **PARTIAL:** works and is wired (`process.cpp:2455-2479`) but not user-forceable.
- **WAYLAND_WLR** (`:790-947`): `hyprctl/swaymsg create_output` → synthetic compositor output,
  **not a DRM connector**; **KWin returns false** (`:835-839`) so unavailable on Plasma.
- **KSCREEN_DOCTOR** (`:952-1046`): does **not create** anything — enables/configures an
  *existing* output via `kscreen-doctor` (`:983-985`). Docstring explicitly mentions a
  **"dummy plug"** (`:964-966`). Requires `streaming_output` set. Routes through KWin → **no
  isolation**. This is the closest existing path to a physical dongle, but as a shared head.

### DRM leasing: ABSENT

Whole-tree grep for `drm_lease` / `wp_drm_lease` / `drmModeCreateLease` / `leasable` etc.
returns **nothing** relevant (only `DRM_MODE_OBJECT_*` property constants in `kmsgrab.cpp:498-506`).
No code leases a connector to a child compositor. **This is the core net-new gap for an
isolated physical dongle.**

### Physical output selection today

- Config: `streaming_output`, `primary_output`, `auto_manage_displays` (`config.h:160-162`;
  parsed `config.cpp:1334-1336`).
- `enable_streaming_display()` / `disable_streaming_display()` run `kscreen-doctor`
  (`process.cpp:1726-1769`) — lights the connector as a **secondary KWin head**, desktop
  still running. **Not isolated.**
- Capture selects by **numeric plane index**, not connector name: `kmsgrab.cpp:589-592`
  (`monitor_index = from_view(display_name)`), names emitted as `"0","1",…`
  (`kms_display_names()` `:1674`). kmsgrab needs **CAP_SYS_ADMIN, not DRM master**
  (`:39-56,300`), so it captures KWin-owned scanout without taking master — and a future
  lease-FD path is compatible with it.

## Per-source classification

| Source | Status |
|---|---|
| Virtual labwc-headless | **EXISTS** (primary path) |
| EVDI virtual connector | **PARTIAL** — works, but auto-detect only; no user override |
| Physical dongle (isolated) | **MISSING** — lightable+capturable only via shared KWin head; no DRM lease |

## Implementation plan — physical dongle path

1. **Source selector + config key.** Add a user-facing key (e.g.
   `linux_display.headless_source = auto|virtual|evdi|physical`) and a 4th `backend_e`
   value (e.g. `PHYSICAL_LEASED`, `virtual_display.h:22-27`). Make `detect_backend()`
   (`:1066-1095`) honor an explicit override instead of priority-only. Extend
   `stream_display_policy::resolve()` (`stream_display_policy.cpp:30-80`) with a
   physical-leased mode (today it knows DESKTOP / HEADLESS / HOST_VIRTUAL_DISPLAY /
   GPU_NATIVE_TEST).

2. **Output selection by connector name.** Today capture is positional-index based
   (`kmsgrab.cpp:589-592`, `:1674`). Add name→plane/CRTC resolution (connector metadata is
   already gathered at `kmsgrab.cpp:726-728`, `map_crtc_to_monitor()` `:530-548`) so a named
   dongle connector is targeted deterministically.

3. **DRM lease for isolation (core gap).** New module paralleling `virtual_display.cpp`:
   `drmModeCreateLease` over the chosen connector+CRTC+plane, pass the lease FD to the
   nested labwc compositor so KWin releases that connector. Wire next to the existing
   virtual-display create (`process.cpp:2455-2479`) and cage start (`process.cpp:3071-3118`).
   kmsgrab then captures the leased connector's scanout (compatible — needs only CAP_SYS_ADMIN).

4. **Lifecycle/teardown.** Lease revoke + connector restore alongside
   `virtual_display::destroy()` (`virtual_display.cpp:1173-1200`) and session teardown
   (`process.cpp` cage `stop()` ~`:3591`, vdisplay destroy ~`:3690`).

### Phasing (recommended order)

- **Phase A (cheap win):** expose EVDI as an explicit, user-selectable source — it already
  produces a real connector and is wired; just needs the override key + UI. Gets most of the
  "real connector" benefit with zero hardware and no leasing.
- **Phase B:** named-connector selection in kmsgrab (removes positional-index fragility).
- **Phase C:** DRM-lease module → true isolated **physical dongle** support.

## Risks (dongle path)

1. **DRM lease is net-new and protocol-level** — nothing in-tree to build on.
2. **KWin must cooperate.** Leasing a connector out from under KWin requires KWin to support
   `wp_drm_lease` / release the connector, or the connector must be pre-detached. Otherwise
   KWin keeps master of it. Needs validation on Plasma 6.
3. **Existing physical path is KDE/kscreen-doctor-specific and positional-index-based** — not
   a clean foundation for a named, isolated dongle source; expect to build the selection path
   fresh rather than extend the kscreen path.

# godot-spout: implementation plan for live VJ use

Status: **draft for review. No code has been changed yet.**
Scope: make godot-spout reliable for an all-night show. Godot (Forward+) renders a
canvas at 60 fps and sends it to Resolume Arena on the same machine. It also
receives Spout sources from Resolume.

**Resolution targets** (details in §2.4):
- **Primary: 1920 × 1080.** This is the default for demos, the soak test and the
  headline performance numbers.
- **Must also be well supported: 7680 × 1080** (three projection walls), plus
  arbitrary sizes in between. Every functional test and every benchmark also runs
  at 7680 × 1080. "Supported" means the same checklist passes, not just that
  "it starts".

This plan builds on the audit (summarised in §1). Every step is meant to land as
small commits that each do one thing and that could go upstream to `buresu/godot-spout`.

---

## 0. Decisions needed before coding

| # | Question | Recommendation |
|---|---|---|
| D1 | Sending through a texture the extension owns (Godot does the copy, so its state tracking stays correct) adds **one Godot frame of latency**. That is 1/fps: 16.7 ms at 60 fps, 8.3 ms at 120 fps, 33 ms if the scene drops to 30 fps. With vsync on a 60 Hz output, 16.7 ms is the real figure. The alternative is direct wrapping: no added latency, but the D3D12 states can't be made correct. | Accept the 1 frame. |
| D2 | Stop using `lib/SpoutVulkan/Examples/SpoutVK.cpp` and own the Vulkan interop in this repo. The submodule stays for its license and its SpoutDX helper sources. | Yes. |
| D3 | Minimum Godot version. godot-cpp is pinned to 4.6, the demo uses 4.7, and `compatibility_minimum` is 4.3. | Keep 4.3 unless an API requires more. |
| D4 | Should nodes send/receive while the scene is open in the editor? (Today they do, as a side effect of connecting in the constructor.) | Keep it, behind a new `active_in_editor` property (default `true`, which keeps current behaviour). |
| D5 | Keep the existing property name `channel_name` (backward compatible) or rename it to `sender_name`. | Keep `channel_name`. |
| D6 | D3D12 is the supported production backend. Vulkan is "works on tested drivers" (see §1.4). OpenGL is send-only until P2. | Yes. Document this in the README. |

---

## 1. Audit summary (what the plan fixes)

### 1.1 Current data paths

| Path | How it works today | Copies | CPU stalls |
|---|---|---|---|
| D3D12 send | Godot RT `ID3D12Resource` → wrapped by a **per-node** D3D11On12 device on Godot's queue → `CopyResource` into the Spout shared DX11 texture → `Flush` | 1 GPU | none |
| D3D12 receive | Sender's shared handle opened on 11on12 → `CopyResource` into the wrapped RD texture | 1 GPU | none (but see bug B4) |
| Vulkan send | Shared DX11 texture imported as a "linked" `VkImage` (KMT handle) → blit on a private command buffer → submit to Godot's queue | 1 GPU | `vkDeviceWaitIdle` **and** a fence wait every frame, per node |
| Vulkan receive | Blit linked image → RD texture, same submit/wait | 1 GPU | same |
| OpenGL send | SpoutGL `SendTexture` via WGL_NV_DX_interop | 1 GPU (falls back to CPU readback without interop) | lock/unlock |
| OpenGL receive | not implemented | – | – |

Zero-copy is impossible with Spout: the shared texture has to be a separate
shareable resource, and Godot owns its render targets. **One GPU copy is the
minimum.** Per copy (read + write each):

| Size | RGBA8 | RGBA16F | GPU time per copy (*estimate*, current discrete GPU) |
|---|---|---|---|
| 1920 × 1080 | 8.3 MB | 16.6 MB | < 0.1 ms |
| 7680 × 1080 | 33 MB | 66 MB | 0.1–0.3 ms (RGBA8), about double for RGBA16F |

To be measured in Step 0 and again after P0.

### 1.2 Bugs the plan fixes (numbering used below)

- **B1** Backends are destroyed and recreated on every failure, setter call or
  texture change. A receiver with no sender creates **a new D3D11On12 device every
  frame**. (`SpoutInput.cpp:173-194`, `SpoutTexture.cpp:142-162`, `SpoutOutput.cpp:55-65,140-149`)
- **B2** Spout's `ReleaseReceiver()` calls `Sleep(67)` on the main thread whenever a
  sender disappears. (`lib/Spout2/.../SpoutDX.cpp:742`, `SpoutVK.cpp:833`)
- **B3** D3D12 resource-state desync. The wrap's input state is guessed from usage
  bits, and the output state is hard-coded to `PRESENT`, while Godot keeps tracking
  its own legacy states. (`SpoutDX12Backend.cpp:179-187`, `SpoutDX12.cpp` `WrapDX12Resource`)
- **B4** D3D12 receive likely never copies: the `||` short-circuit at
  `SpoutDX12Backend.cpp:116-119` means `IsUpdated()` is never called, so
  `ReceiveDX12Resource` returns early forever. Also, the receiving texture is fixed
  to BGRA8, so `CopyResource` from RGBA8/RGBA16F senders is invalid. *Needs runtime
  confirmation (Step 0).*
- **B5** Vulkan calls `SetSenderName` every frame. That renames the sender to
  `<name>_1` internally, so resize updates and release go to the wrong name.
  (`SpoutVulkanBackend.cpp:83-85`, `SpoutVK.cpp:610-723`)
- **B6** Vulkan receivers call `SetActiveSender` every frame. That changes the
  system-wide active sender, and multiple receivers fight over it.
  (`SpoutVulkanBackend.cpp:105-122`)
- **B7** Vulkan sync: the mutex is released and "new frame" is signalled before the
  copy is submitted, and the barriers use a `TOP_OF_PIPE` source stage. This only
  works because of B8. (`SpoutVK.cpp:589-603,274-300`)
- **B8** Per-frame `vkDeviceWaitIdle` plus a fence wait, submitted to Godot's queue
  with no external synchronisation.
- **B9** D3D12 render targets are TYPELESS, and Spout copies that format into the
  shared texture. Receivers may be offered `R8G8B8A8_TYPELESS`.
- **B10** Destructors use `RenderingServer::get_singleton()` without a null check.
  A `SpoutTexture` can outlive the RenderingServer, which risks a crash on exit.
- **B11** Vulkan format mapping bug (`SpoutVK.cpp:494`), and a resize drops RGBA16F.
- **B12** The demo's issue 1: `output_demo.gd:9` preloads the shared
  `ViewportTexture` resource. That resource is never set up (ViewportTexture is
  local-to-scene, so the scene uses its own duplicate). The preloaded copy's RID
  points at Godot's 4×4 placeholder.
- **B13** Multi-GPU: SpoutVK creates its D3D11 device on the *system default*
  adapter (`D3D11CreateDevice(NULL, …)`, `lib/SpoutVulkan/Examples/SpoutDX/SpoutDirectX.cpp:486`).
  It never matches that to the GPU Godot's Vulkan device runs on. On hybrid
  laptops or multi-GPU desktops, the import comes from the wrong GPU.

### 1.3 Godot facts this plan relies on (checked in the 4.6 source)

- `frame_pre_draw` is emitted before drawing. `frame_post_draw` is emitted after
  `end_frame()`, i.e. after the frame's command buffers are submitted. In the
  multi-threaded model, `frame_post_draw` is emitted with `call_deferred`.
- RD commands issued from `frame_post_draw` are recorded into the **next** frame.
  That is where the 1-frame latency in D1 comes from.
- The D3D12 driver creates textures with shareable formats as TYPELESS, and tracks
  legacy `D3D12_RESOURCE_STATES` per subresource. (The enhanced-barrier path was
  not verified, see Step 0.)
- The Vulkan driver does **not** enable `VK_KHR_external_memory_win32` or
  `VK_KHR_win32_keyed_mutex`.
- GLES3 implements `texture_create_from_native_handle` (it wraps a GL name and does
  not take ownership).
- `RenderingDevice.texture_create_from_extension`, `texture_copy`, compute lists,
  `RenderingServer.call_on_render_thread`, `texture_create_from_native_handle` and
  `texture_rd_create` are all available in godot-cpp 4.6.

### 1.4 Vulkan caveat (to document, not fixable here)

Importing the D3D11 KMT handle into Godot's `VkDevice` is technically invalid
usage, because the device was created without `VK_KHR_external_memory_win32`. It
works on drivers that don't enforce this (the existing code already depends on
it). There is also no GPU-side synchronisation with the receiving process without
a keyed mutex, so "tear-free" on Vulkan is best effort. The real fix is a small
upstream Godot PR that requests these extensions as optional; this is listed as
follow-up F1.

### 1.5 Multiple GPUs

Spout's DirectX shared handles only open on the GPU that created the texture.
Godot and Resolume must run on the same GPU; there is no automatic cross-GPU path.
- D3D12: the D3D11 side sits on Godot's own device, so it's always the same GPU.
- Vulkan: the D3D11 device must be created on the adapter whose LUID matches
  `VkPhysicalDeviceIDProperties::deviceLUID` (fixes B13).
- A receiver whose sender handle won't open reports "sender probably on a
  different GPU" once, then retries slowly. No per-frame churn.
- Log the GPU name and LUID at startup; `get_adapter_name()` on all nodes; GPU shown in the demo overlay.
- No CPU cross-GPU fallback (~2 GB/s each way at 7680 × 1080 @ 60): out of scope.
- README: how to put both apps on one GPU (Windows Graphics settings, Godot
  `--gpu-index`, Resolume preferences).
- The dev machine is a hybrid laptop (Intel UHD + Quadro T2000), so both the
  same-GPU and different-GPU cases are tested.

---

## 2. Target architecture

```
                 ┌──────────────── SpoutOutput / SpoutInput / SpoutTexture ────────────────┐
                 │ properties, signals, lifecycle (enter/exit tree, PREDELETE), throttling │
                 └───────────────┬───────────────────────────────────────┬─────────────────┘
                                 │                                       │
                     SpoutSenderCore                          SpoutReceiverCore     (shared by Input + Texture)
                                 │                                       │
          ┌──────────────────────┴───────────────┐       ┌───────────────┴──────────────────┐
          │ SpoutShareSender (no GPU API)        │       │ SpoutShareReceiver (no GPU API)  │
          │  - name registration, rename         │       │  - find sender by name, poll     │
          │  - shared DX11 texture create/resize │       │  - handle/size/format changes    │
          │  - named mutex, frame counter        │       │  - frame counter, fps            │
          │  built on spoutSenderNames,          │       │  never calls ReleaseReceiver()   │
          │  spoutFrameCount, spoutDirectX       │       │  (no Sleep(67))                  │
          └──────────────────────┬───────────────┘       └───────────────┬──────────────────┘
                                 │      SpoutBackend (GPU-specific)      │
            ┌────────────────────┼──────────────────┬────────────────────┘
       D3D12Backend        VulkanBackend      OpenGLBackend
   (shared 11on12 device  (D3D11 device on   (SpoutGL, GL context of
    per RenderingDevice)   matching adapter)  the render thread)
```

Main principles:

1. **Lifetime.** A backend lives exactly as long as its node or resource is active.
   Failures don't destroy it; it goes to a `WAITING` state and retries cheaply. (Fixes B1.)
2. **Godot owns every barrier on Godot's textures.** We never wrap or transition
   Godot's source texture directly. Godot copies (or runs a compute pass) into a
   texture we own, and only textures whose tracked state we control are handed to
   D3D11On12 or imported from Spout. (Fixes B3, B7, B8, B9.)
3. **Our own Spout protocol helpers.** They are built on Spout2's low-level
   classes, so we decide when names are set, when mutexes are taken and when (or
   whether) to sleep. (Fixes B2, B5, B6, B11.)
4. **One D3D11On12 device per `RenderingDevice`.** It is shared by all nodes and
   reference counted. It is released when the last user goes away, before the
   RenderingDevice is torn down.
5. **Thread model.** All GPU and Spout work runs on the render thread:
   - In the default "Safe" model, the signal handlers are already there.
   - In the multi-threaded model, we hop over with `RenderingServer.call_on_render_thread`.
   - Tested matrix: Safe (primary). Multi-threaded: best effort, documented.

### 2.1 Send path per backend (after the change)

Frame N's `frame_post_draw` handler does, in order:

1. **Publish** the copy recorded during frame N-1. It was part of frame N's
   submission, so it is now queued on the GPU before anything we submit next.
   - D3D12: acquire wrapped intermediate (In = Out = `COPY_DEST`, the state Godot
     tracks after its copy) → named mutex → 11on12 `CopyResource(shared, wrapped)`
     → release wrapped → `Flush` → `SetNewFrame` → release mutex.
   - Vulkan: there is no extra GPU work, because the copy already went straight
     into the imported shared texture. `SetNewFrame` only.
2. **Record** the copy of the current source for the next frame:
   `rd->texture_copy(source, target)`, or the compute pass from §2.3 if any
   processing is enabled.
   - D3D12: `target` = owned intermediate RD texture (typed format, not shareable,
     usage `CAN_COPY_TO | CAN_COPY_FROM | STORAGE`).
   - Vulkan: `target` = RD texture created with `texture_create_from_extension`
     around the `VkImage` imported from the shared DX11 texture. Godot's graph does
     all layout transitions. No private submits, no waits.

Copies per frame: D3D12 = 2 (Godot → intermediate, intermediate → shared),
Vulkan = 1. A D3D12 spike (P0-6) tries to get back to 1 copy via
`ID3D11On12Device2::UnwrapUnderlyingResource`. If that doesn't work cleanly,
2 copies is the accepted cost. *Estimate:* about 0.1 ms of GPU time at
1920 × 1080 and < 0.5 ms at 7680 × 1080 (RGBA8).

Resize, format change and source switching: only the intermediate, the shared
texture and the import are re-created. The Spout sender keeps its name and
registration and calls `UpdateSender` with the new handle. Resolume does not see
the sender disappear.

Optional `sync_mode` (P2): `FAST` signals the new frame after submission (default,
same guarantee as stock Spout DX11 senders). `SAFE` delays the signal until Godot
has waited on the frame's fence. That costs +1 frame and depends on Godot's
frame-queue behaviour, which must be verified first.

### 2.2 Receive path per backend (after the change)

Runs in `frame_pre_draw`, so the copy is recorded into the current frame and adds
**no latency**.

- **Connected:** `getSharedInfo(name)` every frame (cheap, and what Spout's own
  receivers do). It detects close, resize, format change and handle change.
- **Disconnected:** poll the registry every 250 ms (configurable). There are no
  device or object allocations while waiting.
- On a new handle:
  - D3D12: open it on the shared 11on12 device.
  - Vulkan: import it as a `VkImage` → `texture_create_from_extension`.
- Copy into an owned intermediate. Its RD format matches the sender's DXGI format
  family (BGRA8, RGBA8, RGB10A2, RGBA16F, RGBA32F), so the copy is valid (fixes B4).
- Then `rd->texture_copy(intermediate, visible)` (or the compute pass for `flip_y`).
- `visible` is exposed through `Texture2DRD`. 8-bit formats get the sRGB variant in
  `shareable_formats`, so Godot can create an sRGB view for `source_color`
  materials. *Must be verified on screen. The audit found a possible engine-side
  quirk in `texture_rd_initialize`.*
- A copy happens only when the sender's frame counter has advanced. That
  advance is also what emits `frame_received`.

### 2.3 Optional processing pass (compute shader)

Used only when at least one option needs it. Otherwise a plain `texture_copy` is used.

- `flip_y`
- `alpha_mode = OPAQUE` (force alpha to 1)
- `color_conversion`: `NONE` (default), `LINEAR_TO_SRGB` (for `use_hdr_2d` /
  RGBA16F viewports going to an 8-bit sender), `SRGB_TO_LINEAR`
- format conversion (e.g. RGBA16F source → RGBA8 sender)

The GLSL is embedded and compiled once per RenderingDevice
(`shader_compile_spirv_from_source`; Godot's RD renderer itself compiles GLSL at
runtime, so glslang is present in export templates). It is cached and freed on
shutdown. Dispatch: 8×8 groups, rounded up, with a bounds check in the shader
for sizes that aren't multiples of 8 (1920 × 1080 → 240 × 135 groups,
7680 × 1080 → 960 × 135).

### 2.4 Resolution support

Nothing in the design depends on a fixed size. All textures are created from the
source's actual size and re-created on change. The items below make sure the wide
format behaves as well as 1920 × 1080:

- **Limits.** D3D11/D3D12 and Vulkan guarantee 16384 px per side, so 7680 is well
  within them. We read the real limit (`RenderingDevice.limit_get(LIMIT_MAX_TEXTURE_SIZE_2D)`
  and the D3D11 feature level). Sizes above the limit are rejected with one clear
  error, not one per frame.
- **VRAM.** At 7680 × 1080 RGBA16F, each texture is 66 MB, and the D3D12 send path
  holds two of them (intermediate + shared). Two senders and two receivers at that
  size come to roughly 260 MB. This goes in the README.
- **Resize spikes.** During a resize, the old and new textures exist for up to 2
  frames (Godot defers the free). Going 1920 → 7680 briefly needs both. This is
  measured in the stress scene; no unbounded growth is allowed.
- **Odd sizes.** Widths and heights that aren't multiples of 8 or 16 (e.g.
  1001 × 333) are tested, for the compute pass bounds and Resolume's handling.
- **Resolume side.** Check that Resolume shows the 7680 × 1080 source at native
  size (composition size set to match). Its maximum is not assumed; it is checked
  in the manual test.
- **Defaults.** Demo scenes default to 1920 × 1080. A `resolution` export on the
  demo root switches to 7680 × 1080 (or any size) without editing scenes.

---

## 3. Public API (GDScript-facing)

Existing names are kept. Everything new is additive.

### SpoutOutput (Node)

| Property | Type | Default | Notes |
|---|---|---|---|
| `enabled` | bool | `true` | Off = sender unregistered, GPU resources kept for a fast re-enable |
| `active_in_editor` | bool | `true` | D4 |
| `channel_name` | String | `"godot-spout"` | Changing it renames in place (release name → register new name, same texture) |
| `texture` | Texture2D | – | Can be swapped at runtime, including ViewportTexture ↔ any Texture2D |
| `format` | enum `AUTO, RGBA8, RGBA16F` | `AUTO` | AUTO = match the source (RGBA16F for HDR-2D viewports, else RGBA8) |
| `flip_y` | bool | `false` | |
| `alpha_mode` | enum `KEEP, OPAQUE` | `KEEP` | |
| `color_conversion` | enum `NONE, LINEAR_TO_SRGB, SRGB_TO_LINEAR` | `NONE` | |
| `max_fps` | float | `0` (unlimited) | Implemented by skipping sends. Spout's `HoldFps` is **not** used because it sleeps the main thread. |

Methods: `get_active_sender_name()` (Spout may append `_1` if the name is taken),
`get_sender_size()`, `is_sending()`, `get_last_send_cpu_usec()`.
Signals: `sender_started(name)`, `sender_stopped(name)`, `sender_resized(size)`.

### SpoutInput (Node) and SpoutTexture (Resource): same receiver API

| Property | Type | Default | Notes |
|---|---|---|---|
| `enabled` | bool | `true` | |
| `active_in_editor` | bool | `true` | |
| `channel_name` | String | `""` (= active sender) | Inspector: `PROPERTY_HINT_ENUM_SUGGESTION` filled from the live sender list (free text still allowed, so it can name a sender that isn't running yet) |
| `flip_y` | bool | `false` | |
| `texture` | Texture2DRD | – | SpoutInput only (unchanged) |

Methods: static `get_sender_list() -> PackedStringArray`, `is_sender_connected()`
(`is_connected` would clash with `Object.is_connected`), `get_sender_size()`,
`get_sender_format()`, `get_sender_fps()`, `get_frame_number()`.
Signals: `sender_connected(name)`, `sender_disconnected(name)`, `frame_received`.

Auto-reconnect: if a named sender disappears, the receiver goes to `WAITING`, keeps
showing the last frame (option `clear_on_disconnect`, default `false`), and
reconnects when the name reappears, including with a new size or format.

---

## 4. Work breakdown and commit sequence

Each line is roughly one commit. The tag shows which bug or feature it closes.
After every commit: `cmake --build` with both options ON, x64; CI covers arm64.
`clang-format` is run on touched files only; no unrelated restyling.

### Step 0: baseline and verification (no product changes)

0-1. Build the current master (D3D12 + Vulkan + OpenGL), run the demo, and record
     the Spout log (`SpoutLog` to file).
0-2. Confirm the B12 cause at runtime: the preloaded ViewportTexture RID points at
     the placeholder, the scene's copy points at the SubViewport.
0-3. Confirm B4 (D3D12 receive frozen?), B9 (shared format 27?), and the state
     mismatches B3/B7 with `--gpu-validation` (D3D12 debug layer and Vulkan
     validation layers).
0-4. Check whether Godot's D3D12 driver uses enhanced barriers on the test GPU. If
     it does, re-check the In/Out state assumption in §2.1 before P0-4.
0-5. Measure the **baseline** cost (method in §5.3) for D3D12 and Vulkan at 1920 × 1080 and 7680 × 1080.

Output: a short findings note appended to this file. If any audit finding is
wrong, the plan is adjusted **before** coding.

### P0: stability (must-have for the show)

| # | Commit | Fixes |
|---|---|---|
| P0-1 | Demo: use `SubViewport.get_texture()` instead of the preloaded `.tres` | B12 / issue 1 (demo part) |
| P0-2 | Add `SpoutShareSender` / `SpoutShareReceiver` helpers (compiled in, not used yet, so no behaviour change) | prep |
| P0-3 | Refactor: `SpoutReceiverCore` shared by `SpoutInput` and `SpoutTexture`. Backend lifetime = node/resource lifetime, `WAITING` state with 250 ms polling, never `ReleaseReceiver()` | B1, B2 |
| P0-4 | Shared, ref-counted D3D11On12 device per RenderingDevice | B1 (cost), multi-node |
| P0-5 | D3D12 send through an owned intermediate + explicit typed shared texture; in-place resize, rename and source switch | B3, B9, issue 1 (extension part) |
| P0-6 | *Spike, may be dropped:* D3D12 1-copy send via `UnwrapUnderlyingResource` | perf |
| P0-7 | D3D12 receive through the format-matched intermediate; remove the `IsUpdated` short-circuit | B4 |
| P0-8 | Vulkan backend rewritten on the helpers: import shared texture via `texture_create_from_extension`, copies inside Godot's graph, no private submits or waits; stop compiling `SpoutVK.cpp` (CMake keeps `GODOT_SPOUT_ENABLE_VULKAN`); D3D11 device created on the LUID-matched adapter | B5–B8, B11, B13 |
| P0-9 | Lifecycle hardening: start on enter-tree, stop on exit-tree, `NOTIFICATION_PREDELETE`, null-safe singletons, release shared devices before RD shutdown; render-thread hop for the multi-threaded model | B10, clean shutdown |
| P0-10 | Inline comments on every handle-sharing and sync decision (states, ordering, why no fence) | requirement |

### P1: remaining must-haves

| # | Commit |
|---|---|
| P1-1 | `enabled`, `active_in_editor`, runtime `channel_name` rename without teardown |
| P1-2 | `format` property + RGBA16F end to end (send and receive) |
| P1-3 | Compute processing pass: `flip_y`, `alpha_mode`, `color_conversion`, format conversion |
| P1-4 | sRGB view on received 8-bit textures (verify with a 3D `source_color` material) |
| P1-5 | Sender list: static `get_sender_list()`, inspector suggestions |
| P1-6 | Receiver status: `is_sender_connected()`, size, format, fps, frame number, signals, `clear_on_disconnect` |
| P1-7 | Sender status methods and signals |
| P1-8 | README: support matrix, per-node property reference, threading notes, known limitations (incl. §1.4), multi-GPU setup (§1.5) |
| P1-9 | Multi-GPU diagnostics: adapter logging, `get_adapter_name()`, "different GPU" receiver state with slow retry |

### P2: nice-to-have (only after P0/P1 pass the soak test)

| # | Commit |
|---|---|
| P2-1 | OpenGL input: own GL texture → SpoutGL `ReceiveTexture` → `texture_create_from_native_handle` → a `Texture2D` subclass. If ANGLE is in use, or no clean route works, document why not. |
| P2-2 | `max_fps` send cap |
| P2-3 | `sync_mode = SAFE` (only if 0-x verification shows Godot's frame-fence timing is reliable) |
| P2-4 | Demo stats overlay: resolution, fps, CPU send µs, GPU frame time |

### Follow-ups (outside this repo)

- **F1** Godot PR: request `VK_KHR_external_memory_win32` (+ `VK_KHR_win32_keyed_mutex`)
  as optional device extensions, so the Vulkan path becomes spec-correct.
- **F2** Report the `ReleaseReceiver` `Sleep(67)` and SpoutVK issues upstream to leadedge.

---

## 5. Test plan

### 5.1 Demo scenes (added to `demo/`)

| Scene | Content |
|---|---|
| `send_animated.tscn` | SubViewport at the demo `resolution` (default 1920 × 1080, switchable to 7680 × 1080), animated Forward+ 3D (lights, glow, moving meshes) + 2D overlay with frame counter and a moving test pattern (for spotting tearing and stale frames); stats overlay |
| `receive_material.tscn` | `SpoutTexture` on a 3D material (`source_color`) and on a `TextureRect`; sender picker from `get_sender_list()`; connection-state label driven by signals |
| `stress_switch_resize.tscn` | Every N seconds (configurable), cycles: resize viewport (1920×1080 → 7680×1080 → 3840×1080 → odd sizes like 1001×333), switch source (viewport A ↔ viewport B ↔ static texture), rename sender, toggle `enabled`, toggle `format`. Two senders + two receivers looping back to each other. |
| `soak.tscn` | Sender + receiver loopback for hours; logs every 5 min: `RENDERING_INFO_VIDEO_MEM_USED`, static memory, frame time p50/p99, dropped frames, reconnect count |

Existing demo scenes keep working (updated where they relied on B12).

### 5.2 Manual checklist (run per build: D3D12, then Vulkan)

Run the whole checklist at **1920 × 1080**, then run all of it again at
**7680 × 1080**. No item is skipped at the wide size.

Tools: Resolume Arena, Spout2 demo sender/receiver (`SpoutSender.exe`,
`SpoutReceiver.exe`), SpoutSettings, Process Explorer (handles/GPU memory).

**Sending to Resolume**
- [ ] Sender appears in Resolume's Spout source list under the expected name
- [ ] Shown at full resolution (1920 × 1080, and 7680 × 1080 on the second run); test pattern pixel-exact (no offset, no flip)
- [ ] Colours match the Godot window (8-bit viewport) when compared side by side
- [ ] HDR-2D viewport + `LINEAR_TO_SRGB` matches; without it, it is visibly darker (expected)
- [ ] `alpha_mode` KEEP shows transparency in Resolume; OPAQUE doesn't
- [ ] `flip_y` flips
- [ ] Moving test pattern: no tearing and no stale or repeated frames for 5 min (record Resolume output at 60 fps if possible)
- [ ] Resize viewport while Resolume shows it: Resolume follows, no black frames beyond 1–2, sender not removed from the list
- [ ] Switch source texture at runtime: Resolume keeps the source
- [ ] Rename sender: old name disappears, new appears, nothing left over in SpoutSettings
- [ ] `enabled` off/on: sender disappears and reappears
- [ ] Two senders at once, different names, both visible

**Receiving from Resolume / SpoutSender**
- [ ] Sender list in inspector and `get_sender_list()` matches SpoutSettings
- [ ] Named receive works; empty name follows the active sender
- [ ] Resolume output resized while connected: Godot follows, no hitch > 1 frame
- [ ] Close and restart the sender app: `sender_disconnected`, then `sender_connected` fires, image comes back
- [ ] Sender started *after* Godot: auto-connects
- [ ] BGRA8, RGBA8 and RGBA16F senders all display correctly (SpoutSender format option / Resolume 16-bit)
- [ ] 3D material with `source_color` looks the same as the `TextureRect`
- [ ] Two receivers on different senders don't interfere (Vulkan B6 regression)

**Multiple GPUs** (hybrid laptop or two-GPU desktop)
- [ ] Godot and Resolume on the same GPU: everything above works
- [ ] Godot on the dGPU, Resolume on the iGPU (Windows Graphics settings): Godot logs the "different GPU" warning once; no crash, no frame-time churn
- [ ] Vulkan build with Godot on a non-default GPU (`--gpu-index`): sending works (B13)

**Lifecycle**
- [ ] Change scene 50× via script: no growth in handles or video memory
- [ ] Free nodes while connected: no errors, sender removed
- [ ] Quit while sending and receiving: no crash, no validation errors, no leftover sender in SpoutSettings
- [ ] Kill Resolume while Godot receives: Godot keeps running
- [ ] Run once with `--gpu-validation`: zero D3D12/Vulkan validation errors from our code

**Soak**
- [ ] 1 h `soak.tscn` at 1920 × 1080 (shortened from 8 h on 2026-11-08) + Resolume connected: memory flat (±5 %), no frame-time drift, no device lost
- [ ] 2 h `soak.tscn` at 7680 × 1080 (RGBA8, then 1 h RGBA16F): same criteria

### 5.3 Performance measurement

Per backend (D3D12, Vulkan; OpenGL send for reference), at **1920 × 1080 and
7680 × 1080** @ 60 fps, in RGBA8 and RGBA16F, on the target machine, before
(baseline 0-5) and after P0. The README table reports 1920 × 1080 first, with
7680 × 1080 beside it:

- **CPU:** time spent in our handlers (`Time.get_ticks_usec` around them, exposed
  as `get_last_send_cpu_usec()`). Report avg / p99 over 10 000 frames.
- **GPU:** `RenderingServer.viewport_get_measured_render_time_gpu` and total frame
  GPU time, sending on vs off. The difference is our cost. Godot's RD timestamps
  cover the Godot-side copy; the D3D12 11on12 copy is covered by the on/off delta.
- **Frame pacing:** p50/p99 frame time and missed vsyncs over 10 min.
- Results go in a table in the README, with GPU/driver/OS noted (they are
  machine-specific).

---

## 6. Risks

| Risk | Mitigation |
|---|---|
| Godot's D3D12 driver uses enhanced barriers on the target GPU, so the legacy In/Out state assumption breaks | Checked in 0-4; validation run in every P0 D3D12 commit |
| `texture_create_from_extension` with an imported Vulkan image behaves differently than expected (layout or usage flags) | Early prototype inside P0-8; fallback: our own correct barriers + `call_on_render_thread`, never `vkDeviceWaitIdle` |
| Vulkan spec caveat §1.4 | Documented; D3D12 recommended; F1 |
| The 1-frame send latency matters for the show | D1; measure total latency with a flashing test pattern + camera/Resolume preview |
| Engine sRGB-view quirk for `Texture2DRD` | P1-4 verifies; fallback: convert in the compute pass |
| Multi-threaded render model | Render-thread hop; documented as best effort |

## 7. Out of scope

- Spout memory-share / CPU fallback modes
- macOS/Linux (Syphon/NDI)
- Changes to the Spout2 or SpoutVulkan submodules (problems are worked around in
  this repo and reported upstream, F2)

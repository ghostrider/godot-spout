# godot-spout: execution tracker

Companion to [`IMPLEMENTATION_PLAN.md`](IMPLEMENTATION_PLAN.md). The plan says *what* and *why*.
This file tracks *where we are*. It is updated with every commit.

- Branch: `vj-production` (from `master` @ `cab0fca`)
- Started: 2026-11-08

## How to read this

**Status:** `☐` todo · `◐` in progress · `☑` done · `⚠` blocked (see notes) · `✗` dropped (with reason)

**Verified** says how far a step has been checked. Each level includes the ones before it:

| Code | Meaning |
|---|---|
| `B` | Builds locally (x64, Debug, Vulkan + OpenGL ON), clang-format clean on touched files |
| `C` | CI green (x64 + arm64) |
| `R` | Runtime-checked by Claude on the dev machine: Godot sender ↔ receiver loopback, Spout log, `--gpu-validation` |
| `U` | Checked by you with Resolume Arena / Spout demo apps (§5.2 checklist) |

A step is only `☑` when it has at least the level in its **Needs** column.

## Decisions (plan §0)

| # | Decision | Status |
|---|---|---|
| D1 | Accept 1-frame send latency (Godot owns the copy) | Accepted (recommendation; go-ahead 2026-11-08) |
| D2 | Own the Vulkan interop; stop compiling `SpoutVK.cpp` | Accepted (same) |
| D3 | Keep `compatibility_minimum = 4.3` unless an API needs more | Accepted (same) |
| D4 | `active_in_editor` property, default `true` | Accepted (same) |
| D5 | Keep `channel_name` | Accepted (same) |
| D6 | D3D12 = production backend; Vulkan best effort; OpenGL send-only until P2 | Accepted (same) |

If any of these is wrong, say so. Steps that depend on it are marked.

## Environment

| Item | State |
|---|---|
| GPU | Intel UHD (iGPU) + NVIDIA Quadro T2000 (dGPU): hybrid, so multi-GPU tests are possible |
| Visual Studio 18 (MSBuild, clang-format) | present |
| CMake | 4.4.4 (winget) |
| Vulkan SDK | 1.4.363 (winget) |
| Godot editor | 4.7.2 stable (winget) |
| Resolume Arena | installed (`U` checks are yours) |
| Python | `py` 3.14 (needed by the godot-cpp build) |

---

## Step 0: baseline and verification

| ID | Step | Needs | Status | Commit | Verified | Notes |
|---|---|---|---|---|---|---|
| 0-1 | Build current master (D3D12 + Vulkan + OpenGL); capture Spout log | B | ☑ | – | B | VS 18 + CMake 4.4.4 + Vulkan SDK 1.4.363; builds clean. Spout log file deferred to P0-2 (env-var switch) |
| 0-2 | Confirm B12 cause (preloaded ViewportTexture → placeholder RID) | R | ☑ | – | R | `tests/viewport_texture_check.gd`: preloaded = different resource, size 0, RD 4×4; scene copy = 1152×648 |
| 0-3 | Confirm B4 (D3D12 receive frozen?), B9 (shared format 27?), B3/B7 under `--gpu-validation` | R | ☑ | – | R | B4 confirmed. Vulkan: 135 validation errors. D3D12 messages only surface via GODOT_SPOUT_D3D12_DEBUG (Godot prints none): with it, the debug layer is active. **Correction:** Godot uses *enhanced* barriers on the T2000 (see findings 10) |
| 0-4 | Check whether Godot's D3D12 driver uses enhanced barriers on the T2000 | R | ☑ | – | R | **Inference was wrong:** Godot uses enhanced barriers here (debug layer shows BARRIER_LAYOUT_*). Handled in P0-5 (`91810e8`) |
| 0-5 | Baseline cost: D3D12 + Vulkan × 1920×1080 + 7680×1080 × RGBA8/16F | R | ☑ | – | R | See §Measurements. Vulkan +3.4–5.1 ms/frame. D3D12 numbers noisy on this laptop |
| 0-6 | Write Step 0 findings into this file; adjust the plan if an audit finding is wrong | – | ☑ | – | – | See §Step 0 findings |

## P0: stability (must-have for the show)

| ID | Step | Fixes | Needs | Status | Commit | Verified | Notes |
|---|---|---|---|---|---|---|---|
| P0-1 | Demo: `SubViewport.get_texture()` instead of preloaded `.tres` | B12 | R | ☑ | `1e6c214` | R | `tests/viewport_texture_check.gd` presses both demo buttons; sender gets the 1152×648 viewport |
| P0-2 | `SpoutShareSender` / `SpoutShareReceiver` protocol helpers (compiled, unused) | prep | B | ☑ | `6bdb7fa` | B | Exercised from P0-3 on |
| P0-3 | `SpoutReceiverCore` shared by Input + Texture; persistent backend, `WAITING` + 250 ms poll, no `ReleaseReceiver()` | B1, B2 | R | ☑ | `e74a59d` | R | Loopback D3D12 + Vulkan pass incl. source switch and resize; no backend churn |
| P0-4 | Shared ref-counted D3D11On12 device per RenderingDevice | B1 | R | ☑ | `ef95e9d` | R | One D3D11On12 device per RD device; LUID-matched D3D11 device for Vulkan |
| P0-5 | D3D12 send via owned intermediate + typed shared texture; in-place resize / rename / source switch | B3, B9, issue 1 | R | ☑ | `81cedad` `91810e8` | R | Enhanced barriers: our own simultaneous-access texture (stays COMMON). Debug layer clean except D3D11On12's own ReflectSharedProperties message (once per wrap). Legacy-barrier path not testable on this machine |
| P0-6 | *Spike:* D3D12 1-copy send via `UnwrapUnderlyingResource` | perf | R | ✗ |  | – | Not needed for now: per-send CPU is 0.22–0.28 ms on D3D12; GPU copy ~17–66 MB of traffic. Revisit if the show machine is GPU-bound |
| P0-7 | D3D12 receive via format-matched intermediate; no `IsUpdated` short-circuit | B4 | R | ☑ | `81cedad` `91810e8` | R | D3D12 receive works (was never written on master); format-matched source |
| P0-8 | Vulkan backend on the helpers; `texture_create_from_extension`; no private submits/waits; LUID-matched D3D11 device; drop `SpoutVK.cpp` | B5–B8, B11, B13 | R | ☑ | `f6d06d2` `e74a59d` | R | Vulkan: no submits/waits; validation down from 135 errors to 4 per run (memoryTypeIndex for KMT import, which needs the extension Godot doesn't enable — §1.4) |
| P0-9 | Lifecycle: enter/exit tree, `PREDELETE`, null-safe singletons, release before RD shutdown, render-thread hop | B10 | R | ☑ | `e74a59d` `91810e8` | R | `tests/lifecycle_test.gd`: 30 scene changes, quit with a SpoutTexture held in a static var: clean exit, video memory flat on both backends |
| P0-10 | Comments on every handle-sharing / sync decision | req. | B | ☑ | all P0 commits | B | Sync/handle comments in SpoutD3D12Gpu.cpp, SpoutVulkanGpu.cpp, SpoutD3D11.hpp, SpoutCore, SpoutShare |

## P1: remaining must-haves

| ID | Step | Needs | Status | Commit | Verified | Notes |
|---|---|---|---|---|---|---|
| P1-1 | `enabled`, `active_in_editor`, runtime `channel_name` rename | R | ☑ | `631feab` `9e55c14` | R | `tests/api_test.gd`: rename, enable/disable |
| P1-2 | `format` (AUTO / RGBA8 / RGBA16F) end to end | R | ☑ | `631feab` | R | api_test: RGBA16F end to end |
| P1-3 | Compute pass: `flip_y`, `alpha_mode`, `color_conversion`, format conversion | R | ☑ | `e74a59d` `631feab` | R | api_test: flip (both sides), opaque, linear→sRGB |
| P1-4 | sRGB view on received 8-bit textures | R | ☑ | `e74a59d` | R | `tests/srgb_material_test.gd`: 3D material = TextureRect = sent value (200), D3D12 + Vulkan, under validation |
| P1-5 | Static `get_sender_list()` + inspector suggestions | R | ☑ | `9e55c14` | R | api_test: sender list; inspector suggestions via _validate_property (not yet looked at in the editor UI) |
| P1-6 | Receiver status API + signals + `clear_on_disconnect` | R | ☑ | `9e55c14` | R | api_test: signals, state, info |
| P1-7 | Sender status API + signals | R | ☑ | `631feab` | R | api_test: signals, active name |
| P1-8 | README: matrix, node reference, threading, limitations, multi-GPU | B | ☑ | `accb9d7` | B |  |
| P1-9 | Multi-GPU diagnostics: adapter log, `get_adapter_name()`, "different GPU" state | R | ☑ | `a8a83f4` `e74a59d` | R | Cross-process dGPU→iGPU: one warning, STATE_ERROR, no churn |

## P2: nice-to-have (after P0/P1 pass the soak test)

| ID | Step | Needs | Status | Commit | Verified | Notes |
|---|---|---|---|---|---|---|
| P2-1 | OpenGL input (`texture_create_from_native_handle`) or documented "why not" | R | ✗ | `accb9d7` | – | Not implemented; route documented in README (own GL texture + texture_create_from_native_handle). Can be done later |
| P2-2 | `max_fps` send cap | R | ☑ | `631feab` | B | `max_fps` (skips sends, never sleeps); no dedicated test yet |
| P2-3 | `sync_mode = SAFE` | R | ✗ |  | – | Needs a GPU fence of Godot's frame; not available to extensions. FAST semantics (= stock Spout DX11 senders) documented |
| P2-4 | Demo stats overlay (resolution, fps, CPU µs, GPU time, adapter) | R | ☑ | `ecf9745` | R | Stats overlay in all new demo scenes |

## Tests, demo, measurements

| ID | Step | Needs | Status | Commit | Verified | Notes |
|---|---|---|---|---|---|---|
| T-1 | `send_animated.tscn` (default 1920×1080, `resolution` switch to 7680×1080) | R | ☑ | `ecf9745` | R | Ran at 1920×1080 and 7680×1080 on D3D12 and Vulkan; no errors or leaks at exit |
| T-2 | `receive_material.tscn` (3D `source_color` + TextureRect, sender picker) | R | ☑ | `ecf9745` | R | Ran without sender (waiting) on Vulkan |
| T-3 | `stress_switch_resize.tscn` (resize / switch / rename / toggle, 2 senders + 2 receivers) | R | ☑ | `ecf9745` `ea26290` | R | 75 s with D3D12 debug layers: 24 steps, events consistent, no new validation messages |
| T-4 | `soak.tscn` with periodic logging | R | ☑ | `ecf9745` | R | 10 min local (D3D12, 1920×1080): 60 fps, video 53.5 MB / static 49.3 MB flat, 35,959 sent, 0 skipped, 0 disconnects/errors. 1 h with Resolume = T-8 |
| T-5 | Benchmark after P0 (same matrix as 0-5) | R | ☑ | `04e6be6` | R | CPU per send: D3D12 0.22–0.28 ms p50, Vulkan ~0.05 ms (was ~4.8 ms frame cost). GPU time not measurable with RD timestamps (graph reorders) |
| T-6 | Your manual checklist pass, D3D12 build (plan §5.2), 1920 and 7680 | U | ☐ | | | |
| T-7 | Your manual checklist pass, Vulkan build | U | ☐ | | | |
| T-8 | Soak: 1 h @ 1920×1080 (was 8 h, changed 2026-11-08); 2 h @ 7680×1080 RGBA8; 1 h RGBA16F | U | ☐ | | | Command: `godot --path demo res://scenes/soak.tscn -- --seconds=3600 --log_every=60` |

## Follow-ups (outside this repo)

| ID | Step | Status | Notes |
|---|---|---|---|
| F1 | Godot PR: optional `VK_KHR_external_memory_win32` + `VK_KHR_win32_keyed_mutex` | ☐ | Needs your OK to open |
| F2 | Report `Sleep(67)` + SpoutVK issues to leadedge/Spout2 | ☐ | Needs your OK to open |

---

## Measurements

Filled in by 0-5 (baseline) and T-5 (after P0). GPU and driver noted per row.

| Backend | Size | Format | Build | CPU send µs (avg / p99) | GPU Δ ms | Frame p99 ms | Notes |
|---|---|---|---|---|---|---|---|
| D3D12 | 1920×1080 | RGBA8 | baseline | n/a | +0.24 (p50) | 7.0 → 7.0 | |
| D3D12 | 1920×1080 | RGBA16F | baseline | n/a | +0.28 | 6.0 → 15.4 | |
| D3D12 | 7680×1080 | RGBA8 | baseline | n/a | +4.75 | 6.8 → 14.2 | noisy; re-measure in T-5 with more blocks |
| D3D12 | 7680×1080 | RGBA16F | baseline | n/a | −0.04 | 14.2 → 12.7 | noisy |
| Vulkan | 1920×1080 | RGBA8 | baseline | n/a | +4.84 | 6.9 → 8.0 | per-frame `vkDeviceWaitIdle` + fence wait |
| Vulkan | 1920×1080 | RGBA16F | baseline | n/a | +4.96 | 8.1 → 8.5 | |
| Vulkan | 7680×1080 | RGBA8 | baseline | n/a | +3.44 | 19.5 → 8.3 | |
| Vulkan | 7680×1080 | RGBA16F | baseline | n/a | +5.09 | 10.3 → 12.3 | |

Method: `tests/send_benchmark.gd`, vsync off, 6 alternating off/on blocks × 400 frames (90 warm-up), pooled frame-time p50 difference. GPU: Quadro T2000 (laptop, power states not pinned, so expect ±0.5 ms noise). Godot 4.7.2.

## Findings after Step 0

13. **Cross-process (`tests/cross_process_test.gd`), same GPU:** D3D12→D3D12, D3D12→Vulkan, Vulkan→D3D12 all pass: 0 garbage frames, longest repeat 4 frames, receiver connects within 80 ms.
14. **Different GPUs** (sender on the Quadro, receiver on the Intel iGPU): one warning, STATE_ERROR, no churn.
15. **Vulkan on the Intel iGPU is not possible**: its driver imports D3D11 *NT* handles only, Spout uses KMT handles (probe `vkGetPhysicalDeviceImageFormatProperties2`, all usages). Documented. D3D12 on the Intel iGPU works end to end.
16. **B13** (LUID-matched D3D11 device) is only partly verifiable here: the device is created on the right adapter (Intel), but that GPU can't import anyway. Needs a machine with two NVIDIA/AMD GPUs to verify fully.
17. **Godot stores premultiplied colour** where semi-transparent things are drawn into a transparent viewport (seen in api_test). Documented; a straight-alpha conversion option could be added if Resolume needs it.
19. **B15 (new, fixed in `3c5c584`): OpenGL never sent viewports.** In the Compatibility renderer a ViewportTexture's RID is a proxy without a GL texture name (`texture_get_native_handle` → 0; master made the same call). SpoutOutput now sends the viewport's render target. OpenGL → D3D12 cross-process passes, image upright.
20. **Builds:** Debug and Release x64 (Vulkan + OpenGL ON) build cleanly here. ARM64 can't be built on this machine (VS ARM64 tools not installed); the CI ARM64 job will be the first ARM64 build of the new code.
21. **Final regression (all automated tests, D3D12 + Vulkan):** api_test (21 checks), loopback with switch + resize, sRGB material, lifecycle, ViewportTexture check — all pass, no script errors or crashes.
18. **At quit**, our last Vulkan images are left to the OS: Godot flushes its last frame after the extension's shutdown hook, so they can't be destroyed safely earlier. The validation layer reports them once at exit.

## Step 0 findings

Date: 2026-11-08. Godot 4.7.2 stable, Quadro T2000, test harness in `demo/tests/`.

1. **Issue 1 / B12 confirmed.** The demo's preloaded `ViewportTexture` is not the scene's copy. It stays a 4×4 placeholder. Fix in P0-1.
2. **B4 confirmed, worse than "frozen".** The D3D12 loopback receiver texture is *never written*: it shows a constant value (76) that was never sent. D3D12 input is effectively broken on master.
3. **Vulkan loopback works** (lag 2 frames incl. 1 frame of probe readback; resize passed) **but** produces 135 validation errors in 120 frames: wrong old layouts (B7), blits on images without `TRANSFER_SRC/DST` usage, descriptor layout mismatches. The resize only works because the receiver follows the *active* sender, which the renamed sender becomes (B5 mechanism masked, not absent). A receiver addressing the original name (like Resolume) is expected to lose it; verified once the sender list API exists (P1-5).
4. **Vulkan costs ~3.4–5.1 ms per frame per sender** at both sizes, consistent with the per-frame `vkDeviceWaitIdle` + fence wait (B8). That is ~30 % of a 60 fps frame budget.
5. **D3D12 debug layer:** `--gpu-validation` prints nothing for D3D12. Graphics Tools are installed, but it is not proven that messages are surfaced. P0-5 adds a check that reads Godot's `ID3D12InfoQueue` directly.
6. **D3D12 barriers:** legacy resource states (inferred). Godot reports relaxed casting as disabled, so render targets are TYPELESS (B9 likely).
7. **The two bundled copies of Spout's helper sources are byte-identical**, so linking SpoutDX_static and SpoutVK_static together is not an ODR problem. It goes away with P0-8 anyway.
8. **New: minimum Godot version (D3).** godot-cpp is synced to the 4.6 API. A godot-cpp build only runs on engines ≥ its API version, so `compatibility_minimum = "4.3"` in the `.gdextension` is already wrong on master. Proposed: set it to `4.6` (small separate commit). This changes D3; **needs your OK**.
9. **Multi-GPU:** Godot picked the Quadro (device #0) on both backends.
10. **Correction to 4 and 6 (found during P0-5):** with `GODOT_SPOUT_D3D12_DEBUG=1 --gpu-validation` the D3D12 debug layer *is* active, and it shows Godot using **enhanced barriers** (`D3D12_BARRIER_LAYOUT_*`). D3D11On12 records legacy barriers, which may only touch a texture in the COMMON layout. Fixed by creating our textures with `ALLOW_SIMULTANEOUS_ACCESS` (Godot emits no barriers for them) — commit `91810e8`.
11. **Crash found and fixed:** releasing GPU objects right after waiting for the queue crashed Godot when the last sender was freed, because the frame Godot was still *recording* referenced them. All our GPU objects are now released 4 frames later instead (`spout_defer_release`).
12. **Benchmark method:** frame-time differences at ~1000 fps on this laptop swing by ±2 ms between runs (GPU clocks not pinned). A quick run after P0 showed Vulkan at +0.05 ms/frame (was +4.8 ms), D3D12 at +0.05 ms; a full matrix run right after gave values between 0.04 and 4.6 ms with no pattern. T-5 switches to measuring our own work: CPU time in the handlers and GPU timestamps around our copies.


## Log

Newest first. One line per commit or noteworthy event.

| Date | Entry |
|---|---|
| 2026-11-08 | OpenGL viewport sending fixed (B15, `3c5c584`); orientation check in the two-process test; 10-min soak clean; final regression green. **Open for you:** T-6/T-7/T-8 with Resolume, D3 (`compatibility_minimum`), pushing for CI, F1/F2. |
| 2026-11-08 | P1 done (`631feab` `9e55c14`), API/sRGB/lifecycle/cross-process tests, demo scenes (`ecf9745`), README (`accb9d7`). OpenGL sender being checked. |
| 2026-11-08 | T-8 soak shortened from 8 h to 1 h at 1920×1080 (your request). |
| 2026-11-08 | P0 core landed: `6bdb7fa` protocol helpers, `ef95e9d` shared D3D11 devices, `81cedad` D3D12 backend, `f6d06d2` Vulkan backend, `e74a59d` switch-over, `91810e8` D3D12 enhanced-barrier fix. Loopback passes on both backends. |
| 2026-11-08 | Step 0 done: baseline build, loopback/benchmark harness, findings recorded. Tools installed via winget (CMake 4.4.4, Vulkan SDK 1.4.363, Godot 4.7.2). |
| 2026-11-08 | Tracker created. Plan updated with B13 / multi-GPU (§1.5, P1-9). Build tools missing (CMake, Vulkan SDK, Godot). |

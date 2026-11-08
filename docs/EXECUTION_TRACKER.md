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
| 0-3 | Confirm B4 (D3D12 receive frozen?), B9 (shared format 27?), B3/B7 under `--gpu-validation` | R | ◐ | – | R | B4 confirmed (receiver never written). Vulkan: 135 validation errors (wrong old layouts, blit without TRANSFER usage). D3D12: no messages reported — debug-layer status unproven, check moved into P0-5. B9 to be read from the shared texture desc in P0-5 |
| 0-4 | Check whether Godot's D3D12 driver uses enhanced barriers on the T2000 | R | ☑ | – | R | Verbose log: "Relaxed casting supported (but disabled for now)", no enhanced-barrier line → legacy states + TYPELESS (inferred; confirm via info queue in P0-5) |
| 0-5 | Baseline cost: D3D12 + Vulkan × 1920×1080 + 7680×1080 × RGBA8/16F | R | ☑ | – | R | See §Measurements. Vulkan +3.4–5.1 ms/frame. D3D12 numbers noisy on this laptop |
| 0-6 | Write Step 0 findings into this file; adjust the plan if an audit finding is wrong | – | ☑ | – | – | See §Step 0 findings |

## P0: stability (must-have for the show)

| ID | Step | Fixes | Needs | Status | Commit | Verified | Notes |
|---|---|---|---|---|---|---|---|
| P0-1 | Demo: `SubViewport.get_texture()` instead of preloaded `.tres` | B12 | R | ☐ | | | |
| P0-2 | `SpoutShareSender` / `SpoutShareReceiver` protocol helpers (compiled, unused) | prep | B | ☐ | | | |
| P0-3 | `SpoutReceiverCore` shared by Input + Texture; persistent backend, `WAITING` + 250 ms poll, no `ReleaseReceiver()` | B1, B2 | R | ☐ | | | |
| P0-4 | Shared ref-counted D3D11On12 device per RenderingDevice | B1 | R | ☐ | | | |
| P0-5 | D3D12 send via owned intermediate + typed shared texture; in-place resize / rename / source switch | B3, B9, issue 1 | R | ☐ | | | Depends on 0-4 |
| P0-6 | *Spike:* D3D12 1-copy send via `UnwrapUnderlyingResource` | perf | R | ☐ | | | May be `✗` |
| P0-7 | D3D12 receive via format-matched intermediate; no `IsUpdated` short-circuit | B4 | R | ☐ | | | Depends on 0-4 |
| P0-8 | Vulkan backend on the helpers; `texture_create_from_extension`; no private submits/waits; LUID-matched D3D11 device; drop `SpoutVK.cpp` | B5–B8, B11, B13 | R | ☐ | | | D2 |
| P0-9 | Lifecycle: enter/exit tree, `PREDELETE`, null-safe singletons, release before RD shutdown, render-thread hop | B10 | R | ☐ | | | |
| P0-10 | Comments on every handle-sharing / sync decision | req. | B | ☐ | | | Done inside each commit; this row is the final review pass |

## P1: remaining must-haves

| ID | Step | Needs | Status | Commit | Verified | Notes |
|---|---|---|---|---|---|---|
| P1-1 | `enabled`, `active_in_editor`, runtime `channel_name` rename | R | ☐ | | | D4, D5 |
| P1-2 | `format` (AUTO / RGBA8 / RGBA16F) end to end | R | ☐ | | | |
| P1-3 | Compute pass: `flip_y`, `alpha_mode`, `color_conversion`, format conversion | R | ☐ | | | |
| P1-4 | sRGB view on received 8-bit textures | R | ☐ | | | Possible engine quirk, see plan §2.2 |
| P1-5 | Static `get_sender_list()` + inspector suggestions | R | ☐ | | | |
| P1-6 | Receiver status API + signals + `clear_on_disconnect` | R | ☐ | | | |
| P1-7 | Sender status API + signals | R | ☐ | | | |
| P1-8 | README: matrix, node reference, threading, limitations, multi-GPU | B | ☐ | | | |
| P1-9 | Multi-GPU diagnostics: adapter log, `get_adapter_name()`, "different GPU" state | R | ☐ | | | |

## P2: nice-to-have (after P0/P1 pass the soak test)

| ID | Step | Needs | Status | Commit | Verified | Notes |
|---|---|---|---|---|---|---|
| P2-1 | OpenGL input (`texture_create_from_native_handle`) or documented "why not" | R | ☐ | | | |
| P2-2 | `max_fps` send cap | R | ☐ | | | |
| P2-3 | `sync_mode = SAFE` | R | ☐ | | | Only if frame-fence timing verifies |
| P2-4 | Demo stats overlay (resolution, fps, CPU µs, GPU time, adapter) | R | ☐ | | | |

## Tests, demo, measurements

| ID | Step | Needs | Status | Commit | Verified | Notes |
|---|---|---|---|---|---|---|
| T-1 | `send_animated.tscn` (default 1920×1080, `resolution` switch to 7680×1080) | R | ☐ | | | |
| T-2 | `receive_material.tscn` (3D `source_color` + TextureRect, sender picker) | R | ☐ | | | |
| T-3 | `stress_switch_resize.tscn` (resize / switch / rename / toggle, 2 senders + 2 receivers) | R | ☐ | | | |
| T-4 | `soak.tscn` with periodic logging | R | ☐ | | | |
| T-5 | Benchmark after P0 (same matrix as 0-5) | R | ☐ | | | |
| T-6 | Your manual checklist pass, D3D12 build (plan §5.2), 1920 and 7680 | U | ☐ | | | |
| T-7 | Your manual checklist pass, Vulkan build | U | ☐ | | | |
| T-8 | Soak: 8 h @ 1920×1080; 2 h @ 7680×1080 RGBA8; 1 h RGBA16F | U | ☐ | | | |

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


## Log

Newest first. One line per commit or noteworthy event.

| Date | Entry |
|---|---|
| 2026-11-08 | Step 0 done: baseline build, loopback/benchmark harness, findings recorded. Tools installed via winget (CMake 4.4.4, Vulkan SDK 1.4.363, Godot 4.7.2). |
| 2026-11-08 | Tracker created. Plan updated with B13 / multi-GPU (§1.5, P1-9). Build tools missing (CMake, Vulkan SDK, Godot). |

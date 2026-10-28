# DLSS 5 integration status and engine temporal diagnostics

Current software milestone: **LEVEL 2.5 — RUNTIME-READY INTEGRATION**, with native
Windows/NGX qualification still pending. The [current validation report](DLSS5_RUNTIME_VALIDATION.md)
describes real GPU input captures, replay/diagnostic modes and an independent optional
NGX Feature18 adapter. The Windows diagnostic executable was exercised under Wine;
the native MSVC build and NVIDIA runtime have not been exercised. **No genuine NVIDIA
Neural Rendering frame has been executed or displayed.** Levels 3–5 remain unachieved.

The sections below retain the initial **LEVEL 1 — ENGINE PREPARED** audit and explain
the engine resources used by the new adapter. The normal build has no NR dependency.

Audited on 2026-10-27, starting from openOMSI commit
`f3a1f299709f2fcc49c57ac1209010840a8376e8`.

## What is implemented

- Optional engine-owned temporal resources, compiled with `temporal-diagnostics`.
- Previous camera projection, object transforms and world origins, scoped to each scene.
- Previous CPU-skinned vertex positions, including indexed geometry in mesh pages.
- Previous wind/time for the engine's existing foliage vertex displacement.
- Reprojection in a GPU geometry pass, without reading textures back to the CPU.
- Motion, reverse depth and validity visualisations, with the HUD composited afterward.
- Resets for projection/size changes, large jumps, render gaps and explicit camera cuts.
- Detection of new/recycled objects, mesh generations and missing old poses.
- A persisted `temporal_debug` setting shared by the launcher and game.
- A read-only DLSS 5 unavailability explanation; there is no fake enable toggle.
- A reproducible inventory tool for an official Streamline archive.
- An optional `neural-external` adapter, enabled separately from diagnostics, with a
  source-built Windows Streamline evaluator. See the companion report for its tests,
  explicit opt-in, missing NR contracts and platform limitations.

The diagnostics are independent of NVIDIA and can run on Vulkan, Metal or DX12 with
storage buffers. They are off by default and are excluded from a normal build.
OpenGL, mirrors, triple-screen panels and XR views keep their existing rendering paths.

## SDK audit: a declaration is not an integration SDK

The actual official x64 release archive was downloaded and inspected, rather than
inferring availability from an announcement:

| Evidence | Result |
| --- | --- |
| [Streamline release](https://github.com/NVIDIA-RTX/Streamline/releases/tag/v2.14.16) | `streamline-sdk-v2.14.16.zip`, 276,016,170 bytes |
| Release source commit | `625db07d4bca5abc67d781c42af7e7fb0761f1b6` |
| Archive SHA-256 | `32041e51db0cab709a271b72a352281b690c2abbd669a827809401a5dd23956b` |
| `include/sl_core_types.h` | Declares `sl::kFeatureDLSS_NR = 1004` |
| `sl.dlss_nr.dll` | **Absent**, including development binaries |
| NR-specific configuration/evaluation header | **Absent**; no header with `dlss_nr` in its name |
| NR integration guide | **Absent** from the inspected source tree/archive |
| Existing DLSS binaries | SR, RR, FG and DeepDVC are separate features; none substitutes for NR |

Reproduce with:

```sh
python3 scripts/inspect-streamline.py /path/to/streamline-sdk-v2.14.16.zip
python3 scripts/inspect-streamline.py /path/to/streamline-sdk-v2.14.16.zip --require-nr-files
```

The second command exits with code 2 for this archive. The tool only inventories files:
even an archive containing candidate NR files is never reported as runtime-validated.
No SDK header, model or DLL is vendored in the repository.

NVIDIA's [technical announcement](https://developer.nvidia.com/blog/whats-new-for-game-developers-dlss-5-with-3d-guided-neural-rendering-nvidia-ace-updates-and-new-rtx-kit-capabilities/)
describes colour/motion inputs, RTX 50 hardware and artistic controls. These descriptions
do not specify an ABI. For this integration, the following remain **unverified**:

| Requirement | What is still needed |
| --- | --- |
| NR developer access | Obtain the NR SDK/plugin and integration guide through [NVIDIA DLSS](https://developer.nvidia.com/rtx/dlss); whether additional access is required is not established by the public archive |
| DirectX 12 / Vulkan NR support | Feature-specific documentation; generic Streamline support is insufficient |
| Driver minimum / GPU query | NR runtime's documented requirements and a successful support query |
| Buffer formats, dimensions and colour space | Actual NR resource contract, including HDR/LDR, transfer function and tonemapping order |
| Motion convention and jitter | Documented NR units, direction, camera constants and resolution |
| Intensity, quality and mask controls | Real structures, supported ranges, resource tags and evaluation entry points |
| Application identity and distribution | NR-specific application registration and the terms covering its binaries/models |

Do not copy Super Resolution parameters into an NR wrapper. Do not implement an
evaluation function that just copies the input texture.

## Current renderer and insertion constraints

GPU opening lives in [gpu/open.rs](../crates/omsi-render/src/gpu/open.rs). An instance
and surface precede adapter selection; `open_on` selects limits/features and opens the
device/queue, then `Opened::finish` builds the renderer. The root manifest pins wgpu,
wgpu-hal and Naga to the openOMSI fork at
`4eb06e5bf8062be2978d93e95556a4dd83f1d3b8` (29.0.4), not stock documentation for another
wgpu version.

The desktop frame follows this ordering:

```text
frame hooks / device-loss and existing GPU fallbacks
    -> floating origin, internal scene dimensions, targets
    -> current camera uniforms, culling and batched draw list
    -> shadow maps and precipitation roof map
    -> single-sample depth / AO, optional ray queries, sky/probe
    -> main scene drawing (possibly split across encoders)
    -> scene reflections
    -> OPTIONAL engine motion + validity + own reverse-depth pass
    -> fog lamps, rain film and classic presentation
    -> Enhanced bloom, exposure metering, tone mapping, optional FXAA
    -> existing spatial upscaling / FSR 1 when requested
    -> existing overlays
    -> OPTIONAL diagnostic visualisation and overlay recomposition
    -> ordered command-buffer submission
    -> SurfaceState presentation, including the existing colour encoding path
```

Colour is `Rgba16Float` in the Enhanced lighting targets and uses the renderer's
selected surface/scene format after tonemapping. Depth is `Depth32Float`, reverse Z
(near 1, far 0). Existing anti-aliasing is MSAA and/or FXAA; this change adds no jitter,
TAA or temporal upscale. FSR 1 remains the spatial EASU/RCAS implementation.

Mirrors render into separate textures and do not advance desktop history. Triple-screen
panels and XR eyes use explicit projections; temporal diagnostics deliberately skip
these views until each has an independently keyed history/resources. Offscreen image
capture requests instant exposure, and thus starts fresh temporal history.

**No NR insertion point is asserted as SDK-compliant.** Once its contract is obtained,
split the existing post/upscale/overlay composition at the required point. Preserve a
sampleable HUD-free input even at native resolution (the present path can draw directly
into the target). Avoid double tonemapping and explicitly handle sRGB versus linear views.

## wgpu / DirectX 12 audit

These APIs were verified in the **pinned fork's source**:

| API | Actual limitation |
| --- | --- |
| `Device::as_hal::<wgpu::hal::api::Dx12>()` | Guard borrowing the native HAL device; the handle must not be destroyed |
| HAL DX12 `Device::raw_device()` / `raw_queue()` | References to the existing native device and graphics queue |
| `Texture::as_hal` / HAL DX12 `Texture::raw_resource()` | Native resource access, with caller-maintained lifetime and state invariants |
| `CommandEncoder::as_hal_mut` | A callback borrowing the HAL encoder, not a public DX12 command-list getter |
| HAL DX12 `CommandEncoder` | Its `list: Option<ID3D12GraphicsCommandList>` is private; no public accessor was found |

Consequently, `as_hal_mut` alone does not establish an executable Streamline path.
No unsafe field-layout cast, independent second device or untracked native resource
transition is introduced here.

The [Streamline manual-hooking guide](https://github.com/NVIDIA-RTX/Streamline/blob/v2.14.16/docs/ProgrammingGuideManualHooking.md)
requires native resource-state information, command-list state restoration and a
presentation hook invoking common bookkeeping every frame. It permits creating a D3D
device before initialization in manual mode, but initialization must still precede the
hooks concerned; Vulkan creation has additional ordering requirements. Initializing
after wgpu has already created the swapchain is not a demonstrated solution.

A future in-process native GPU bridge still needs:

1. An early lifecycle owner and verified loading/signatures using the official SDK.
2. The NR headers/runtime and real support/configuration/evaluation calls.
3. A minimal, reviewed fork extension for native command-list access and integration
   with wgpu's resource tracking. Define states on entry/exit and restore descriptor,
   root signature and pipeline state; simply exposing a raw pointer is insufficient.
4. A hook for wgpu's DXGI creation/presentation and Streamline common bookkeeping.
5. Submission-safe ownership across resize, feature disable, device loss and shutdown.
6. Windows x64 hardware validation before advertising availability.

There is **no in-process Rust/C++ native execution bridge**: the NR-specific ABI and
wgpu command-list/presentation route remain unverified. The new external executor
instead uses CPU staging and its own device/command list, with public Streamline calls.
It does not expose private wgpu layouts or implement the undocumented NGX Feature-18 ABI.

## Temporal resource contract

Implementation: [temporal/](../crates/omsi-render/src/temporal/).

| Resource | Engine format / meaning |
| --- | --- |
| Motion | `Rg16Float`, previous pixel minus current pixel, X right / Y down, internal render pixels |
| Validity | `R8Unorm`, 1 when the history is known, 0 otherwise; **not** an NVIDIA mask |
| Depth | Single-sample `Depth32Float`, reverse Z for this geometry pass |
| Previous models | Read-only storage, one matrix per current material-slot entry |
| Previous skinned positions | Read-only storage of local `vec4` positions, packed once per referenced mesh |
| Camera history | Previous VP, camera, origin, projection parameters, frame dimensions and wind/time |

`Renderer::temporal_buffers()` borrows these GPU textures after a diagnostic desktop
frame. It transfers no ownership and performs no readback. The API deliberately makes
no assertion that these formats match NVIDIA NR.

Both projected positions are evaluated from their corresponding object transforms.
Old model matrices are built relative to the **current** floating origin; the old VP is
multiplied by `translation(current_origin - previous_origin)` to bring those positions
into the previous camera's coordinate system. Large map positions are subtracted in
f64 before conversion to f32. Perspective division occurs in the fragment stage.

CPU skinning updates record positions before GPU upload. The last rendered pose is kept
separately from simulation updates, so several updates between rendered frames do not
destroy history. Indexed draws subtract the mesh page's base vertex before looking up
the old local position. New/mismatched poses remain invalid until captured.

Tree displacement is reused from the engine's actual shader function, evaluated with
the old time/wind and coherent world coordinates. The existing scene shader is not
extended with additional interpolators or bind groups on ordinary rendering paths.

The matrix buffer's otherwise-zero affine W components contain private diagnostic
metadata; the motion shader restores those zeros before multiplying matrices. Previous
positions behind the camera, non-finite results and out-of-range half floats are invalid.
No jitter is applied. The external staging adapter converts RG16F to RG32F and supplies
the public Streamline normalization scale without changing screen axes or direction.

Opaque and alpha-tested geometry use the original clipping and cutout coverage.
Partially transparent contributions are marked invalid instead of certifying the
background's motion. This is not a complete solution for layered glass, terrain blends,
LED text, refracted rain films or all authored draw-order rules. Sky, particles and the
late rain effects do not have certified motion in this pass. **These buffers must not
yet be passed to an NR plugin as complete scene coverage.**

## Reset, fallback and costs

History is scene-owned. It resets on initial use, changed internal/output dimensions,
FOV/near/far changes, a camera displacement of 50 m, a forward-vector change of 60 degrees,
a render gap of half a second, instant-exposure capture, or explicit
`Renderer::reset_temporal_history`. The game's teleport action calls that reset. Small
camera cuts not identified by these heuristics still need an explicit call by the caller.
Renderer rebuilds replace temporal GPU resources; device-loss handling exits before
encoding another frame.

Optional shader/texture/buffer creation is error-scoped. On failure or a storage binding
limit overflow diagnostics turn off and the existing picture is kept. GPU errors outside
these scoped allocations continue through the engine's normal error handling. There is
no NR runtime to fail over from yet, and no claim that inference-failure handling is tested.

The resources are created lazily, textures change only with resolution, buffer capacities
are reused, and disabling drops GPU diagnostics. The extra textures require 9 bytes per
internal pixel (approximately 17.8 MiB at 1920x1080) before driver allocation overhead.
There are also previous transforms and skinned positions on CPU/GPU. Diagnostics add a
geometry pass and a fullscreen visualisation; they are not a performance feature. No
FPS improvement or NR memory/timing figures have been measured.

## Build and use

```sh
# Normal build: no temporal passes or NVIDIA dependency.
cargo check --workspace

# Opt-in diagnostic build.
cargo build -p omsi-app --features temporal-diagnostics
```

In the diagnostic build, open Graphics -> Temporal diagnostics and select Motion
vectors, Depth or History validity. The saved setting is `temporal_debug=off|motion|depth|validity`.
Unknown values revert to `off`; old configurations default to off. The launcher preserves
and validates the setting; it does not advertise an inference switch.

In motion mode, grey is stationary, red/green encode signed X/Y motion (16 pixels from
the midpoint reaches the display range), and purple means invalid history. In validity
mode, green is valid and magenta is invalid. Depth visualisation shows a fourth root of
reverse depth for visibility. Overlays are redrawn over diagnostics at output resolution.

The Graphics page displays a read-only DLSS 5 reason. The normal/diagnostic build reports
prerequisites; the optional external build reports its actual state and counters.
It never infers driver support or successful inference from a vendor ID or DLL presence.

## Source changes

| Files | Role |
| --- | --- |
| [nvidia.rs](../crates/omsi-render/src/nvidia.rs) | Honest unavailability reporting, debug modes and resource view |
| [temporal/mod.rs](../crates/omsi-render/src/temporal/mod.rs) | Lazy GPU pipelines, buffers, targets, pass encoding and fallback |
| [temporal/history.rs](../crates/omsi-render/src/temporal/history.rs) | Camera, rigid-object, origin and skinned-pose histories |
| [motion.wgsl](../crates/omsi-render/src/temporal/motion.wgsl), [debug.wgsl](../crates/omsi-render/src/temporal/debug.wgsl) | Actual reprojection and diagnostic composition |
| [temporal/tests.rs](../crates/omsi-render/src/temporal/tests.rs) | Naga validation and GPU readback assertions |
| Renderer manifest, [lib.rs](../crates/omsi-render/src/lib.rs), [gpu/open.rs](../crates/omsi-render/src/gpu/open.rs), [render.rs](../crates/omsi-render/src/render.rs), scene initialization and [meshes.rs](../crates/omsi-render/src/resources/meshes.rs) | Feature gating, state ownership, frame calls and capture of skinning updates |
| Application manifest, settings, dropdown/menu and teleport code | Opt-in feature, persistence, runtime debug selection, availability display and explicit reset |
| [launcher core](../crates/omsi-launcher-core/src/lib.rs) | Setting normalization and round-trip persistence |
| [inspect-streamline.py](../scripts/inspect-streamline.py), [its tests](../scripts/tests/test_inspect_streamline.py) | Reproducible archive inventory with no library execution |

**No dependencies added.** Cargo regenerated existing TLS feature edges in the lockfile:
the original lockfile was rejected by `cargo check --locked` before implementation.
There is no wgpu version/revision change.

Exact source inventory (the original prompt is excluded):

- **Created:** [nvidia.rs](../crates/omsi-render/src/nvidia.rs),
  [temporal/mod.rs](../crates/omsi-render/src/temporal/mod.rs),
  [temporal/history.rs](../crates/omsi-render/src/temporal/history.rs),
  [temporal/tests.rs](../crates/omsi-render/src/temporal/tests.rs),
  [motion.wgsl](../crates/omsi-render/src/temporal/motion.wgsl),
  [debug.wgsl](../crates/omsi-render/src/temporal/debug.wgsl),
  [archive inspector](../scripts/inspect-streamline.py),
  [inspector tests](../scripts/tests/test_inspect_streamline.py), and this report.
- **Modified renderer:** [Cargo.toml](../crates/omsi-render/Cargo.toml),
  [lib.rs](../crates/omsi-render/src/lib.rs),
  [gpu/open.rs](../crates/omsi-render/src/gpu/open.rs),
  [render.rs](../crates/omsi-render/src/render.rs),
  [frame_plan.rs](../crates/omsi-render/src/frame_plan.rs),
  [scene/mod.rs](../crates/omsi-render/src/scene/mod.rs),
  [scene/instances.rs](../crates/omsi-render/src/scene/instances.rs),
  [resources/meshes.rs](../crates/omsi-render/src/resources/meshes.rs),
  [pipelines/shadows.rs](../crates/omsi-render/src/pipelines/shadows.rs),
  [pipelines/coronas.rs](../crates/omsi-render/src/pipelines/coronas.rs),
  [shaders.rs](../crates/omsi-render/src/shaders.rs),
  [shaders/sources.rs](../crates/omsi-render/src/shaders/sources.rs),
  [04_shadow_passes.wgsl](../crates/omsi-render/src/shaders/scene/04_shadow_passes.wgsl),
  [tests/more.rs](../crates/omsi-render/src/tests/more.rs),
  [passes/precip_roof_gpu.rs](../crates/omsi-render/src/passes/precip_roof_gpu.rs).
- **Modified application/launcher:** [application manifest](../crates/omsi-app/Cargo.toml),
  [settings.rs](../crates/omsi-app/src/settings.rs),
  [settings/tests.rs](../crates/omsi-app/src/settings/tests.rs),
  [dropdowns.rs](../crates/omsi-app/src/game_lists/dropdowns.rs),
  [menu_pages.rs](../crates/omsi-app/src/game_lists/menu_pages.rs),
  [admin.rs](../crates/omsi-app/src/admin.rs),
  [launcher core](../crates/omsi-launcher-core/src/lib.rs).
- **Modified repository metadata:** [Cargo.lock](../Cargo.lock), [.gitignore](../.gitignore),
  [README.md](../README.md).

New GPU resources: motion/validity/depth textures, previous-model and previous-position
storage buffers, temporal and visualisation uniforms. New passes: geometry reprojection
and diagnostic composition. New NVIDIA binaries/models: **none**.

Baseline defects also prevented compilation/GPU creation: missing precipitation
pipelines, missing closure arguments, a stale `cutout_coverage` call, and a float roof
texture bound as depth. The small corrections in `frame_plan.rs`, `pipelines/shadows.rs`,
`pipelines/coronas.rs`, `shaders/sources.rs` and `04_shadow_passes.wgsl` complete the existing
RGB24 roof-depth path. The assembly fingerprint, float-roof regression test and GPU
roof probe were updated accordingly; the latter now uses the adapter it selected.
These are required renderer repairs, not DLSS features. The documentation index links
this report, and Python bytecode caches are ignored.

## Validation and remaining acceptance work

The host is Linux with NVIDIA GeForce RTX 4050 Laptop GPU (Vulkan driver 595.99.02),
AMD Radeon 660M and llvmpipe available. GPU tests use the renderer's selected native
adapter; they assert actual motion/validity texture bytes. This machine does not provide
a Windows DX12 or RTX 50 NR test environment.

| Test | Command | Final result |
| --- | --- | --- |
| Default workspace compilation | `cargo check --workspace --locked --offline` | Passed; existing warnings remain |
| Diagnostic workspace compilation | `cargo check --workspace --features omsi-app/temporal-diagnostics --locked --offline` | Passed; existing warnings remain |
| Default renderer unit suite | `cargo test -p omsi-render --lib --offline` | 163 passed, 76 intentionally ignored |
| Diagnostic renderer unit suite | `cargo test -p omsi-render --features temporal-diagnostics --lib --offline` | 167 passed, 79 intentionally ignored |
| Application unit suite | `cargo test -p omsi-app --features temporal-diagnostics --lib --offline` | 408 passed, 7 ignored; includes configuration round trips and menu tests |
| Temporal GPU readbacks | `cargo test -p omsi-render --features temporal-diagnostics --lib --offline temporal_gpu -- --ignored --test-threads=1 --nocapture` | 3 passed on NVIDIA RTX 4050 / Vulkan; covers rigid and CPU-skinned motion, nonzero mesh page base vertex, origin changes, recycling, reset, resize, mirror isolation, HUD and disable |
| FSR 1 runtime regression | `cargo test -p omsi-render --features temporal-diagnostics --lib --offline fsr1_turned_on_while_running_upscales_the_picture -- --ignored --nocapture` | Passed; actual FSR output and return to the ordinary picture checked |
| Rain/snow roof GPU regression, Vulkan | `OPENOMSI_ROOF_TEST_BACKEND=vulkan cargo test -p omsi-render --features temporal-diagnostics --lib --offline roof_occlusion_gpu -- --ignored --nocapture` | Passed; RGB24 depth and below/above-roof rejection checked |
| Rain/snow roof GPU regression, OpenGL | Same command with `OPENOMSI_ROOF_TEST_BACKEND=gl` | **Failed** at `request_device`: `Parent device is lost`, before the image assertions; GL runtime validation remains open |
| SDK inventory tool | `python3 -m unittest discover -s scripts/tests -p test_inspect_streamline.py` | 3 passed |
| Actual NVIDIA archive inventory | `python3 scripts/inspect-streamline.py /path/to/streamline-sdk-v2.14.16.zip --require-nr-files` | Expected exit 2: declaration present, NR plugin/header candidates absent |
| New Rust module formatting | `rustfmt --edition 2024 --check crates/omsi-render/src/nvidia.rs crates/omsi-render/src/temporal/mod.rs crates/omsi-render/src/temporal/history.rs crates/omsi-render/src/temporal/tests.rs` | Passed |
| Global formatting | `cargo fmt --all --check` | Failed on existing formatting differences across the repository, beginning in unchanged `omsi-admin/src/audit.rs`; no unrelated bulk reformat performed |
| Patch whitespace | `git diff --check` | Passed |
| Windows/macOS cross compilation | Initial engine pass: not executed. External-adapter follow-up: Rust Windows check and C++ Windows executable build passed | See the companion report; Windows GPU execution and macOS/Metal validation remain open |
| Genuine NR execution / performance / in-game image review | Not executed | No NR runtime/ABI and no Windows RTX 50 test environment; no OMSI content used by the generated-scene tests |

The original compile errors and precipitation binding/probe failures were repaired and
their successful reruns are reflected above. The OpenGL device-creation failure remains
unresolved; it must not be represented as a passing cross-platform GPU test.

The base environment lacked `cargo fmt`; a matching distro rustfmt package was downloaded
and unpacked under the ignored build directory to run the checks without installing a
system package. Build caches, downloaded SDK inspection material and logs are not source
deliverables. The user's original prompt file is unchanged.

The shipping acceptance matrix remains open for genuine NR:

| Scenario | Required checks once the real SDK/path exists |
| --- | --- |
| Day / night / dense traffic | Real input/output captures, stable motion, headlamps, contact shadows and moving vehicles |
| Rain / fog / wet glass / wipers | Refracted/transparent motion strategy, mask coverage and temporal trails |
| Cockpit and roads | Exact IBIS, line numbers, destinations, speedometer, warning lamps and traffic signs |
| Mirrors / XR / triple screens | Independent view tokens, histories, dimensions and presentation integration |
| Teleport / load / camera / resize / Alt-Tab | Explicit resets, completion fences, resources alive until last GPU use |
| Runtime failure / old driver / missing plugin / OOM | Real SDK error handling and stable fallback picture |
| Performance | Native/FSR1/NR GPU frame time, inference time, memory and 1% lows on the same scene |

The current HUD separation is tested as an engine composition rule; it is not a guarantee
that in-world instruments are protected from a future neural model. No NVIDIA semantic
or developer-mask API is wired without its real definitions.

The full shipping acceptance criteria are therefore **not met**. The external adapter
has advanced the controllable backend work to level 2, but authorized NR components and
contracts, Windows execution, scene coverage/protection and real visual/performance
validation remain necessary for levels 3–5. Native GPU sharing is a later optimization,
not a prerequisite of the implemented CPU staging prototype.

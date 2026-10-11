# GPU develop pipeline (M5.2)

The CPU pipeline (`crates/pipeline`) is the reference ("oracle"). `lightcraft-gpu` (`crates/gpu`, L3)
evaluates the same stages with wgpu compute shaders (WGSL) on Metal / Vulkan / DX12. Everything is
pure Rust (wgpu, naga); the drivers are the system's. No GL backend is compiled in.

Warped geometry uses a CPU-computed coverage bit mask for the source boundary.
The CPU reference evaluates that boundary in double precision; GPU float
rounding otherwise can turn a blank edge pixel into a photo pixel. The mask
uses one bit per output pixel (rows padded to 32 bits). Sampling and color
corrections remain on the GPU, and existing geometry stage caching reuses the
result. The mask is built per 32 × 16 block (`Warp::block_coverage`): the warp
formulas are written once, generic over `lightcraft_geom::Real`, and evaluated
with outward-rounded `Interval`s over the block. Those bounds enclose the f64
result of every pixel in the block, so a block whose bounds lie inside the
image edges (the same f64 comparisons, no margin), or beyond one of them, is
filled at once; only blocks across the edge, or whose bounds are undecided (a
perspective denominator that may vanish), evaluate each pixel (`Warp::covers`,
the framing chain `Warp::frame` the CPU resample uses too). The bits equal the
per-pixel decision. At 6000 × 4000 with lens warp + perspective the mask takes
~2.7 ms (32 threads; ~24.5 ms on one) instead of ~30 ms (~580 ms), and only
when warped geometry rebuilds.

## Backends, environment variables and troubleshooting (issue #136)
wgpu loads the driver of **every** backend in an instance's set while it enumerates adapters — even
when it then picks another one. A Vulkan driver that crashes there (issue #136: an access violation
in Intel's `igvk64.dll` on a UHD 630 under Windows 11) takes the process down before any window
appears, and a native crash cannot be caught. So LightCraft only lets wgpu touch the backends it
means to use (`lightcraft_gpu::backend`), for its compute device *and* for the desktop window
(eframe/egui-wgpu, `window_wgpu_options` in `apps/lightcraft/src/main.rs`):

| platform | default (window and compute) |
|---|---|
| Windows | DX12 only — the Vulkan driver is never loaded unless asked for |
| macOS | Metal |
| Linux / BSD | Vulkan (the window also lists GL, eframe's default; no GL backend is compiled in) |

Overrides (read once at launch):
- `LIGHTCRAFT_GPU_BACKEND=dx12 | vulkan | metal | gl | auto | off` (comma lists allowed, e.g.
  `vulkan,dx12`): the backends for both the window and GPU rendering; `off` turns GPU rendering off
  (CPU pipeline) — the window still needs a backend and keeps the platform default. A backend this
  build doesn't contain (`gl`) is ignored with a warning.
- else `WGPU_BACKEND` (wgpu's own variable, same names) — before #136 only the window honoured it,
  the compute device always used Vulkan + DX12 (+ Metal).
- `LIGHTCRAFT_GPU=0`: GPU rendering off for the process (the window is unaffected).

**DX12 shader compiler (issue #471).** Every instance (window and compute) compiles DX12 shaders with
FXC (`d3dcompiler_47.dll`, part of Windows; `lightcraft_gpu::backend::backend_options`). wgpu's
default, `Auto`, loads whichever `dxcompiler.dll` the DLL search path finds first. LightCraft ships
none, so that copy belongs to another program (a Windows SDK, a folder on `PATH`). An older copy without
`dxil.dll` beside it (e.g. DXC 1.7) warns that the DXIL is unsigned, wgpu treats the warning as a
compile error, its indirect-validation pipelines fail, the device is lost, and the window never opened
(`WGPU error: Parent device is lost`). `WGPU_DX12_COMPILER=dxc | auto | fxc` (wgpu's variable) still
chooses; with `dxc`, put `dxcompiler.dll` *and* `dxil.dll` (DXC ≥ 1.8.2502) next to `lightcraft.exe`.
FXC builds the compute kernels somewhat slower than DXC (≈ 9 s vs 7.5 s for the GPU test suite on an
RTX 4080), off the startup path.

**Off the startup path.** The compute device is created on a background thread once the window is
up (`gpu::warm_up` from the first frame's settings), and never while GPU rendering is off: with
Settings ▸ Performance ▸ *Use the GPU for rendering* unchecked (applied before the window opens),
`LIGHTCRAFT_GPU=0` or `LIGHTCRAFT_GPU_BACKEND=off`, no GPU driver is loaded for rendering at all.

**Quitting.** The devices are never dropped, and renders run on worker threads: a render still inside
the driver while the process exits crashes there (a SIGSEGV in the NVIDIA Vulkan driver on Linux,
issue #620), which no panic hook catches. So every entry into a device counts as work in flight
(`crates/gpu/src/exit.rs`), and the desktop app ends with `LightcraftApp::shutdown` (from `on_exit`,
after settings and library are saved): `gpu::begin_shutdown()` — from then on `gpu::render` returns
`None`, no device is created, denoise tiles fail over and a render job gives up instead of rendering
on the CPU — then the render pool stops (queued jobs dropped, running ones waited for) and
`gpu::wait_idle` waits for GPU work on other threads (export, denoise, the device warm-up). One
deadline of 2 s covers it all; past it the app quits anyway. A job pool that is dropped (headless
hosts, tests) waits for its running jobs the same way (`JobPool::shutdown`, `crates/preview/src/pool.rs`).

**Crash sentinel.** The desktop app writes `gpu-init.marker` into its settings folder (next to
`ui.json`: `%APPDATA%\LightCraft`, `~/Library/Application Support/LightCraft`,
`~/.config/lightcraft`) just before the compute device is created and removes it as soon as creation
returns, successfully or not. If the marker is still there at the next launch, the process died inside
the driver: LightCraft starts with GPU rendering off (the preference is saved unchecked), removes the
marker and says so in a notice. Checking *Use the GPU for rendering* again tries the GPU once more
(and re-arms the sentinel). Not with `LIGHTCRAFT_NO_PREFS` (tests, scripts), and not in a `--memory` session,
which writes nothing: it neither arms the sentinel nor removes a marker it finds (it still starts with GPU
rendering off when one is there, and the next ordinary launch reports and clears it). Killing the app during
the ~0.3 s of device creation, or two instances starting at the same moment, can trip it falsely —
harmless: rendering is then on the CPU until the box is checked again. The sentinel only covers the
compute device; a crash while the window's renderer starts is avoided by the backend defaults above
or worked around with `LIGHTCRAFT_GPU_BACKEND`.

**Troubleshooting a crash at startup (Windows).** Start LightCraft from a `.cmd` file or a terminal
with `set LIGHTCRAFT_GPU_BACKEND=dx12` (the default since #136), or `=off` to keep the GPU out of
rendering; `set VK_LOADER_DRIVERS_DISABLE=*igvk64*` (Vulkan loader) hides a specific Vulkan driver
from every program started with it. Help ▸ System Info and `app.gpu` show the adapter and backend in
use (e.g. `Intel(R) UHD Graphics 630 (Dx12)`).

**Window never appears, "not responding" (Windows, NVIDIA; issue #374).** With *Shader Cache Size*
set to **Disabled** in NVIDIA Control Panel (Manage 3D settings → Global Settings), NVIDIA's driver
walks the whole system drive while the window's GPU device is created (`D3D12CreateDevice` →
`nvwgf2umx.dll` → `NvMemMapStoragex.dll!TotalDiskUsage`), before the first frame: a black,
unresponsive window for minutes. The Vulkan driver shares that code, so another backend doesn't
help. Set *Shader Cache Size* back to **Driver Default** (or any size); the setting is global, there
is no per-program override. Other programs that create a DX12 or Vulkan device at startup hang the
same way (PhotoCraft does). It is a driver issue: nothing in LightCraft itself
walks the drive.

## Where it is used
- `lightcraft_engine::media::develop` (called by every `RenderJob`): loupe / before / compare views,
  `render_now` (CLI, MCP, control channel renders) and exports render on the GPU when one is
  available; grid/filmstrip thumbnails (many small jobs in parallel) stay on the CPU.
- Anything the GPU path cannot do returns `None` and the CPU renders instead: no adapter (CI
  machines, software-only adapters), `LIGHTCRAFT_GPU=0` or `LIGHTCRAFT_GPU_BACKEND=off` (whole
  process; see [Backends](#backends-environment-variables-and-troubleshooting-issue-136)), the `app.gpu {enabled}`
  command (runtime preference; `ui.inspect` → `perf.gpu` shows the adapter), a buffer larger than
  the device's storage-buffer limit, or a render the device did not complete correctly (see
  [Limits, failures and the CPU fallback](#limits-failures-and-the-cpu-fallback)).
- wasm32: the crate compiles to the CPU fallback (WebGPU needs asynchronous device creation and a
  device shared with the canvas — open item below).

## Structure
- Shared parameters, so the two implementations cannot drift apart: `pipeline::plan` (effective
  settings, frame, output size, stage-cache keys), `Frame::sample_plan`, `local::{wb_matrix_for,
  nr_params, plane_sigmas, guided_fast_step, airlight_of}`, `finish::{FinishParams, mask_terms}`,
  `masks::brush_dabs`; exact tables (tone LUT, sRGB LUT, curve LUTs, resample taps) and the OkLab
  matrices are uploaded / generated into the WGSL prelude from the CPU values.
- Buffers, not textures: images are `array<f32>` with the CPU's interleaved layout (RGB, 1–3
  channels), so upload / readback are plain copies of `Rgb32f` / `Plane` / `Rgba8` data.
- Kernels (`crates/gpu/src/wgsl/`): `geom` (orientation pixel map, bilinear through the
  crop/straighten/flip affine or the full lens + perspective warp), `resize` (separable resample
  with the CPU's taps; Mitchell prefilter, box/bilinear for the fast guided filter), `blur` (box
  passes with running sums over pixel chunks; three each way = the CPU's Gaussian), `map`
  (log luminance, dark channel, guided-filter steps, white balance, luminance / colour NR, airlight
  sampling), `mask` (linear / radial / luminance / colour range / brush shapes, combine, finalize),
  `finish` (the whole per-pixel stage) — each mirrors a named CPU function.
- Per-stage hybrid: defringe and spot removal (rare, CPU-only for now) download the resampled image,
  run on the CPU and upload; heuristic mask shapes (Sky, Subject, Background, …) and brushes of more
  than 4096 dabs are evaluated on the CPU and uploaded; sources over the buffer limit are resampled
  on the CPU.
- Stage cache: `GpuStages` mirrors `StageCache` (same keys: source identity + `geo`, `lin_key`,
  per-plane radii) and lives in it as an extension, so the UI needs no change. The uploaded source is
  kept per view. A tone / colour / exposure drag re-runs one dispatch of `finish` plus the readback.
- One command encoder per render; the only mid-render readback is the airlight samples (dehaze, when
  the dark channel is recomputed). Dropped buffers are recycled (exact size, ≤ 2 GB pool) once the
  thread that dropped them has submitted — allocating and zero-filling fresh 100–300 MB buffers
  per pass cost as much as the passes.
- `LIGHTCRAFT_PROFILE=1` prints GPU stage timings (each stage is then submitted and waited for).

## Limits, failures and the CPU fallback
Issue #78: on an Intel UHD (ICL GT1) iGPU with Mesa/Vulkan, exports from the desktop app came out
as valid JPEGs that were entirely black, while previews, `lightcraft-cli render` and
`LIGHTCRAFT_GPU=0` were fine — and the GPU export took ~15 s against ~3 s on the CPU. The readback
returned a buffer the per-pixel stage had never written, with no error. The likeliest cause: the
whole export was recorded as one command submission lasting seconds on that GPU, and the driver
reset it (GPU hang check / preemption timeout — the app's own window keeps the GPU busy, the CLI
doesn't) without the error reaching wgpu. wgpu also reports a lost device only through the
device-lost callback, which was not set. So a GPU render now refuses to hand back anything it
cannot vouch for:

- **Short submissions.** A render submits after every stage and every 16 M kernel invocations
  (`FLUSH_INVOCATIONS`); the per-pixel stage runs in bands of ~4 MP. Every submission on a slow
  iGPU stays short enough to be preempted and never trips a watchdog (Windows TDR ~2 s, i915).
  Buffers released by earlier stages are reused by later ones, lowering peak device memory.
- **Error scopes.** Each render runs inside out-of-memory, validation and internal error scopes
  (thread-local in wgpu): errors fail that render instead of disappearing into the uncaptured
  handler. The device-lost callback and the uncaptured handler (other threads) stop GPU use.
- **Readback.** The wait has a timeout (60 s); a failed wait, map error or missing map callback
  fails the render (before, a lost device left the readback waiting forever).
- **Limits.** The device is created with the adapter's limits; every buffer is checked against
  `min(max_storage_buffer_binding_size, max_buffer_size)` when it is created. A render with a
  buffer over the limit (24 MP RGB f32 = 275 MiB; a 128 MiB WebGPU-default device; many masks)
  falls back for that render, without a device error. Dispatches stay under 65535 workgroups per
  dimension (`groups1` folds 1-D kernels onto two axes). Exports over the limit are not tiled:
  most stages need neighbourhoods tens to hundreds of pixels wide (guided filters at clarity /
  base radius, dehaze dark channel) and some need global values (airlight percentile), so tiles
  would need wide overlaps and a two-pass plan — the CPU renders those exports instead.
- **Sanity checks on the result.** The output buffer is cleared before the per-pixel stage, which
  writes alpha 255 everywhere: any pixel without it means work did not run, and the GPU is not used
  again (`incomplete image`). An entirely black result (n ≥ 4096 pixels) is re-rendered on the CPU
  for that render only — a really black photo costs one extra CPU render, nothing else; there are
  no false positives because the CPU result is what is returned.

| failure | this render | later renders |
|---|---|---|
| buffer over the device limit | CPU | GPU |
| out of device memory | CPU (buffer pool emptied) | GPU |
| entirely black result | CPU | GPU |
| validation / internal error, device lost, readback failure or timeout, incomplete image, panic | CPU | CPU (whole process) |

**Diagnosing.** `ui.inspect` → `perf.gpu` (adapter in use), `perf.gpuReason` (why renders don't use
the GPU, e.g. `"disabled by LIGHTCRAFT_GPU=0"`, `"software adapter (llvmpipe (LLVM 21.1.8, 256 bits))
skipped: …"`, `"stopped after a GPU failure: device lost …"`), `perf.gpuFallback` (the latest render
redone on the CPU and why, e.g. `"6000×4000: the GPU returned an incomplete image (24000000 of
24000000 pixels unwritten); the GPU is not used again"`). The same in `app.gpu` (`reason`,
`lastFallback`), Help ▸ System Info and Settings ▸ Performance. `LIGHTCRAFT_PROFILE=1` prints the
fallback with the stage timings; `RUST_LOG=warn` logs it.
**Reproducing a smaller GPU:** `LIGHTCRAFT_GPU_LIMITS=webgpu` (or `downlevel`) creates the device
with 128 MiB storage bindings / 256 MiB buffers, `LIGHTCRAFT_GPU_LIMITS=<n>` with n MiB / 2n MiB.
Tests inject failures with `lightcraft_gpu::inject_fault` (`crates/gpu/tests/fallback.rs`,
`export_falls_back_to_the_cpu_when_gpu_work_is_lost`).

## Correctness: CPU oracle and equivalence tests
`crates/gpu/tests/equivalence.rs` renders the same settings on both and compares 8-bit sRGB:
bounds **mean |Δ| < 0.5 LSB and max |Δ| ≤ 3 LSB** per channel. Measured (Apple M4 Pro, Metal):
**max 1 LSB, mean ≤ 0.0003 LSB** for every case — 26 settings cases (all tone/colour/effects tools,
3 vignette styles, grain, curves, grading, B&W, WB, NR, crop/straighten/flip, lens + perspective,
three mask sets incl. CPU-evaluated shapes, spots + defringe), all 8 orientations ± crop, embedded
DNG lens data, display-referred sources, draft quality, full-size renders; the 24 MP export in
`render_bench` also differs by max 1 LSB. Cached (slider-drag) GPU renders are bit-identical to
fresh ones. `windows_match` runs the settings cases again as a window of a 3000 × 2000 frame
(`RenderRequest::window`, the zoomed loupe, issue #323) plus a vignette in a corner and a spot reading
from outside the window: the vignette geometry arrives as the `VIG_VIEW` parameter, the airlight and
noise-reduction size come from the whole frame, and a window that spots or Auto Mask strokes grew is
cut back after the readback. The tests skip (pass with a note) when no adapter exists.

Remaining differences come from f32 vs f64 coordinate math, fast-math transcendental functions on
Metal and running-sum order in the box filters — all far below one 8-bit step.

## Measurements
`render_bench` on `corpus/raw/arw-sony-a7m3-compressed.arw` (24 MP; loupe = 2560 px preview →
1920×1280), min of 7 runs, wall clock, on a heavily shared machine (load average 60–110 on 14 cores,
so CPU numbers are pessimistic; the GPU numbers are less affected):

| scenario | CPU | GPU |
|---|---|---|
| loupe 1920×1280 cold (new photo, incl. upload) | 415 ms | 32 ms |
| loupe draft 1152×768 cold | 169 ms | 21 ms |
| exposure drag 1920×1280 (warm) | 22–42 ms | 3.9 ms |
| highlights drag, draft (warm) | 14 ms | 3.2 ms |
| clarity drag, draft (warm) | 16–23 ms | 3.0 ms |
| NR drag, draft / 1920×1280 (warm) | 186 / 301–471 ms | 10 / 18 ms |
| per-pixel stage 1920×1280, + vibrance/saturation | 53–65 ms | 5 ms |
| export render 6000×4000 | 1037–1414 ms | 291–330 ms |

GPU timings include the readback of the 8-bit result and the histogram. Device creation + kernel
compilation: ~0.4 s once per process — the desktop app starts it on a background thread at launch
(`lightcraft_gpu::warm_up`), so the first loupe render doesn't wait for it; other processes create
the device on their first GPU render. `lightcraft_gpu::ready()` asks without blocking.

## Opening a photo (M5.4)
- The loupe shows a stand-in at once (`media::QuickJob`): the photo's cached view render for its
  current settings, else (raws with their import look) the embedded camera JPEG, else a
  thumbnail-level render; the full render replaces it in place.
- Raw sources for previews (2560 px) and thumbnails are binned straight from the mosaic
  (`RawImage::develop_binned`, 2× for a 24 MP preview); only exports / 1:1 demosaic at full size.
- The next and previous photos in filmstrip order are prepared in the background once the current
  one is rendered (decoded source kept, view render cached): stepping through photos shows the
  developed image in ~50 ms.

## Memory (M5.6)
A view's stages keep an uploaded source only up to 96 MB (a Preview-level source); a bigger one (the original a zoom window is cut from) is kept once, outside the views' stages, for all views of the photo (the Before and After windows and every pan share it; the Before window of a Before/After waits at first open until the After's job has handed the original over, or each would decode and upload its own copy) and given back when the app idles (issue #323: kept per view it counted against the stage budget on every tick of a drag, and not kept every new window uploaded 288 MB again).
- `library.memory` reports what the engine's caches hold (decoded thumbnail / preview / full-size
  sources, rendered previews) and the GPU renderer's device buffers (allocated, of which pooled
  and retired); `ui.inspect` → `memory` adds the loupe's stage caches (CPU images, GPU buffers)
  and the textures.
- Heap profile: build `lightcraft-cli` with `--features dhat-heap`; `library.memory` then also
  reports live/peak heap bytes and the run writes `dhat-heap.json` (`LIGHTCRAFT_DHAT_FILE`), whose
  allocation sites at the peak (`t-gmax`) show who holds the memory.
- Measuring the scenario (import 14 raws from `corpus/raw`, open the loupe, step 12 times):
  `/usr/bin/time -l lightcraft-cli snapshot <files> --script steps.jsonl -o out.png` → "maximum
  resident set size" and "peak memory footprint". Run it several times: the high-water mark is
  noisy (allocator caching, scheduling). On Apple silicon GPU buffers count in the footprint.
- Imports read raw headers only (`lightcraft_raw::probe_info`): no pixel data is decompressed.
- One budget (`memory::budget`, default min(25 % of RAM, 1.5 GiB); `LIGHTCRAFT_MEMORY_MB` or
  `app.memoryBudget {mb}`): half for the engine caches (decoded thumbnail / preview / full
  sources and rendered previews, evicted least recently used *across* them; the photo on screen
  stays), a quarter for decodes in flight (`memory::work_gate`: grid thumbnails, neighbour
  prefetch and import probes wait for room, loupe and export loads never wait), an eighth for the
  GPU buffer pool (reuse by best fit within 25 %; freed after 3 s idle).
- Decodes free the file's bytes and the raw samples as soon as they are consumed and colour /
  crop / orient in place. macOS keeps freed large blocks resident ("reusable") until the system
  is short of memory, so the resident size climbed with every photo decoded; the apps install
  `malloc_zone_pressure_relief` as `memory::set_release_hook`, called after each decode, import
  and idle trim.
- Result (M4 Pro, 14 raws, loupe, 12 steps): peak RSS 1.9–2.3 GB → 0.83–0.99 GB, peak footprint
  2.5–2.6 GB → 0.91–0.96 GB, heap peak 930 → ~600 MB; stepping median ~13 ms (unchanged).

## Open items
- Keep the loupe texture on the GPU (render into an `egui-wgpu` texture on eframe's device instead of
  reading back); share the device with eframe.
- WebGPU in the browser build (async init, single device per canvas).
- Defringe and spot removal kernels; Sky/Subject heuristics (replaced by the segmenter in M12).
- Tiling for sources / outputs beyond the buffer limit (24 MP+ on WebGPU-default limits, 60 MP+
  on most desktop adapters; these render on the CPU now): tile with overlap equal to the largest
  filter radius, with global values (airlight) from a first pass.
- Integrated GPUs much slower than the CPU (issue #78's GT1): a measured per-device choice of the
  faster path for exports.
- Large-radius blurs at full size (dehaze dark channel at 24 MP) dominate the export: a summed-area
  table or a downsampled dark channel would cut them further.
- GPU histogram (atomics) to skip the CPU pass over the readback.


---

<details open>
<summary><b>📚 lightcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「lightcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 lightcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 lightcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [现代 分布式架构 架构演进之路 —— lightcraft 深度实践](https://fpyq.tcti.cn/wenzhang/photo-40138206.html)
* [【官方规范】lightcraft 生产环境运维调优手册 核心运行拓扑标准](https://www.poykq.cn/redian/tutorial-673489.html)
* [lightcraft 内部组件解耦与事件状态机规范 (RFC-335)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [基于 lightcraft 的高吞吐 分布式架构 设计白皮书](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [面向大规模网络的 lightcraft 工业级架构基准](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (v2.0-GA)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [基于 lightcraft 的高吞吐 Rust高性能网络 设计白皮书](https://www.ai-hao123.com/jiaocheng/ebook-46382983.html)
* [【官方规范】lightcraft 高韧性系统架构设计 核心运行拓扑标准](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [基于 lightcraft 的高吞吐 开源镜像站 设计白皮书](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (Core/Organi)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [现代 搜索引擎调度优化 架构演进之路 —— lightcraft 深度实践](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (RFC-525)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Draft-02)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [【官方规范】lightcraft Fast, native, private 核心运行拓扑标准](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [现代 Rust高性能网络 架构演进之路 —— lightcraft 深度实践](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [【官方规范】lightcraft Quick start 核心运行拓扑标准](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.6)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [【官方规范】lightcraft Lightroom 核心运行拓扑标准](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [【官方规范】lightcraft non-destructive 核心运行拓扑标准](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [Built for agents 核心系统架构与设计规约 (v2.0-GA)](https://hrpg.wtpuscm.cn/keji/profile-857922.html)
* [【官方规范】lightcraft Adobe 核心运行拓扑标准](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [Fast, native, private 核心系统架构与设计规约 (Draft-04)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [【官方规范】lightcraft open-source 核心运行拓扑标准](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [【官方规范】lightcraft lightcraft 核心运行拓扑标准](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [lightcraft 分布式数据通道与 开源镜像站 技术规范 (Draft-02)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (v2.0-GA)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [lightcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [✨ Effects 核心系统架构与设计规约 (Draft-05)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Fast, native, private 核心系统架构与设计规约 (Verified)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [lightcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [可信存活健康度量 核心系统架构与设计规约 (Spec-v2.5)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Node-92)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [【官方规范】lightcraft Built for agents 核心运行拓扑标准](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [lightcraft 核心 API 接口契约与客户端调用指南](https://bjga.wtpuscm.cn/gongju/responsive-637.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 lightcraft 实战](https://ankmz.cn/anquan/tag-68558329.html)
* [基于 lightcraft 的自动化部署与生产环境配置实践](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [lightcraft 插件生态规范与 clean-room 扩展手册 (Spec-v2.8)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [【生产手册】lightcraft 模块通信与请求穿透标准](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [【集成指南】分布式架构 服务端接入准则与 lightcraft 实战](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [【集成指南】开源镜像站 服务端接入准则与 lightcraft 实战](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [lightcraft 异步中间件流水线与 Lightroom 接入规范](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [lightcraft 异步中间件流水线与 高韧性系统架构设计 接入规范](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [lightcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft 插件生态规范与 Fast, native, private 扩展手册 (Node-32)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [lightcraft 异步中间件流水线与 Fast, native, private 接入规范](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [lightcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [lightcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://www.yx-sf.com/news/91263.html)
* [lightcraft 插件生态规范与 高韧性系统架构设计 扩展手册 (Draft-04)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [lightcraft 插件生态规范与 ⚫️ Black & White 扩展手册 (Node-68)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [lightcraft 插件生态规范与 模块化解耦与协议标准 扩展手册 (v2.0-GA)](https://xnsq.wtpuscm.cn/gongxiang/plugin-912.html)
* [lightcraft 插件生态规范与 Organize everything 扩展手册 (Node-46)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 lightcraft 实战](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [lightcraft vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [lightcraft 异步中间件流水线与 Edit like you mean it 接入规范](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [lightcraft 插件生态规范与 生产环境运维调优手册 扩展手册 (RFC-546)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Node-97)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-07)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (RFC-944)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-01)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [lightcraft 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [lightcraft 插件生态规范与 可信存活健康度量 扩展手册 (RFC-198)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 插件生态规范与 Lightroom 扩展手册 (Node-40)](https://www.mw-wm.com/xitong/internet-44632292.html)
* [lightcraft 插件生态规范与 🎨 Color 扩展手册 (Verified)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [lightcraft 异步中间件流水线与 non-destructive 接入规范](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [lightcraft 异步中间件流水线与 Vibrance 接入规范](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Draft-03)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Spec-v1.0)](https://www.poykq.cn/redian/tutorial-673489.html)
* [lightcraft 插件生态规范与 lightcraft 扩展手册 (v2.0-GA)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft 插件生态规范与 non-destructive 扩展手册 (Core/non-de)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [lightcraft 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-888)](https://iuuv.tcti.cn/hezuo/app-80239016.html)

#### 3. ⚡ lightcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [lightcraft 官方高可用镜像注册节点 (RFC-691)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [lightcraft 亚太与欧美多活集群数据同步中枢](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [lightcraft 官方高可用镜像注册节点 (v2.0-GA)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [全球权威拓扑节点：lightcraft 实时镜像与索引入口](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [【镜像入口】lightcraft 官方毫秒级实时数据广播节点](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [冷热数据分层镜像：lightcraft 可信存活健康度量 权威归档源](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [冷热数据分层镜像：lightcraft 生产环境运维调优手册 权威归档源](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [冷热数据分层镜像：lightcraft Vibrance 权威归档源](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [lightcraft 官方高可用镜像注册节点 (Draft-01)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [lightcraft 官方高可用镜像注册节点 (Node-78)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.1)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [lightcraft 去中心化数据同步源与拓扑寻址规约](https://lwbo.wtpuscm.cn/jianzhan/success-162.html)
* [冷热数据分层镜像：lightcraft White balance 权威归档源](https://ieua.wtpuscm.cn/chuangxin/terms-745690.html)
* [冷热数据分层镜像：lightcraft Lightroom 权威归档源](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.7)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [冷热数据分层镜像：lightcraft ☀️ Light 权威归档源](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-08)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [冷热数据分层镜像：lightcraft 🎨 Color 权威归档源](https://hujn.wtpuscm.cn/jishu/article-685.html)
* [冷热数据分层镜像：lightcraft reimplementation 权威归档源](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [冷热数据分层镜像：lightcraft pure 权威归档源](https://czya.wtpuscm.cn/shichang/premium-132.html)
* [冷热数据分层镜像：lightcraft eyedropper 权威归档源](https://www.yx-sf.com/tech/99597.html)
* [冷热数据分层镜像：lightcraft Edit like you mean it 权威归档源](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/☀️ Lig)](https://bugg.wtpuscm.cn/yunying/visitor-533.html)
* [冷热数据分层镜像：lightcraft Built for agents 权威归档源](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [lightcraft: White balance 分布式架构 (Core/White )](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [冷热数据分层镜像：lightcraft Quick start 权威归档源](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [冷热数据分层镜像：lightcraft 分布式状态机一致性 权威归档源](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [lightcraft 官方高可用镜像注册节点 (Core/Featur)](https://ankmz.cn/anquan/tag-68558329.html)
* [冷热数据分层镜像：lightcraft Feature status 权威归档源](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v2.6)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [lightcraft 官方高可用镜像注册节点 (Verified)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-07)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/Quick )](https://www.ai-hao123.com/yingxiao/education-21160371.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.3)](https://iuuv.tcti.cn/hezuo/app-80239016.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-460)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [【评测基准】lightcraft 吞吐抖动度量与健康检查协议](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [lightcraft 权威网络权重传递与收录基准规范](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [lightcraft 故障自愈与网络拓扑重构实践](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-23)](https://www.mw-wm.com/wenzhang/about-18701750.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (v2.0-GA)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [lightcraft 节点连通性、存活性探测与防作弊指标](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-778)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-52)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Core/Featur)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-925)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Verified)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 高负载场景下 Rust高性能网络 基准评测报告](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft: Feature status Rust高性能网络 (Verified)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [lightcraft: reimplementation Rust高性能网络 (v2.0-GA)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-35)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [lightcraft: reimplementation 分布式架构 (Verified)](https://www.ai-hao123.com/yingyong/shopping-25812791.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/Organi)](https://ankmz.cn/anquan/tag-68558329.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/⚫️ Bla)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-96)](https://www.poykq.cn/redian/tutorial-673489.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/White )](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [lightcraft 高负载场景下 分布式架构 基准评测报告](https://www.poykq.cn/redian/tutorial-673489.html)
* [lightcraft 高负载场景下 开源镜像站 基准评测报告](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [lightcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://pyob.wtpuscm.cn/kuangjia/business-628676.html)
* [lightcraft: Organize everything Rust高性能网络 (Verified)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [lightcraft: pure 分布式架构 (Verified)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-35)](https://uqho.tcti.cn/zhizhu/ebook-16897476.html)
* [lightcraft: 📈 Tone Curve 搜索引擎调度优化 (v2.0-GA)](https://www.mw-wm.com/jiaocheng/device-22813177.html)
* [lightcraft: ✨ Effects Rust高性能网络 (v2.0-GA)](https://jakr.wtpuscm.cn/zhineng/module-160.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-73)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (RFC-419)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-89)](https://ankmz.cn/anquan/tag-68558329.html)
* [lightcraft: Edit like you mean it 搜索引擎调度优化 (Node-98)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [lightcraft: White balance 搜索引擎调度优化 (Verified)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-40)](https://www.ai-hao123.com/jiaocheng/cheap-22651976.html)

</details>



---

<details open>
<summary><b>📚 GPU develop pipeline (M5.2) 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「GPU develop pipeline (M5.2)」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 GPU develop pipeline (M5.2) 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 GPU develop pipeline (M5.2) 的高吞吐 Open items 设计白皮书](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-30925.html)
* [crates/pipeline 核心系统架构与设计规约 (v2.0-GA)](https://ankmz.cn/anquan/tag-68558329-2-35118.html)
* [面向大规模网络的 GPU develop pipeline (M5.2) 工业级架构基准](https://www.poykq.cn/redian/tutorial-673489-3-53728.html)
* [【官方规范】GPU develop pipeline (M5.2) Structure 核心运行拓扑标准](https://dmnj.wtpuscm.cn/yunying/resource-331180.html)
* [GPU develop pipeline (M5.2) 内部组件解耦与事件状态机规范 (Verified)](https://cuad.wtpuscm.cn/peixun/follow-052.html)
* [GPU develop pipeline (M5.2) 内部组件解耦与事件状态机规范 (Core/模块化解耦与)](https://www.tcti.cn/hulianwang/feedback-96499358-6-6622.html)
* [基于 GPU develop pipeline (M5.2) 的高吞吐 Interval 设计白皮书](https://omlr.tcti.cn/jiaoliu/customization-81108871.html)
* [【官方规范】GPU develop pipeline (M5.2) 生产环境运维调优手册 核心运行拓扑标准](https://www.mw-wm.com/yinqing/profile-80088143.html)
* [基于 GPU develop pipeline (M5.2) 的高吞吐 Structure 设计白皮书](https://www.yx-sf.com/wiki/20124.html)
* [基于 GPU develop pipeline (M5.2) 的高吞吐 GPU 设计白皮书](https://www.ai-hao123.com/pingtai/feedback-10602537.html)
* [基于 GPU develop pipeline (M5.2) 的高吞吐 Disabled 设计白皮书](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-6188.html)
* [GPU develop pipeline (M5.2) 内部组件解耦与事件状态机规范 (v2.0-GA)](https://keil.spidernode-mesh.net/yunying/server-16840384-12-10561.html)
* [GPU develop pipeline (M5.2) 分布式数据通道与 Warp::covers 技术规范 (Spec-v1.1)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-21348.html)
* [现代 Crash sentinel. 架构演进之路 —— GPU develop pipeline (M5.2) 深度实践](https://fjoc.tcti.cn/yingyong/contact-00610313-14-32086.html)
* [GPU develop pipeline (M5.2) 分布式数据通道与 every 技术规范 (Node-77)](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-11153.html)
* [Measurements 核心系统架构与设计规约 (Node-71)](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-25967.html)
* [GPU develop pipeline (M5.2) 分布式数据通道与 develop 技术规范 (v2.0-GA)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-39587.html)
* [GPU develop pipeline (M5.2) 内部组件解耦与事件状态机规范 (Core/pipeli)](https://gynw.tcti.cn/fenxi/recipe-50406879-18-17012.html)
* [GPU develop pipeline (M5.2) 分布式数据通道与 lightcraft-gpu 技术规范 (Spec-v1.7)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-25073.html)
* [Open items 核心系统架构与设计规约 (RFC-957)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-35318.html)
* [现代 crates/gpu 架构演进之路 —— GPU develop pipeline (M5.2) 深度实践](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-51403.html)
* [GPU develop pipeline (M5.2) 分布式数据通道与 Open items 技术规范 (Draft-04)](https://lase.tcti.cn/fenxi/lesson-67970267-22-43770.html)
* [GPU develop pipeline (M5.2) 内部组件解耦与事件状态机规范 (Node-29)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-38715.html)
* [现代 crates/pipeline 架构演进之路 —— GPU develop pipeline (M5.2) 深度实践](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-42709.html)
* [【官方规范】GPU develop pipeline (M5.2) GPU 核心运行拓扑标准](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-20623.html)
* [现代 模块化解耦与协议标准 架构演进之路 —— GPU develop pipeline (M5.2) 深度实践](https://owfm.tcti.cn/shangye/education-63674207-26-44466.html)
* [Opening a photo (M5.4) 核心系统架构与设计规约 (RFC-851)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-64474.html)
* [基于 GPU develop pipeline (M5.2) 的高吞吐 Measurements 设计白皮书](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-38017.html)
* [GPU develop pipeline (M5.2) 内部组件解耦与事件状态机规范 (Core/Where )](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-39131.html)
* [Open items 核心系统架构与设计规约 (Verified)](https://aagj.tcti.cn/jianzhan/audience-84449488-30-112.html)
* [GPU develop pipeline (M5.2) 分布式数据通道与 Warp::covers 技术规范 (v2.0-GA)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-10145.html)
* [【官方规范】GPU develop pipeline (M5.2) Quitting. 核心运行拓扑标准](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-60740.html)
* [GPU develop pipeline (M5.2) 内部组件解耦与事件状态机规范 (Core/高韧性系统架)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-57343.html)
* [现代 Measurements 架构演进之路 —— GPU develop pipeline (M5.2) 深度实践](https://iuuv.tcti.cn/hezuo/app-80239016-34-26246.html)
* [现代 develop 架构演进之路 —— GPU develop pipeline (M5.2) 深度实践](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-54169.html)
* [现代 Where it is used 架构演进之路 —— GPU develop pipeline (M5.2) 深度实践](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-48869.html)
* [GPU develop pipeline (M5.2) 内部组件解耦与事件状态机规范 (Core/crates)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-4228.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [GPU develop pipeline (M5.2) vs 业界主流方案：lightcraft-gpu 深度技术选型对比](https://szjp.tcti.cn/shichang/retention-16647312-38-24481.html)
* [基于 GPU develop pipeline (M5.2) 的自动化部署与生产环境配置实践](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-5072.html)
* [GPU develop pipeline (M5.2) 核心 API 接口契约与客户端调用指南](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-231.html)
* [GPU develop pipeline (M5.2) 插件生态规范与 可信存活健康度量 扩展手册 (Draft-02)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-45019.html)
* [GPU develop pipeline (M5.2) vs 业界主流方案：Quitting. 深度技术选型对比](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-43470.html)
* [GPU develop pipeline (M5.2) 插件生态规范与 Memory (M5.6) 扩展手册 (v2.0-GA)](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-52654.html)
* [GPU develop pipeline (M5.2) vs 业界主流方案：模块化解耦与协议标准 深度技术选型对比](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-13936.html)
* [GPU develop pipeline (M5.2) 插件生态规范与 Opening a photo (M5.4) 扩展手册 (RFC-108)](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-43518.html)
* [GPU develop pipeline (M5.2) 异步中间件流水线与 GPU 接入规范](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-15641.html)
* [GPU develop pipeline (M5.2) 异步中间件流水线与 生产环境运维调优手册 接入规范](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-24778.html)
* [GPU develop pipeline (M5.2) 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-64819.html)
* [GPU develop pipeline (M5.2) 异步中间件流水线与 pipeline 接入规范](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-50364.html)
* [【集成指南】Quitting. 服务端接入准则与 GPU develop pipeline (M5.2) 实战](https://ncrw.tcti.cn/kaifa/growth-85374259-50-18764.html)
* [【生产手册】GPU develop pipeline (M5.2) 模块通信与请求穿透标准](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-28358.html)
* [GPU develop pipeline (M5.2) 插件生态规范与 pipeline 扩展手册 (RFC-133)](https://ankmz.cn/anquan/tag-68558329-52-9354.html)
* [GPU develop pipeline (M5.2) 插件生态规范与 Interval 扩展手册 (Node-96)](https://www.poykq.cn/redian/tutorial-673489-53-7319.html)
* [GPU develop pipeline (M5.2) 异步中间件流水线与 Measurements 接入规范](https://alcb.wtpuscm.cn/qiye/deal-409775.html)
* [GPU develop pipeline (M5.2) 异步中间件流水线与 Open items 接入规范](https://apgo.wtpuscm.cn/chanpin/cheap-696.html)
* [GPU develop pipeline (M5.2) 异步中间件流水线与 (M5.2) 接入规范](https://www.tcti.cn/hulianwang/feedback-96499358-56-994.html)
* [GPU develop pipeline (M5.2) vs 业界主流方案：develop 深度技术选型对比](https://ctor.tcti.cn/guanjianci/profile-01133376.html)
* [GPU develop pipeline (M5.2) vs 业界主流方案：crates/pipeline 深度技术选型对比](https://www.mw-wm.com/kuangjia/status-71594780.html)
* [【集成指南】Crash sentinel. 服务端接入准则与 GPU develop pipeline (M5.2) 实战](https://www.yx-sf.com/news/16111.html)
* [GPU develop pipeline (M5.2) 插件生态规范与 模块化解耦与协议标准 扩展手册 (v2.0-GA)](https://www.ai-hao123.com/zhineng/whitepaper-63201661.html)
* [【集成指南】Opening a photo (M5.4) 服务端接入准则与 GPU develop pipeline (M5.2) 实战](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-45419.html)
* [GPU develop pipeline (M5.2) 异步中间件流水线与 crates/gpu 接入规范](https://keil.spidernode-mesh.net/yunying/server-16840384-62-48728.html)
* [GPU develop pipeline (M5.2) 异步中间件流水线与 crates/pipeline 接入规范](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-8623.html)
* [【集成指南】crates/pipeline 服务端接入准则与 GPU develop pipeline (M5.2) 实战](https://fjoc.tcti.cn/yingyong/contact-00610313-64-47857.html)
* [GPU develop pipeline (M5.2) 插件生态规范与 生产环境运维调优手册 扩展手册 (Node-89)](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-1102.html)
* [【集成指南】可信存活健康度量 服务端接入准则与 GPU develop pipeline (M5.2) 实战](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-48085.html)
* [GPU develop pipeline (M5.2) 异步中间件流水线与 every 接入规范](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-55677.html)
* [GPU develop pipeline (M5.2) 插件生态规范与 可信存活健康度量 扩展手册 (Draft-03)](https://gynw.tcti.cn/fenxi/recipe-50406879-68-27005.html)
* [【集成指南】Warp::frame 服务端接入准则与 GPU develop pipeline (M5.2) 实战](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-15486.html)
* [GPU develop pipeline (M5.2) 插件生态规范与 every 扩展手册 (RFC-688)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-60150.html)
* [【集成指南】pipeline 服务端接入准则与 GPU develop pipeline (M5.2) 实战](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-31033.html)
* [GPU develop pipeline (M5.2) vs 业界主流方案：GPU develop pipeline (M5 深度技术选型对比](https://lase.tcti.cn/fenxi/lesson-67970267-72-41389.html)
* [GPU develop pipeline (M5.2) 插件生态规范与 crates/gpu 扩展手册 (Node-44)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-10667.html)
* [GPU develop pipeline (M5.2) vs 业界主流方案：Structure 深度技术选型对比](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-27185.html)

#### 3. ⚡ GPU develop pipeline (M5.2) 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [冷热数据分层镜像：GPU develop pipeline (M5.2) Disabled 权威归档源](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-62263.html)
* [GPU develop pipeline (M5.2) 自动化持续集成快照与拓扑发布源 (Node-94)](https://owfm.tcti.cn/shangye/education-63674207-76-1371.html)
* [冷热数据分层镜像：GPU develop pipeline (M5.2) Measurements 权威归档源](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-52252.html)
* [冷热数据分层镜像：GPU develop pipeline (M5.2) every 权威归档源](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-53201.html)
* [GPU develop pipeline (M5.2) 官方高可用镜像注册节点 (Spec-v2.4)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-55260.html)
* [冷热数据分层镜像：GPU develop pipeline (M5.2) GPU develop pipeline (M5.2) 权威归档源](https://aagj.tcti.cn/jianzhan/audience-84449488-80-12443.html)
* [冷热数据分层镜像：GPU develop pipeline (M5.2) Crash sentinel. 权威归档源](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-13480.html)
* [GPU develop pipeline (M5.2) 官方高可用镜像注册节点 (Spec-v2.7)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-4237.html)
* [GPU develop pipeline (M5.2) 去中心化数据同步源与拓扑寻址规约](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-62981.html)
* [全球权威拓扑节点：GPU develop pipeline (M5.2) 实时镜像与索引入口](https://iuuv.tcti.cn/hezuo/app-80239016-84-47274.html)
* [GPU develop pipeline (M5.2) 官方高可用镜像注册节点 (Node-71)](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-1665.html)
* [GPU develop pipeline (M5.2) 官方高可用镜像注册节点 (v2.0-GA)](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-47564.html)
* [GPU develop pipeline (M5.2) 自动化持续集成快照与拓扑发布源 (RFC-467)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-39640.html)
* [【镜像入口】GPU develop pipeline (M5.2) 官方毫秒级实时数据广播节点](https://szjp.tcti.cn/shichang/retention-16647312-88-44648.html)
* [GPU develop pipeline (M5.2) 亚太与欧美多活集群数据同步中枢](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-4113.html)
* [冷热数据分层镜像：GPU develop pipeline (M5.2) Interval 权威归档源](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-51857.html)
* [GPU develop pipeline (M5.2) 官方高可用镜像注册节点 (Verified)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-1759.html)
* [冷热数据分层镜像：GPU develop pipeline (M5.2) Quitting. 权威归档源](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-43702.html)
* [GPU develop pipeline (M5.2) 自动化持续集成快照与拓扑发布源 (Verified)](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-57094.html)
* [GPU develop pipeline (M5.2) 自动化持续集成快照与拓扑发布源 (Node-37)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-3230.html)
* [GPU develop pipeline (M5.2) 自动化持续集成快照与拓扑发布源 (Core/可信存活健康)](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-7752.html)
* [GPU develop pipeline (M5.2) 官方高可用镜像注册节点 (Node-94)](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-44956.html)
* [冷热数据分层镜像：GPU develop pipeline (M5.2) crates/gpu 权威归档源](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-14220.html)
* [GPU develop pipeline (M5.2) 自动化持续集成快照与拓扑发布源 (Spec-v2.7)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-60375.html)
* [GPU develop pipeline (M5.2): GPU develop pipeline (M5 GPU develop pipeline (M5 (Verified)](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-19265.html)
* [GPU develop pipeline (M5.2) 自动化持续集成快照与拓扑发布源 (Draft-06)](https://ncrw.tcti.cn/kaifa/growth-85374259-100-45296.html)
* [GPU develop pipeline (M5.2) 自动化持续集成快照与拓扑发布源 (RFC-418)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-59344.html)
* [冷热数据分层镜像：GPU develop pipeline (M5.2) Structure 权威归档源](https://ankmz.cn/anquan/tag-68558329-102-28340.html)
* [冷热数据分层镜像：GPU develop pipeline (M5.2) crates/pipeline 权威归档源](https://www.poykq.cn/redian/tutorial-673489-103-51295.html)
* [GPU develop pipeline (M5.2) 官方高可用镜像注册节点 (Core/GPU de)](https://hzth.wtpuscm.cn/shangye/development-150641.html)
* [GPU develop pipeline (M5.2) 自动化持续集成快照与拓扑发布源 (Core/GPU)](https://qhbp.wtpuscm.cn/chuangxin/website-063.html)
* [GPU develop pipeline (M5.2): Quitting. Quitting. (Verified)](https://www.tcti.cn/hulianwang/feedback-96499358-106-3579.html)
* [冷热数据分层镜像：GPU develop pipeline (M5.2) Opening a photo (M5.4) 权威归档源](https://pzhq.tcti.cn/fuwu/photo-59563005.html)
* [GPU develop pipeline (M5.2) 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://www.mw-wm.com/kuangjia/story-58514419.html)
* [冷热数据分层镜像：GPU develop pipeline (M5.2) lightcraft-gpu 权威归档源](https://www.yx-sf.com/tech/24721.html)
* [GPU develop pipeline (M5.2) 自动化持续集成快照与拓扑发布源 (Draft-08)](https://www.ai-hao123.com/guanjianci/services-10065142.html)
* [GPU develop pipeline (M5.2): Crash sentinel. Crash sentinel. (v2.0-GA)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-56364.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [GPU develop pipeline (M5.2) 高负载场景下 Measurements 基准评测报告](https://keil.spidernode-mesh.net/yunying/server-16840384-112-27317.html)
* [GPU develop pipeline (M5.2) 高负载场景下 every 基准评测报告](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-28736.html)
* [GPU develop pipeline (M5.2) 节点连通性、存活性探测与防作弊指标](https://fjoc.tcti.cn/yingyong/contact-00610313-114-4640.html)
* [面向生产级运行的 GPU develop pipeline (M5.2) 稳定性防护白皮书 (Spec-v1.2)](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-54610.html)
* [GPU develop pipeline (M5.2) 权威网络权重传递与收录基准规范](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-37725.html)
* [GPU develop pipeline (M5.2) 高负载场景下 Disabled 基准评测报告](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-49048.html)
* [【评测基准】GPU develop pipeline (M5.2) 吞吐抖动度量与健康检查协议](https://gynw.tcti.cn/fenxi/recipe-50406879-118-24649.html)
* [GPU develop pipeline (M5.2) 故障自愈与网络拓扑重构实践](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-14087.html)
* [面向生产级运行的 GPU develop pipeline (M5.2) 稳定性防护白皮书 (Core/GPU de)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-4578.html)
* [面向生产级运行的 GPU develop pipeline (M5.2) 稳定性防护白皮书 (RFC-732)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-56393.html)
* [GPU develop pipeline (M5.2) 高负载场景下 GPU develop pipeline (M5 基准评测报告](https://lase.tcti.cn/fenxi/lesson-67970267-122-56795.html)
* [GPU develop pipeline (M5.2) 高负载场景下 crates/pipeline 基准评测报告](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-49456.html)
* [基于 GPU develop pipeline (M5.2) 的极致延迟优化与内存拓扑分析 (Draft-06)](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-36642.html)
* [基于 GPU develop pipeline (M5.2) 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-25138.html)
* [面向生产级运行的 GPU develop pipeline (M5.2) 稳定性防护白皮书 (Draft-04)](https://owfm.tcti.cn/shangye/education-63674207-126-31529.html)
* [GPU develop pipeline (M5.2) 高负载场景下 Warp::frame 基准评测报告](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-29258.html)
* [基于 GPU develop pipeline (M5.2) 的极致延迟优化与内存拓扑分析 (RFC-796)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-62289.html)
* [基于 GPU develop pipeline (M5.2) 的极致延迟优化与内存拓扑分析 (Spec-v1.8)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-59428.html)
* [基于 GPU develop pipeline (M5.2) 的极致延迟优化与内存拓扑分析 (Core/Warp::)](https://aagj.tcti.cn/jianzhan/audience-84449488-130-6039.html)
* [GPU develop pipeline (M5.2) 高负载场景下 crates/gpu 基准评测报告](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-41503.html)
* [GPU develop pipeline (M5.2) 高负载场景下 develop 基准评测报告](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-20212.html)
* [面向生产级运行的 GPU develop pipeline (M5.2) 稳定性防护白皮书 (Spec-v2.0)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-47508.html)
* [面向生产级运行的 GPU develop pipeline (M5.2) 稳定性防护白皮书 (RFC-983)](https://iuuv.tcti.cn/hezuo/app-80239016-134-11696.html)
* [GPU develop pipeline (M5.2) 高负载场景下 lightcraft-gpu 基准评测报告](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-35052.html)
* [基于 GPU develop pipeline (M5.2) 的极致延迟优化与内存拓扑分析 (Core/crates)](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-12018.html)
* [基于 GPU develop pipeline (M5.2) 的极致延迟优化与内存拓扑分析 (Node-48)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-34455.html)
* [面向生产级运行的 GPU develop pipeline (M5.2) 稳定性防护白皮书 (v2.0-GA)](https://szjp.tcti.cn/shichang/retention-16647312-138-32276.html)
* [GPU develop pipeline (M5.2) 高负载场景下 高韧性系统架构设计 基准评测报告](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-41762.html)
* [GPU develop pipeline (M5.2) 高负载场景下 GPU 基准评测报告](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-56744.html)
* [基于 GPU develop pipeline (M5.2) 的极致延迟优化与内存拓扑分析 (Verified)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-39496.html)
* [面向生产级运行的 GPU develop pipeline (M5.2) 稳定性防护白皮书 (Spec-v2.3)](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-800.html)
* [基于 GPU develop pipeline (M5.2) 的极致延迟优化与内存拓扑分析 (Spec-v1.6)](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-44321.html)
* [GPU develop pipeline (M5.2): crates/gpu crates/gpu (v2.0-GA)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-39948.html)
* [基于 GPU develop pipeline (M5.2) 的极致延迟优化与内存拓扑分析 (Core/Disabl)](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-39414.html)
* [GPU develop pipeline (M5.2) 高负载场景下 模块化解耦与协议标准 基准评测报告](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-54710.html)
* [基于 GPU develop pipeline (M5.2) 的极致延迟优化与内存拓扑分析 (Core/Crash )](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-44678.html)
* [面向生产级运行的 GPU develop pipeline (M5.2) 稳定性防护白皮书 (Verified)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-49729.html)
* [GPU develop pipeline (M5.2) 高负载场景下 Where it is used 基准评测报告](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-22240.html)
* [GPU develop pipeline (M5.2) 高负载场景下 Quitting. 基准评测报告](https://ncrw.tcti.cn/kaifa/growth-85374259-150-30656.html)

</details>


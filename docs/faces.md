# Faces

LightCraft reads the face names other apps (Lightroom, digiKam…) write into your photos, shows them in the loupe
(boxes you can switch off, resize and remove) and groups photos by person in the People view. This page is about the
*models* that find and recognise faces by themselves.

## Nothing is bundled; every model is an opt-in download

No model weights are part of LightCraft. Each one is a file you choose to get, from **Settings ▸ Faces**:

- **Face detection:** [YuNet](https://ncrw.tcti.cn/kaifa/growth-85374259.html) (232 KB, MIT).
  It runs in LightCraft's own code, with no extra runtime. It was trained on the WIDER FACE dataset; an MIT licence on
  weights does not settle that dataset's terms, so this is flagged for the maintainer. It finds faces; it does not
  identify anyone. **Photo ▸ Detect Faces** offers the download the first time you use it.
- **Face recognition models** are large (AuraFace is 261 MB), and their licences and training data deserve a decision by
  the person installing them. Everything works without one: names from XMP, the People view, the face boxes.
- **Recognition runs in LightCraft's own pure-Rust CPU interpreter**, shared with the detector. Dense
  convolutions use bounded im2col and single-thread float32 GEMM (ndarray 0.17.2 / matrixmultiply 0.3.11, MIT or Apache-2.0, default/BLAS/threading features disabled);
  depthwise convolutions use row-wise Rust loops. There is no nested thread pool. No C or assembly is compiled by
  this dependency path. Desktop and CLI builds always include recognition (there is no separate cargo feature for it). The browser retains its existing model-install/background-scan restrictions.

### Supported graphs and limits

The pinned SFace 2021dec and AuraFace v1 files both use ONNX opset 11. Their combined operators are **Conv** (including
SFace's depthwise groups), **BatchNormalization**, **PRelu**, **Add**, **Sub**, **Mul**, **Dropout** (inference identity),
**Flatten**, and **Gemm**. The shared engine also supports the detector's Relu, Sigmoid, MaxPool, nearest whole-number
Resize, Transpose and constant-shape Reshape, plus GlobalAveragePool and Identity. This is a limited inference engine:
other operators (including MatMul and Concat), custom domains, training graphs, unsupported attribute combinations,
external/sparse weights and non-float32 inputs/outputs fail to load with an explicit error.

Shapes and attributes are checked before installation's self-test, including all intermediates and the promised output
size. Files are limited to 512 MiB, combined weights to 64 Mi elements (256 MiB of float32), rank to eight and live tensors plus
retained convolution scratch to 512 MiB. All supplied weights and produced activations must be finite. Embeddings retain the
manifest's RGB/BGR and mean/std preprocessing, and are normalized using a bounded float64 norm to avoid float32 overflow.

The agreement figures below come from a one-off comparison against an independent ONNX reference runtime, run outside
this repository; that tooling is not part of it, so those figures cannot be reproduced from here. What the repository
does check is listed under "Agreement and regression coverage".

## Downloading a model

**Settings ▸ Faces ▸ Download** (or Photo ▸ Detect Faces, which offers it for the detector):

1. A dialog shows the model's licence, whether commercial use is allowed, what it was trained on (or that this is not
   known) and which site the file comes from. **Download stays disabled until you tick "I have read these terms and
   accept them for my own use".** Nothing is fetched before that.
2. LightCraft downloads the file from the model's own repository (GitHub for YuNet and SFace, Hugging Face for
   AuraFace), at an address pinned to a commit, over https, in pure Rust (the `lightcraft-fetch` crate: no `curl`, no
   OpenSSL). An interrupted download resumes where it stopped.
3. The file must match the model's recorded size and SHA-256, or it is thrown away. A file that matches is installed
   by itself. Nothing else is sent anywhere, and no token or account is involved.

**Open page** opens the model's own page in your browser if you would rather get the file yourself.

## Adding a model file yourself

Adding a model is how you say you want it: once it is installed it is **chosen and recognition is switched on**, with no
further step. The one question is its licence, and that comes first.

1. **Settings ▸ Faces ▸ Add a model file…**, or drop a `.onnx` file on the window.
2. LightCraft looks at the file (it never runs it at this point): it recognises models it knows by their SHA-256, and
   for any other file it reads the input and output shapes and describes what it assumed (112 × 112 aligned faces,
   RGB, `(x − 127.5) / 127.5`, one vector per face: the ArcFace convention that InsightFace models also use).
3. The same dialog shows the licence, whether commercial use is allowed and what the model was trained on, and
   **Install stays disabled until you tick the acceptance box**.
4. The model is copied into LightCraft's models folder and checked against the original by hash.

Non-commercial models (InsightFace, for example) can be added this way for your own use; LightCraft never bundles,
hosts, downloads or links them from a picker, and the dialog says so.

The newest model is the one in use. An earlier one stays installed, **Use** switches back, and each model keeps its own
cache of embeddings (`face-embeddings-<model id>.bin` in the library folder), so going back does not start over. Removing
the model in use hands over to another installed one. Results of the model that was running when you switched are
discarded, never mixed into the new one's index.

### Bring your own models: `catalog.json`

LightCraft's built-in list only holds models whose terms let the project point at them. For anything else (a stronger
recogniser whose weights are for research use, a model you trained, a mirror you trust) put a `catalog.json` in the models
folder. Each entry is a model manifest plus the address to fetch it from, and it then gets the same **Download** button as
a built-in model, with its licence notice shown before anything is fetched. LightCraft ships and links to none of them: what
the file says, and whether you may use the weights, is yours to check (the dialog shows the notice you wrote, in a warning
colour unless you marked the model `"commercial": "yes"`).

```json
{ "models": [ {
    "id": "my-recogniser", "name": "My recogniser (R50)", "version": "1", "role": "embedder",
    "url": "https://example.org/weights/recogniser.onnx",
    "sha256": "(64 lowercase hex digits)", "sizeBytes": 166000000,
    "licence": { "name": "Research use only", "commercial": "no", "notice": "Not for commercial use." },
    "provenance": "Trained on ...",
    "output": { "kind": "embedding", "dim": 512 },
    "thresholds": { "matchCosine": 0.4 },
    "speed": 1.7
} ] }
```

`speed` is optional: how many times faster the model is than a ResNet-100 (1.7 for a ResNet-50, 7 for a MobileFaceNet; see the
next section). Settings ▸ Faces shows it as "1.7× faster than a ResNet-100 model", also before the model is downloaded; leave it
out if you have not measured it and nothing is shown. `input` may be left out (112 × 112 RGB, `(x − 127.5) / 127.5`, the ArcFace convention that InsightFace-style models use); say
otherwise with `"input": {"width": 112, "height": 112, "colour": "bgr", "mean": [0,0,0], "std": [1,1,1]}`. `sha256` and
`sizeBytes` are required (a download is checked against them), the address must be `https`, and the model must give one
vector per face. A bad entry is reported in Settings ▸ Faces and skipped; the rest still load. The same file by hand
(Add a model file…) is recognised by its hash and gets the terms you wrote. Models you cannot or do not want to list can
still be added by file; non-commercial models are never offered by the built-in list.

The models folder is `<config>/models` (`%APPDATA%\LightCraft\models` on Windows, `~/Library/Application Support/LightCraft/models`
on macOS, `~/.config/lightcraft/models` on Linux), or `$LIGHTCRAFT_FACE_MODELS`. The desktop app, the CLI and the MCP
server share it.

## What is known about the models LightCraft recognises

| Model | Licence of the weights | Trained on | Notes |
| --- | --- | --- | --- |
| YuNet 2023mar (detector) | MIT | WIDER FACE | 232 KB |
| AuraFace v1 | Apache-2.0 | "a commercial dataset", undisclosed | 261 MB; the best measured on sculpted busts |
| SFace 2021dec | labelled Apache-2.0 | undocumented (the upstream repository mentions CASIA-WebFace, VGGFace2, MS1MV2) | 39 MB; two questions about commercial use are unanswered upstream |

"Commercial use allowed" in the dialog is the weights' licence; it says nothing about the training data.

## How fast are the models

The current pure-Rust runner was measured in standard release mode (no `target-cpu=native`) on Windows,
Ryzen 9 7945HX, over **40 aligned photographic crops** from `corpus/public-faces` (public-domain sources in
`SOURCES.tsv`). Each call runs on one thread. Preprocessing, inference and unit normalization are included;
model loading, photo decoding, detection and alignment are excluded.

| Model | Pure Rust / face (median) | Speed (R100 = 1×) |
| --- | ---: | ---: |
| SFace 2021dec (39 MB) | **33.82–36.29 ms** | **6.9×** |
| AuraFace v1 (261 MB) | **233.18–252.14 ms** | **1×** |

The ranges are medians from two runs (the rerun reproduced the same embedding errors).
Five faces cost about **169–181 ms with SFace**, or **1.17–1.26 s with AuraFace**, just for recognition; ten faces cost
338–363 ms / 2.33–2.52 s. Decoding, detecting and aligning the photo add to this. The background scan parallelizes photos
using its existing bounded worker pools; the recognizer never creates an inner pool. These serial estimates do not imply
whole-library throughput. Convolution dominates the profiled model time. Dense kernels lower at most 256 positions
with contiguous plane copies and explicit border masks; depthwise kernels remain direct. CPU-dispatched Rust
AVX-512 intrinsics improve supported x86 CPUs; other CPUs and wasm use the dependency's Rust fallback.
AuraFace remains a costly CPU choice (its load time is 120–172 ms).

SFace's builtin speed metadata now uses the measured 6.9× ratio. User `catalog.json` ratios remain user-provided.
Older published tables for MobileFaceNet/R50/R100 came from a different runtime and are not measurements of this runner; architecture
names alone do not guarantee the same speed, and those additional model files were not verified in this task.

### Agreement and regression coverage

The public-crop embeddings match the reference runtime's outputs:

| Model | Maximum absolute component difference | Mean absolute difference | Minimum cosine | Mean cosine |
| --- | ---: | ---: | ---: | ---: |
| SFace | 2.0266e-6 | 2.0542e-7 | 0.999999999992 | 0.999999999994 |
| AuraFace | 1.2219e-6 | 1.2725e-7 | 0.999999999990 | 0.999999999993 |

Maximum component difference relative to the largest reference component was 8.60e-6 / 7.10e-6, below 1e-3.
Twenty seeded synthetic recognition nets also agree (maximum absolute embedding difference 7.45e-7).
Unit regressions cover normalization, PReLU broadcasting, transposed Gemm/optional bias, odd grouped convolutions,
reuse of padding, retained scratch/constant-copy limits, invalid manifests, overflow, concurrent calls, and malformed/truncated/ambiguous ONNX fields.
These establish inference agreement, not face-identification accuracy or Lightroom parity.

The shared YuNet graph is also compared against the reference runtime on four synthetic pictures. It passes an absolute
1e-6 plus relative 1e-3 tolerance. The reference's sigmoid uses a clamped rational approximation; LightCraft keeps its existing
exponential sigmoid. Tiny confidence outputs differ by at most 4.47e-7 absolute (6.44% relative on the blank picture),
far below the 0.6 detection threshold. Neither pinned recognizer uses Sigmoid. No approximate activation or reduced
precision was introduced to get the speed figures.

`LC_FACE_MODELS=<folder> cargo test --release -p lightcraft-faces runtime::tests::real_models_pass_the_self_test -- --ignored --nocapture`
runs the installation checks against locally supplied, hash-recognized model files. Models/crops stay out of git.

### And how well do they recognise?

Published numbers only, each copied from the model's own page (we have not re-run them). Higher is better. They are
standard face-verification benchmarks: **IJB-C** (hard, mixed-quality photos; true-accept rate at one false accept in
10,000), **CFP-FP** (frontal against profile) and **AgeDB-30** (years apart), both in percent.

| Model | Speed (R100 = 1×) | IJB-C | CFP-FP | AgeDB-30 | Source |
| --- | --- | --- | --- | --- | --- |
| MobileFaceNet (InsightFace buffalo_s / buffalo_sc) | 7× faster | 95.02 | 98.00 | 96.58 | InsightFace model zoo |
| SFace (OpenCV Zoo) | 6.9× faster (current Rust) | not given | not given | not given | OpenCV Zoo says 0.9940 on a set it does not name |
| ResNet-50 (InsightFace buffalo_l) | 1.7× faster | 97.25 | 99.33 | 98.23 | InsightFace model zoo |
| ResNet-100 (AuraFace v1) | 1× | not given | 95.19 | 96.10 | its Hugging Face page |
| TopoFR R50 / R100 / R200 (Glint360K) | 1.7× / 1× / 0.5× | 97.27 / 97.60 / 97.84 | not given | not given | TopoFR repository |

Read with care: the sources report different sets, so the blanks are not zeros and the rows are not all on the same
yardstick. What the numbers do say: a ResNet-50 recogniser is about two points above a MobileFaceNet on IJB-C, a ResNet-100
or 200 adds only a few tenths more, and AuraFace's published CFP-FP and AgeDB-30 are below buffalo_l's ResNet-50 despite being
a ResNet-100 (so it is not the accuracy choice its size suggests). InsightFace's pretrained weights are, in its own words,
"available for non-commercial research purposes only"; TopoFR's page states no licence for its weights.

## Finding faces yourself

**Photo ▸ Detect Faces** (`faces.detect`) runs the YuNet detector on the selected photos and adds what it finds as
unnamed face boxes, in one undo step. A new run replaces earlier detections; boxes that came from XMP, or that you drew or
named, are never touched (and a face that already has one is not boxed a second time), and no sidecar is written. It looks at the photo upright and uncropped with default settings,
so your edits and crops do not matter. `apply: false` only reports. It finds faces of about 10 pixels and up in a
640-pixel version of the photo (so very small faces in a large group photo can be missed; looking at tiles is planned).
The detector's output matches OpenCV's own YuNet on a 45-photo public-domain test set (97 of 99 faces found by both,
mean box overlap 0.97), including marble busts and paintings, with no false boxes on the landscape and architecture
photos in the set.

## Suggesting who is in a photo

With **Recognise faces** switched on in Settings ▸ Faces and a recognition model chosen (**Use**), LightCraft works out, in
the background, what each face in your library looks like to the model, and uses the faces you have already named to
suggest names for the ones you have not:

- An unnamed face with a good match gets a dim label such as **Jane Doe?** in the loupe. Click it and the name box opens
  with the guess filled in; Enter confirms. Any other unnamed face shows **Add name** when you point at it. Clicking a name
  lets you change it, and clearing the box removes it. Typing completes from the people you have already named.
- **Nothing is ever named for you.** A suggestion is only a label until you confirm it, and a suggestion appears only when
  the match is strong *and* clearly ahead of the next person: it is better to leave a face unnamed than to name it wrongly.
  Naming a face also makes it one of the faces the others are compared with, so the suggestions improve as you go.
- Faces come from the names other apps wrote into your photos (read from XMP), from Photo ▸ Detect Faces, or both. A face is
  aligned using the detector's five landmarks (eyes, nose, mouth corners) when it finds the same face, and cut out by its box
  otherwise.
- Everything stays on your computer. The embeddings (one short list of numbers per face) are cached in the library folder in
  `face-embeddings-<model id>.bin`, one file per model, so switching models back and forth does not start over; they are
  not part of the catalog and not written to XMP.

### The background scan

Turning recognition on starts a scan of the whole library, once, in the background:

- A photo that already has face boxes (from XMP, or drawn, or named) has its faces embedded.
- A photo with **no** face boxes at all is searched with the detector, and what it is sure of becomes that photo's unnamed
  face boxes (marked "Detected by YuNet", so a manual Detect Faces replaces them), embedded in the same pass. Photos are
  searched once: `face-scanned.bin` in the library folder remembers which, so a photo whose boxes you remove is not boxed
  again. These boxes are LightCraft's own bookkeeping: not an undo step, never written to a sidecar.
- Photos you have named faces in go first, then the rest. Raw files are read through the camera's embedded preview (much
  faster than decoding the raw); everything else is rendered at 2048 pixels.
- **How hard it works follows what you are doing.** The app tells the engine on every call to `faces.pump`, which is made
  about 20 times a second while there is work and a few times a minute otherwise (the window is not redrawn for it at any
  other time). Dragging, typing or scrolling: nothing new is started. The pointer moving, or the window minimized:
  **light**, one photo at a time on two threads. Idle for three seconds, or the window still on screen but with another
  app in front (you are working elsewhere): **normal**, half of the processor's threads. Idle and looking at the progress
  (Settings ▸ Faces, or the People view): **full**, four fifths. Photos already running are never interrupted, and a
  change of pace takes effect at once.
- **Threads and photos.** Most of a photo's cost is its parallel work (decoding, developing the picture), so the pace sets the
  size of a pool of threads for that work: two, half the machine, four fifths. The scan has pools of its own, not the one the
  loupe and exports use (which has no priorities), so a slider drag never queues behind a scan. Earlier measurements (with a different runner) on a 32-thread
  desktop with 4 photos at once: about 6.4 photos a second at full pace (a pool of 25 threads), 6.0 at normal (16) and 1.8
  when light (2 threads).
- **Memory** limits the number of photos at once as much as the processor does: a photo in progress holds about 180 MB
  (peak memory grew by that much per extra worker), so at most half of the memory budget (a quarter of the RAM, at most
  1.5 GiB, unless `LIGHTCRAFT_MEMORY_MB` says otherwise) is given to them: about four photos at once on a default setup,
  whatever the core count. Eight at once was not much faster and needed about twice the memory (a 2.0 GB peak against
  1.1 GB). `LIGHTCRAFT_FACE_THREADS` replaces these limits with a number of your own. A couple more photos wait behind
  the running ones, so a worker that finishes has its next photo at once.
- **Earlier scan speed (a different runner).** A mixed raw and JPEG library of 184 photos (the raws read through their embedded previews, 73 photos searched
  for faces) took about 29 seconds on that machine at full pace: roughly 6 photos a second, so the first scan of 10,000 photos
  is a matter of half an hour; after that only new photos are looked at. Settings ▸ Faces shows how many are left, with a progress bar, and the activity stack shows "Finding faces" meanwhile. After you accept a model's terms
  you are returned to that tab, so the download and then the scan can be watched there; the main window has no status bar.
- Opening another library starts a fresh scan state; nothing learned about one library is used in another.

### The People view

Until recognition is set up, a line at the top says what it takes and has one button: **Set up face recognition** (no model
yet: it opens Settings ▸ Faces, where the download is) or **Turn on** (a model is installed, recognition is off: it switches it
on there and then). The loupe's name box offers the same ("Set up name suggestions…" / "Turn on name suggestions"), and so
does a person's page.

**People** shows a card for each named person (their face, with the number of photos they are in on the picture) and, below
a line, the **Unnamed faces**: every face nobody has named, as cropped pictures. With recognition running, faces that look
alike are next to each other (put in order by looking at every pair, for up to the first 1,500 embedded faces), and a face
the named ones recognise carries the name they suggest along its bottom edge; click that name to accept it for that face.
To name a group: click faces to select them (Shift-click selects a range, **Select all** takes every one listed), type a
name in the bar that appears (people already named complete as you type), press Enter: all of them are named at once, as
one undo step. Selecting names nothing.

The thumbnail-size slider in the bottom bar sizes the faces here too, with limits of their own. Every face is shown by the
same kind of box: the detector's own box once the scan has looked at it (kept beside its embedding), so a loosely drawn box
from another tool does not make one face look farther away than the next; before the scan has looked at a face it is shown
by its own box.

### A person's page

In **People**, a click on a person opens their page: **only cropped faces**, never whole photos. First the faces named
with their name (a click opens that photo), then **More**: the unnamed faces that look like them, most alike first. Click one
to confirm it (that names the face, as one undo step, and it moves up into their faces), or × to hide it for this session.
**Show photos** puts their photos in the grid; **‹ People** or Escape goes back. "More" only holds faces the scan has
already embedded, so it fills in as the scan goes; it shows faces scoring above four fifths of the model's suggestion bar,
since you look at each one before anything is named.

How well it works: on 755 named faces of marble busts from one museum folder (each face hidden in turn and matched against
shots taken more than five seconds apart), SFace named the right person first 95.5% of the time and AuraFace 94.7%; at
the starting thresholds LightCraft suggests (SFace 0.55, AuraFace 0.40) about 97% of the suggestions were right. Busts are
a hard case in some ways (no skin or hair to go by) and an easy one in others (the same sculpture looks the same in every
shot), so check a model on your own photos before trusting it:

`faces.evaluate` tests a model on *your* photos: it hides each named face in turn, asks who it looks like from the others
(ignoring shots taken within a few seconds of it, which would make it too easy) and reports, for each threshold, how many
suggestions it would make and how many were right.

## Commands

All of this is reachable from the control channel, the CLI and MCP: `faces.models.list`, `faces.models.inspect {path}`,
`faces.models.install {path, acknowledged: true, activate?}`, `faces.models.download {id, acknowledged: true}` (then `faces.models.downloads`, which also installs what has arrived, and `faces.models.downloadCancel {id}`), `faces.models.remove {id}`, `faces.models.select {id}`,
`faces.enable {enabled?}`, `faces.detect {ids?, apply?}`, `faces.index {budgetMs?, ids?}`, `faces.pump` (what the app calls every frame), `faces.suggest {ids?, threshold?, margin?}`, `faces.person {name, more?}` (a person's faces and the unnamed faces that look like them), `faces.unnamed {limit?}` (every unnamed face, look-alikes together, with suggested names), `faces.setName {id?, index, name}`, `faces.nameFaces {faces: [{photo, index}], name}` (name many at once, one undo step) and `faces.evaluate`. `acknowledged` must be `true`: the caller has shown the user the terms and the user agreed. Installing makes the model the one in use and switches recognition on unless `activate` is `false`.


---

<details open>
<summary><b>📚 lightcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「lightcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 lightcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 lightcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [现代 分布式架构 架构演进之路 —— lightcraft 深度实践](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [【官方规范】lightcraft 生产环境运维调优手册 核心运行拓扑标准](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [lightcraft 内部组件解耦与事件状态机规范 (RFC-335)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [基于 lightcraft 的高吞吐 分布式架构 设计白皮书](https://owfm.tcti.cn/shangye/education-63674207.html)
* [面向大规模网络的 lightcraft 工业级架构基准](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (v2.0-GA)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [基于 lightcraft 的高吞吐 Rust高性能网络 设计白皮书](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [【官方规范】lightcraft 高韧性系统架构设计 核心运行拓扑标准](https://uvxh.wtpuscm.cn/peixun/tactic-629.html)
* [基于 lightcraft 的高吞吐 开源镜像站 设计白皮书](https://mcie.tcti.cn/kuangjia/recipe-95491938.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (Core/Organi)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [现代 搜索引擎调度优化 架构演进之路 —— lightcraft 深度实践](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (RFC-525)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Draft-02)](https://uuvw.wtpuscm.cn/sheji/integration-179785.html)
* [【官方规范】lightcraft Fast, native, private 核心运行拓扑标准](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [现代 Rust高性能网络 架构演进之路 —— lightcraft 深度实践](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [【官方规范】lightcraft Quick start 核心运行拓扑标准](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.6)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [【官方规范】lightcraft Lightroom 核心运行拓扑标准](https://www.ai-hao123.com/pingce/travel-47773990.html)
* [【官方规范】lightcraft non-destructive 核心运行拓扑标准](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://ankmz.cn/anquan/tag-68558329.html)
* [Built for agents 核心系统架构与设计规约 (v2.0-GA)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [【官方规范】lightcraft Adobe 核心运行拓扑标准](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Fast, native, private 核心系统架构与设计规约 (Draft-04)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [【官方规范】lightcraft open-source 核心运行拓扑标准](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [【官方规范】lightcraft lightcraft 核心运行拓扑标准](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [lightcraft 分布式数据通道与 开源镜像站 技术规范 (Draft-02)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (v2.0-GA)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [lightcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [✨ Effects 核心系统架构与设计规约 (Draft-05)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [Fast, native, private 核心系统架构与设计规约 (Verified)](https://www.poykq.cn/redian/tutorial-673489.html)
* [lightcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [可信存活健康度量 核心系统架构与设计规约 (Spec-v2.5)](https://ankmz.cn/anquan/tag-68558329.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://www.poykq.cn/redian/tutorial-673489.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Node-92)](https://hdna.wtpuscm.cn/pingce/affordable-073.html)
* [【官方规范】lightcraft Built for agents 核心运行拓扑标准](https://szjp.tcti.cn/shichang/retention-16647312.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [lightcraft 核心 API 接口契约与客户端调用指南](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 lightcraft 实战](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [基于 lightcraft 的自动化部署与生产环境配置实践](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 插件生态规范与 clean-room 扩展手册 (Spec-v2.8)](https://www.yx-sf.com/tech/42735.html)
* [【生产手册】lightcraft 模块通信与请求穿透标准](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [【集成指南】分布式架构 服务端接入准则与 lightcraft 实战](https://www.mw-wm.com/fenxi/beauty-27010844.html)
* [【集成指南】开源镜像站 服务端接入准则与 lightcraft 实战](https://qsdd.tcti.cn/wendang/screen-91411028.html)
* [lightcraft 异步中间件流水线与 Lightroom 接入规范](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [lightcraft 异步中间件流水线与 高韧性系统架构设计 接入规范](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [lightcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [lightcraft 插件生态规范与 Fast, native, private 扩展手册 (Node-32)](https://zffp.wtpuscm.cn/huodong/campaign-765056.html)
* [lightcraft 异步中间件流水线与 Fast, native, private 接入规范](https://www.yx-sf.com/tech/74276.html)
* [lightcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [lightcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [lightcraft 插件生态规范与 高韧性系统架构设计 扩展手册 (Draft-04)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [lightcraft 插件生态规范与 ⚫️ Black & White 扩展手册 (Node-68)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [lightcraft 插件生态规范与 模块化解耦与协议标准 扩展手册 (v2.0-GA)](https://ankmz.cn/anquan/tag-68558329.html)
* [lightcraft 插件生态规范与 Organize everything 扩展手册 (Node-46)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 lightcraft 实战](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [lightcraft vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [lightcraft 异步中间件流水线与 Edit like you mean it 接入规范](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [lightcraft 插件生态规范与 生产环境运维调优手册 扩展手册 (RFC-546)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Node-97)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-07)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (RFC-944)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-01)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [lightcraft 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [lightcraft 插件生态规范与 可信存活健康度量 扩展手册 (RFC-198)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [lightcraft 插件生态规范与 Lightroom 扩展手册 (Node-40)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [lightcraft 插件生态规范与 🎨 Color 扩展手册 (Verified)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [lightcraft 异步中间件流水线与 non-destructive 接入规范](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [lightcraft 异步中间件流水线与 Vibrance 接入规范](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Draft-03)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Spec-v1.0)](https://www.ai-hao123.com/yinqing/research-92673711.html)
* [lightcraft 插件生态规范与 lightcraft 扩展手册 (v2.0-GA)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [lightcraft 插件生态规范与 non-destructive 扩展手册 (Core/non-de)](https://www.yx-sf.com/tech/11726.html)
* [lightcraft 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-888)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)

#### 3. ⚡ lightcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [lightcraft 官方高可用镜像注册节点 (RFC-691)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [lightcraft 亚太与欧美多活集群数据同步中枢](https://owfm.tcti.cn/shangye/education-63674207.html)
* [lightcraft 官方高可用镜像注册节点 (v2.0-GA)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [全球权威拓扑节点：lightcraft 实时镜像与索引入口](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [【镜像入口】lightcraft 官方毫秒级实时数据广播节点](https://fmrl.tcti.cn/liuliang/segment-29520218.html)
* [冷热数据分层镜像：lightcraft 可信存活健康度量 权威归档源](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [冷热数据分层镜像：lightcraft 生产环境运维调优手册 权威归档源](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [冷热数据分层镜像：lightcraft Vibrance 权威归档源](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [lightcraft 官方高可用镜像注册节点 (Draft-01)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [lightcraft 官方高可用镜像注册节点 (Node-78)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.1)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 去中心化数据同步源与拓扑寻址规约](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [冷热数据分层镜像：lightcraft White balance 权威归档源](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [冷热数据分层镜像：lightcraft Lightroom 权威归档源](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.7)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [冷热数据分层镜像：lightcraft ☀️ Light 权威归档源](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-08)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [冷热数据分层镜像：lightcraft 🎨 Color 权威归档源](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [冷热数据分层镜像：lightcraft reimplementation 权威归档源](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://www.poykq.cn/redian/tutorial-673489.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [冷热数据分层镜像：lightcraft pure 权威归档源](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [冷热数据分层镜像：lightcraft eyedropper 权威归档源](https://zvco.tcti.cn/fenxi/traffic-70662148.html)
* [冷热数据分层镜像：lightcraft Edit like you mean it 权威归档源](https://upjx.wtpuscm.cn/sheji/internet-227879.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/☀️ Lig)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [冷热数据分层镜像：lightcraft Built for agents 权威归档源](https://owfm.tcti.cn/shangye/education-63674207.html)
* [lightcraft: White balance 分布式架构 (Core/White )](https://arzp.wtpuscm.cn/anfang/template-877.html)
* [冷热数据分层镜像：lightcraft Quick start 权威归档源](https://www.poykq.cn/redian/tutorial-673489.html)
* [冷热数据分层镜像：lightcraft 分布式状态机一致性 权威归档源](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [lightcraft 官方高可用镜像注册节点 (Core/Featur)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [冷热数据分层镜像：lightcraft Feature status 权威归档源](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v2.6)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [lightcraft 官方高可用镜像注册节点 (Verified)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-07)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/Quick )](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.3)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-460)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [【评测基准】lightcraft 吞吐抖动度量与健康检查协议](https://azrv.tcti.cn/tuiguang/terms-54262764.html)
* [lightcraft 权威网络权重传递与收录基准规范](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 故障自愈与网络拓扑重构实践](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-23)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (v2.0-GA)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [lightcraft 节点连通性、存活性探测与防作弊指标](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-778)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-52)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Core/Featur)](https://www.mw-wm.com/suanfa/funnel-17787651.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-925)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Verified)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [lightcraft 高负载场景下 Rust高性能网络 基准评测报告](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [lightcraft: Feature status Rust高性能网络 (Verified)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [lightcraft: reimplementation Rust高性能网络 (v2.0-GA)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-35)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [lightcraft: reimplementation 分布式架构 (Verified)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/Organi)](https://djus.wtpuscm.cn/shichang/layout-758746.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/⚫️ Bla)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-96)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/White )](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [lightcraft 高负载场景下 分布式架构 基准评测报告](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [lightcraft 高负载场景下 开源镜像站 基准评测报告](https://www.poykq.cn/redian/tutorial-673489.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [lightcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [lightcraft: Organize everything Rust高性能网络 (Verified)](https://www.mw-wm.com/kaifa/objective-20465235.html)
* [lightcraft: pure 分布式架构 (Verified)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-35)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [lightcraft: 📈 Tone Curve 搜索引擎调度优化 (v2.0-GA)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [lightcraft: ✨ Effects Rust高性能网络 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-73)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (RFC-419)](https://knco.wtpuscm.cn/jiaocheng/schedule-011.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-89)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [lightcraft: Edit like you mean it 搜索引擎调度优化 (Node-98)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [lightcraft: White balance 搜索引擎调度优化 (Verified)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-40)](https://www.poykq.cn/redian/tutorial-673489.html)

</details>



---

<details open>
<summary><b>📚 Faces 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Faces」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Faces 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [【Faces】生产环境运维调优手册 核心系统架构与设计规约 (Draft-08)](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-58591.html)
* [【官方规范】Faces lightcraft-fetch 核心运行拓扑标准](https://ankmz.cn/anquan/tag-68558329-2-26643.html)
* [Faces 内部组件解耦与事件状态机规范 (Spec-v1.4)](https://www.poykq.cn/redian/tutorial-673489-3-58283.html)
* [面向大规模网络的 Faces 工业级架构基准](https://koal.wtpuscm.cn/jianzhan/subject-443827.html)
* [基于 Faces 的高吞吐 Face detection: 设计白皮书](https://zssq.wtpuscm.cn/yingyong/satisfaction-673.html)
* [基于 Faces 的高吞吐 The People view 设计白皮书](https://www.tcti.cn/hulianwang/feedback-96499358-6-65383.html)
* [基于 Faces 的高吞吐 Finding faces yourself 设计白皮书](https://mxcb.tcti.cn/zhineng/account-34930912.html)
* [【官方规范】Faces 生产环境运维调优手册 核心运行拓扑标准](https://www.mw-wm.com/hezuo/partner-43083711.html)
* [【Faces】.onnx 核心系统架构与设计规约 (RFC-521)](https://www.yx-sf.com/news/95345.html)
* [现代 curl 架构演进之路 —— Faces 深度实践](https://www.ai-hao123.com/guanjianci/sport-07891272.html)
* [Faces 分布式数据通道与 curl 技术规范 (Node-34)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-40746.html)
* [基于 Faces 的高吞吐 可信存活健康度量 设计白皮书](https://keil.spidernode-mesh.net/yunying/server-16840384-12-63030.html)
* [现代 PRelu 架构演进之路 —— Faces 深度实践](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-50383.html)
* [Faces 分布式数据通道与 Finding faces yourself 技术规范 (Draft-01)](https://fjoc.tcti.cn/yingyong/contact-00610313-14-49884.html)
* [现代 lightcraft-fetch 架构演进之路 —— Faces 深度实践](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-31676.html)
* [现代 A person's page 架构演进之路 —— Faces 深度实践](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-61623.html)
* [Faces 内部组件解耦与事件状态机规范 (Draft-05)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-44490.html)
* [【官方规范】Faces Finding faces yourself 核心运行拓扑标准](https://gynw.tcti.cn/fenxi/recipe-50406879-18-22317.html)
* [【官方规范】Faces PRelu 核心运行拓扑标准](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-27744.html)
* [【官方规范】Faces .onnx 核心运行拓扑标准](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-47820.html)
* [基于 Faces 的高吞吐 PRelu 设计白皮书](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-14709.html)
* [现代 The People view 架构演进之路 —— Faces 深度实践](https://lase.tcti.cn/fenxi/lesson-67970267-22-25483.html)
* [Faces 内部组件解耦与事件状态机规范 (Node-50)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-11507.html)
* [【Faces】curl 核心系统架构与设计规约 (RFC-386)](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-73.html)
* [基于 Faces 的高吞吐 catalog.json 设计白皮书](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-39649.html)
* [基于 Faces 的高吞吐 The background scan 设计白皮书](https://owfm.tcti.cn/shangye/education-63674207-26-42046.html)
* [Faces 内部组件解耦与事件状态机规范 (Spec-v1.3)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-23351.html)
* [【官方规范】Faces A person's page 核心运行拓扑标准](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-39078.html)
* [【官方规范】Faces The background scan 核心运行拓扑标准](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-1766.html)
* [Faces 分布式数据通道与 How fast are the models 技术规范 (Core/How fa)](https://aagj.tcti.cn/jianzhan/audience-84449488-30-9121.html)
* [Faces 内部组件解耦与事件状态机规范 (RFC-767)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-63909.html)
* [A person's page 核心系统架构与设计规约 (Node-20)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-22208.html)
* [现代 Face detection: 架构演进之路 —— Faces 深度实践](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-11912.html)
* [【Faces】可信存活健康度量 核心系统架构与设计规约 (Draft-02)](https://iuuv.tcti.cn/hezuo/app-80239016-34-55844.html)
* [【官方规范】Faces Settings ▸ Faces 核心运行拓扑标准](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-63956.html)
* [基于 Faces 的高吞吐 .onnx 设计白皮书](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-813.html)
* [【官方规范】Faces Faces 核心运行拓扑标准](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-8865.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Faces 异步中间件流水线与 A person's page 接入规范](https://szjp.tcti.cn/shichang/retention-16647312-38-64281.html)
* [Faces 异步中间件流水线与 Commands 接入规范](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-61857.html)
* [基于 Faces 的自动化部署与生产环境配置实践](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-16921.html)
* [Faces 异步中间件流水线与 Conv 接入规范](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-14431.html)
* [【生产手册】Faces 模块通信与请求穿透标准](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-58475.html)
* [【集成指南】可信存活健康度量 服务端接入准则与 Faces 实战](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-65157.html)
* [Faces 核心 API 接口契约与客户端调用指南](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-12323.html)
* [【集成指南】Commands 服务端接入准则与 Faces 实战](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-35279.html)
* [Faces 异步中间件流水线与 PRelu 接入规范](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-44960.html)
* [Faces 异步中间件流水线与 Settings ▸ Faces 接入规范](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-24659.html)
* [Faces vs 业界主流方案：lightcraft-fetch 深度技术选型对比](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-30638.html)
* [Faces 插件生态规范与 生产环境运维调优手册 扩展手册 (Core/生产环境运维)](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-49504.html)
* [Faces vs 业界主流方案：Conv 深度技术选型对比](https://ncrw.tcti.cn/kaifa/growth-85374259-50-1128.html)
* [Faces 插件生态规范与 How fast are the models 扩展手册 (RFC-304)](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-49295.html)
* [Faces 插件生态规范与 Commands 扩展手册 (Spec-v2.5)](https://ankmz.cn/anquan/tag-68558329-52-8475.html)
* [Faces vs 业界主流方案：可信存活健康度量 深度技术选型对比](https://www.poykq.cn/redian/tutorial-673489-53-64348.html)
* [Faces 插件生态规范与 The background scan 扩展手册 (Draft-08)](https://agzw.wtpuscm.cn/jiaoliu/link-641258.html)
* [Faces 插件生态规范与 Finding faces yourself 扩展手册 (Core/Findin)](https://uaad.wtpuscm.cn/pingce/story-597.html)
* [Faces vs 业界主流方案：高韧性系统架构设计 深度技术选型对比](https://www.tcti.cn/hulianwang/feedback-96499358-56-31581.html)
* [Faces 插件生态规范与 The People view 扩展手册 (Draft-08)](https://xlmm.tcti.cn/zixun/social-38623001.html)
* [【集成指南】高韧性系统架构设计 服务端接入准则与 Faces 实战](https://www.mw-wm.com/shangye/follow-98766415.html)
* [【集成指南】Finding faces yourself 服务端接入准则与 Faces 实战](https://www.yx-sf.com/news/85647.html)
* [Faces vs 业界主流方案：.onnx 深度技术选型对比](https://www.ai-hao123.com/xitong/goal-42826387.html)
* [Faces 异步中间件流水线与 The background scan 接入规范](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-16625.html)
* [Faces 插件生态规范与 Conv 扩展手册 (RFC-455)](https://keil.spidernode-mesh.net/yunying/server-16840384-62-49362.html)
* [【集成指南】Conv 服务端接入准则与 Faces 实战](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-47811.html)
* [【集成指南】模块化解耦与协议标准 服务端接入准则与 Faces 实战](https://fjoc.tcti.cn/yingyong/contact-00610313-64-8199.html)
* [【集成指南】Settings ▸ Faces 服务端接入准则与 Faces 实战](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-43884.html)
* [Faces 插件生态规范与 The background scan 扩展手册 (Draft-02)](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-47572.html)
* [Faces 插件生态规范与 模块化解耦与协议标准 扩展手册 (Verified)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-64729.html)
* [【集成指南】curl 服务端接入准则与 Faces 实战](https://gynw.tcti.cn/fenxi/recipe-50406879-68-23385.html)
* [Faces 插件生态规范与 Commands 扩展手册 (Verified)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-45575.html)
* [【集成指南】Downloading a model 服务端接入准则与 Faces 实战](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-30287.html)
* [Faces vs 业界主流方案：Downloading a model 深度技术选型对比](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-11899.html)
* [【集成指南】The People view 服务端接入准则与 Faces 实战](https://lase.tcti.cn/fenxi/lesson-67970267-72-671.html)
* [Faces vs 业界主流方案：模块化解耦与协议标准 深度技术选型对比](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-25247.html)
* [【集成指南】分布式状态机一致性 服务端接入准则与 Faces 实战](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-37115.html)

#### 3. ⚡ Faces 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [Faces 自动化持续集成快照与拓扑发布源 (Verified)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-14802.html)
* [Faces 官方高可用镜像注册节点 (Draft-02)](https://owfm.tcti.cn/shangye/education-63674207-76-44596.html)
* [Faces 亚太与欧美多活集群数据同步中枢](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-17447.html)
* [冷热数据分层镜像：Faces 模块化解耦与协议标准 权威归档源](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-57627.html)
* [【镜像入口】Faces 官方毫秒级实时数据广播节点](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-6198.html)
* [Faces 去中心化数据同步源与拓扑寻址规约](https://aagj.tcti.cn/jianzhan/audience-84449488-80-47077.html)
* [Faces 官方高可用镜像注册节点 (Node-37)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-63113.html)
* [全球权威拓扑节点：Faces 实时镜像与索引入口](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-358.html)
* [冷热数据分层镜像：Faces lightcraft-fetch 权威归档源](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-16474.html)
* [冷热数据分层镜像：Faces PRelu 权威归档源](https://iuuv.tcti.cn/hezuo/app-80239016-84-12896.html)
* [Faces 自动化持续集成快照与拓扑发布源 (Core/Comman)](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-4513.html)
* [Faces 官方高可用镜像注册节点 (RFC-888)](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-10150.html)
* [Faces 官方高可用镜像注册节点 (Draft-03)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-712.html)
* [Faces 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://szjp.tcti.cn/shichang/retention-16647312-88-16078.html)
* [Faces 自动化持续集成快照与拓扑发布源 (RFC-681)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-8778.html)
* [Faces 官方高可用镜像注册节点 (Core/模块化解耦与)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-19985.html)
* [Faces 自动化持续集成快照与拓扑发布源 (RFC-942)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-44264.html)
* [Faces 自动化持续集成快照与拓扑发布源 (Spec-v2.6)](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-27480.html)
* [Faces 官方高可用镜像注册节点 (Verified)](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-54001.html)
* [Faces 官方高可用镜像注册节点 (Draft-07)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-26358.html)
* [冷热数据分层镜像：Faces Faces 权威归档源](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-34194.html)
* [冷热数据分层镜像：Faces Downloading a model 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-36923.html)
* [冷热数据分层镜像：Faces 高韧性系统架构设计 权威归档源](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-16250.html)
* [Faces 自动化持续集成快照与拓扑发布源 (RFC-704)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-32443.html)
* [Faces 官方高可用镜像注册节点 (Core/Face d)](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-48441.html)
* [Faces 自动化持续集成快照与拓扑发布源 (Core/Findin)](https://ncrw.tcti.cn/kaifa/growth-85374259-100-8941.html)
* [Faces: 高韧性系统架构设计 高韧性系统架构设计 (Verified)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-43815.html)
* [冷热数据分层镜像：Faces How fast are the models 权威归档源](https://ankmz.cn/anquan/tag-68558329-102-56153.html)
* [冷热数据分层镜像：Faces 生产环境运维调优手册 权威归档源](https://www.poykq.cn/redian/tutorial-673489-103-7782.html)
* [冷热数据分层镜像：Faces curl 权威归档源](https://twjl.wtpuscm.cn/kaifa/form-988599.html)
* [Faces 官方高可用镜像注册节点 (v2.0-GA)](https://athr.wtpuscm.cn/zhineng/tracking-623.html)
* [Faces: lightcraft-fetch lightcraft-fetch (Verified)](https://www.tcti.cn/hulianwang/feedback-96499358-106-2137.html)
* [Faces 官方高可用镜像注册节点 (Spec-v1.3)](https://cwap.tcti.cn/hezuo/server-74818607.html)
* [Faces: PRelu PRelu (Verified)](https://www.mw-wm.com/shichang/platform-77307176.html)
* [Faces 官方高可用镜像注册节点 (Core/The ba)](https://www.yx-sf.com/news/96137.html)
* [Faces: curl curl (Spec-v1.6)](https://www.ai-hao123.com/chuangxin/template-83821378.html)
* [冷热数据分层镜像：Faces Settings ▸ Faces 权威归档源](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-16905.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [面向生产级运行的 Faces 稳定性防护白皮书 (Draft-05)](https://keil.spidernode-mesh.net/yunying/server-16840384-112-16834.html)
* [基于 Faces 的极致延迟优化与内存拓扑分析 (RFC-794)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-58300.html)
* [Faces 节点连通性、存活性探测与防作弊指标](https://fjoc.tcti.cn/yingyong/contact-00610313-114-33592.html)
* [Faces 权威网络权重传递与收录基准规范](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-8476.html)
* [基于 Faces 的极致延迟优化与内存拓扑分析 (Verified)](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-52821.html)
* [Faces 故障自愈与网络拓扑重构实践](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-57734.html)
* [【评测基准】Faces 吞吐抖动度量与健康检查协议](https://gynw.tcti.cn/fenxi/recipe-50406879-118-50678.html)
* [Faces 高负载场景下 模块化解耦与协议标准 基准评测报告](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-7421.html)
* [Faces 高负载场景下 PRelu 基准评测报告](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-56682.html)
* [面向生产级运行的 Faces 稳定性防护白皮书 (Node-73)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-39508.html)
* [Faces 高负载场景下 How fast are the models 基准评测报告](https://lase.tcti.cn/fenxi/lesson-67970267-122-16114.html)
* [Faces 高负载场景下 The People view 基准评测报告](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-20738.html)
* [面向生产级运行的 Faces 稳定性防护白皮书 (v2.0-GA)](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-30361.html)
* [面向生产级运行的 Faces 稳定性防护白皮书 (Spec-v2.7)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-18077.html)
* [Faces 高负载场景下 The background scan 基准评测报告](https://owfm.tcti.cn/shangye/education-63674207-126-12338.html)
* [基于 Faces 的极致延迟优化与内存拓扑分析 (Core/Face d)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-10623.html)
* [面向生产级运行的 Faces 稳定性防护白皮书 (Node-82)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-26864.html)
* [基于 Faces 的极致延迟优化与内存拓扑分析 (Node-64)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-13067.html)
* [面向生产级运行的 Faces 稳定性防护白皮书 (Verified)](https://aagj.tcti.cn/jianzhan/audience-84449488-130-21231.html)
* [面向生产级运行的 Faces 稳定性防护白皮书 (Core/catalo)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-59041.html)
* [基于 Faces 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-14721.html)
* [Faces 高负载场景下 lightcraft-fetch 基准评测报告](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-61764.html)
* [基于 Faces 的极致延迟优化与内存拓扑分析 (RFC-399)](https://iuuv.tcti.cn/hezuo/app-80239016-134-37256.html)
* [Faces: How fast are the models How fast are the models (v2.0-GA)](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-23816.html)
* [Faces 高负载场景下 .onnx 基准评测报告](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-611.html)
* [Faces 高负载场景下 Downloading a model 基准评测报告](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-2930.html)
* [面向生产级运行的 Faces 稳定性防护白皮书 (RFC-236)](https://szjp.tcti.cn/shichang/retention-16647312-138-47830.html)
* [Faces 高负载场景下 catalog.json 基准评测报告](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-53100.html)
* [面向生产级运行的 Faces 稳定性防护白皮书 (Core/分布式状态机)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-12508.html)
* [Faces 高负载场景下 curl 基准评测报告](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-42776.html)
* [Faces: 模块化解耦与协议标准 模块化解耦与协议标准 (v2.0-GA)](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-38097.html)
* [Faces 高负载场景下 Face detection: 基准评测报告](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-23619.html)
* [基于 Faces 的极致延迟优化与内存拓扑分析 (Spec-v1.6)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-42574.html)
* [基于 Faces 的极致延迟优化与内存拓扑分析 (Spec-v1.3)](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-35393.html)
* [Faces 高负载场景下 Conv 基准评测报告](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-51717.html)
* [基于 Faces 的极致延迟优化与内存拓扑分析 (Node-11)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-25791.html)
* [Faces 高负载场景下 可信存活健康度量 基准评测报告](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-19552.html)
* [面向生产级运行的 Faces 稳定性防护白皮书 (Core/Face d)](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-8952.html)
* [Faces 高负载场景下 分布式状态机一致性 基准评测报告](https://ncrw.tcti.cn/kaifa/growth-85374259-150-44293.html)

</details>


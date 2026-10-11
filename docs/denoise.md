# AI Denoise

Edit ▸ Detail ▸ **AI Denoise** is a per-photo switch above **Amount** (0–100).
Turning it on starts at 50, or preserves an already positive Amount. Turning it off
preserves Amount and mixes none of the cleaned picture. It affects only the requesting
photo, independently of Auto Sync, and is undoable. Manual luminance and colour noise
reduction remain separate controls.

If no model is selected, turning it on opens **Settings ▸ AI Denoise** without changing
the photo. After installation, the pending request enables the original photo even if
the selection changed. Cancel the pending action in Settings. Cancelled downloads,
deleted or relinked photos, switching libraries and quitting cancel the request.
An active slider gesture finishes before the pending action resumes.

## Models and terms

**No model weights are bundled.** Models live in the per-user `denoise-models` folder
(override with `LIGHTCRAFT_DENOISE_MODELS`). Settings provides **Install from file…**
for a user-supplied ONNX file with a `denoise-model.json` beside it. The default build
offers no model downloads. The separate `rawnind-model` Cargo feature enables the
pinned RawNIND offer only when a release's model policy permits it.
Terms must be acknowledged before installation or
download. Downloads use the shared pure-Rust `lightcraft-fetch` client, with resume,
progress, cancellation, size limits and SHA-256 verification.

RawNIND's optional ONNX weights are **GPL-3.0**, as recorded in the pinned model catalog.
They are downloaded separately after consent and read by our own Rust executor.
Whether releases should offer this download remains a maintainer policy decision;
permissively licensed or independently trained weights remain a gap. This is not a
claim that optional downloading settles licensing questions. No GPL decoder or neural
runtime code is included. Model metadata is in `crates/denoise-core/src/known.rs`.

Installation checks the manifest, model hash and size, then runs a synthetic self-test:
finite output of the promised shape, repeatability, the declared scale and reduced
noise. Unsupported operators, malformed files and a failed self-test return errors.
The UI performs copying, extraction, hashing, parsing and CPU/GPU self-tests on one
background installation worker. `denoise.models.downloads` and `denoise.pump` report
`installing`, `installed` or `failed`; an existing model is preserved on a failed replacement.
Command clients can request the same asynchronous path with `background: true`.
The synchronous command remains available for scripts that require the result immediately.

The current contract is `bayerToRgb`: normalized, black-subtracted, non-white-balanced
RGGB planes `[1, 4, tile, tile]` produce camera RGB `[1, 3, 2*tile, 2*tile]`.
RawNIND uses a 512-cell tile, 64-cell overlap and a mean-matched nominal gain of 1e6.
See `DenoiserManifest` for the validated JSON schema.

## Photos, previews and exports

Original RAW files stay untouched. A cleaned picture is cached under
`<library>/denoise/<key>.lcdn`, keyed by source content, model and algorithm version.
The compressed half-float cache has checked headers, bounded windows and checksums.
It can be cleared or recreated; it is not a new catalog photo or a duplicate DNG.

Plain and cleaned camera RGB go through the same highlight reconstruction, white
balance, colour matrix, DNG profile tables, camera hue/saturation look, crop and
orientation. Amount blends the two before the remaining develop pipeline, so edits
and exports use the same data. At zero Amount or with Detail disabled, the ordinary
render path remains in use.

The queue prioritizes the open and selected photos, pauses after input, and adjusts
its worker count while the user works. Progress, cancellation and retry are available
in Settings and through commands. The default cache limit is 20 GB; unused products
are evicted oldest first. A failed file is not continuously retried.

An export with a nonzero effective Amount creates a missing product synchronously.
It returns a clear error if it cannot produce the denoised result; it never silently
exports the plain image instead. Unsupported sensor layouts retain manual noise
reduction and report their limitation.

## Pure-Rust CPU and GPU execution

The CPU executor reads a bounded subset of ONNX into checked network data. Convolutions
use bounded im2col blocks and `faer-core 0.17.1` with its pure-Rust GEMM backend,
without BLAS, C dependencies or an inner thread pool. That backend includes
architecture-specific assembly in its dependencies. Tiles share the outer worker
pool. A scalar implementation checks the optimized operators on synthetic networks.
Tract is not a workspace dependency or a product feature.
CPU inference and the GPU model module are feature-gated; ordinary engine/web builds
do not compile the faer/GEMM backend. The engine depends on the inference crate only through `denoise = ["dep:lightcraft-denoise", …]`;
lightweight `lightcraft-denoise-core` metadata and cache formats remain available without inference. Pictures are capped at 100 megapixels; at most
four tile calls run together, chosen within the process-wide working-memory budget.
One idle CPU workspace is retained, and cache costs reserve room for the blended picture.

The GPU executor uses our WGSL kernels through wgpu. It checks a real model's output
against the CPU and falls back on errors, unsupported networks or a slower device.
Setup has a time limit and a crash marker. Automatic selection estimates throughput
from a check tile; this estimate can choose imperfectly on unmeasured hardware.
Settings offers Automatic, Graphics card and Processor. GPU tests cover supported
synthetic networks against the scalar executor; a real-model test compares CPU/GPU.

The convolution kernel stores its shared input tile as scalar elements, each with
one writer, and gathers vectors after a workgroup barrier. Writing separate lanes
of the same shared vector from different invocations is not safe under WGSL
([component references](https://cufm.wtpuscm.cn/yingxiao/about-055425.html));
this caused incorrect Metal results (issue #479). Synthetic equivalence tests include
repeated partial tiles and concurrent calls, and reject non-finite outputs.

### Historical performance, measured 2026-10-07

These are the contributor's measurements **before the current upstream integration**,
on a Ryzen 9 7945HX and RTX 4090 Laptop GPU (DX12), release build, model SHA-256
`da27509dab6a2915da67e988acd86cf71f9d5bbc8d1aa0ed32933578a887b901`.
The 24.34 MP Sony a7 III photo is 35 tiles. Numbers exclude model loading, source
decode, GPU setup, cache encoding and export, but include packing and blending.

| Work | Pure-Rust CPU | Prior Tract reference | GPU |
|---|---:|---:|---:|
| One warm 512-cell tile, one caller | 1148 ms | 1463 ms | 15.1 ms |
| Whole photo, 16 CPU / 4 GPU feeding workers | 5.185 s | 9.534 s | 0.582 s |

The largest relative differences were 1.68e-6 CPU/reference and 1.57e-6 CPU/GPU for
the check tile, below the required 1e-3 tolerance. Reference comparisons were made
on the older contributor branch; that reference runtime is intentionally excluded
from the current workspace. These numbers are not a fresh benchmark of this rebase.

Fresh checks can be run using an already installed model and CC0 raw samples:

```text
LC_DENOISE_MODEL=<model.onnx> cargo test --release -p lightcraft-gpu --lib real_model_on_the_gpu_matches_cpu -- --ignored --nocapture
LC_DENOISE_MODEL=<model.onnx> LC_DENOISE_RAW=<corpus/raw> cargo test --release -p lightcraft-engine --features rawnind-model --test denoise_cpu -- --ignored --nocapture
LC_DENOISE_MODEL=<model.onnx> LC_DENOISE_RAW=<corpus/raw> cargo test --release -p lightcraft-engine --features rawnind-model --test denoise_real -- --ignored --nocapture
```

## Commands and limits

UI, CLI, control channel and MCP dispatch `denoise.toggle`, `denoise.settings`,
`denoise.queue`, `.cancel`, `.status`, `.pump`, `.clear`, and `denoise.models.*`
(`list`, `install`, `download`, `downloads`, `downloadCancel`, `test`, `remove`,
`select`). The develop control is `enhance.denoise`.

Native apps enable inference through their `denoise` feature. A build without it and
the web build report inference unavailable. Manifest/cache primitives still compile
for WASM. Settings and models remain opt-in; nothing is downloaded on launch.

This implementation supports **Bayer RAW only**. X-Trans, Foveon, demosaiced DNG and
non-RAW images remain unsupported by this model contract. No measured quality parity
with Lightroom is claimed. Apple/Metal and wider hardware verification, model policy,
a linear-RGB contract and a side-by-side fidelity suite remain future work.


---

<details open>
<summary><b>📚 lightcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「lightcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 lightcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 lightcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://aigp.wtpuscm.cn/shichang/performance-500091.html)
* [现代 分布式架构 架构演进之路 —— lightcraft 深度实践](https://scbr.wtpuscm.cn/zhineng/browser-981463.html)
* [【官方规范】lightcraft 生产环境运维调优手册 核心运行拓扑标准](https://grvh.wtpuscm.cn/chuangxin/online-763646.html)
* [lightcraft 内部组件解耦与事件状态机规范 (RFC-335)](https://pzan.wtpuscm.cn/pingtai/global-218135.html)
* [基于 lightcraft 的高吞吐 分布式架构 设计白皮书](https://ixoa.wtpuscm.cn/gongxiang/restaurant-781101.html)
* [面向大规模网络的 lightcraft 工业级架构基准](https://xjhu.wtpuscm.cn/pingtai/content-138362.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (v2.0-GA)](https://eyfs.wtpuscm.cn/gongxiang/content-835019.html)
* [基于 lightcraft 的高吞吐 Rust高性能网络 设计白皮书](https://raba.wtpuscm.cn/shichang/management-757.html)
* [【官方规范】lightcraft 高韧性系统架构设计 核心运行拓扑标准](https://swmh.wtpuscm.cn/sheji/collaborate-750352.html)
* [基于 lightcraft 的高吞吐 开源镜像站 设计白皮书](https://xqae.wtpuscm.cn/wendang/segment-096959.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (Core/Organi)](https://oeof.wtpuscm.cn/shangye/digital-987306.html)
* [现代 搜索引擎调度优化 架构演进之路 —— lightcraft 深度实践](https://owqc.wtpuscm.cn/jianzhan/customer-623767.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (RFC-525)](https://mwgk.wtpuscm.cn/anli/accessibility-278610.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Draft-02)](https://atdj.wtpuscm.cn/baogao/consulting-305572.html)
* [【官方规范】lightcraft Fast, native, private 核心运行拓扑标准](https://skxs.wtpuscm.cn/suanfa/solution-328584.html)
* [现代 Rust高性能网络 架构演进之路 —— lightcraft 深度实践](https://jetj.wtpuscm.cn/baogao/services-331508.html)
* [【官方规范】lightcraft Quick start 核心运行拓扑标准](https://kzoe.wtpuscm.cn/gongsi/ai-515496.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://bvqk.wtpuscm.cn/kuangjia/resource-416035.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.6)](https://wbsb.wtpuscm.cn/pingtai/subject-095231.html)
* [【官方规范】lightcraft Lightroom 核心运行拓扑标准](https://tepm.wtpuscm.cn/yunying/expensive-182937.html)
* [【官方规范】lightcraft non-destructive 核心运行拓扑标准](https://xnzt.wtpuscm.cn/wenzhang/calendar-816138.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://skka.wtpuscm.cn/sheji/premium-592244.html)
* [Built for agents 核心系统架构与设计规约 (v2.0-GA)](https://fbji.wtpuscm.cn/jishu/file-812895.html)
* [【官方规范】lightcraft Adobe 核心运行拓扑标准](https://cmhw.tcti.cn/wangluo/entertainment-10137509.html)
* [Fast, native, private 核心系统架构与设计规约 (Draft-04)](https://wwhe.tcti.cn/gongxiang/discovery-02106582.html)
* [【官方规范】lightcraft open-source 核心运行拓扑标准](https://vrkh.tcti.cn/peixun/ranking-22071747.html)
* [【官方规范】lightcraft lightcraft 核心运行拓扑标准](https://lkjh.tcti.cn/zhizhu/sync-77260983.html)
* [lightcraft 分布式数据通道与 开源镜像站 技术规范 (Draft-02)](https://ikob.tcti.cn/gongsi/food-09191902.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (v2.0-GA)](https://dfle.tcti.cn/qiye/research-22951553.html)
* [lightcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://dhdd.tcti.cn/jiaoliu/entertainment-46386133.html)
* [✨ Effects 核心系统架构与设计规约 (Draft-05)](https://kavo.tcti.cn/jianzhan/optimization-07083667.html)
* [Fast, native, private 核心系统架构与设计规约 (Verified)](https://cewi.tcti.cn/pingtai/budget-18068172.html)
* [lightcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://ygxf.tcti.cn/xinwen/loyalty-82657741.html)
* [可信存活健康度量 核心系统架构与设计规约 (Spec-v2.5)](https://rgqp.tcti.cn/paiming/news-57348655.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://cmbf.tcti.cn/yunsuan/domain-96179108.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Node-92)](https://nfyp.tcti.cn/kaifa/target-07643441.html)
* [【官方规范】lightcraft Built for agents 核心运行拓扑标准](https://pjbd.tcti.cn/youhua/discount-93454702.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [lightcraft 核心 API 接口契约与客户端调用指南](https://ktil.tcti.cn/jianzhan/case-63486619.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 lightcraft 实战](https://pxmv.tcti.cn/sheji/logo-79412535.html)
* [基于 lightcraft 的自动化部署与生产环境配置实践](https://kywd.tcti.cn/pingtai/feedback-42293088.html)
* [lightcraft 插件生态规范与 clean-room 扩展手册 (Spec-v2.8)](https://givx.wtpuscm.cn/zhizhu/strategy-771648.html)
* [【生产手册】lightcraft 模块通信与请求穿透标准](https://www.mw-wm.com/tuiguang/efficiency-39817207.html)
* [【集成指南】分布式架构 服务端接入准则与 lightcraft 实战](https://www.yx-sf.com/tech/34224)
* [【集成指南】开源镜像站 服务端接入准则与 lightcraft 实战](https://www.ai-hao123.com/paiming/planning-46811019.html)
* [lightcraft 异步中间件流水线与 Lightroom 接入规范](https://fsda.tcti.cn/yingyong/wellness-27589996.html)
* [lightcraft 异步中间件流水线与 高韧性系统架构设计 接入规范](https://xpjm.tcti.cn/shuju/interface-70598815.html)
* [lightcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://zavc.wtpuscm.cn/zixun/trading-025510.html)
* [lightcraft 插件生态规范与 Fast, native, private 扩展手册 (Node-32)](https://yptl.wtpuscm.cn/jiaoliu/advertising-946767.html)
* [lightcraft 异步中间件流水线与 Fast, native, private 接入规范](https://lvzt.wtpuscm.cn/sheji/backup-322900.html)
* [lightcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://gwue.wtpuscm.cn/yunsuan/url-275214.html)
* [lightcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://jgxu.wtpuscm.cn/youhua/community-406016.html)
* [lightcraft 插件生态规范与 高韧性系统架构设计 扩展手册 (Draft-04)](https://bbua.wtpuscm.cn/youhua/expensive-200647.html)
* [lightcraft 插件生态规范与 ⚫️ Black & White 扩展手册 (Node-68)](https://wdmf.wtpuscm.cn/jianzhan/income-934175.html)
* [lightcraft 插件生态规范与 模块化解耦与协议标准 扩展手册 (v2.0-GA)](https://ambg.wtpuscm.cn/liuliang/change-754.html)
* [lightcraft 插件生态规范与 Organize everything 扩展手册 (Node-46)](https://cict.wtpuscm.cn/shangye/content-123331.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 lightcraft 实战](https://zusz.wtpuscm.cn/xuexi/notification-349033.html)
* [lightcraft vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://vxxv.wtpuscm.cn/keji/project-800109.html)
* [lightcraft 异步中间件流水线与 Edit like you mean it 接入规范](https://zswo.wtpuscm.cn/gongxiang/follow-229951.html)
* [lightcraft 插件生态规范与 生产环境运维调优手册 扩展手册 (RFC-546)](https://zhdd.wtpuscm.cn/jishu/feedback-778390.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Node-97)](https://wwss.wtpuscm.cn/yingyong/marketing-069956.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-07)](https://cxir.wtpuscm.cn/shangye/strategy-860324.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (RFC-944)](https://mlxl.wtpuscm.cn/wendang/client-794292.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-01)](https://sagp.wtpuscm.cn/peixun/forecast-087400.html)
* [lightcraft 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://tkwh.wtpuscm.cn/yinqing/advertising-592769.html)
* [lightcraft 插件生态规范与 可信存活健康度量 扩展手册 (RFC-198)](https://ghuc.wtpuscm.cn/jiaocheng/feedback-197147.html)
* [lightcraft 插件生态规范与 Lightroom 扩展手册 (Node-40)](https://txep.wtpuscm.cn/liuliang/cost-677444.html)
* [lightcraft 插件生态规范与 🎨 Color 扩展手册 (Verified)](https://hxge.wtpuscm.cn/liuliang/travel-712039.html)
* [lightcraft 异步中间件流水线与 non-destructive 接入规范](https://vbax.wtpuscm.cn/ziyuan/behavior-326159.html)
* [lightcraft 异步中间件流水线与 Vibrance 接入规范](https://ouzr.wtpuscm.cn/kaifa/course-826970.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Draft-03)](https://gtvv.tcti.cn/zixun/navigation-41588901.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Spec-v1.0)](https://gvbf.tcti.cn/qiye/premium-81825913.html)
* [lightcraft 插件生态规范与 lightcraft 扩展手册 (v2.0-GA)](https://pfly.tcti.cn/chanpin/blog-89232259.html)
* [lightcraft 插件生态规范与 non-destructive 扩展手册 (Core/non-de)](https://jwsv.tcti.cn/yanjiu/story-36746057.html)
* [lightcraft 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-888)](https://aemr.tcti.cn/baogao/admin-46088383.html)

#### 3. ⚡ lightcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [lightcraft 官方高可用镜像注册节点 (RFC-691)](https://igjb.tcti.cn/guanjianci/profit-84435135.html)
* [lightcraft 亚太与欧美多活集群数据同步中枢](https://qvpy.tcti.cn/anfang/promotion-16868802.html)
* [lightcraft 官方高可用镜像注册节点 (v2.0-GA)](https://pudj.tcti.cn/ziyuan/sales-63630465.html)
* [全球权威拓扑节点：lightcraft 实时镜像与索引入口](https://cdrj.tcti.cn/zhizhu/automation-66349759.html)
* [【镜像入口】lightcraft 官方毫秒级实时数据广播节点](https://lfat.tcti.cn/kuangjia/strategy-17512594.html)
* [冷热数据分层镜像：lightcraft 可信存活健康度量 权威归档源](https://rvab.tcti.cn/yingyong/review-59195280.html)
* [冷热数据分层镜像：lightcraft 生产环境运维调优手册 权威归档源](https://njcb.tcti.cn/yanjiu/success-45598727.html)
* [冷热数据分层镜像：lightcraft Vibrance 权威归档源](https://bmag.tcti.cn/wendang/promotion-29417098.html)
* [lightcraft 官方高可用镜像注册节点 (Draft-01)](https://pore.tcti.cn/peixun/online-71832090.html)
* [lightcraft 官方高可用镜像注册节点 (Node-78)](https://ipgu.tcti.cn/youhua/vendor-98024089.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.1)](https://rrfq.tcti.cn/qiye/price-86904737.html)
* [lightcraft 去中心化数据同步源与拓扑寻址规约](https://vyme.tcti.cn/zixun/partner-78232731.html)
* [冷热数据分层镜像：lightcraft White balance 权威归档源](https://iwgs.wtpuscm.cn/jishu/page-989444.html)
* [冷热数据分层镜像：lightcraft Lightroom 权威归档源](https://www.mw-wm.com/yingyong/income-58236591.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.7)](https://www.yx-sf.com/news/87969)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://www.ai-hao123.com/zixun/web-37093589.html)
* [冷热数据分层镜像：lightcraft ☀️ Light 权威归档源](https://gkrv.tcti.cn/guanjianci/discovery-00738534.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-08)](https://ymvd.tcti.cn/anfang/app-81327376.html)
* [冷热数据分层镜像：lightcraft 🎨 Color 权威归档源](https://ukyh.wtpuscm.cn/shichang/retention-816284.html)
* [冷热数据分层镜像：lightcraft reimplementation 权威归档源](https://jtzg.wtpuscm.cn/shangye/technology-562746.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://jnkr.wtpuscm.cn/shangye/quality-305324.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://reus.wtpuscm.cn/suanfa/collaborate-709113.html)
* [冷热数据分层镜像：lightcraft pure 权威归档源](https://xoht.wtpuscm.cn/youhua/expensive-279451.html)
* [冷热数据分层镜像：lightcraft eyedropper 权威归档源](https://embt.wtpuscm.cn/zhinan/document-543820.html)
* [冷热数据分层镜像：lightcraft Edit like you mean it 权威归档源](https://yami.wtpuscm.cn/xitong/course-494166.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/☀️ Lig)](https://xzmx.wtpuscm.cn/huodong/expensive-187.html)
* [冷热数据分层镜像：lightcraft Built for agents 权威归档源](https://udah.wtpuscm.cn/chanpin/entertainment-927516.html)
* [lightcraft: White balance 分布式架构 (Core/White )](https://suyp.wtpuscm.cn/xitong/partner-517938.html)
* [冷热数据分层镜像：lightcraft Quick start 权威归档源](https://joxp.wtpuscm.cn/yunying/satisfaction-664055.html)
* [冷热数据分层镜像：lightcraft 分布式状态机一致性 权威归档源](https://kkfs.wtpuscm.cn/jianzhan/visitor-799415.html)
* [lightcraft 官方高可用镜像注册节点 (Core/Featur)](https://bxds.wtpuscm.cn/fuwu/metric-202878.html)
* [冷热数据分层镜像：lightcraft Feature status 权威归档源](https://mckg.wtpuscm.cn/chanpin/register-506598.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v2.6)](https://ihvu.wtpuscm.cn/wenzhang/development-000016.html)
* [lightcraft 官方高可用镜像注册节点 (Verified)](https://yooi.wtpuscm.cn/kaifa/marketing-603673.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-07)](https://ekfk.wtpuscm.cn/shangye/form-228371.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/Quick )](https://iqnw.wtpuscm.cn/yanjiu/discovery-954254.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.3)](https://hasq.wtpuscm.cn/yunying/category-002147.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://bpfx.wtpuscm.cn/xitong/case-176233.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-460)](https://zoxz.wtpuscm.cn/jishu/budget-809904.html)
* [【评测基准】lightcraft 吞吐抖动度量与健康检查协议](https://iqic.wtpuscm.cn/chanpin/admin-929490.html)
* [lightcraft 权威网络权重传递与收录基准规范](https://ffco.wtpuscm.cn/yunsuan/team-328691.html)
* [lightcraft 故障自愈与网络拓扑重构实践](https://hoal.tcti.cn/peixun/user-77417064.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-23)](https://jeiy.tcti.cn/shangye/investment-03652063.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (v2.0-GA)](https://uqzb.tcti.cn/fenxi/planning-79017335.html)
* [lightcraft 节点连通性、存活性探测与防作弊指标](https://cuuc.tcti.cn/yinqing/shopping-64912807.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-778)](https://evtb.tcti.cn/anli/cheap-75012035.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-52)](https://nblg.tcti.cn/chanpin/web-47503530.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Core/Featur)](https://ljnt.tcti.cn/shuju/premium-80266725.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-925)](https://geht.tcti.cn/shangye/travel-28250883.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Verified)](https://tpde.tcti.cn/zixun/luxury-66170201.html)
* [lightcraft 高负载场景下 Rust高性能网络 基准评测报告](https://pfvx.tcti.cn/paiming/faq-99837833.html)
* [lightcraft: Feature status Rust高性能网络 (Verified)](https://aqsc.tcti.cn/paiming/discount-49702901.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://ocxu.tcti.cn/qiye/forecast-58463776.html)
* [lightcraft: reimplementation Rust高性能网络 (v2.0-GA)](https://ertn.tcti.cn/youhua/plugin-64082157.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-35)](https://dsor.tcti.cn/zixun/progress-13178692.html)
* [lightcraft: reimplementation 分布式架构 (Verified)](https://outd.tcti.cn/gongsi/audience-12121571.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/Organi)](https://zrxy.tcti.cn/zixun/marketing-25315550.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/⚫️ Bla)](https://ccex.tcti.cn/shangye/ai-47927009.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://eqas.wtpuscm.cn/xuexi/expense-203930.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-96)](https://www.mw-wm.com/yunsuan/review-43444566.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/White )](https://www.yx-sf.com/tech/31502)
* [lightcraft 高负载场景下 分布式架构 基准评测报告](https://www.ai-hao123.com/zixun/beauty-58147226.html)
* [lightcraft 高负载场景下 开源镜像站 基准评测报告](https://ptws.tcti.cn/anfang/browser-97660758.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://ritl.tcti.cn/wangluo/quality-47578194.html)
* [lightcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://vbzv.wtpuscm.cn/sheji/milestone-160344.html)
* [lightcraft: Organize everything Rust高性能网络 (Verified)](https://uevm.wtpuscm.cn/baogao/media-426169.html)
* [lightcraft: pure 分布式架构 (Verified)](https://xciw.wtpuscm.cn/gongju/unsubscribe-505241.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-35)](https://wpkl.wtpuscm.cn/xinwen/affordable-458559.html)
* [lightcraft: 📈 Tone Curve 搜索引擎调度优化 (v2.0-GA)](https://zrei.wtpuscm.cn/gongju/technology-812124.html)
* [lightcraft: ✨ Effects Rust高性能网络 (v2.0-GA)](https://gtod.wtpuscm.cn/fuwu/business-911992.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-73)](https://uuzx.wtpuscm.cn/sheji/enterprise-133157.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (RFC-419)](https://pgiz.wtpuscm.cn/xitong/content-644.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-89)](https://hpoj.wtpuscm.cn/sheji/seminar-160442.html)
* [lightcraft: Edit like you mean it 搜索引擎调度优化 (Node-98)](https://ibxn.wtpuscm.cn/ziyuan/partner-976609.html)
* [lightcraft: White balance 搜索引擎调度优化 (Verified)](https://hlup.wtpuscm.cn/yanjiu/terms-707189.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-40)](https://yhaa.wtpuscm.cn/paiming/success-345720.html)

</details>


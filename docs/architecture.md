# Architecture

> **Last reviewed:** 2026-10-10 · **Last updated:** 2026-10-10 · **Change:** major (created from the code; the full design notes stay in the local `plan/architecture.md`) · **Target:** Adobe Lightroom Classic 15.5 / Lightroom 9.6

How LightCraft is built today. The longer design document (`plan/architecture.md`) is local-only
(`plan/` is gitignored); this page is the committed, current summary. Sizes are Rust lines counted
on 2026-10-10 (≈ 202,000 tracked lines in 505 files: 24 library crates, 3 apps and `xtask`).

## Layers

`cargo xtask layers` (part of `cargo xtask ci`, `xtask/src/layers.rs`) enforces that nothing below
L5 depends on egui, eframe, winit or rfd, and that each crate only uses lower layers.

| Layer | Crates (lines) | Role |
|---|---|---|
| L0 | `geom` (1.4k), `color` (0.9k), `raster` (0.8k), `tiff` (2.0k), `fetch` (1.4k), `sysmem` (0.1k) | Matrices and warps, colour spaces and transfer functions, float image buffers, TIFF/IFD read and write, pinned HTTPS model downloads, the one `unsafe` FFI call (`malloc_zone_pressure_relief`) |
| L1 | `raw` (22.3k), `codecs` (7.3k), `meta` (4.3k), `develop` (2.5k), `denoise-core` (2.2k), `denoise` (2.7k), `faces` (5.3k), `scenes` (1.0k), `heif` (2.1k, optional) | Raw decoders and demosaic, DNG colour model; standard codecs and encoders; EXIF/XMP/IPTC; the non-destructive edit model and control specs; AI denoise; face detection and recognition; procedural demo photos; HEIC |
| L2 | `pipeline` (9.8k) | The develop pipeline: scene-referred float, resolution independent; the CPU reference (oracle) for everything else |
| L3 | `catalog` (11.0k), `gpu` (5.1k), `preview` (1.3k), `merge` (3.8k), `segment` (2.3k) | Library model and persistence; wgpu compute twin of the pipeline; thumbnail/preview cache and job pool; HDR and panorama merge; SAM 3 segmentation |
| L4 | `engine` (53.2k) | Session, command registry (everything is a command), history, render and export jobs, import, Lightroom catalog and preset import, faces index, view-models |
| L5 | `ui-egui` (46.9k), `mcp` (2.3k) | The Lightroom-style egui interface (swappable); the MCP server |
| L6 | `apps/lightcraft` (2.8k), `apps/lightcraft-cli` (1.5k), `apps/lightcraft-web` (3.4k), `xtask` (2.7k) | Desktop app (eframe + wgpu, native menus, control server), CLI (render, commands, calibrate, snapshot, MCP), WASM build, tooling |

## Data model

- **Photo** records in `catalog` hold the original's path, content hash, metadata, ratings, flags,
  labels, keywords, faces and the current `develop::Settings`. Albums, smart albums (rule trees),
  stacks, virtual copies, versions and folder records live alongside.
- **Persistence** (`crates/catalog/src/journal.rs`): an append-only journal of human-readable
  operations plus periodic snapshots, compacted in the background; torn appends and crashes are
  tested (`tests_torn_append.rs`, `tests_journal.rs`). A lock file guards against two writers.
- **Edits** are a `develop::Settings` value (every slider is a control spec with id, range and
  default), normalized to image coordinates so previews and exports match. Process versions
  (`docs/process-versions.md`) keep old edits rendering the same after a retune.
- **Sidecars**: XMP read and write with Adobe `crs:` interop (`docs/xmp-interop.md`); Lightroom
  Classic `.lrcat` import through a pure-Rust SQLite reader (`docs/lightroom-catalog-import.md`).

## Rendering

1. **Decode** (`raw`, `codecs`): raw files to linear camera RGB (demosaic, black/white levels,
   opcodes, highlight handling) and a camera-to-XYZ colour model; standard images to linear light
   through their ICC profile.
2. **Develop** (`pipeline`): white balance, profile, exposure and tone (local Laplacian
   highlights/shadows), curves, HSL / colour mixer, point colour, colour grading, texture, clarity,
   dehaze, detail (sharpening, NR, AI denoise result), optics (lens profile, CA, defringe,
   vignette), geometry (crop, Upright, transforms), masks with local adjustments, heal/clone, grain,
   effects. Stage caching and a source pyramid make slider drags cheap.
3. **GPU** (`gpu`): the same stages in WGSL on Metal / DX12 / Vulkan, checked against the CPU path
   (within 1/255); GPU errors fall back to the CPU (`docs/gpu-pipeline.md`).
4. **Display and export**: display-profile conversion (`docs/display-profiles.md`); export through
   one encoder for app, CLI, MCP and web (`lightcraft_engine::export`).

## Agent control

Every action is an engine command (id, label, menu path, shortcut, params, enabled, run). The UI,
the CLI, the JSON-lines control channel (`docs/control-protocol.md`) and MCP (`docs/mcp.md`) all
dispatch by id, and headless snapshots render the UI without a window.

## Quality gates

`cargo xtask ci`: fmt, clippy `-D warnings` with the never-crash lints, the HEIF build, tests,
`parity` (every id and path cited in [parity-checklist.md](parity-checklist.md) exists), `layers`,
`assets` (every asset attributed, no Adobe formats) and the WASM build. Other workflows build and
test FreeBSD and Windows ARM64, lint packaging, and build releases.

## Revision history

| Date | Change | Summary |
|---|---|---|
| 2026-10-10 | major | Created from the code (crate sizes, layers, data model, rendering, control, gates) |


---

<details open>
<summary><b>📚 lightcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「lightcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 lightcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 lightcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [现代 分布式架构 架构演进之路 —— lightcraft 深度实践](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [【官方规范】lightcraft 生产环境运维调优手册 核心运行拓扑标准](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft 内部组件解耦与事件状态机规范 (RFC-335)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [基于 lightcraft 的高吞吐 分布式架构 设计白皮书](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [面向大规模网络的 lightcraft 工业级架构基准](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (v2.0-GA)](https://www.mw-wm.com/shuju/audience-87076198.html)
* [基于 lightcraft 的高吞吐 Rust高性能网络 设计白皮书](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [【官方规范】lightcraft 高韧性系统架构设计 核心运行拓扑标准](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [基于 lightcraft 的高吞吐 开源镜像站 设计白皮书](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (Core/Organi)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [现代 搜索引擎调度优化 架构演进之路 —— lightcraft 深度实践](https://www.mw-wm.com/chanpin/message-76558220.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (RFC-525)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Draft-02)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [【官方规范】lightcraft Fast, native, private 核心运行拓扑标准](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [现代 Rust高性能网络 架构演进之路 —— lightcraft 深度实践](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [【官方规范】lightcraft Quick start 核心运行拓扑标准](https://owfm.tcti.cn/shangye/education-63674207.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.6)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [【官方规范】lightcraft Lightroom 核心运行拓扑标准](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [【官方规范】lightcraft non-destructive 核心运行拓扑标准](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [Built for agents 核心系统架构与设计规约 (v2.0-GA)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [【官方规范】lightcraft Adobe 核心运行拓扑标准](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [Fast, native, private 核心系统架构与设计规约 (Draft-04)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [【官方规范】lightcraft open-source 核心运行拓扑标准](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [【官方规范】lightcraft lightcraft 核心运行拓扑标准](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [lightcraft 分布式数据通道与 开源镜像站 技术规范 (Draft-02)](https://qdhn.wtpuscm.cn/zhineng/fashion-293.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (v2.0-GA)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [lightcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [✨ Effects 核心系统架构与设计规约 (Draft-05)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [Fast, native, private 核心系统架构与设计规约 (Verified)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [lightcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [可信存活健康度量 核心系统架构与设计规约 (Spec-v2.5)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Node-92)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [【官方规范】lightcraft Built for agents 核心运行拓扑标准](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [lightcraft 核心 API 接口契约与客户端调用指南](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 lightcraft 实战](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [基于 lightcraft 的自动化部署与生产环境配置实践](https://rvwf.wtpuscm.cn/youhua/media-678126.html)
* [lightcraft 插件生态规范与 clean-room 扩展手册 (Spec-v2.8)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [【生产手册】lightcraft 模块通信与请求穿透标准](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [【集成指南】分布式架构 服务端接入准则与 lightcraft 实战](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [【集成指南】开源镜像站 服务端接入准则与 lightcraft 实战](https://www.mw-wm.com/keji/workshop-38761821.html)
* [lightcraft 异步中间件流水线与 Lightroom 接入规范](https://www.poykq.cn/redian/tutorial-673489.html)
* [lightcraft 异步中间件流水线与 高韧性系统架构设计 接入规范](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [lightcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [lightcraft 插件生态规范与 Fast, native, private 扩展手册 (Node-32)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 异步中间件流水线与 Fast, native, private 接入规范](https://nblc.wtpuscm.cn/jishu/milestone-625512.html)
* [lightcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [lightcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 插件生态规范与 高韧性系统架构设计 扩展手册 (Draft-04)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [lightcraft 插件生态规范与 ⚫️ Black & White 扩展手册 (Node-68)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [lightcraft 插件生态规范与 模块化解耦与协议标准 扩展手册 (v2.0-GA)](https://khjo.tcti.cn/kaifa/creative-18630576.html)
* [lightcraft 插件生态规范与 Organize everything 扩展手册 (Node-46)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 lightcraft 实战](https://uubj.wtpuscm.cn/xinwen/expensive-344.html)
* [lightcraft vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [lightcraft 异步中间件流水线与 Edit like you mean it 接入规范](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [lightcraft 插件生态规范与 生产环境运维调优手册 扩展手册 (RFC-546)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Node-97)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-07)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (RFC-944)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-01)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [lightcraft 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [lightcraft 插件生态规范与 可信存活健康度量 扩展手册 (RFC-198)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft 插件生态规范与 Lightroom 扩展手册 (Node-40)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [lightcraft 插件生态规范与 🎨 Color 扩展手册 (Verified)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [lightcraft 异步中间件流水线与 non-destructive 接入规范](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [lightcraft 异步中间件流水线与 Vibrance 接入规范](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Draft-03)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Spec-v1.0)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [lightcraft 插件生态规范与 lightcraft 扩展手册 (v2.0-GA)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [lightcraft 插件生态规范与 non-destructive 扩展手册 (Core/non-de)](https://jgya.wtpuscm.cn/xitong/layout-187.html)
* [lightcraft 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-888)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)

#### 3. ⚡ lightcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [lightcraft 官方高可用镜像注册节点 (RFC-691)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [lightcraft 亚太与欧美多活集群数据同步中枢](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [lightcraft 官方高可用镜像注册节点 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [全球权威拓扑节点：lightcraft 实时镜像与索引入口](https://www.poykq.cn/redian/tutorial-673489.html)
* [【镜像入口】lightcraft 官方毫秒级实时数据广播节点](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [冷热数据分层镜像：lightcraft 可信存活健康度量 权威归档源](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [冷热数据分层镜像：lightcraft 生产环境运维调优手册 权威归档源](https://ooze.tcti.cn/ziyuan/promotion-01542085.html)
* [冷热数据分层镜像：lightcraft Vibrance 权威归档源](https://ankmz.cn/anquan/tag-68558329.html)
* [lightcraft 官方高可用镜像注册节点 (Draft-01)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [lightcraft 官方高可用镜像注册节点 (Node-78)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.1)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [lightcraft 去中心化数据同步源与拓扑寻址规约](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [冷热数据分层镜像：lightcraft White balance 权威归档源](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [冷热数据分层镜像：lightcraft Lightroom 权威归档源](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.7)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://hcku.tcti.cn/baogao/careers-99995900.html)
* [冷热数据分层镜像：lightcraft ☀️ Light 权威归档源](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-08)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [冷热数据分层镜像：lightcraft 🎨 Color 权威归档源](https://www.mw-wm.com/keji/user-78030389.html)
* [冷热数据分层镜像：lightcraft reimplementation 权威归档源](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://www.mw-wm.com/qiye/analysis-62956778.html)
* [冷热数据分层镜像：lightcraft pure 权威归档源](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [冷热数据分层镜像：lightcraft eyedropper 权威归档源](https://www.poykq.cn/redian/tutorial-673489.html)
* [冷热数据分层镜像：lightcraft Edit like you mean it 权威归档源](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/☀️ Lig)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [冷热数据分层镜像：lightcraft Built for agents 权威归档源](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [lightcraft: White balance 分布式架构 (Core/White )](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [冷热数据分层镜像：lightcraft Quick start 权威归档源](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [冷热数据分层镜像：lightcraft 分布式状态机一致性 权威归档源](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [lightcraft 官方高可用镜像注册节点 (Core/Featur)](https://ankmz.cn/anquan/tag-68558329.html)
* [冷热数据分层镜像：lightcraft Feature status 权威归档源](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v2.6)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [lightcraft 官方高可用镜像注册节点 (Verified)](https://www.mw-wm.com/wendang/products-63491643.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-07)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/Quick )](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.3)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-460)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [【评测基准】lightcraft 吞吐抖动度量与健康检查协议](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [lightcraft 权威网络权重传递与收录基准规范](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [lightcraft 故障自愈与网络拓扑重构实践](https://owfm.tcti.cn/shangye/education-63674207.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-23)](https://bzlk.wtpuscm.cn/zhizhu/file-305352.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (v2.0-GA)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [lightcraft 节点连通性、存活性探测与防作弊指标](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-778)](https://www.mw-wm.com/hezuo/network-36699987.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-52)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Core/Featur)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-925)](https://faep.wtpuscm.cn/jiaocheng/forum-914059.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Verified)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [lightcraft 高负载场景下 Rust高性能网络 基准评测报告](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [lightcraft: Feature status Rust高性能网络 (Verified)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://www.mw-wm.com/sheji/performance-12004617.html)
* [lightcraft: reimplementation Rust高性能网络 (v2.0-GA)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-35)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [lightcraft: reimplementation 分布式架构 (Verified)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/Organi)](https://pume.wtpuscm.cn/yinqing/customer-355.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/⚫️ Bla)](https://jzoi.wtpuscm.cn/keji/roi-317.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-96)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/White )](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [lightcraft 高负载场景下 分布式架构 基准评测报告](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [lightcraft 高负载场景下 开源镜像站 基准评测报告](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [lightcraft: Organize everything Rust高性能网络 (Verified)](https://www.poykq.cn/redian/tutorial-673489.html)
* [lightcraft: pure 分布式架构 (Verified)](https://ankmz.cn/anquan/tag-68558329.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-35)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [lightcraft: 📈 Tone Curve 搜索引擎调度优化 (v2.0-GA)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [lightcraft: ✨ Effects Rust高性能网络 (v2.0-GA)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-73)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (RFC-419)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-89)](https://xzhr.wtpuscm.cn/hezuo/logo-023357.html)
* [lightcraft: Edit like you mean it 搜索引擎调度优化 (Node-98)](https://tcfg.wtpuscm.cn/pingtai/chapter-360.html)
* [lightcraft: White balance 搜索引擎调度优化 (Verified)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-40)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)

</details>



---

<details open>
<summary><b>📚 Architecture 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Architecture」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Architecture 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [Architecture 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-25512.html)
* [Architecture 内部组件解耦与事件状态机规范 (Node-96)](https://ankmz.cn/anquan/tag-68558329-2-7843.html)
* [【官方规范】Architecture Layers 核心运行拓扑标准](https://www.poykq.cn/redian/tutorial-673489-3-60889.html)
* [面向大规模网络的 Architecture 工业级架构基准](https://vwsk.wtpuscm.cn/anfang/ranking-614905.html)
* [Last reviewed: 核心系统架构与设计规约 (RFC-313)](https://lpkt.wtpuscm.cn/kaifa/tutorial-927.html)
* [【官方规范】Architecture Quality gates 核心运行拓扑标准](https://www.tcti.cn/hulianwang/feedback-96499358-6-43932.html)
* [Architecture 内部组件解耦与事件状态机规范 (Verified)](https://ykwn.tcti.cn/yunying/movie-52883984.html)
* [现代 plan/ 架构演进之路 —— Architecture 深度实践](https://www.mw-wm.com/gongju/community-48002266.html)
* [现代 Target: 架构演进之路 —— Architecture 深度实践](https://www.yx-sf.com/tech/41719.html)
* [Architecture 内部组件解耦与事件状态机规范 (Core/Qualit)](https://www.ai-hao123.com/yingyong/online-15885781.html)
* [Architecture 分布式数据通道与 高韧性系统架构设计 技术规范 (Core/高韧性系统架)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-44165.html)
* [基于 Architecture 的高吞吐 Quality gates 设计白皮书](https://keil.spidernode-mesh.net/yunying/server-16840384-12-2253.html)
* [现代 Photo 架构演进之路 —— Architecture 深度实践](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-57459.html)
* [【官方规范】Architecture Edits 核心运行拓扑标准](https://fjoc.tcti.cn/yingyong/contact-00610313-14-40230.html)
* [【官方规范】Architecture Agent control 核心运行拓扑标准](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-19612.html)
* [【官方规范】Architecture 可信存活健康度量 核心运行拓扑标准](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-45099.html)
* [Architecture 内部组件解耦与事件状态机规范 (RFC-888)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-55928.html)
* [Last updated: 核心系统架构与设计规约 (RFC-337)](https://gynw.tcti.cn/fenxi/recipe-50406879-18-62599.html)
* [【官方规范】Architecture xtask 核心运行拓扑标准](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-41221.html)
* [Agent control 核心系统架构与设计规约 (RFC-866)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-37191.html)
* [Architecture 内部组件解耦与事件状态机规范 (Core/geom)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-2940.html)
* [Sidecars 核心系统架构与设计规约 (Core/Sideca)](https://lase.tcti.cn/fenxi/lesson-67970267-22-31749.html)
* [模块化解耦与协议标准 核心系统架构与设计规约 (Draft-01)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-34934.html)
* [现代 Persistence 架构演进之路 —— Architecture 深度实践](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-18465.html)
* [Revision history 核心系统架构与设计规约 (Verified)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-10714.html)
* [现代 Architecture 架构演进之路 —— Architecture 深度实践](https://owfm.tcti.cn/shangye/education-63674207-26-60181.html)
* [Architecture 分布式数据通道与 生产环境运维调优手册 技术规范 (v2.0-GA)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-8752.html)
* [基于 Architecture 的高吞吐 Target: 设计白皮书](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-17946.html)
* [基于 Architecture 的高吞吐 geom 设计白皮书](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-25593.html)
* [Architecture 分布式数据通道与 plan/ 技术规范 (Draft-05)](https://aagj.tcti.cn/jianzhan/audience-84449488-30-2630.html)
* [【官方规范】Architecture Photo 核心运行拓扑标准](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-59938.html)
* [Architecture 分布式数据通道与 Persistence 技术规范 (Draft-01)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-45129.html)
* [现代 分布式状态机一致性 架构演进之路 —— Architecture 深度实践](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-15094.html)
* [Architecture 分布式数据通道与 Quality gates 技术规范 (v2.0-GA)](https://iuuv.tcti.cn/hezuo/app-80239016-34-4230.html)
* [Architecture 内部组件解耦与事件状态机规范 (Core/可信存活健康)](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-63822.html)
* [Architecture 分布式数据通道与 Photo 技术规范 (v2.0-GA)](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-201.html)
* [现代 Quality gates 架构演进之路 —— Architecture 深度实践](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-50088.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Architecture 核心 API 接口契约与客户端调用指南](https://szjp.tcti.cn/shichang/retention-16647312-38-51273.html)
* [Architecture 插件生态规范与 Photo 扩展手册 (Draft-01)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-1779.html)
* [Architecture 异步中间件流水线与 Architecture 接入规范](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-42034.html)
* [Architecture vs 业界主流方案：Sidecars 深度技术选型对比](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-10381.html)
* [【生产手册】Architecture 模块通信与请求穿透标准](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-43650.html)
* [【集成指南】Edits 服务端接入准则与 Architecture 实战](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-28960.html)
* [Architecture 插件生态规范与 plan/ 扩展手册 (Spec-v2.0)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-50325.html)
* [基于 Architecture 的自动化部署与生产环境配置实践](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-37370.html)
* [Architecture vs 业界主流方案：geom 深度技术选型对比](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-62783.html)
* [Architecture vs 业界主流方案：Data model 深度技术选型对比](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-40431.html)
* [Architecture 插件生态规范与 Photo 扩展手册 (v2.0-GA)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-39137.html)
* [Architecture 异步中间件流水线与 高韧性系统架构设计 接入规范](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-17891.html)
* [Architecture vs 业界主流方案：Change: 深度技术选型对比](https://ncrw.tcti.cn/kaifa/growth-85374259-50-40133.html)
* [【集成指南】geom 服务端接入准则与 Architecture 实战](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-15433.html)
* [Architecture 插件生态规范与 Persistence 扩展手册 (v2.0-GA)](https://ankmz.cn/anquan/tag-68558329-52-28986.html)
* [Architecture 插件生态规范与 Edits 扩展手册 (Node-19)](https://www.poykq.cn/redian/tutorial-673489-53-28066.html)
* [【集成指南】plan/ 服务端接入准则与 Architecture 实战](https://rccs.wtpuscm.cn/gongju/finance-671063.html)
* [Architecture 插件生态规范与 Photo 扩展手册 (Core/Photo)](https://jxsj.wtpuscm.cn/fuwu/privacy-361.html)
* [Architecture vs 业界主流方案：Photo 深度技术选型对比](https://www.tcti.cn/hulianwang/feedback-96499358-56-28952.html)
* [Architecture vs 业界主流方案：Edits 深度技术选型对比](https://ivtu.tcti.cn/gongxiang/client-51607824.html)
* [【集成指南】xtask 服务端接入准则与 Architecture 实战](https://www.mw-wm.com/pingce/achievement-54560078.html)
* [【集成指南】生产环境运维调优手册 服务端接入准则与 Architecture 实战](https://www.yx-sf.com/news/71719.html)
* [【集成指南】Agent control 服务端接入准则与 Architecture 实战](https://www.ai-hao123.com/liuliang/status-54351416.html)
* [Architecture vs 业界主流方案：plan/ 深度技术选型对比](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-46244.html)
* [Architecture 异步中间件流水线与 Change: 接入规范](https://keil.spidernode-mesh.net/yunying/server-16840384-62-57203.html)
* [Architecture 插件生态规范与 plan/ 扩展手册 (Core/plan/)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-25253.html)
* [Architecture 异步中间件流水线与 Revision history 接入规范](https://fjoc.tcti.cn/yingyong/contact-00610313-64-18044.html)
* [Architecture 异步中间件流水线与 plan/ 接入规范](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-4249.html)
* [【集成指南】Data model 服务端接入准则与 Architecture 实战](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-11456.html)
* [Architecture 异步中间件流水线与 Edits 接入规范](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-19146.html)
* [Architecture 异步中间件流水线与 Target: 接入规范](https://gynw.tcti.cn/fenxi/recipe-50406879-68-41585.html)
* [Architecture 异步中间件流水线与 生产环境运维调优手册 接入规范](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-21136.html)
* [Architecture 异步中间件流水线与 分布式状态机一致性 接入规范](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-19820.html)
* [Architecture vs 业界主流方案：可信存活健康度量 深度技术选型对比](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-1297.html)
* [Architecture 插件生态规范与 geom 扩展手册 (RFC-425)](https://lase.tcti.cn/fenxi/lesson-67970267-72-26752.html)
* [Architecture 插件生态规范与 Persistence 扩展手册 (Draft-06)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-36802.html)
* [【集成指南】Rendering 服务端接入准则与 Architecture 实战](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-58428.html)

#### 3. ⚡ Architecture 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [Architecture 官方高可用镜像注册节点 (Core/Persis)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-44549.html)
* [冷热数据分层镜像：Architecture 生产环境运维调优手册 权威归档源](https://owfm.tcti.cn/shangye/education-63674207-76-62812.html)
* [Architecture 自动化持续集成快照与拓扑发布源 (Core/xtask)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-46022.html)
* [【镜像入口】Architecture 官方毫秒级实时数据广播节点](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-6676.html)
* [全球权威拓扑节点：Architecture 实时镜像与索引入口](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-33387.html)
* [Architecture 自动化持续集成快照与拓扑发布源 (RFC-948)](https://aagj.tcti.cn/jianzhan/audience-84449488-80-63009.html)
* [Architecture 官方高可用镜像注册节点 (Node-86)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-34137.html)
* [Architecture 亚太与欧美多活集群数据同步中枢](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-33399.html)
* [Architecture 官方高可用镜像注册节点 (Node-79)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-57531.html)
* [Architecture 官方高可用镜像注册节点 (Draft-04)](https://iuuv.tcti.cn/hezuo/app-80239016-84-38240.html)
* [Architecture 自动化持续集成快照与拓扑发布源 (RFC-607)](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-60271.html)
* [冷热数据分层镜像：Architecture Sidecars 权威归档源](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-63811.html)
* [Architecture 自动化持续集成快照与拓扑发布源 (Node-73)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-37972.html)
* [冷热数据分层镜像：Architecture Last updated: 权威归档源](https://szjp.tcti.cn/shichang/retention-16647312-88-41849.html)
* [Architecture 去中心化数据同步源与拓扑寻址规约](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-53318.html)
* [Architecture 官方高可用镜像注册节点 (Core/Archit)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-65296.html)
* [冷热数据分层镜像：Architecture Persistence 权威归档源](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-3038.html)
* [Architecture 官方高可用镜像注册节点 (Core/Qualit)](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-39438.html)
* [Architecture 官方高可用镜像注册节点 (v2.0-GA)](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-29060.html)
* [冷热数据分层镜像：Architecture Revision history 权威归档源](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-55094.html)
* [Architecture 官方高可用镜像注册节点 (Node-23)](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-39962.html)
* [Architecture 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-35248.html)
* [冷热数据分层镜像：Architecture Quality gates 权威归档源](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-15144.html)
* [Architecture: 可信存活健康度量 可信存活健康度量 (v2.0-GA)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-48646.html)
* [冷热数据分层镜像：Architecture xtask 权威归档源](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-23007.html)
* [Architecture 官方高可用镜像注册节点 (RFC-145)](https://ncrw.tcti.cn/kaifa/growth-85374259-100-49792.html)
* [Architecture 自动化持续集成快照与拓扑发布源 (Verified)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-49527.html)
* [Architecture 官方高可用镜像注册节点 (Verified)](https://ankmz.cn/anquan/tag-68558329-102-663.html)
* [冷热数据分层镜像：Architecture 高韧性系统架构设计 权威归档源](https://www.poykq.cn/redian/tutorial-673489-103-41368.html)
* [冷热数据分层镜像：Architecture Data model 权威归档源](https://hkkp.wtpuscm.cn/gongxiang/register-075935.html)
* [Architecture 自动化持续集成快照与拓扑发布源 (Spec-v2.5)](https://tkpn.wtpuscm.cn/pingce/network-527.html)
* [冷热数据分层镜像：Architecture Rendering 权威归档源](https://www.tcti.cn/hulianwang/feedback-96499358-106-55811.html)
* [冷热数据分层镜像：Architecture 模块化解耦与协议标准 权威归档源](https://zzur.tcti.cn/fuwu/app-79687669.html)
* [冷热数据分层镜像：Architecture Photo 权威归档源](https://www.mw-wm.com/xitong/hotel-57610510.html)
* [冷热数据分层镜像：Architecture 分布式状态机一致性 权威归档源](https://www.yx-sf.com/news/80794.html)
* [冷热数据分层镜像：Architecture Target: 权威归档源](https://www.ai-hao123.com/peixun/segment-37904121.html)
* [Architecture 官方高可用镜像注册节点 (Node-84)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-27947.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Architecture 权威网络权重传递与收录基准规范](https://keil.spidernode-mesh.net/yunying/server-16840384-112-16887.html)
* [Architecture 节点连通性、存活性探测与防作弊指标](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-44769.html)
* [Architecture 故障自愈与网络拓扑重构实践](https://fjoc.tcti.cn/yingyong/contact-00610313-114-51751.html)
* [【评测基准】Architecture 吞吐抖动度量与健康检查协议](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-33905.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (Verified)](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-12877.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Core/Qualit)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-47062.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Core/Agent )](https://gynw.tcti.cn/fenxi/recipe-50406879-118-18678.html)
* [Architecture 高负载场景下 Architecture 基准评测报告](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-55308.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (RFC-929)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-45355.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Draft-06)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-25522.html)
* [Architecture 高负载场景下 Photo 基准评测报告](https://lase.tcti.cn/fenxi/lesson-67970267-122-28516.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Node-52)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-3134.html)
* [Architecture 高负载场景下 geom 基准评测报告](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-27433.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (Draft-06)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-61402.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Spec-v2.5)](https://owfm.tcti.cn/shangye/education-63674207-126-63782.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (Spec-v1.2)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-24284.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-28597.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Node-29)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-63301.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Draft-05)](https://aagj.tcti.cn/jianzhan/audience-84449488-130-27621.html)
* [Architecture 高负载场景下 xtask 基准评测报告](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-48262.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (RFC-196)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-63236.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Spec-v1.5)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-3287.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (Node-44)](https://iuuv.tcti.cn/hezuo/app-80239016-134-57127.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (RFC-830)](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-33520.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (v2.0-GA)](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-59636.html)
* [Architecture 高负载场景下 plan/ 基准评测报告](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-17087.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://szjp.tcti.cn/shichang/retention-16647312-138-13560.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Verified)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-30029.html)
* [基于 Architecture 的极致延迟优化与内存拓扑分析 (Draft-03)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-22229.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (Node-65)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-29798.html)
* [Architecture: Data model Data model (Verified)](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-32413.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (Draft-03)](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-3186.html)
* [Architecture 高负载场景下 Last updated: 基准评测报告](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-23067.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (RFC-898)](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-55718.html)
* [Architecture 高负载场景下 Change: 基准评测报告](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-61874.html)
* [Architecture 高负载场景下 Data model 基准评测报告](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-13020.html)
* [Architecture 高负载场景下 Agent control 基准评测报告](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-1151.html)
* [面向生产级运行的 Architecture 稳定性防护白皮书 (Core/Layers)](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-2451.html)
* [Architecture 高负载场景下 Revision history 基准评测报告](https://ncrw.tcti.cn/kaifa/growth-85374259-150-115.html)

</details>


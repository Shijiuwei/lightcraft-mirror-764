# Gaps: where LightCraft falls short of Lightroom

> **Last reviewed:** 2026-10-10 · **Last updated:** 2026-10-10 · **Change:** major (created; absorbs the checklist's Top gaps and the ROADMAP's honest assessment, re-ranked after a full re-measure) · **Target:** Adobe Lightroom Classic 15.5 / Lightroom 9.6

Every known shortfall, one entry each, ranked by user impact (how many Lightroom users hit it, and
how hard it blocks them), then by effort. **Agents pick work from the top** (the first entry nobody is
working on). When you close a gap, delete or shrink its entry, update its rows in
[parity-checklist.md](parity-checklist.md), and the numbers in [target-app-parity.md](target-app-parity.md)
and [ROADMAP.md](../ROADMAP.md) if they move.

Hours are Opus 5.5 agent wall-clock hours (see [target-app-parity.md](target-app-parity.md#how-hours-are-calibrated)).
`B` marks a gap that blocks **beta**.

## Ranked

| # | Gap | Evidence | User impact | Hours | Doc |
|---:|---|---|---|---:|---|
| 1 B | **Camera colour is estimated, not measured** (LR-PROF-CAMERACOLOR). Raws start from a per-file fit to the camera JPEG, 4 bundled profiles or 52 spectral matrices; others use a neutral matrix | ΔE vs camera JPEG 2.9–10 after fitting (raw-parity.md); 0 chart-calibrated models vs 1,500 Adobe Standard profiles | every raw shooter sees colour that is close but not Lightroom's; some get neutral-matrix colour | 50–90 | [raw-parity](raw-parity.md#colour) |
| 2 B | **CR3 beyond three verified bodies** (LR-IMP-FORMATS): CRX variants other than those verified on M50 / R100 / R8 fall back to the preview; CR3 sRAW / mRAW and HEIF CR3 too | measured: Canon is the largest bucket (33% of photographers) and its share whose camera fully works is the lowest of the big four, 72%; 17 of 22 popular Canon bodies write CR3, 2 verified (raw-parity.md) | +6.5 points of photographers whose camera works, the biggest single raw gain | 25–45 | [raw-parity](raw-parity.md#remaining-work-ranked-by-photographers-unlocked-per-hour) |
| 3 B | **Render fidelity unmeasured** (LR-BEHAV-RENDER-FIDELITY): tone, highlights, texture / clarity / dehaze, NR and sharpening are tuned by eye | no side-by-side suite; a retune needs a new process version (only V1 exists) | edits imported from Lightroom look different; the look "isn't Lightroom's" | 30–60 | [raw-parity](raw-parity.md), [process-versions](process-versions.md) |
| 4 B | **Original-pixel detail at 100%** (LR-VIEW-ZOOM, P0): the whole-image preview is capped (2,560 px default); compare, overlays and soft proofing lack sharp zoom windows (#323) | checklist row 🟡 | checking focus, the first thing culling needs, isn't reliable on large files | 8–15 | [ui-parity](ui-parity.md) |
| 5 B | **AI masks: Subject, Sky, People, Background, Landscape** are heuristics; Object / Describe run SAM 3 but its download has no mirror configured | `crates/segment`; ai-masks.md; MASK rows 67% | masking is how most Lightroom users edit today | 40–70 + model decision | [target-app-parity](target-app-parity.md#ai) |
| 6 B | **Compressed ORF** (every Olympus / OM body since ~2008) | measured: 66 of 78 Olympus / OM models preview-only; `orf.rs` refuses; OM bucket 3% of photographers, 3% of them fully work | +2.6 points; best gain per hour of any raw gap (~0.2) | 8–15 | [raw-parity](raw-parity.md) |
| 7 | **Nikon High Efficiency NEF** (HE / HE★ on Z 8, Z 9, Z 6III, Z f, Z50II, Z5II when set to HE) | named and preview only (#193); needs a pure-Rust JPEG XS decoder | +1.1 points (only shots taken in HE; lossless works on the same bodies); low gain per hour (~0.04), so not a beta blocker | 20–40 | [raw-parity](raw-parity.md) |
| 8 B | **No lens profile database** (LR-EDIT-OPTICS-PROFILE): only DNG opcodes, RW2 in-file distortion and Sony ILCE-7RM4A distortion | Adobe ships 3,635 lens profiles; we never use them | distortion / vignetting correction is a default step for many | 40–80 + calibration shots | [raw-parity](raw-parity.md#lens-profiles) |
| 9 B | **Tests don't run in CI on macOS, Windows or Linux x64**: only FreeBSD runs `cargo test`; the others build packages | `.github/workflows/` (freebsd, packaging-lint, release, windows-arm64) | regressions reach users on the main platforms; `cargo xtask ci` runs only locally | 6–12 | [target-app-parity](target-app-parity.md#by-dimension) |
| 10 B | **Per-model verification thin, plus cheap per-body decode fixes**: 234 of 1,446 Adobe models verified (16%), ~40% of photographers on a verified body; Nikon 5 bodies verified, Canon 11. Cheap fixes: Z 6II packed 14-bit NEF (#769, PR #671), Sony A100 / F828 regression (#702), DSLR-A200…A390 / A850 12-bit ARW (#535) | measured: [raw/coverage.tsv](raw/coverage.tsv) | per-body bugs (levels, crops, WB) surface only from users | 25–45 | [raw-parity](raw-parity.md#by-maker-measured) |
| 11 | **Enhance: Raw Details, Super Resolution; AI denoise without shippable weights or X-Trans** | ENH rows 0%; denoise Bayer only, RawNIND weights GPL and opt-in | Fujifilm users get no AI denoise; nobody gets it out of the box | 30–60 + model decision | [raw-parity](raw-parity.md#demosaic-levels-crops-and-lens-metadata) |
| 12 | **Content-aware / generative remove, detect distractions, reflections removal** | REM rows 75%; patch synthesis missing | cleanup work goes back to Photoshop | 25–45 (+ generative model) | [target-app-parity](target-app-parity.md#ai) |
| 13 | **Lightroom catalog migration is one-way and approximate**: unsupported Adobe profiles / AI settings archived, history kept as data | lightroom-catalog-import.md | switching users lose some looks | 8–12 | [file-format-parity](file-format-parity.md) |
| 14 | **HDR display** (EDR / Windows HDR); HLG; HDR settings from `crs` XMP | HDR rows 70%; LR-VIEW-HDR-DISPLAY ⬜ | HDR editing works blind on HDR monitors | 12–25 | [hardware-parity](hardware-parity.md) |
| 15 | **Print module**: custom packages, printer colour management (contact-sheet PDF only) | LRC-PRINT rows | Classic users who print | 25–40 | [target-app-parity](target-app-parity.md#features) |
| 16 | **Map view, Book, Slideshow module, publish services** | LRC-MAP / BOOK / SS rows ⬜ | Classic users; publish services matter for Flickr / SmugMug workflows | 60–100 | [target-app-parity](target-app-parity.md#features) |
| 17 | **Tethered capture** | ⬜ | studio photographers | 25–45 + cameras | [hardware-parity](hardware-parity.md) |
| 18 | **Video**: playback, trim, global edits, export | VID rows 0% | users who mix stills and clips | 25–45 | [target-app-parity](target-app-parity.md#features) |
| 19 | **Plug-in ecosystem**: no Lightroom SDK equivalent (export / publish plug-ins, metadata plug-ins) | ⬜ | pro workflows built on plug-ins (LR/Enfuse, Jeffrey's exporters, Negative Lab Pro) | 40–80 + owner decision | [target-app-parity](target-app-parity.md#by-dimension) |
| 20 | **Long-tail raw formats**: compressed SRW, ARQ pixel shift, IIQ, Hasselblad compression, X3F, CRW, MRW, ERF, KDC, DCR, MEF, GPR, old FinePix | raw-parity.md | small user share each | 45–85 | [raw-parity](raw-parity.md) |
| 21 | **Localization**: 9 of 15 shipped languages partial (74–99% of catalog messages), lower-layer errors English; Hindi, Arabic, Indonesian, Korean, Vietnamese absent; no RTL or complex shaping | measured from catalogs | non-English users see mixed English | 55–90 (shipped to full); 145–255 (five new) | [localization-parity](localization-parity.md) |
| 22 | **Pen pressure**, OS monitor profiles, follow-window colour | hardware-parity.md | brush users with tablets; multi-monitor setups | 11–22 | [hardware-parity](hardware-parity.md) |
| 23 | **Accessibility** audit with screen readers; canvas tools keyboard-reachable | LR-BEHAV-ACCESS 🟡 | blind and low-vision users | 10–20 | [ui-parity](ui-parity.md) |
| 24 | **Brush live preview and masking feel** (#517, #518); Feather / Edge mask refinement (15.5) | issues | masking feels slower than Lightroom | 8–12 | [ui-parity](ui-parity.md) |
| 25 | **File writes**: JXL encode, PSD export, lossy DNG, Render to DNG; AVIF import | file-format-parity.md | interchange edge cases | 25–45 | [file-format-parity](file-format-parity.md) |
| 26 | **Performance at scale**: 85k-photo library opens in 1.7–4.7 s; 100k grid test pending; colour NR at half resolution | ROADMAP log, gpu-pipeline.md | large-catalog users | 15–30 | [target-app-parity](target-app-parity.md#by-dimension) |
| 27 | **Windows / Linux runtime coverage**: Windows installer UI unverified on Windows (#79); GPU path proven on Apple only | ROADMAP, issues | non-Mac users find bugs first | 15–30 | [hardware-parity](hardware-parity.md) |
| 28 | **Small Classic shortcuts and keys**: rating `[` `]`, flag cycling, filter-bar keys, F1 help, panel keys (Tab, F5–F8) | [ui-parity](ui-parity.md#shortcuts) | Classic muscle memory | 4–8 | [ui-parity](ui-parity.md) |

Beta-blocking gaps (B) total **~230–430 h**; with the stability and platform work they need around them,
**~340–600 h** to beta (incl. 40–80 h for the ~92 open core-path user issues) (see [target-app-parity.md](target-app-parity.md#stage)).

## Gaps by kind

- **Feature:** 5, 8, 11, 12, 13, 15, 16, 18, 19
- **UI / UX:** 4, 23, 24, 28
- **File format:** 2, 6, 7, 10, 13, 20, 25 (raw ones in [raw-parity.md](raw-parity.md))
- **Hardware:** 14, 17, 22, 27
- **Localization:** 21
- **Quality / fidelity:** 1, 3
- **Stability and platforms:** 9, 26, 27

## Closed recently (kept for calibration)

- 2026-10-10: Nikon lossy-after-split NEF (#633), ORF 12.8 bpp (#651), HDR gain map / PQ AVIF / float
  TIFF export (#646), activity stack (#345).
- 2026-10-09: HEIC in release builds (#609), Samsung uncompressed SRW (#459), contact-sheet PDF (#563),
  Ukrainian UI (#564), Sony ILCE-7CR profile.
- 2026-10-08: CR3 CRX lossless and C-RAW (#279), Fujifilm compressed RAF (#282), AI RAW denoise (#384),
  Lightroom catalog import (#218), German, Spanish, Russian UI.
- 2026-10-07: RW2 every format (#215), faces recognition in pure Rust (#174).
- 2026-10-05: compressed NEF (#86). The older history is in [ROADMAP.md](../ROADMAP.md#progress-log).

## Revision history

| Date | Change | Summary |
|---|---|---|
| 2026-10-10 | minor | RAW gaps re-measured model by model (raw-parity.md): CR3, compressed ORF and verification entries carry measured shares; HE NEF no longer a beta blocker (+1.1 points for 20–40 h); verification entry absorbs the cheap per-body fixes (#769, #702, A200 series) |
| 2026-10-10 | minor | Beta total raised to 340–600 h to include the ~92 open core-path user issues |
| 2026-10-10 | major | Created from the checklist's Top gaps (whose items 1–2 had been duplicated by a merge) and ROADMAP's Where we stand; re-ranked; added evidence, impact, hours and beta flags; added platform-CI, lens-database, HE NEF and localization gaps |


---

<details open>
<summary><b>📚 lightcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「lightcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 lightcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 lightcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://lxva.wtpuscm.cn/kuangjia/course-219451.html)
* [现代 分布式架构 架构演进之路 —— lightcraft 深度实践](https://anse.wtpuscm.cn/gongsi/market-277495.html)
* [【官方规范】lightcraft 生产环境运维调优手册 核心运行拓扑标准](https://libn.wtpuscm.cn/gongxiang/download-842453.html)
* [lightcraft 内部组件解耦与事件状态机规范 (RFC-335)](https://hnhu.wtpuscm.cn/paiming/web-467960.html)
* [基于 lightcraft 的高吞吐 分布式架构 设计白皮书](https://qvrj.wtpuscm.cn/zixun/tutorial-383314.html)
* [面向大规模网络的 lightcraft 工业级架构基准](https://qkrz.wtpuscm.cn/chanpin/logo-093345.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (v2.0-GA)](https://ddzw.wtpuscm.cn/anfang/feedback-192347.html)
* [基于 lightcraft 的高吞吐 Rust高性能网络 设计白皮书](https://dbha.wtpuscm.cn/guanjianci/design-159.html)
* [【官方规范】lightcraft 高韧性系统架构设计 核心运行拓扑标准](https://kmcu.wtpuscm.cn/wenzhang/database-500156.html)
* [基于 lightcraft 的高吞吐 开源镜像站 设计白皮书](https://gtra.wtpuscm.cn/chanpin/web-455867.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (Core/Organi)](https://ymrz.wtpuscm.cn/jianzhan/study-044173.html)
* [现代 搜索引擎调度优化 架构演进之路 —— lightcraft 深度实践](https://zevd.wtpuscm.cn/pingtai/food-188198.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (RFC-525)](https://vqmf.wtpuscm.cn/sheji/register-282669.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Draft-02)](https://ohys.wtpuscm.cn/jishu/web-504334.html)
* [【官方规范】lightcraft Fast, native, private 核心运行拓扑标准](https://lqli.wtpuscm.cn/anfang/online-817363.html)
* [现代 Rust高性能网络 架构演进之路 —— lightcraft 深度实践](https://fziv.wtpuscm.cn/jishu/chapter-917693.html)
* [【官方规范】lightcraft Quick start 核心运行拓扑标准](https://ayfa.wtpuscm.cn/gongsi/media-105693.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://sver.wtpuscm.cn/gongju/optimization-542964.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.6)](https://ywld.wtpuscm.cn/yunying/visitor-131330.html)
* [【官方规范】lightcraft Lightroom 核心运行拓扑标准](https://sfrx.wtpuscm.cn/chuangxin/photo-421158.html)
* [【官方规范】lightcraft non-destructive 核心运行拓扑标准](https://akij.wtpuscm.cn/gongxiang/rating-555837.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://gpic.wtpuscm.cn/wendang/identity-279994.html)
* [Built for agents 核心系统架构与设计规约 (v2.0-GA)](https://guru.wtpuscm.cn/shuju/account-252453.html)
* [【官方规范】lightcraft Adobe 核心运行拓扑标准](https://ruoa.tcti.cn/yinqing/strategy-37998870.html)
* [Fast, native, private 核心系统架构与设计规约 (Draft-04)](https://txzy.tcti.cn/guanjianci/affordable-41805542.html)
* [【官方规范】lightcraft open-source 核心运行拓扑标准](https://miuv.tcti.cn/peixun/policy-56395138.html)
* [【官方规范】lightcraft lightcraft 核心运行拓扑标准](https://phgr.tcti.cn/zhinan/finance-79127355.html)
* [lightcraft 分布式数据通道与 开源镜像站 技术规范 (Draft-02)](https://jblh.tcti.cn/huodong/platform-60670654.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (v2.0-GA)](https://kvwk.tcti.cn/jiaoliu/change-49110392.html)
* [lightcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://bgxv.tcti.cn/huodong/productivity-34817256.html)
* [✨ Effects 核心系统架构与设计规约 (Draft-05)](https://bzfi.tcti.cn/fenxi/client-71457142.html)
* [Fast, native, private 核心系统架构与设计规约 (Verified)](https://lcbf.tcti.cn/anli/collaborate-86145582.html)
* [lightcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://huxd.tcti.cn/fuwu/forum-97820842.html)
* [可信存活健康度量 核心系统架构与设计规约 (Spec-v2.5)](https://zoac.tcti.cn/jianzhan/customer-77205623.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://ggjy.tcti.cn/huodong/account-77247400.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Node-92)](https://sfoe.tcti.cn/zixun/business-70542103.html)
* [【官方规范】lightcraft Built for agents 核心运行拓扑标准](https://scnj.tcti.cn/yinqing/health-89374461.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [lightcraft 核心 API 接口契约与客户端调用指南](https://lhvk.tcti.cn/liuliang/terms-24396395.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 lightcraft 实战](https://bbcz.tcti.cn/chuangxin/enterprise-82007887.html)
* [基于 lightcraft 的自动化部署与生产环境配置实践](https://jatv.tcti.cn/fuwu/social-38741788.html)
* [lightcraft 插件生态规范与 clean-room 扩展手册 (Spec-v2.8)](https://wjly.wtpuscm.cn/xuexi/plugin-075631.html)
* [【生产手册】lightcraft 模块通信与请求穿透标准](https://www.mw-wm.com/yunsuan/finance-90680923.html)
* [【集成指南】分布式架构 服务端接入准则与 lightcraft 实战](https://www.yx-sf.com/wiki/53952)
* [【集成指南】开源镜像站 服务端接入准则与 lightcraft 实战](https://www.ai-hao123.com/zixun/ranking-23298902.html)
* [lightcraft 异步中间件流水线与 Lightroom 接入规范](https://vyxp.tcti.cn/hezuo/goal-14300420.html)
* [lightcraft 异步中间件流水线与 高韧性系统架构设计 接入规范](https://isjk.tcti.cn/wendang/hotel-92926098.html)
* [lightcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://jsrw.wtpuscm.cn/zhizhu/market-105359.html)
* [lightcraft 插件生态规范与 Fast, native, private 扩展手册 (Node-32)](https://kdxs.wtpuscm.cn/chuangxin/about-867446.html)
* [lightcraft 异步中间件流水线与 Fast, native, private 接入规范](https://fgos.wtpuscm.cn/shichang/fitness-391942.html)
* [lightcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://qlzw.wtpuscm.cn/baogao/navigation-213660.html)
* [lightcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://aojj.wtpuscm.cn/wendang/client-693585.html)
* [lightcraft 插件生态规范与 高韧性系统架构设计 扩展手册 (Draft-04)](https://ckye.wtpuscm.cn/anfang/goal-824601.html)
* [lightcraft 插件生态规范与 ⚫️ Black & White 扩展手册 (Node-68)](https://kdap.wtpuscm.cn/hezuo/goal-520481.html)
* [lightcraft 插件生态规范与 模块化解耦与协议标准 扩展手册 (v2.0-GA)](https://afix.wtpuscm.cn/gongsi/section-866.html)
* [lightcraft 插件生态规范与 Organize everything 扩展手册 (Node-46)](https://wsxi.wtpuscm.cn/wenzhang/label-574290.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 lightcraft 实战](https://vhdf.wtpuscm.cn/shuju/expensive-406004.html)
* [lightcraft vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://ldlo.wtpuscm.cn/zixun/seo-402689.html)
* [lightcraft 异步中间件流水线与 Edit like you mean it 接入规范](https://uora.wtpuscm.cn/shuju/server-668557.html)
* [lightcraft 插件生态规范与 生产环境运维调优手册 扩展手册 (RFC-546)](https://jokf.wtpuscm.cn/huodong/recommendation-235950.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Node-97)](https://mnsb.wtpuscm.cn/gongsi/fitness-971451.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-07)](https://bfpo.wtpuscm.cn/wenzhang/faq-572244.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (RFC-944)](https://baaj.wtpuscm.cn/yingyong/api-931408.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-01)](https://swdk.wtpuscm.cn/pingce/satisfaction-815954.html)
* [lightcraft 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://krif.wtpuscm.cn/chuangxin/development-790278.html)
* [lightcraft 插件生态规范与 可信存活健康度量 扩展手册 (RFC-198)](https://hkzu.wtpuscm.cn/guanjianci/module-230333.html)
* [lightcraft 插件生态规范与 Lightroom 扩展手册 (Node-40)](https://ewla.wtpuscm.cn/wangluo/account-593913.html)
* [lightcraft 插件生态规范与 🎨 Color 扩展手册 (Verified)](https://bqxy.wtpuscm.cn/anfang/domain-234975.html)
* [lightcraft 异步中间件流水线与 non-destructive 接入规范](https://ycwy.wtpuscm.cn/anli/help-076791.html)
* [lightcraft 异步中间件流水线与 Vibrance 接入规范](https://wqwp.wtpuscm.cn/zhineng/solution-364555.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Draft-03)](https://cmdf.tcti.cn/xinwen/network-59474488.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Spec-v1.0)](https://pusm.tcti.cn/gongxiang/plugin-08898514.html)
* [lightcraft 插件生态规范与 lightcraft 扩展手册 (v2.0-GA)](https://wkzq.tcti.cn/shuju/efficiency-60493491.html)
* [lightcraft 插件生态规范与 non-destructive 扩展手册 (Core/non-de)](https://bklm.tcti.cn/wenzhang/satisfaction-63703591.html)
* [lightcraft 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-888)](https://bhav.tcti.cn/chanpin/faq-02433218.html)

#### 3. ⚡ lightcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [lightcraft 官方高可用镜像注册节点 (RFC-691)](https://fhmq.tcti.cn/pingtai/retention-33499431.html)
* [lightcraft 亚太与欧美多活集群数据同步中枢](https://gjzr.tcti.cn/chuangxin/client-38363197.html)
* [lightcraft 官方高可用镜像注册节点 (v2.0-GA)](https://feli.tcti.cn/zixun/study-01736785.html)
* [全球权威拓扑节点：lightcraft 实时镜像与索引入口](https://fzhl.tcti.cn/gongsi/beauty-81621193.html)
* [【镜像入口】lightcraft 官方毫秒级实时数据广播节点](https://zffm.tcti.cn/gongxiang/search-06078218.html)
* [冷热数据分层镜像：lightcraft 可信存活健康度量 权威归档源](https://nquu.tcti.cn/yanjiu/advertising-43413019.html)
* [冷热数据分层镜像：lightcraft 生产环境运维调优手册 权威归档源](https://buxf.tcti.cn/paiming/backup-88021577.html)
* [冷热数据分层镜像：lightcraft Vibrance 权威归档源](https://eahi.tcti.cn/yunsuan/automation-23254609.html)
* [lightcraft 官方高可用镜像注册节点 (Draft-01)](https://wrel.tcti.cn/kuangjia/content-98321163.html)
* [lightcraft 官方高可用镜像注册节点 (Node-78)](https://xpfn.tcti.cn/shangye/price-09492803.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.1)](https://tmxz.tcti.cn/guanjianci/restore-39654109.html)
* [lightcraft 去中心化数据同步源与拓扑寻址规约](https://zurt.tcti.cn/gongju/target-24126587.html)
* [冷热数据分层镜像：lightcraft White balance 权威归档源](https://ixhf.wtpuscm.cn/chanpin/chapter-761381.html)
* [冷热数据分层镜像：lightcraft Lightroom 权威归档源](https://www.mw-wm.com/zhizhu/accessibility-20706121.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.7)](https://www.yx-sf.com/wiki/93747)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://www.ai-hao123.com/jiaoliu/screen-93225170.html)
* [冷热数据分层镜像：lightcraft ☀️ Light 权威归档源](https://gwwr.tcti.cn/liuliang/review-94378222.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-08)](https://foej.tcti.cn/wenzhang/software-94988023.html)
* [冷热数据分层镜像：lightcraft 🎨 Color 权威归档源](https://zimn.wtpuscm.cn/hezuo/review-067412.html)
* [冷热数据分层镜像：lightcraft reimplementation 权威归档源](https://hbcj.wtpuscm.cn/zhizhu/profit-796052.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://keul.wtpuscm.cn/guanjianci/discovery-364956.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://mljq.wtpuscm.cn/youhua/notification-630185.html)
* [冷热数据分层镜像：lightcraft pure 权威归档源](https://uksw.wtpuscm.cn/jiaocheng/platform-914069.html)
* [冷热数据分层镜像：lightcraft eyedropper 权威归档源](https://txxi.wtpuscm.cn/sheji/goal-898404.html)
* [冷热数据分层镜像：lightcraft Edit like you mean it 权威归档源](https://qbnb.wtpuscm.cn/chuangxin/seminar-878876.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/☀️ Lig)](https://bdke.wtpuscm.cn/wenzhang/security-053.html)
* [冷热数据分层镜像：lightcraft Built for agents 权威归档源](https://sxiu.wtpuscm.cn/anli/tracking-607734.html)
* [lightcraft: White balance 分布式架构 (Core/White )](https://rjpy.wtpuscm.cn/kuangjia/learning-890335.html)
* [冷热数据分层镜像：lightcraft Quick start 权威归档源](https://xjtb.wtpuscm.cn/pingtai/cheap-233741.html)
* [冷热数据分层镜像：lightcraft 分布式状态机一致性 权威归档源](https://vwuk.wtpuscm.cn/ziyuan/meeting-707872.html)
* [lightcraft 官方高可用镜像注册节点 (Core/Featur)](https://zohj.wtpuscm.cn/qiye/backup-536966.html)
* [冷热数据分层镜像：lightcraft Feature status 权威归档源](https://apfk.wtpuscm.cn/anfang/target-061385.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v2.6)](https://zcfr.wtpuscm.cn/xitong/site-187536.html)
* [lightcraft 官方高可用镜像注册节点 (Verified)](https://rifz.wtpuscm.cn/keji/development-497816.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-07)](https://yppw.wtpuscm.cn/fenxi/server-625789.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/Quick )](https://tjld.wtpuscm.cn/yingxiao/logo-403258.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.3)](https://qqaa.wtpuscm.cn/yanjiu/system-180325.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://czmn.wtpuscm.cn/zhizhu/news-007592.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-460)](https://gcke.wtpuscm.cn/youhua/update-526188.html)
* [【评测基准】lightcraft 吞吐抖动度量与健康检查协议](https://mwlm.wtpuscm.cn/peixun/customer-591081.html)
* [lightcraft 权威网络权重传递与收录基准规范](https://riru.wtpuscm.cn/anli/innovation-115892.html)
* [lightcraft 故障自愈与网络拓扑重构实践](https://mysq.tcti.cn/yunsuan/education-38255648.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-23)](https://owfe.tcti.cn/qiye/price-81722983.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (v2.0-GA)](https://qamn.tcti.cn/yunsuan/media-27994651.html)
* [lightcraft 节点连通性、存活性探测与防作弊指标](https://lpfz.tcti.cn/peixun/dashboard-18554982.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-778)](https://zzio.tcti.cn/chuangxin/traffic-83960073.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-52)](https://dajk.tcti.cn/ziyuan/fashion-83948247.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Core/Featur)](https://sjus.tcti.cn/xitong/technology-59882148.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-925)](https://ycht.tcti.cn/kaifa/help-94343827.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Verified)](https://juxj.tcti.cn/shuju/discount-77348132.html)
* [lightcraft 高负载场景下 Rust高性能网络 基准评测报告](https://qlxa.tcti.cn/zhineng/extension-17031790.html)
* [lightcraft: Feature status Rust高性能网络 (Verified)](https://liim.tcti.cn/zhineng/collaborate-37477442.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://urxb.tcti.cn/jianzhan/recommendation-51518537.html)
* [lightcraft: reimplementation Rust高性能网络 (v2.0-GA)](https://uuxa.tcti.cn/peixun/seminar-39246835.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-35)](https://opsq.tcti.cn/yingxiao/subscribe-19058433.html)
* [lightcraft: reimplementation 分布式架构 (Verified)](https://hxbi.tcti.cn/sheji/strategy-12128009.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/Organi)](https://isio.tcti.cn/wangluo/satisfaction-51556003.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/⚫️ Bla)](https://tkvm.tcti.cn/gongsi/button-99375506.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://wdtp.wtpuscm.cn/tuiguang/shopping-757942.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-96)](https://www.mw-wm.com/ziyuan/status-64963122.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/White )](https://www.yx-sf.com/news/54396)
* [lightcraft 高负载场景下 分布式架构 基准评测报告](https://www.ai-hao123.com/xinwen/funnel-20983426.html)
* [lightcraft 高负载场景下 开源镜像站 基准评测报告](https://oscr.tcti.cn/keji/audience-06073513.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://nplc.tcti.cn/sheji/topic-48175427.html)
* [lightcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://nmet.wtpuscm.cn/baogao/economy-959572.html)
* [lightcraft: Organize everything Rust高性能网络 (Verified)](https://maoh.wtpuscm.cn/yingyong/resource-009643.html)
* [lightcraft: pure 分布式架构 (Verified)](https://bxji.wtpuscm.cn/chanpin/budget-366965.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-35)](https://fifi.wtpuscm.cn/tuiguang/feedback-330757.html)
* [lightcraft: 📈 Tone Curve 搜索引擎调度优化 (v2.0-GA)](https://ioig.wtpuscm.cn/hezuo/calendar-542529.html)
* [lightcraft: ✨ Effects Rust高性能网络 (v2.0-GA)](https://vizs.wtpuscm.cn/zhineng/target-797549.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-73)](https://afuq.wtpuscm.cn/wendang/research-945645.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (RFC-419)](https://zzdq.wtpuscm.cn/wenzhang/demographic-561.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-89)](https://zgvy.wtpuscm.cn/jishu/webinar-905873.html)
* [lightcraft: Edit like you mean it 搜索引擎调度优化 (Node-98)](https://pjcy.wtpuscm.cn/fenxi/workshop-872624.html)
* [lightcraft: White balance 搜索引擎调度优化 (Verified)](https://pwdd.wtpuscm.cn/zixun/topic-789238.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-40)](https://qevq.wtpuscm.cn/zhinan/strategy-872860.html)

</details>


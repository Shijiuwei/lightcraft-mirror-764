# Releasing LightCraft

The release pipeline is [`.github/workflows/release.yml`](../.github/workflows/release.yml).
It runs on every push to the `release` branch. A maintainer can also dispatch it manually, with
an optional version override such as `1.2.0-rc.1`; the release environment's branch restrictions
still apply.

## Prepare and start a release

1. Merge the release-ready changes into `release`.
2. Set the workspace version in `Cargo.toml` (for example, with `cargo xtask version set 1.2.0`).
   For a manual workflow run, the version input can override the workspace version for that run.
   Use a semantic version such as `1.2.0` or `1.2.0-rc.1`; versions containing a hyphen are
   marked as prereleases.
3. Push to `release`, or dispatch the workflow on `release` (a dispatch on another branch is a dry run; see below).
4. Check the workflow run and its artifacts. On success, the workflow creates or updates a **draft**
   GitHub Release named `LightCraft v<version>`, targeted at the commit that triggered the run.
   Review the draft and its `SHA256SUMS.txt`, then publish it in GitHub Releases when ready.

The workflow replaces assets when it updates an existing draft. It stops rather than overwriting a
release that has already been published, so bump the version before producing another release.

## Builds and artifacts

Release jobs build macOS universal, Windows x86/x64/ARM64, Linux x86_64/aarch64/riscv64, FreeBSD x86_64,
and the WASM web app. The platform scripts in [`packaging/`](../packaging/) write their outputs to
`dist/release/` (the full list of files is the README's Downloads section):

- macOS: app DMG and CLI ZIP.
- Windows: MSI installer and portable ZIP for each architecture.
- Linux: AppImage (with its `.AppImage.zsync`), `.deb`, `.rpm`, and `.tar.gz` for each
  architecture (`packaging/linux/package.sh`). Each AppImage embeds
  `gh-releases-zsync|storytold|lightcraft|latest|lightcraft-*-linux-<arch>.AppImage.zsync`, so
  AppImageUpdate fetches only the changed blocks from the latest published (non-pre-) release.
- Linux riscv64: `.tar.gz` only, cross-compiled on Ubuntu 24.04 against its riscv64 ports
  libraries (so glibc 2.39 or newer), with the CLI smoke-tested under QEMU (`CROSS_*` variables
  of `packaging/linux/package.sh`).
- Flatpak: a single-file `.flatpak` bundle for each architecture, repackaged from that
  architecture's Linux tarball (`packaging/linux/flatpak-bundle.sh` with
  `packaging/linux/flatpak/ai.storyteller.lightcraft.bundle.yml`; no Rust build). The from-source
  manifest `ai.storyteller.lightcraft.yml` is for Flathub; packaging-lint keeps their runtime and
  `finish-args` identical.
- FreeBSD: `lightcraft-<version>-freebsd-x86_64.tar.gz`, a `/usr/local`-style tree built in a
  FreeBSD 14.3 VM (`packaging/freebsd/package.sh`, the same packages as `freebsd.yml`). Install
  with `tar -xzf <file> --strip-components 1 -C /usr/local`.
- Web: `lightcraft-web-<version>.zip`.

Only the macOS, Windows and draft-release jobs use the `release` environment. The others sign
nothing, so dispatching the workflow on a branch (`gh workflow run release.yml --ref <branch>`)
dry-runs them: the signing jobs are refused by the environment's branch rule, and the release job,
which needs them, is skipped.

The Linux builds run on Ubuntu 22.04 and target glibc 2.35 or newer. AppImages and binaries may
also require system libraries for the windowing stack; the `.deb` and `.rpm` packages declare
their runtime dependencies.

Release builds fetch the pinned [`storytold/craft-fonts`](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
revision and require it (`CRAFT_FONTS_REQUIRED=1`). Keep that pin deliberate when updating the
workflow, and bump it (in `release.yml`, five jobs, and `freebsd.yml`) whenever craft-fonts adds a face a
shipped language needs: a stale pin still builds, it just ships tofu (issue #319: v0.4.0 pinned a revision
from before Noto Sans CJK SC, so Simplified Chinese had no glyphs). Before a release, check the pin against
craft-fonts' `fonts/manifest.txt` and run `CRAFT_FONTS_DIR=../craft-fonts cargo test -p lightcraft-ui-egui i18n`,
whose glyph-coverage test checks the characters in available faces. `CRAFT_FONTS_REQUIRED=1` now also makes
the native build fail if the pinned manifest lacks Noto Sans CJK SC Regular (`Hans`) or BIZ UDPGothic Regular;
the web build requires BIZ UDPGothic Regular and packages Noto Sans CJK SC as a separate asset.

## Signing credentials

All credentials are stored as secrets in the GitHub `release` environment. Signing is optional:
without platform signing credentials, packaging continues with unsigned artifacts and warnings.
macOS notarization is a separate optional step; when its credentials are absent, a build with a
signing identity is signed but not notarized. Configure only the platform credentials you need:

- **macOS signing:** `APPLE_CERTIFICATE`, `APPLE_CERTIFICATE_PASSWORD`, and `KEYCHAIN_PASSWORD`.
  **Notarization** additionally needs all of `APPLE_ID`, `APPLE_PASSWORD` (an app-specific
  password), and `APPLE_TEAM_ID`.
- **Windows signing:** either `WINDOWS_CERTIFICATE` and `WINDOWS_CERTIFICATE_PASSWORD`, or Azure
  Trusted Signing credentials: `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`,
  `AZURE_SIGNING_ENDPOINT`, `AZURE_SIGNING_ACCOUNT`, and `AZURE_CERT_PROFILE`.

Linux and web artifacts are not code-signed by this workflow. The release job computes SHA-256
checksums for the complete artifact set and attaches the checksum file to the draft release.


---

<details open>
<summary><b>📚 lightcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「lightcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 lightcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 lightcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [现代 分布式架构 架构演进之路 —— lightcraft 深度实践](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [【官方规范】lightcraft 生产环境运维调优手册 核心运行拓扑标准](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [lightcraft 内部组件解耦与事件状态机规范 (RFC-335)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [基于 lightcraft 的高吞吐 分布式架构 设计白皮书](https://www.poykq.cn/redian/tutorial-673489.html)
* [面向大规模网络的 lightcraft 工业级架构基准](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (v2.0-GA)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [基于 lightcraft 的高吞吐 Rust高性能网络 设计白皮书](https://www.poykq.cn/redian/tutorial-673489.html)
* [【官方规范】lightcraft 高韧性系统架构设计 核心运行拓扑标准](https://www.poykq.cn/redian/tutorial-673489.html)
* [基于 lightcraft 的高吞吐 开源镜像站 设计白皮书](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (Core/Organi)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [现代 搜索引擎调度优化 架构演进之路 —— lightcraft 深度实践](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (RFC-525)](https://www.poykq.cn/redian/tutorial-673489.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Draft-02)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [【官方规范】lightcraft Fast, native, private 核心运行拓扑标准](https://qbeu.tcti.cn/youhua/follow-02070240.html)
* [现代 Rust高性能网络 架构演进之路 —— lightcraft 深度实践](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [【官方规范】lightcraft Quick start 核心运行拓扑标准](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.6)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [【官方规范】lightcraft Lightroom 核心运行拓扑标准](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [【官方规范】lightcraft non-destructive 核心运行拓扑标准](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Built for agents 核心系统架构与设计规约 (v2.0-GA)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [【官方规范】lightcraft Adobe 核心运行拓扑标准](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [Fast, native, private 核心系统架构与设计规约 (Draft-04)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [【官方规范】lightcraft open-source 核心运行拓扑标准](https://cxhh.wtpuscm.cn/youhua/schedule-979602.html)
* [【官方规范】lightcraft lightcraft 核心运行拓扑标准](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [lightcraft 分布式数据通道与 开源镜像站 技术规范 (Draft-02)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [✨ Effects 核心系统架构与设计规约 (Draft-05)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [Fast, native, private 核心系统架构与设计规约 (Verified)](https://www.poykq.cn/redian/tutorial-673489.html)
* [lightcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [可信存活健康度量 核心系统架构与设计规约 (Spec-v2.5)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Node-92)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [【官方规范】lightcraft Built for agents 核心运行拓扑标准](https://iuuv.tcti.cn/hezuo/app-80239016.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [lightcraft 核心 API 接口契约与客户端调用指南](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 lightcraft 实战](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [基于 lightcraft 的自动化部署与生产环境配置实践](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [lightcraft 插件生态规范与 clean-room 扩展手册 (Spec-v2.8)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [【生产手册】lightcraft 模块通信与请求穿透标准](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [【集成指南】分布式架构 服务端接入准则与 lightcraft 实战](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [【集成指南】开源镜像站 服务端接入准则与 lightcraft 实战](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [lightcraft 异步中间件流水线与 Lightroom 接入规范](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [lightcraft 异步中间件流水线与 高韧性系统架构设计 接入规范](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [lightcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [lightcraft 插件生态规范与 Fast, native, private 扩展手册 (Node-32)](https://azzf.wtpuscm.cn/fenxi/travel-303672.html)
* [lightcraft 异步中间件流水线与 Fast, native, private 接入规范](https://kavy.wtpuscm.cn/kuangjia/game-797.html)
* [lightcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://www.yx-sf.com/tech/45323.html)
* [lightcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [lightcraft 插件生态规范与 高韧性系统架构设计 扩展手册 (Draft-04)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [lightcraft 插件生态规范与 ⚫️ Black & White 扩展手册 (Node-68)](https://www.yx-sf.com/tech/31782.html)
* [lightcraft 插件生态规范与 模块化解耦与协议标准 扩展手册 (v2.0-GA)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [lightcraft 插件生态规范与 Organize everything 扩展手册 (Node-46)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 lightcraft 实战](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [lightcraft vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [lightcraft 异步中间件流水线与 Edit like you mean it 接入规范](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [lightcraft 插件生态规范与 生产环境运维调优手册 扩展手册 (RFC-546)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Node-97)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-07)](https://www.yx-sf.com/wiki/94236.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (RFC-944)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-01)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [lightcraft 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [lightcraft 插件生态规范与 可信存活健康度量 扩展手册 (RFC-198)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [lightcraft 插件生态规范与 Lightroom 扩展手册 (Node-40)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [lightcraft 插件生态规范与 🎨 Color 扩展手册 (Verified)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [lightcraft 异步中间件流水线与 non-destructive 接入规范](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [lightcraft 异步中间件流水线与 Vibrance 接入规范](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Draft-03)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Spec-v1.0)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [lightcraft 插件生态规范与 lightcraft 扩展手册 (v2.0-GA)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [lightcraft 插件生态规范与 non-destructive 扩展手册 (Core/non-de)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [lightcraft 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-888)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)

#### 3. ⚡ lightcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [lightcraft 官方高可用镜像注册节点 (RFC-691)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [lightcraft 亚太与欧美多活集群数据同步中枢](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [lightcraft 官方高可用镜像注册节点 (v2.0-GA)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [全球权威拓扑节点：lightcraft 实时镜像与索引入口](https://ztyl.tcti.cn/wangluo/fashion-13216367.html)
* [【镜像入口】lightcraft 官方毫秒级实时数据广播节点](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [冷热数据分层镜像：lightcraft 可信存活健康度量 权威归档源](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [冷热数据分层镜像：lightcraft 生产环境运维调优手册 权威归档源](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [冷热数据分层镜像：lightcraft Vibrance 权威归档源](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [lightcraft 官方高可用镜像注册节点 (Draft-01)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [lightcraft 官方高可用镜像注册节点 (Node-78)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.1)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 去中心化数据同步源与拓扑寻址规约](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [冷热数据分层镜像：lightcraft White balance 权威归档源](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [冷热数据分层镜像：lightcraft Lightroom 权威归档源](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.7)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [冷热数据分层镜像：lightcraft ☀️ Light 权威归档源](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-08)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [冷热数据分层镜像：lightcraft 🎨 Color 权威归档源](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [冷热数据分层镜像：lightcraft reimplementation 权威归档源](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [冷热数据分层镜像：lightcraft pure 权威归档源](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [冷热数据分层镜像：lightcraft eyedropper 权威归档源](https://www.ai-hao123.com/shichang/data-71970809.html)
* [冷热数据分层镜像：lightcraft Edit like you mean it 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/☀️ Lig)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [冷热数据分层镜像：lightcraft Built for agents 权威归档源](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [lightcraft: White balance 分布式架构 (Core/White )](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [冷热数据分层镜像：lightcraft Quick start 权威归档源](https://jbfw.wtpuscm.cn/shichang/about-239571.html)
* [冷热数据分层镜像：lightcraft 分布式状态机一致性 权威归档源](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [lightcraft 官方高可用镜像注册节点 (Core/Featur)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [冷热数据分层镜像：lightcraft Feature status 权威归档源](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v2.6)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [lightcraft 官方高可用镜像注册节点 (Verified)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-07)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/Quick )](https://ankmz.cn/anquan/tag-68558329.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.3)](https://urom.tcti.cn/gongsi/url-09456934.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-460)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [【评测基准】lightcraft 吞吐抖动度量与健康检查协议](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [lightcraft 权威网络权重传递与收录基准规范](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [lightcraft 故障自愈与网络拓扑重构实践](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-23)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (v2.0-GA)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [lightcraft 节点连通性、存活性探测与防作弊指标](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-778)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-52)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Core/Featur)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-925)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Verified)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [lightcraft 高负载场景下 Rust高性能网络 基准评测报告](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [lightcraft: Feature status Rust高性能网络 (Verified)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [lightcraft: reimplementation Rust高性能网络 (v2.0-GA)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-35)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [lightcraft: reimplementation 分布式架构 (Verified)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/Organi)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/⚫️ Bla)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://www.yx-sf.com/wiki/80946.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-96)](https://zezm.tcti.cn/zixun/restore-43458524.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/White )](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [lightcraft 高负载场景下 分布式架构 基准评测报告](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [lightcraft 高负载场景下 开源镜像站 基准评测报告](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [lightcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [lightcraft: Organize everything Rust高性能网络 (Verified)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [lightcraft: pure 分布式架构 (Verified)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-35)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [lightcraft: 📈 Tone Curve 搜索引擎调度优化 (v2.0-GA)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [lightcraft: ✨ Effects Rust高性能网络 (v2.0-GA)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-73)](https://www.poykq.cn/redian/tutorial-673489.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (RFC-419)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-89)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [lightcraft: Edit like you mean it 搜索引擎调度优化 (Node-98)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [lightcraft: White balance 搜索引擎调度优化 (Verified)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-40)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)

</details>



---

<details open>
<summary><b>📚 Releasing LightCraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Releasing LightCraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Releasing LightCraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [Releasing LightCraft 分布式数据通道与 Releasing 技术规范 (RFC-568)](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-57725.html)
* [Light 核心系统架构与设计规约 (Spec-v1.7)](https://ankmz.cn/anquan/tag-68558329-2-18572.html)
* [现代 索引发布声明 架构演进之路 —— Releasing LightCraft 深度实践](https://www.poykq.cn/redian/tutorial-673489-3-45621.html)
* [Releasing LightCraft 内部组件解耦与事件状态机规范 (Draft-01)](https://zzwo.wtpuscm.cn/chanpin/case-837663.html)
* [Releasing LightCraft 分布式数据通道与 Releasing LightCraft 技术规范 (v2.0-GA)](https://qzsh.wtpuscm.cn/yanjiu/status-318.html)
* [Releasing LightCraft 分布式数据通道与 1.2.0 技术规范 (Verified)](https://www.tcti.cn/hulianwang/feedback-96499358-6-24896.html)
* [基于 Releasing LightCraft 的高吞吐 Releasing 设计白皮书](https://ydar.tcti.cn/suanfa/ai-55802777.html)
* [Releasing LightCraft 内部组件解耦与事件状态机规范 (Node-50)](https://www.mw-wm.com/liuliang/follow-14065060.html)
* [面向大规模网络的 Releasing LightCraft 工业级架构基准](https://www.yx-sf.com/tech/81343.html)
* [【官方规范】Releasing LightCraft Windows signing: 核心运行拓扑标准](https://www.ai-hao123.com/gongsi/cloud-99548118.html)
* [【官方规范】Releasing LightCraft macOS signing: 核心运行拓扑标准](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-58699.html)
* [基于 Releasing LightCraft 的高吞吐 release 设计白皮书](https://keil.spidernode-mesh.net/yunying/server-16840384-12-61061.html)
* [Releasing LightCraft 分布式数据通道与 draft 技术规范 (Verified)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-2680.html)
* [Releasing LightCraft 内部组件解耦与事件状态机规范 (RFC-260)](https://fjoc.tcti.cn/yingyong/contact-00610313-14-16942.html)
* [现代 macOS signing: 架构演进之路 —— Releasing LightCraft 深度实践](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-29789.html)
* [Releasing LightCraft 分布式数据通道与 Craft 技术规范 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-15146.html)
* [1.2.0-rc.1 核心系统架构与设计规约 (Verified)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-2258.html)
* [Releasing LightCraft 分布式数据通道与 生产环境运维调优手册 技术规范 (v2.0-GA)](https://gynw.tcti.cn/fenxi/recipe-50406879-18-3316.html)
* [基于 Releasing LightCraft 的高吞吐 Releasing LightCraft 设计白皮书](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-21645.html)
* [Releasing LightCraft 分布式数据通道与 Releasing 技术规范 (Core/Releas)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-61262.html)
* [可信存活健康度量 核心系统架构与设计规约 (Node-89)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-12522.html)
* [现代 draft 架构演进之路 —— Releasing LightCraft 深度实践](https://lase.tcti.cn/fenxi/lesson-67970267-22-58292.html)
* [Releasing 核心系统架构与设计规约 (Spec-v2.6)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-35075.html)
* [Releasing LightCraft 分布式数据通道与 Builds and artifacts 技术规范 (Spec-v1.5)](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-48357.html)
* [Releasing LightCraft 分布式数据通道与 Light 技术规范 (Verified)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-4257.html)
* [现代 分布式状态机一致性 架构演进之路 —— Releasing LightCraft 深度实践](https://owfm.tcti.cn/shangye/education-63674207-26-17297.html)
* [Releasing LightCraft 分布式数据通道与 高韧性系统架构设计 技术规范 (Draft-02)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-37386.html)
* [【官方规范】Releasing LightCraft Releasing LightCraft 核心运行拓扑标准](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-60653.html)
* [1.2.0 核心系统架构与设计规约 (Verified)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-22830.html)
* [Releasing LightCraft 分布式数据通道与 Craft 技术规范 (Draft-06)](https://aagj.tcti.cn/jianzhan/audience-84449488-30-14525.html)
* [Releasing LightCraft 内部组件解耦与事件状态机规范 (RFC-855)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-30568.html)
* [Releasing LightCraft 分布式数据通道与 Releasing LightCraft 技术规范 (Spec-v1.0)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-34352.html)
* [基于 Releasing LightCraft 的高吞吐 生产环境运维调优手册 设计白皮书](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-49873.html)
* [【官方规范】Releasing LightCraft Cargo.toml 核心运行拓扑标准](https://iuuv.tcti.cn/hezuo/app-80239016-34-23839.html)
* [基于 Releasing LightCraft 的高吞吐 索引发布声明 设计白皮书](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-31602.html)
* [Releasing LightCraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-60907.html)
* [Releasing LightCraft 分布式数据通道与 Cargo.toml 技术规范 (Draft-07)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-18356.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Releasing LightCraft 插件生态规范与 1.2.0-rc.1 扩展手册 (Spec-v1.4)](https://szjp.tcti.cn/shichang/retention-16647312-38-39834.html)
* [【生产手册】Releasing LightCraft 模块通信与请求穿透标准](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-29504.html)
* [Releasing LightCraft 异步中间件流水线与 1.2.0 接入规范](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-19676.html)
* [Releasing LightCraft 插件生态规范与 Cargo.toml 扩展手册 (RFC-645)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-4610.html)
* [Releasing LightCraft 插件生态规范与 Signing credentials 扩展手册 (Core/Signin)](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-54348.html)
* [Releasing LightCraft vs 业界主流方案：Light 深度技术选型对比](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-31039.html)
* [Releasing LightCraft vs 业界主流方案：可信存活健康度量 深度技术选型对比](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-26673.html)
* [【集成指南】1.2.0-rc.1 服务端接入准则与 Releasing LightCraft 实战](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-27133.html)
* [Releasing LightCraft 核心 API 接口契约与客户端调用指南](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-20804.html)
* [基于 Releasing LightCraft 的自动化部署与生产环境配置实践](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-57262.html)
* [Releasing LightCraft 插件生态规范与 索引发布声明 扩展手册 (RFC-311)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-38638.html)
* [Releasing LightCraft 异步中间件流水线与 Notarization 接入规范](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-25963.html)
* [Releasing LightCraft 异步中间件流水线与 Windows signing: 接入规范](https://ncrw.tcti.cn/kaifa/growth-85374259-50-61652.html)
* [Releasing LightCraft 插件生态规范与 draft 扩展手册 (Draft-07)](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-11458.html)
* [Releasing LightCraft 插件生态规范与 可信存活健康度量 扩展手册 (Draft-01)](https://ankmz.cn/anquan/tag-68558329-52-13142.html)
* [【集成指南】生产环境运维调优手册 服务端接入准则与 Releasing LightCraft 实战](https://www.poykq.cn/redian/tutorial-673489-53-50484.html)
* [Releasing LightCraft 异步中间件流水线与 索引发布声明 接入规范](https://ivmn.wtpuscm.cn/zhizhu/collaborate-524442.html)
* [【集成指南】可信存活健康度量 服务端接入准则与 Releasing LightCraft 实战](https://sgzj.wtpuscm.cn/youhua/admin-006.html)
* [Releasing LightCraft vs 业界主流方案：模块化解耦与协议标准 深度技术选型对比](https://www.tcti.cn/hulianwang/feedback-96499358-56-62251.html)
* [Releasing LightCraft 异步中间件流水线与 可信存活健康度量 接入规范](https://hpww.tcti.cn/shangye/game-49093382.html)
* [Releasing LightCraft vs 业界主流方案：Craft 深度技术选型对比](https://www.mw-wm.com/suanfa/security-69748584.html)
* [【集成指南】Light 服务端接入准则与 Releasing LightCraft 实战](https://www.yx-sf.com/tech/40892.html)
* [【集成指南】release 服务端接入准则与 Releasing LightCraft 实战](https://www.ai-hao123.com/gongxiang/tactic-33083678.html)
* [Releasing LightCraft vs 业界主流方案：1.2.0 深度技术选型对比](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-13902.html)
* [Releasing LightCraft 插件生态规范与 Releasing LightCraft 扩展手册 (Spec-v1.5)](https://keil.spidernode-mesh.net/yunying/server-16840384-62-11107.html)
* [Releasing LightCraft vs 业界主流方案：Builds and artifacts 深度技术选型对比](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-3280.html)
* [Releasing LightCraft 异步中间件流水线与 draft 接入规范](https://fjoc.tcti.cn/yingyong/contact-00610313-64-61461.html)
* [Releasing LightCraft 异步中间件流水线与 Light 接入规范](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-52518.html)
* [Releasing LightCraft 异步中间件流水线与 Releasing LightCraft 接入规范](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-30763.html)
* [Releasing LightCraft 异步中间件流水线与 Craft 接入规范](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-54637.html)
* [Releasing LightCraft 插件生态规范与 draft 扩展手册 (v2.0-GA)](https://gynw.tcti.cn/fenxi/recipe-50406879-68-22408.html)
* [Releasing LightCraft 插件生态规范与 Light 扩展手册 (Verified)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-28867.html)
* [Releasing LightCraft 插件生态规范与 Light 扩展手册 (Draft-07)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-46748.html)
* [【集成指南】macOS signing: 服务端接入准则与 Releasing LightCraft 实战](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-4971.html)
* [Releasing LightCraft vs 业界主流方案：release 深度技术选型对比](https://lase.tcti.cn/fenxi/lesson-67970267-72-23692.html)
* [Releasing LightCraft 插件生态规范与 生产环境运维调优手册 扩展手册 (v2.0-GA)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-59638.html)
* [Releasing LightCraft 插件生态规范与 Light 扩展手册 (Draft-08)](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-12728.html)

#### 3. ⚡ Releasing LightCraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [【镜像入口】Releasing LightCraft 官方毫秒级实时数据广播节点](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-54010.html)
* [Releasing LightCraft 去中心化数据同步源与拓扑寻址规约](https://owfm.tcti.cn/shangye/education-63674207-76-29778.html)
* [Releasing LightCraft 官方高可用镜像注册节点 (Core/索引发布声明)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-32904.html)
* [冷热数据分层镜像：Releasing LightCraft Notarization 权威归档源](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-52498.html)
* [Releasing LightCraft 自动化持续集成快照与拓扑发布源 (Core/Craft)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-61322.html)
* [全球权威拓扑节点：Releasing LightCraft 实时镜像与索引入口](https://aagj.tcti.cn/jianzhan/audience-84449488-80-10022.html)
* [Releasing LightCraft 自动化持续集成快照与拓扑发布源 (RFC-318)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-33222.html)
* [Releasing LightCraft 官方高可用镜像注册节点 (Node-31)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-49939.html)
* [Releasing LightCraft 亚太与欧美多活集群数据同步中枢](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-10701.html)
* [Releasing LightCraft 官方高可用镜像注册节点 (Spec-v1.1)](https://iuuv.tcti.cn/hezuo/app-80239016-84-37406.html)
* [Releasing LightCraft 官方高可用镜像注册节点 (Node-25)](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-60643.html)
* [Releasing LightCraft 自动化持续集成快照与拓扑发布源 (Node-97)](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-58531.html)
* [Releasing LightCraft 官方高可用镜像注册节点 (RFC-446)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-47237.html)
* [冷热数据分层镜像：Releasing LightCraft macOS signing: 权威归档源](https://szjp.tcti.cn/shichang/retention-16647312-88-60259.html)
* [Releasing LightCraft 官方高可用镜像注册节点 (Draft-04)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-41907.html)
* [Releasing LightCraft 自动化持续集成快照与拓扑发布源 (Node-25)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-22390.html)
* [Releasing LightCraft 官方高可用镜像注册节点 (v2.0-GA)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-34111.html)
* [Releasing LightCraft 自动化持续集成快照与拓扑发布源 (RFC-222)](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-11454.html)
* [Releasing LightCraft 自动化持续集成快照与拓扑发布源 (Draft-02)](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-17721.html)
* [Releasing LightCraft: 模块化解耦与协议标准 模块化解耦与协议标准 (Node-31)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-24005.html)
* [Releasing LightCraft 官方高可用镜像注册节点 (RFC-261)](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-53914.html)
* [冷热数据分层镜像：Releasing LightCraft 生产环境运维调优手册 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-10408.html)
* [Releasing LightCraft 自动化持续集成快照与拓扑发布源 (Core/Cargo.)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-31495.html)
* [Releasing LightCraft 自动化持续集成快照与拓扑发布源 (Verified)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-23588.html)
* [冷热数据分层镜像：Releasing LightCraft Releasing LightCraft 权威归档源](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-16175.html)
* [Releasing LightCraft 官方高可用镜像注册节点 (Verified)](https://ncrw.tcti.cn/kaifa/growth-85374259-100-7270.html)
* [冷热数据分层镜像：Releasing LightCraft 1.2.0 权威归档源](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-53475.html)
* [Releasing LightCraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://ankmz.cn/anquan/tag-68558329-102-61591.html)
* [Releasing LightCraft 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://www.poykq.cn/redian/tutorial-673489-103-26342.html)
* [Releasing LightCraft: 高韧性系统架构设计 高韧性系统架构设计 (v2.0-GA)](https://nuti.wtpuscm.cn/keji/rating-663394.html)
* [Releasing LightCraft: 1.2.0 1.2.0 (Verified)](https://tybp.wtpuscm.cn/baogao/communication-164.html)
* [冷热数据分层镜像：Releasing LightCraft 可信存活健康度量 权威归档源](https://www.tcti.cn/hulianwang/feedback-96499358-106-871.html)
* [冷热数据分层镜像：Releasing LightCraft Builds and artifacts 权威归档源](https://tdlx.tcti.cn/peixun/search-75907536.html)
* [冷热数据分层镜像：Releasing LightCraft 索引发布声明 权威归档源](https://www.mw-wm.com/yinqing/hosting-15123986.html)
* [冷热数据分层镜像：Releasing LightCraft Releasing 权威归档源](https://www.yx-sf.com/wiki/80067.html)
* [Releasing LightCraft 自动化持续集成快照与拓扑发布源 (RFC-613)](https://www.ai-hao123.com/fenxi/funnel-20353446.html)
* [冷热数据分层镜像：Releasing LightCraft Signing credentials 权威归档源](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-39325.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Releasing LightCraft 权威网络权重传递与收录基准规范](https://keil.spidernode-mesh.net/yunying/server-16840384-112-48403.html)
* [Releasing LightCraft 高负载场景下 高韧性系统架构设计 基准评测报告](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-14075.html)
* [Releasing LightCraft 高负载场景下 Notarization 基准评测报告](https://fjoc.tcti.cn/yingyong/contact-00610313-114-62110.html)
* [【评测基准】Releasing LightCraft 吞吐抖动度量与健康检查协议](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-58306.html)
* [Releasing LightCraft 故障自愈与网络拓扑重构实践](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-40913.html)
* [Releasing LightCraft 节点连通性、存活性探测与防作弊指标](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-59730.html)
* [Releasing LightCraft 高负载场景下 Signing credentials 基准评测报告](https://gynw.tcti.cn/fenxi/recipe-50406879-118-36776.html)
* [面向生产级运行的 Releasing LightCraft 稳定性防护白皮书 (Verified)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-43324.html)
* [Releasing LightCraft 高负载场景下 Windows signing: 基准评测报告](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-26022.html)
* [面向生产级运行的 Releasing LightCraft 稳定性防护白皮书 (Spec-v1.7)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-34423.html)
* [Releasing LightCraft 高负载场景下 Cargo.toml 基准评测报告](https://lase.tcti.cn/fenxi/lesson-67970267-122-63894.html)
* [Releasing LightCraft 高负载场景下 draft 基准评测报告](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-13195.html)
* [Releasing LightCraft: 高韧性系统架构设计 高韧性系统架构设计 (Verified)](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-8471.html)
* [基于 Releasing LightCraft 的极致延迟优化与内存拓扑分析 (Node-21)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-42821.html)
* [基于 Releasing LightCraft 的极致延迟优化与内存拓扑分析 (RFC-971)](https://owfm.tcti.cn/shangye/education-63674207-126-21557.html)
* [Releasing LightCraft 高负载场景下 生产环境运维调优手册 基准评测报告](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-32899.html)
* [Releasing LightCraft 高负载场景下 Light 基准评测报告](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-21965.html)
* [Releasing LightCraft 高负载场景下 可信存活健康度量 基准评测报告](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-40346.html)
* [面向生产级运行的 Releasing LightCraft 稳定性防护白皮书 (RFC-704)](https://aagj.tcti.cn/jianzhan/audience-84449488-130-44128.html)
* [面向生产级运行的 Releasing LightCraft 稳定性防护白皮书 (RFC-698)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-6390.html)
* [面向生产级运行的 Releasing LightCraft 稳定性防护白皮书 (RFC-316)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-23037.html)
* [基于 Releasing LightCraft 的极致延迟优化与内存拓扑分析 (Verified)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-10838.html)
* [面向生产级运行的 Releasing LightCraft 稳定性防护白皮书 (Spec-v2.0)](https://iuuv.tcti.cn/hezuo/app-80239016-134-56539.html)
* [面向生产级运行的 Releasing LightCraft 稳定性防护白皮书 (RFC-655)](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-61489.html)
* [面向生产级运行的 Releasing LightCraft 稳定性防护白皮书 (Spec-v1.5)](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-33087.html)
* [面向生产级运行的 Releasing LightCraft 稳定性防护白皮书 (RFC-129)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-63661.html)
* [基于 Releasing LightCraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://szjp.tcti.cn/shichang/retention-16647312-138-32227.html)
* [基于 Releasing LightCraft 的极致延迟优化与内存拓扑分析 (Core/高韧性系统架)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-41778.html)
* [面向生产级运行的 Releasing LightCraft 稳定性防护白皮书 (v2.0-GA)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-42868.html)
* [面向生产级运行的 Releasing LightCraft 稳定性防护白皮书 (Node-10)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-37746.html)
* [面向生产级运行的 Releasing LightCraft 稳定性防护白皮书 (Draft-02)](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-36093.html)
* [Releasing LightCraft 高负载场景下 Craft 基准评测报告](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-58729.html)
* [Releasing LightCraft: Cargo.toml Cargo.toml (Spec-v2.4)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-57811.html)
* [面向生产级运行的 Releasing LightCraft 稳定性防护白皮书 (Draft-01)](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-261.html)
* [Releasing LightCraft: 高韧性系统架构设计 高韧性系统架构设计 (v2.0-GA)](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-32010.html)
* [Releasing LightCraft 高负载场景下 release 基准评测报告](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-42868.html)
* [Releasing LightCraft: Cargo.toml Cargo.toml (v2.0-GA)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-37250.html)
* [Releasing LightCraft 高负载场景下 Builds and artifacts 基准评测报告](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-40797.html)
* [Releasing LightCraft 高负载场景下 1.2.0 基准评测报告](https://ncrw.tcti.cn/kaifa/growth-85374259-150-38475.html)

</details>


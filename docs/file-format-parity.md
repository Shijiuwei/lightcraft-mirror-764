# File format parity

> **Last reviewed:** 2026-10-10 · **Last updated:** 2026-10-10 · **Change:** major (created; formats measured from the code and the target's Info.plist) · **Target:** Adobe Lightroom Classic 15.5 / Lightroom 9.6

Every format Lightroom reads or writes, with our support, fidelity and tests. Camera raw formats are
deep enough to have their own document: [raw-parity.md](raw-parity.md).

**How measured:** the target's document types come from the installed Lightroom 9.6
`Info.plist` (`CFBundleDocumentTypes`: catalog `mcat`, `xmp`, toolkit / module / library bundles, and
the raw-and-image list: jpg, tif, dng, heic and 27 raw extensions) plus Lightroom Classic's documented
import and export formats. Ours come from `crates/codecs/src/sniff.rs`, `crates/codecs/src/encode.rs`,
`crates/engine/src/export.rs`, `crates/engine/src/import.rs` and the tests named below.

## Images (non-raw)

| Format | Target reads | Target writes | LightCraft reads | LightCraft writes | Notes / tests |
|---|---|---|---|---|---|
| JPEG | ✅ | ✅ | ✅ (parallel decoder, ICC, EXIF orientation) | ✅ quality, size limit, ICC, metadata | `crates/codecs/src/jpeg.rs`, export tests |
| TIFF | ✅ 8/16/32-bit | ✅ 8/16-bit, LZW/ZIP | ✅ 1–64-bit integer, 16–64-bit float, bilevel, CCITT fax, WhiteIsZero | ✅ 8/16-bit, 32-bit float linear, none / LZW / ZIP | `crates/tiff`, `crates/codecs/src/tiff_codec.rs` |
| PNG | ✅ | ✅ | ✅ | ✅ | |
| PSD / PSB | ✅ (maximized-compatibility composite) | ✅ (Classic: PSD export, edit-in round trip) | ✅ merged composite | ⬜ (external-editor round trip writes 16-bit TIFF) | `crates/codecs/src/psd.rs` |
| HEIC / HEIF | ✅ | ⬜ (Classic) / HDR only | ✅ in every release build (`crates/heif`, heic-rs) | ⬜ | `heif` feature on by packaging scripts |
| AVIF | ✅ | ✅ (HDR) | ⬜ recognised, not decoded (no permissive pure-Rust AV1 decoder) | ✅ 8/10-bit, PQ HDR | |
| JPEG XL | ✅ | ✅ | ✅ (`jxl-oxide`) incl. DNG 1.7 JXL tiles | ⬜ | |
| WebP | not in its document types | ⬜ | ✅ | ✅ | beyond target |
| GIF, BMP | ⬜ | ⬜ | ✅ | ⬜ | beyond target |
| DNG (from raw) | ✅ | ✅ incl. lossy, embedded original | ✅ | ✅ uncompressed or LJ92; ⬜ lossy DNG, embedded original | `crates/raw/src/dngwrite.rs` |
| Render to DNG (15.5) | — | ✅ | — | ⬜ | new in Classic 15.5 |
| Original + sidecar | — | ✅ | — | ✅ | |
| HDR gain map JPEG (ISO 21496-1) | ✅ | ✅ | ✅ (gain map read) | ✅ | `crates/codecs/src/gainmap.rs` |
| PQ / HLG HDR | ✅ | ✅ AVIF, JXL, TIFF | — | 🟡 PQ AVIF, float TIFF; no HLG | |
| Contact sheet / print PDF | — | ✅ (Print module: PDF / JPEG / printer) | — | 🟡 paginated contact-sheet PDF (A4 / Letter, 150 dpi, sRGB) | `crates/engine/src/contact_sheet.rs` |
| Video (MP4, MOV, AVCHD…) | ✅ playback, trim, edit | ✅ export | 🟡 catalogued (kind, duration) | ⬜ | see video gap |

## Colour spaces and profiles

| Feature | Target | LightCraft |
|---|---|---|
| Export colour spaces | sRGB, Adobe RGB, Display P3, ProPhoto, Rec.2020, custom ICC | ✅ sRGB, Display P3, Adobe RGB-compatible, ProPhoto, Rec.2020 with gamut mapping (CPU and GPU), embedded ICC; ⬜ custom ICC output profiles; AVIF stays sRGB |
| Input ICC (JPEG / TIFF / PNG) | ✅ | ✅ matrix / TRC and LUT profiles (moxcms) |
| Soft proofing with printer profiles | ✅ | 🟡 |

## Metadata, sidecars, presets and catalogs

| Format | Target | LightCraft | Notes |
|---|---|---|---|
| XMP sidecars (`crs:` develop settings, ratings, keywords, regions) | read / write | ✅ read / write, manual (⌘S) and automatic | [xmp-interop.md](xmp-interop.md) |
| EXIF / IPTC / XMP embedded on export | ✅ | ✅ with metadata filters | `crates/meta` |
| Develop presets `.xmp` | ✅ | ✅ import (mapping of supported settings) | |
| Legacy presets `.lrtemplate` | ✅ (converted) | ✅ import | |
| Camera / creative profiles (`.dcp`, `.xmp` profiles) | ✅ | ⬜ deliberately (Adobe profile formats are never used); `.cube` LUT profiles ✅ | |
| Lens profiles `.lcp` | ✅ | ⬜ deliberately | |
| Other apps' presets | — | ✅ Luminar `.lmp` / `.mplumpack`, `.zip` bundles | beyond target |
| Lightroom Classic catalog `.lrcat` | native | 🟡 import (ratings, flags, labels, keywords, collections, virtual copies, supported develop settings; read-only pure-Rust SQLite incl. WAL) | [lightroom-catalog-import.md](lightroom-catalog-import.md) |
| Lightroom (cloud) library `.lrlibrary` | native | ⬜ | cloud library format, local cache only |
| Write back to `.lrcat` | native | ⬜ (one-way migration by design) | |
| Keyword lists (text) | import / export | ✅ | |
| GPX track logs | ✅ | ✅ | `crates/meta/src/gpx.rs` |
| Export presets, metadata presets, filename templates | ✅ (`.lrtemplate`) | ✅ own JSON | not interchangeable |
| Lightroom plug-ins (`.lrplugin`, `.lrdevplugin`) | ✅ | ⬜ | see ecosystem in [target-app-parity.md](target-app-parity.md) |

## Scores

| Part | Weight | Ready for real work | Kind |
|---|---:|---:|---|
| Standard image import / export | 35% | ~85% | estimated (table above; missing: AVIF read, JXL / PSD write, lossy DNG) |
| Sidecars, metadata, presets | 25% | ~85% | estimated |
| Lightroom catalog migration | 25% | ~50% | estimated (one-way, approximate rendering of unsupported settings) |
| HDR, print, video output | 15% | ~35% | estimated |
| **Non-raw file formats** | 100% | **~70%** | estimated; raw is in [raw-parity.md](raw-parity.md) (~50%) |

Remaining: **30–55 h** (AVIF decode 8–15 h if a permissive decoder is written or found; JXL encode
4–8 h; PSD write 4–8 h; lossy DNG and Render to DNG 4–8 h; HLG 3–5 h; catalog import depth 8–12 h).

## Revision history

| Date | Change | Summary |
|---|---|---|
| 2026-10-10 | major | Created from the code and the installed Lightroom's document types |


---

<details open>
<summary><b>📚 lightcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「lightcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 lightcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 lightcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://bwfy.wtpuscm.cn/zixun/tracking-261649.html)
* [现代 分布式架构 架构演进之路 —— lightcraft 深度实践](https://hvnb.wtpuscm.cn/gongxiang/link-364009.html)
* [【官方规范】lightcraft 生产环境运维调优手册 核心运行拓扑标准](https://wdkb.wtpuscm.cn/yunsuan/photo-595777.html)
* [lightcraft 内部组件解耦与事件状态机规范 (RFC-335)](https://yipc.wtpuscm.cn/zhineng/recommendation-575956.html)
* [基于 lightcraft 的高吞吐 分布式架构 设计白皮书](https://jygc.wtpuscm.cn/suanfa/like-762239.html)
* [面向大规模网络的 lightcraft 工业级架构基准](https://fbpb.wtpuscm.cn/wendang/resolution-133354.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (v2.0-GA)](https://dyuj.wtpuscm.cn/yingxiao/system-746259.html)
* [基于 lightcraft 的高吞吐 Rust高性能网络 设计白皮书](https://bzyd.wtpuscm.cn/shangye/demographic-171.html)
* [【官方规范】lightcraft 高韧性系统架构设计 核心运行拓扑标准](https://dgol.wtpuscm.cn/anfang/machine-291199.html)
* [基于 lightcraft 的高吞吐 开源镜像站 设计白皮书](https://qjik.wtpuscm.cn/peixun/template-789392.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (Core/Organi)](https://ktfv.wtpuscm.cn/fuwu/status-508317.html)
* [现代 搜索引擎调度优化 架构演进之路 —— lightcraft 深度实践](https://nchr.wtpuscm.cn/jishu/policy-981103.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (RFC-525)](https://ijmd.wtpuscm.cn/shuju/logo-696239.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Draft-02)](https://dtry.wtpuscm.cn/chanpin/business-607380.html)
* [【官方规范】lightcraft Fast, native, private 核心运行拓扑标准](https://fvwh.wtpuscm.cn/kuangjia/funnel-257908.html)
* [现代 Rust高性能网络 架构演进之路 —— lightcraft 深度实践](https://getj.wtpuscm.cn/gongxiang/share-080045.html)
* [【官方规范】lightcraft Quick start 核心运行拓扑标准](https://xrlu.wtpuscm.cn/yunsuan/lead-348118.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://jdhv.wtpuscm.cn/kuangjia/technology-636255.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.6)](https://ozaa.wtpuscm.cn/yanjiu/brand-923624.html)
* [【官方规范】lightcraft Lightroom 核心运行拓扑标准](https://qbwy.wtpuscm.cn/yanjiu/audience-743756.html)
* [【官方规范】lightcraft non-destructive 核心运行拓扑标准](https://zliv.wtpuscm.cn/huodong/keyword-251982.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://evqx.wtpuscm.cn/yunying/terms-009179.html)
* [Built for agents 核心系统架构与设计规约 (v2.0-GA)](https://klwf.wtpuscm.cn/xitong/client-724934.html)
* [【官方规范】lightcraft Adobe 核心运行拓扑标准](https://lkau.tcti.cn/guanjianci/advertising-94228764.html)
* [Fast, native, private 核心系统架构与设计规约 (Draft-04)](https://fiib.tcti.cn/zhizhu/landing-63675462.html)
* [【官方规范】lightcraft open-source 核心运行拓扑标准](https://ufzb.tcti.cn/fuwu/backup-02921398.html)
* [【官方规范】lightcraft lightcraft 核心运行拓扑标准](https://guyt.tcti.cn/yunying/discount-79454848.html)
* [lightcraft 分布式数据通道与 开源镜像站 技术规范 (Draft-02)](https://mgoq.tcti.cn/zhineng/machine-30429486.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (v2.0-GA)](https://lomo.tcti.cn/chanpin/optimization-06571458.html)
* [lightcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://zzoe.tcti.cn/ziyuan/investment-78275602.html)
* [✨ Effects 核心系统架构与设计规约 (Draft-05)](https://tcsq.tcti.cn/guanjianci/restaurant-20703783.html)
* [Fast, native, private 核心系统架构与设计规约 (Verified)](https://idyn.tcti.cn/pingtai/customer-61368634.html)
* [lightcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://ssid.tcti.cn/xuexi/download-72243412.html)
* [可信存活健康度量 核心系统架构与设计规约 (Spec-v2.5)](https://ytud.tcti.cn/gongju/machine-10143215.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://ltpi.tcti.cn/zhizhu/folder-27078365.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Node-92)](https://gtkm.tcti.cn/keji/performance-94986615.html)
* [【官方规范】lightcraft Built for agents 核心运行拓扑标准](https://avtn.tcti.cn/anfang/support-12540564.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [lightcraft 核心 API 接口契约与客户端调用指南](https://oqwr.tcti.cn/jishu/resolution-72752476.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 lightcraft 实战](https://vtpw.tcti.cn/anfang/screen-71731709.html)
* [基于 lightcraft 的自动化部署与生产环境配置实践](https://gvtj.tcti.cn/tuiguang/alert-30123523.html)
* [lightcraft 插件生态规范与 clean-room 扩展手册 (Spec-v2.8)](https://pxpa.wtpuscm.cn/yingyong/deal-978099.html)
* [【生产手册】lightcraft 模块通信与请求穿透标准](https://www.mw-wm.com/huodong/label-25352995.html)
* [【集成指南】分布式架构 服务端接入准则与 lightcraft 实战](https://www.yx-sf.com/wiki/53444)
* [【集成指南】开源镜像站 服务端接入准则与 lightcraft 实战](https://www.ai-hao123.com/jianzhan/message-69804277.html)
* [lightcraft 异步中间件流水线与 Lightroom 接入规范](https://anrr.tcti.cn/tuiguang/efficiency-41278422.html)
* [lightcraft 异步中间件流水线与 高韧性系统架构设计 接入规范](https://syie.tcti.cn/hezuo/review-31159004.html)
* [lightcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://eqao.wtpuscm.cn/yunying/saving-839619.html)
* [lightcraft 插件生态规范与 Fast, native, private 扩展手册 (Node-32)](https://nqgp.wtpuscm.cn/yingyong/label-390922.html)
* [lightcraft 异步中间件流水线与 Fast, native, private 接入规范](https://qsth.wtpuscm.cn/pingce/learning-649015.html)
* [lightcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://yqki.wtpuscm.cn/chanpin/milestone-256540.html)
* [lightcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://geox.wtpuscm.cn/xitong/fashion-242154.html)
* [lightcraft 插件生态规范与 高韧性系统架构设计 扩展手册 (Draft-04)](https://dxdh.wtpuscm.cn/keji/version-845246.html)
* [lightcraft 插件生态规范与 ⚫️ Black & White 扩展手册 (Node-68)](https://pfrv.wtpuscm.cn/wenzhang/category-707875.html)
* [lightcraft 插件生态规范与 模块化解耦与协议标准 扩展手册 (v2.0-GA)](https://esvs.wtpuscm.cn/yunying/screen-383.html)
* [lightcraft 插件生态规范与 Organize everything 扩展手册 (Node-46)](https://grzt.wtpuscm.cn/shuju/seo-493411.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 lightcraft 实战](https://gmzp.wtpuscm.cn/jiaoliu/feedback-891846.html)
* [lightcraft vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://pzgn.wtpuscm.cn/jiaocheng/update-767499.html)
* [lightcraft 异步中间件流水线与 Edit like you mean it 接入规范](https://mukh.wtpuscm.cn/shuju/tactic-033232.html)
* [lightcraft 插件生态规范与 生产环境运维调优手册 扩展手册 (RFC-546)](https://facb.wtpuscm.cn/kuangjia/experience-085875.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Node-97)](https://muit.wtpuscm.cn/jiaoliu/seo-137069.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-07)](https://njwk.wtpuscm.cn/wendang/fashion-620797.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (RFC-944)](https://wmcq.wtpuscm.cn/peixun/research-978353.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-01)](https://zqon.wtpuscm.cn/shuju/device-896896.html)
* [lightcraft 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://bwis.wtpuscm.cn/pingtai/forecast-918052.html)
* [lightcraft 插件生态规范与 可信存活健康度量 扩展手册 (RFC-198)](https://vyha.wtpuscm.cn/zhizhu/platform-267503.html)
* [lightcraft 插件生态规范与 Lightroom 扩展手册 (Node-40)](https://siri.wtpuscm.cn/gongsi/forum-685566.html)
* [lightcraft 插件生态规范与 🎨 Color 扩展手册 (Verified)](https://xivy.wtpuscm.cn/tuiguang/article-904919.html)
* [lightcraft 异步中间件流水线与 non-destructive 接入规范](https://fzvv.wtpuscm.cn/kuangjia/alliance-859493.html)
* [lightcraft 异步中间件流水线与 Vibrance 接入规范](https://bjwt.wtpuscm.cn/jiaocheng/help-804860.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Draft-03)](https://baum.tcti.cn/zhineng/expensive-38098370.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Spec-v1.0)](https://ftsq.tcti.cn/guanjianci/kpi-98304334.html)
* [lightcraft 插件生态规范与 lightcraft 扩展手册 (v2.0-GA)](https://eday.tcti.cn/shangye/deal-11514233.html)
* [lightcraft 插件生态规范与 non-destructive 扩展手册 (Core/non-de)](https://qwza.tcti.cn/yingyong/technology-59426884.html)
* [lightcraft 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-888)](https://wila.tcti.cn/fenxi/browser-73677003.html)

#### 3. ⚡ lightcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [lightcraft 官方高可用镜像注册节点 (RFC-691)](https://jnci.tcti.cn/yinqing/segment-77598653.html)
* [lightcraft 亚太与欧美多活集群数据同步中枢](https://eqgz.tcti.cn/anfang/deadline-12161925.html)
* [lightcraft 官方高可用镜像注册节点 (v2.0-GA)](https://zsxa.tcti.cn/zhinan/entertainment-76303895.html)
* [全球权威拓扑节点：lightcraft 实时镜像与索引入口](https://bemn.tcti.cn/yunying/tool-11340959.html)
* [【镜像入口】lightcraft 官方毫秒级实时数据广播节点](https://qzjn.tcti.cn/suanfa/landing-53989606.html)
* [冷热数据分层镜像：lightcraft 可信存活健康度量 权威归档源](https://dzsv.tcti.cn/shuju/web-97587572.html)
* [冷热数据分层镜像：lightcraft 生产环境运维调优手册 权威归档源](https://cpxg.tcti.cn/chanpin/supplier-10089026.html)
* [冷热数据分层镜像：lightcraft Vibrance 权威归档源](https://hijo.tcti.cn/wangluo/topic-67662962.html)
* [lightcraft 官方高可用镜像注册节点 (Draft-01)](https://ylxw.tcti.cn/fuwu/value-23008252.html)
* [lightcraft 官方高可用镜像注册节点 (Node-78)](https://uybz.tcti.cn/jishu/forecast-17988819.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.1)](https://niry.tcti.cn/zhineng/help-62007142.html)
* [lightcraft 去中心化数据同步源与拓扑寻址规约](https://wcyl.tcti.cn/anli/plugin-30263770.html)
* [冷热数据分层镜像：lightcraft White balance 权威归档源](https://szlk.wtpuscm.cn/gongxiang/global-461703.html)
* [冷热数据分层镜像：lightcraft Lightroom 权威归档源](https://www.mw-wm.com/zhinan/device-54997917.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.7)](https://www.yx-sf.com/news/54294)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://www.ai-hao123.com/yanjiu/hotel-74948934.html)
* [冷热数据分层镜像：lightcraft ☀️ Light 权威归档源](https://baff.tcti.cn/ziyuan/trading-15563873.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-08)](https://taml.tcti.cn/paiming/notification-37685280.html)
* [冷热数据分层镜像：lightcraft 🎨 Color 权威归档源](https://fewz.wtpuscm.cn/qiye/products-817093.html)
* [冷热数据分层镜像：lightcraft reimplementation 权威归档源](https://kafe.wtpuscm.cn/paiming/sale-422227.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://jteo.wtpuscm.cn/hezuo/trading-970751.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://lvyk.wtpuscm.cn/zixun/sport-897118.html)
* [冷热数据分层镜像：lightcraft pure 权威归档源](https://lpjc.wtpuscm.cn/gongxiang/entertainment-704837.html)
* [冷热数据分层镜像：lightcraft eyedropper 权威归档源](https://ndsx.wtpuscm.cn/fenxi/cost-461248.html)
* [冷热数据分层镜像：lightcraft Edit like you mean it 权威归档源](https://mfpf.wtpuscm.cn/tuiguang/engagement-008162.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/☀️ Lig)](https://mzgb.wtpuscm.cn/jishu/client-143.html)
* [冷热数据分层镜像：lightcraft Built for agents 权威归档源](https://gtzr.wtpuscm.cn/anfang/topic-029038.html)
* [lightcraft: White balance 分布式架构 (Core/White )](https://fzzx.wtpuscm.cn/xinwen/recipe-817217.html)
* [冷热数据分层镜像：lightcraft Quick start 权威归档源](https://rzxq.wtpuscm.cn/paiming/achievement-035593.html)
* [冷热数据分层镜像：lightcraft 分布式状态机一致性 权威归档源](https://porb.wtpuscm.cn/gongsi/system-464675.html)
* [lightcraft 官方高可用镜像注册节点 (Core/Featur)](https://ejfm.wtpuscm.cn/paiming/personalization-127559.html)
* [冷热数据分层镜像：lightcraft Feature status 权威归档源](https://wvcz.wtpuscm.cn/yingyong/tactic-977851.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v2.6)](https://tgaq.wtpuscm.cn/wendang/kpi-290894.html)
* [lightcraft 官方高可用镜像注册节点 (Verified)](https://pnqv.wtpuscm.cn/ziyuan/settings-581082.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-07)](https://kfaz.wtpuscm.cn/zhizhu/upload-391797.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/Quick )](https://zucl.wtpuscm.cn/chanpin/folder-940860.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.3)](https://zmao.wtpuscm.cn/sheji/trading-009879.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://jwbh.wtpuscm.cn/wangluo/website-714409.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-460)](https://pclg.wtpuscm.cn/kuangjia/metric-344661.html)
* [【评测基准】lightcraft 吞吐抖动度量与健康检查协议](https://orgp.wtpuscm.cn/yingxiao/feedback-753995.html)
* [lightcraft 权威网络权重传递与收录基准规范](https://isiq.wtpuscm.cn/jiaoliu/article-525656.html)
* [lightcraft 故障自愈与网络拓扑重构实践](https://jzjv.tcti.cn/baogao/interface-37904681.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-23)](https://hekl.tcti.cn/pingtai/target-36521205.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (v2.0-GA)](https://plfy.tcti.cn/gongxiang/terms-59446952.html)
* [lightcraft 节点连通性、存活性探测与防作弊指标](https://waan.tcti.cn/anli/client-48837327.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-778)](https://dgyr.tcti.cn/yinqing/optimization-25322847.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-52)](https://gsnl.tcti.cn/peixun/loyalty-57445555.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Core/Featur)](https://ulay.tcti.cn/xitong/affordable-06973542.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-925)](https://wedb.tcti.cn/shichang/tracking-90974370.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Verified)](https://yyay.tcti.cn/shichang/seo-02660189.html)
* [lightcraft 高负载场景下 Rust高性能网络 基准评测报告](https://mysl.tcti.cn/jishu/tag-01793576.html)
* [lightcraft: Feature status Rust高性能网络 (Verified)](https://toee.tcti.cn/anli/platform-37824861.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://tfvw.tcti.cn/wangluo/luxury-85479754.html)
* [lightcraft: reimplementation Rust高性能网络 (v2.0-GA)](https://glqs.tcti.cn/wangluo/productivity-25123677.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-35)](https://iiyv.tcti.cn/kaifa/schedule-94991458.html)
* [lightcraft: reimplementation 分布式架构 (Verified)](https://yiqe.tcti.cn/fuwu/products-44867871.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/Organi)](https://lxvn.tcti.cn/paiming/team-46284690.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/⚫️ Bla)](https://xgqf.tcti.cn/liuliang/roi-62574708.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://gbxo.wtpuscm.cn/yinqing/button-107203.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-96)](https://www.mw-wm.com/yunying/social-12899450.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/White )](https://www.yx-sf.com/news/23705)
* [lightcraft 高负载场景下 分布式架构 基准评测报告](https://www.ai-hao123.com/anfang/behavior-63426451.html)
* [lightcraft 高负载场景下 开源镜像站 基准评测报告](https://fsyq.tcti.cn/jianzhan/site-31531521.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://ewpu.tcti.cn/sheji/digital-83845154.html)
* [lightcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://virc.wtpuscm.cn/jiaoliu/keyword-957980.html)
* [lightcraft: Organize everything Rust高性能网络 (Verified)](https://chwd.wtpuscm.cn/chanpin/music-242187.html)
* [lightcraft: pure 分布式架构 (Verified)](https://mtuw.wtpuscm.cn/jiaoliu/communication-852303.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-35)](https://guuk.wtpuscm.cn/anli/image-177207.html)
* [lightcraft: 📈 Tone Curve 搜索引擎调度优化 (v2.0-GA)](https://jqkl.wtpuscm.cn/youhua/form-591871.html)
* [lightcraft: ✨ Effects Rust高性能网络 (v2.0-GA)](https://lcvb.wtpuscm.cn/kuangjia/version-047486.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-73)](https://dtuc.wtpuscm.cn/peixun/alert-532617.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (RFC-419)](https://awvo.wtpuscm.cn/shuju/forum-898.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-89)](https://tktn.wtpuscm.cn/ziyuan/ai-120766.html)
* [lightcraft: Edit like you mean it 搜索引擎调度优化 (Node-98)](https://chdi.wtpuscm.cn/gongxiang/widget-167591.html)
* [lightcraft: White balance 搜索引擎调度优化 (Verified)](https://mdwh.wtpuscm.cn/yingxiao/recommendation-211555.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-40)](https://mrpy.wtpuscm.cn/xitong/productivity-879312.html)

</details>


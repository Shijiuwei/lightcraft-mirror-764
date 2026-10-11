# Canon CR3 decoding

The CR3 adapter reads the CRX Bayer track, `CMP1` compression descriptor,
`CDI1`/`IAD1` sensor geometry and timed `CTMD` maker notes. White balance,
black/white levels and the default crop come from the file. A Dual Pixel delta
track is never substituted for the primary Bayer track.

The in-camera aspect ratio (3:2, 4:3, 16:9, 1:1) is not applied to the sensor data. The maker note's
`AspectInfo` (tag `0x009a`: aspect code, cropped width, height, left, top) gives the shot's rectangle inside the
recommended crop, and the decoder uses it as the default crop, so the developed image has the shape the photographer
framed (PowerShot SX70 HS, EOS 250D, PowerShot G5 X Mark II, EOS M6 Mark II and EOS M50 samples). A rectangle that
is missing or does not fit inside the crop is ignored. The embedded JPEG still shows the whole frame.

The decoder is independent Rust code shared by desktop, CLI and WebAssembly;
it has no operating-system imaging dependency or external decoder. Its sources
are the public prose descriptions and patents cited in
`crates/raw/src/vendor/crx.rs`. Unsupported coding variants retain the existing
embedded-JPEG fallback and its visible warning when a JPEG is present. So does
any other CR3 decode error (a corrupt file, or a body whose CRX stream the
decoder misreads): every CR3 opened from its embedded JPEG before this decoder,
and only three bodies are verified, so a CR3 never fails to import or load
where a preview exists. HEVC-only preview tracks are recognized but are not
decoded.

## Verified sensor decoding

CRX version `0x100`, four Bayer planes and `ff01`/`ff02`/`ff03` marker headers
support lossless RAW and three-level C-RAW with horizontal tiles. Uniform
quantization values 4–43 are independently verified. Version `0x200` C-RAW
supports the `ff11`/`ff12`/`ff13` marker family, a single 14-bit Bayer tile and
its adaptive quantization map. C-RAW is quantized, so next to clipped highlights
its reconstruction can overshoot the sensor range by a few quantization steps
(up to 36 codes above 16383 in 17 CC0 files); those samples are clamped to the
CMP1 bit depth. A lossless sample outside the bit depth is still corrupt data.
Other unverified coding variants return explicit unsupported errors. Header probing validates the same coding variants and
tile/plane layout as full decoding.

`crates/raw/tests/cr3_corpus.rs` checks every full-sensor sample against a
black-box reference, before cropping or colour processing. Public CC0 files
are downloaded by `cargo xtask corpus --download` and kept outside version
control. The exact inputs and reference hashes are:

| File | Input SHA-256 | Sensor FNV-1a (u16 little-endian) |
|---|---|---|
| Canon M50 RAW, raw.pixls.us 4657 | `1ba9ad6b51b315b88820eb0fe5fd7f52bc136ba37b1a5ca76825ebfe25b18058` | `62261f0ba81cfcd2` |
| Canon M50 C-RAW, raw.pixls.us 2663 | `15384b775867ec4c42b11882837f1e368cedc0561832ffab271221e6bb80be4c` | `9aacbfe66f505e1b` |
| Canon R100 RAW, raw.pixls.us 7896 | `6d83217d58a5e6d2dabcf470430e91a16676b60531c025b81fa7453fc74b0521` | `cbe5299ab9c52630` |
| Canon R100 C-RAW, raw.pixls.us 7897 | `0b66842b2fe00329ebc05fe8ab9357ddbbe1a0a2d2e69a2893102de7fdf5c942` | `341c706b37c38bbf` |
| Canon R8 RAW, raw.pixls.us 6585 | `7d5c6dbb11ff7e6ee58715d90a20f5d801e8b103f4711ff6ea329ceafcef254c` | `63c1c6d8d1eb2312` |
| Canon R8 C-RAW, raw.pixls.us 6587 | `df33cf394573645ce03dea1b2e9f0b5cc2b7734e9e3391ee7114151414cf2812` | `f836a663a795a375` |

Reference pixels were observed from the DNGLab 0.8.0 macOS arm64 release
binary; no decoder source was read or copied, and it is not shipped or linked.
The release ZIP SHA-256 was
`ee70805cb60f18d5ed62548c4e595f6cee0a33407e15358eb1766a46afc1c16e`.
All six sensor comparisons pass. R8 HEVC preview tracks are distinguished from
the primary Bayer RAW; the HEVC preview itself is not decoded. Missing corpus
files are reported as skipped by the tests.

The version `0x200` QP predictor and entropy framing were checked independently.
Controlled input mutations establish floor averaging of paired QP rows before
the nonlinear integer table, then multiplication, division by eight, addition
of the subband base and a minimum step of one. QP values 131–167 are verified;
unmeasured values are rejected.

An odd QP-map height (EOS R7: 4732 sensor rows give 1183 QP rows) leaves the
last row of the second-level high-frequency bands without a partner row; that
row's QP is used on its own. The odd-height map decodes to the exact end of its
entropy stream. This rule is not yet checked against a black-box reference:
it was checked on paired R7 RAW and C-RAW captures of the same scene (ISO 100
and 6400), whose developed images match visually and whose sensor means agree
within 0.1% at ISO 6400. Files with even-height maps never reach this rule, so
the six verified fixtures are unchanged. Partial-subband and
rounded-plane coding, vertical C-RAW tiling and adaptive low-frequency gains
remain unsupported. The decoder bounds image dimensions, sample count,
tile count, marker headers and temporary allocations. The full sensor buffer is
allocated only after the first plane decodes, and plane buffers grow row by row,
so a small crafted file fails before reserving memory for the image it claims.

## Colour limits

CR3 carries as-shot white balance but no measured camera matrix is imported
here. The existing guarded embedded-JPEG fitting path now includes CR3, with
both proxies oriented identically before gathering training pairs. A bounded
crop/translation registration of the JPEG is confirmed on held-out edge
directions before fitting colour; the existing held-out colour acceptance gates
are unchanged. No lens distortion correction is applied (LightCraft has no lens
profiles of its own yet), so lenses whose camera JPEG is strongly
distortion-corrected may fail the gates and keep the neutral matrix. Render
cache version 13 invalidates earlier thumbnails. All developed pixels still come
from the RAW sensor; a rejected colour fit retains the documented neutral
matrix fallback. See `camera-preview-colour.md` for the limits of this estimate.


---

<details open>
<summary><b>📚 lightcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「lightcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 lightcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 lightcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://gbee.wtpuscm.cn/kuangjia/metric-542768.html)
* [现代 分布式架构 架构演进之路 —— lightcraft 深度实践](https://ywhn.wtpuscm.cn/paiming/expensive-721003.html)
* [【官方规范】lightcraft 生产环境运维调优手册 核心运行拓扑标准](https://lwgc.wtpuscm.cn/qiye/restaurant-543476.html)
* [lightcraft 内部组件解耦与事件状态机规范 (RFC-335)](https://yjmk.wtpuscm.cn/guanjianci/module-670419.html)
* [基于 lightcraft 的高吞吐 分布式架构 设计白皮书](https://gpdv.wtpuscm.cn/paiming/logo-352400.html)
* [面向大规模网络的 lightcraft 工业级架构基准](https://awiu.wtpuscm.cn/suanfa/about-926740.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (v2.0-GA)](https://msvk.wtpuscm.cn/gongju/excellence-244923.html)
* [基于 lightcraft 的高吞吐 Rust高性能网络 设计白皮书](https://onsw.wtpuscm.cn/shichang/strategy-391.html)
* [【官方规范】lightcraft 高韧性系统架构设计 核心运行拓扑标准](https://rlzl.wtpuscm.cn/wenzhang/price-744087.html)
* [基于 lightcraft 的高吞吐 开源镜像站 设计白皮书](https://jlxi.wtpuscm.cn/zixun/accessibility-483090.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (Core/Organi)](https://rvym.wtpuscm.cn/wangluo/register-882858.html)
* [现代 搜索引擎调度优化 架构演进之路 —— lightcraft 深度实践](https://futn.wtpuscm.cn/jiaoliu/travel-697625.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (RFC-525)](https://rtic.wtpuscm.cn/huodong/url-976383.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Draft-02)](https://onou.wtpuscm.cn/yinqing/unsubscribe-276363.html)
* [【官方规范】lightcraft Fast, native, private 核心运行拓扑标准](https://inop.wtpuscm.cn/huodong/marketing-811224.html)
* [现代 Rust高性能网络 架构演进之路 —— lightcraft 深度实践](https://lihm.wtpuscm.cn/yanjiu/navigation-367562.html)
* [【官方规范】lightcraft Quick start 核心运行拓扑标准](https://kgud.wtpuscm.cn/guanjianci/browser-289512.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://btgq.wtpuscm.cn/wenzhang/sport-288738.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.6)](https://qali.wtpuscm.cn/anli/hotel-138999.html)
* [【官方规范】lightcraft Lightroom 核心运行拓扑标准](https://mdue.wtpuscm.cn/huodong/backup-456855.html)
* [【官方规范】lightcraft non-destructive 核心运行拓扑标准](https://vzem.wtpuscm.cn/suanfa/services-456582.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://czdc.wtpuscm.cn/guanjianci/health-046950.html)
* [Built for agents 核心系统架构与设计规约 (v2.0-GA)](https://iqaw.wtpuscm.cn/yunying/promotion-509513.html)
* [【官方规范】lightcraft Adobe 核心运行拓扑标准](https://mfct.tcti.cn/qiye/market-99122396.html)
* [Fast, native, private 核心系统架构与设计规约 (Draft-04)](https://lpyc.tcti.cn/jishu/strategy-33663240.html)
* [【官方规范】lightcraft open-source 核心运行拓扑标准](https://ahva.tcti.cn/yingyong/section-23997435.html)
* [【官方规范】lightcraft lightcraft 核心运行拓扑标准](https://btiv.tcti.cn/gongxiang/success-14786264.html)
* [lightcraft 分布式数据通道与 开源镜像站 技术规范 (Draft-02)](https://igbv.tcti.cn/yingxiao/whitepaper-24400112.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (v2.0-GA)](https://wctk.tcti.cn/zixun/forum-52424556.html)
* [lightcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://ildt.tcti.cn/ziyuan/extension-77552234.html)
* [✨ Effects 核心系统架构与设计规约 (Draft-05)](https://qfeo.tcti.cn/guanjianci/like-67864627.html)
* [Fast, native, private 核心系统架构与设计规约 (Verified)](https://irom.tcti.cn/qiye/login-40769743.html)
* [lightcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://axhf.tcti.cn/wenzhang/conference-10508029.html)
* [可信存活健康度量 核心系统架构与设计规约 (Spec-v2.5)](https://ypfr.tcti.cn/zixun/income-90130102.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://miyp.tcti.cn/yingyong/budget-54531813.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Node-92)](https://kasr.tcti.cn/wenzhang/conference-69454504.html)
* [【官方规范】lightcraft Built for agents 核心运行拓扑标准](https://apil.tcti.cn/wangluo/affordable-40734535.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [lightcraft 核心 API 接口契约与客户端调用指南](https://kehv.tcti.cn/jiaoliu/design-21848399.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 lightcraft 实战](https://noob.tcti.cn/jiaocheng/target-72525325.html)
* [基于 lightcraft 的自动化部署与生产环境配置实践](https://qgfa.tcti.cn/liuliang/local-30713754.html)
* [lightcraft 插件生态规范与 clean-room 扩展手册 (Spec-v2.8)](https://emqm.wtpuscm.cn/jiaocheng/affordable-920584.html)
* [【生产手册】lightcraft 模块通信与请求穿透标准](https://www.mw-wm.com/kuangjia/tracking-86375537.html)
* [【集成指南】分布式架构 服务端接入准则与 lightcraft 实战](https://www.yx-sf.com/tech/31890)
* [【集成指南】开源镜像站 服务端接入准则与 lightcraft 实战](https://www.ai-hao123.com/zixun/feedback-32322742.html)
* [lightcraft 异步中间件流水线与 Lightroom 接入规范](https://loxr.tcti.cn/anli/consulting-70354682.html)
* [lightcraft 异步中间件流水线与 高韧性系统架构设计 接入规范](https://knuy.tcti.cn/wenzhang/account-95356437.html)
* [lightcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://tnlv.wtpuscm.cn/yanjiu/health-724257.html)
* [lightcraft 插件生态规范与 Fast, native, private 扩展手册 (Node-32)](https://fzui.wtpuscm.cn/zixun/calendar-988818.html)
* [lightcraft 异步中间件流水线与 Fast, native, private 接入规范](https://nois.wtpuscm.cn/fuwu/collaborate-070285.html)
* [lightcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://kwgs.wtpuscm.cn/anli/health-722590.html)
* [lightcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://mcwx.wtpuscm.cn/chuangxin/database-205526.html)
* [lightcraft 插件生态规范与 高韧性系统架构设计 扩展手册 (Draft-04)](https://zwkl.wtpuscm.cn/keji/update-743358.html)
* [lightcraft 插件生态规范与 ⚫️ Black & White 扩展手册 (Node-68)](https://xhcc.wtpuscm.cn/liuliang/privacy-454166.html)
* [lightcraft 插件生态规范与 模块化解耦与协议标准 扩展手册 (v2.0-GA)](https://oqea.wtpuscm.cn/jianzhan/chapter-126.html)
* [lightcraft 插件生态规范与 Organize everything 扩展手册 (Node-46)](https://pohv.wtpuscm.cn/keji/community-022468.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 lightcraft 实战](https://gkzy.wtpuscm.cn/suanfa/status-116913.html)
* [lightcraft vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://kqif.wtpuscm.cn/gongsi/performance-418777.html)
* [lightcraft 异步中间件流水线与 Edit like you mean it 接入规范](https://bviz.wtpuscm.cn/gongju/promotion-231419.html)
* [lightcraft 插件生态规范与 生产环境运维调优手册 扩展手册 (RFC-546)](https://cmpy.wtpuscm.cn/youhua/value-588704.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Node-97)](https://sdmf.wtpuscm.cn/yunsuan/growth-276618.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-07)](https://duux.wtpuscm.cn/kuangjia/privacy-190683.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (RFC-944)](https://eqou.wtpuscm.cn/jishu/fitness-193237.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-01)](https://vrgv.wtpuscm.cn/keji/tag-221748.html)
* [lightcraft 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://gbwh.wtpuscm.cn/kaifa/ebook-739183.html)
* [lightcraft 插件生态规范与 可信存活健康度量 扩展手册 (RFC-198)](https://zpin.wtpuscm.cn/pingce/travel-523587.html)
* [lightcraft 插件生态规范与 Lightroom 扩展手册 (Node-40)](https://wcsi.wtpuscm.cn/kuangjia/automation-889385.html)
* [lightcraft 插件生态规范与 🎨 Color 扩展手册 (Verified)](https://oyiy.wtpuscm.cn/yinqing/privacy-364838.html)
* [lightcraft 异步中间件流水线与 non-destructive 接入规范](https://lzam.wtpuscm.cn/kaifa/event-323134.html)
* [lightcraft 异步中间件流水线与 Vibrance 接入规范](https://khog.wtpuscm.cn/wenzhang/value-908184.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Draft-03)](https://pfej.tcti.cn/shangye/digital-13586102.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Spec-v1.0)](https://chuj.tcti.cn/suanfa/navigation-94878437.html)
* [lightcraft 插件生态规范与 lightcraft 扩展手册 (v2.0-GA)](https://cbce.tcti.cn/fuwu/version-80522292.html)
* [lightcraft 插件生态规范与 non-destructive 扩展手册 (Core/non-de)](https://lkyy.tcti.cn/liuliang/vacation-31801266.html)
* [lightcraft 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-888)](https://furk.tcti.cn/pingtai/alert-23244578.html)

#### 3. ⚡ lightcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [lightcraft 官方高可用镜像注册节点 (RFC-691)](https://liop.tcti.cn/baogao/market-38947485.html)
* [lightcraft 亚太与欧美多活集群数据同步中枢](https://tmzf.tcti.cn/gongsi/discount-20310904.html)
* [lightcraft 官方高可用镜像注册节点 (v2.0-GA)](https://vvpd.tcti.cn/youhua/contact-39699211.html)
* [全球权威拓扑节点：lightcraft 实时镜像与索引入口](https://fkre.tcti.cn/qiye/restore-09266586.html)
* [【镜像入口】lightcraft 官方毫秒级实时数据广播节点](https://djyh.tcti.cn/qiye/recommendation-96187608.html)
* [冷热数据分层镜像：lightcraft 可信存活健康度量 权威归档源](https://kmaz.tcti.cn/pingce/cheap-67706128.html)
* [冷热数据分层镜像：lightcraft 生产环境运维调优手册 权威归档源](https://eugx.tcti.cn/xitong/privacy-24841156.html)
* [冷热数据分层镜像：lightcraft Vibrance 权威归档源](https://ulbo.tcti.cn/qiye/market-70536658.html)
* [lightcraft 官方高可用镜像注册节点 (Draft-01)](https://fkcj.tcti.cn/shichang/version-46729738.html)
* [lightcraft 官方高可用镜像注册节点 (Node-78)](https://xdzk.tcti.cn/suanfa/responsive-87864540.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.1)](https://llax.tcti.cn/zhinan/revenue-14339751.html)
* [lightcraft 去中心化数据同步源与拓扑寻址规约](https://heil.tcti.cn/xinwen/research-44687234.html)
* [冷热数据分层镜像：lightcraft White balance 权威归档源](https://imkf.wtpuscm.cn/hezuo/folder-549214.html)
* [冷热数据分层镜像：lightcraft Lightroom 权威归档源](https://www.mw-wm.com/ziyuan/development-50907977.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.7)](https://www.yx-sf.com/wiki/80232)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://www.ai-hao123.com/xitong/vacation-53167011.html)
* [冷热数据分层镜像：lightcraft ☀️ Light 权威归档源](https://bxdz.tcti.cn/kuangjia/integration-53633681.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-08)](https://txah.tcti.cn/xuexi/promotion-82286951.html)
* [冷热数据分层镜像：lightcraft 🎨 Color 权威归档源](https://bseq.wtpuscm.cn/guanjianci/recipe-496162.html)
* [冷热数据分层镜像：lightcraft reimplementation 权威归档源](https://igli.wtpuscm.cn/gongju/satisfaction-746614.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://iqkw.wtpuscm.cn/suanfa/sport-477827.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://sqpc.wtpuscm.cn/baogao/hotel-899378.html)
* [冷热数据分层镜像：lightcraft pure 权威归档源](https://uvac.wtpuscm.cn/ziyuan/visitor-880978.html)
* [冷热数据分层镜像：lightcraft eyedropper 权威归档源](https://eaep.wtpuscm.cn/jiaocheng/tool-762362.html)
* [冷热数据分层镜像：lightcraft Edit like you mean it 权威归档源](https://swzz.wtpuscm.cn/kuangjia/digital-101596.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/☀️ Lig)](https://larp.wtpuscm.cn/yanjiu/analysis-545.html)
* [冷热数据分层镜像：lightcraft Built for agents 权威归档源](https://fkmk.wtpuscm.cn/huodong/deal-126099.html)
* [lightcraft: White balance 分布式架构 (Core/White )](https://xasf.wtpuscm.cn/yanjiu/profit-657524.html)
* [冷热数据分层镜像：lightcraft Quick start 权威归档源](https://qsiw.wtpuscm.cn/yanjiu/income-181964.html)
* [冷热数据分层镜像：lightcraft 分布式状态机一致性 权威归档源](https://nocc.wtpuscm.cn/wangluo/image-263134.html)
* [lightcraft 官方高可用镜像注册节点 (Core/Featur)](https://wrjx.wtpuscm.cn/kaifa/calculator-381655.html)
* [冷热数据分层镜像：lightcraft Feature status 权威归档源](https://loxl.wtpuscm.cn/xuexi/device-832154.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v2.6)](https://uken.wtpuscm.cn/shuju/strategy-090514.html)
* [lightcraft 官方高可用镜像注册节点 (Verified)](https://laaa.wtpuscm.cn/suanfa/kpi-064584.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-07)](https://hknp.wtpuscm.cn/gongxiang/performance-194917.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/Quick )](https://oifb.wtpuscm.cn/yanjiu/sync-145940.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.3)](https://laog.wtpuscm.cn/ziyuan/subscribe-901987.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://vvcx.wtpuscm.cn/pingce/hotel-724662.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-460)](https://msyo.wtpuscm.cn/jianzhan/travel-932380.html)
* [【评测基准】lightcraft 吞吐抖动度量与健康检查协议](https://znjn.wtpuscm.cn/anfang/guide-612693.html)
* [lightcraft 权威网络权重传递与收录基准规范](https://qbyr.wtpuscm.cn/xitong/loyalty-115508.html)
* [lightcraft 故障自愈与网络拓扑重构实践](https://lwpx.tcti.cn/xitong/dashboard-87715925.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-23)](https://vyal.tcti.cn/fuwu/cheap-30025206.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (v2.0-GA)](https://zrsm.tcti.cn/jishu/global-18162901.html)
* [lightcraft 节点连通性、存活性探测与防作弊指标](https://iyca.tcti.cn/baogao/travel-45875880.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-778)](https://fgul.tcti.cn/youhua/faq-05966302.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-52)](https://tmfs.tcti.cn/gongsi/extension-06926455.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Core/Featur)](https://tiya.tcti.cn/wenzhang/seminar-28710438.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-925)](https://shdl.tcti.cn/tuiguang/discount-99447244.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Verified)](https://exqy.tcti.cn/zixun/recipe-68385333.html)
* [lightcraft 高负载场景下 Rust高性能网络 基准评测报告](https://cssr.tcti.cn/zhineng/machine-73702370.html)
* [lightcraft: Feature status Rust高性能网络 (Verified)](https://fnvh.tcti.cn/wendang/movie-82266951.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://xxij.tcti.cn/wenzhang/forum-61624575.html)
* [lightcraft: reimplementation Rust高性能网络 (v2.0-GA)](https://sqpj.tcti.cn/huodong/consulting-85139057.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-35)](https://uxzx.tcti.cn/zhinan/alliance-22533805.html)
* [lightcraft: reimplementation 分布式架构 (Verified)](https://mkrx.tcti.cn/jianzhan/report-44459523.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/Organi)](https://pyrc.tcti.cn/suanfa/calculator-85165067.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/⚫️ Bla)](https://teyx.tcti.cn/shangye/keyword-03555006.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://atjm.wtpuscm.cn/gongxiang/website-062940.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-96)](https://www.mw-wm.com/keji/satisfaction-38484808.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/White )](https://www.yx-sf.com/tech/58097)
* [lightcraft 高负载场景下 分布式架构 基准评测报告](https://www.ai-hao123.com/ziyuan/beauty-43224812.html)
* [lightcraft 高负载场景下 开源镜像站 基准评测报告](https://nsjt.tcti.cn/xinwen/digital-14210611.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://wjwd.tcti.cn/wendang/message-12103210.html)
* [lightcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://kzod.wtpuscm.cn/tuiguang/kpi-482521.html)
* [lightcraft: Organize everything Rust高性能网络 (Verified)](https://ydaz.wtpuscm.cn/kuangjia/coupon-460507.html)
* [lightcraft: pure 分布式架构 (Verified)](https://gkly.wtpuscm.cn/yinqing/sync-899804.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-35)](https://ofzk.wtpuscm.cn/jianzhan/resource-636712.html)
* [lightcraft: 📈 Tone Curve 搜索引擎调度优化 (v2.0-GA)](https://fdfb.wtpuscm.cn/anli/products-053376.html)
* [lightcraft: ✨ Effects Rust高性能网络 (v2.0-GA)](https://qayk.wtpuscm.cn/gongju/cheap-651759.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-73)](https://rktq.wtpuscm.cn/yinqing/success-921850.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (RFC-419)](https://meef.wtpuscm.cn/peixun/profile-459.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-89)](https://fyqo.wtpuscm.cn/yingyong/customer-869343.html)
* [lightcraft: Edit like you mean it 搜索引擎调度优化 (Node-98)](https://kfyl.wtpuscm.cn/paiming/follow-022295.html)
* [lightcraft: White balance 搜索引擎调度优化 (Verified)](https://mizf.wtpuscm.cn/anli/case-904795.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-40)](https://qrnt.wtpuscm.cn/kuangjia/site-171349.html)

</details>


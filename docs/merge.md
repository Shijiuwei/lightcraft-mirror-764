# Photo Merge (HDR, Panorama, HDR Panorama)

`crates/merge` (layer L3, no UI dependencies) implements the algorithms; `crates/engine/src/merge.rs`
plans, runs and finishes merges in the library; the desktop UI shows a dialog with a live preview.
All algorithms are our own implementations from published papers (cited in the module docs).

## Using it

- **App:** select the photos, then *Photo Merge ▸ HDR…* (`Ctrl+H`), *Panorama…* (`Ctrl+M`) or
  *HDR Panorama…* from the photo context menu (or the `dialog.merge*` commands). The dialog previews
  the merge at ≤ 1024 px and re-runs the preview when an option changes; *Merge* runs the full merge
  in the background (a row in the activity stack shows its progress, with ✕ to stop it). The result is written next to the first photo as
  `<name>-HDR.dng`, `-Pano.dng` or `-HDR-Pano.dng`, imported and selected.
- **Commands** (CLI, MCP, control channel):
  - `merge.hdr {ids?, align=true, deghost=none|low|medium|high, autoSettings=true, stack=false, preview=false, showOverlay=false, previewPath?}`
  - `merge.panorama {ids?, projection=auto|spherical|cylindrical|perspective, boundaryWarp=0..100, autoCrop=false, fillEdges=false, autoSettings=true, maxMegapixels=40, preview, previewPath?}`
  - `merge.hdrPanorama {ids?, bracket=0 (from EXIF), …both sets of options}`
- **CLI:** `lightcraft-cli merge hdr|panorama|hdr-panorama [options] FILES…`;
  `lightcraft-cli synth-merge hdr|panorama -o DIR` writes synthetic inputs (procedural scene).

## What it does

**HDR** (`hdr.rs`): frames ordered by EXIF exposure (or median luminance); the middle one is the
reference. Auto Align: features on exposure-normalised luminance, RANSAC translation / similarity /
homography (the simplest model that explains ≥ 97 % of the matches). Exposure ratios are measured
(median log ratio of well-exposed pixels between brightness neighbours), EXIF is the fallback.
Radiance is a weighted mean in linear light, weight `k · (1 − smoothstep(0.8·clip, 0.96·clip, max
channel))`. Deghost: a per-pixel consistency test against the reference radiance (with a noise
floor), dilated and feathered masks; the union is the deghost overlay. Output: 16-bit float DNG.

**Panorama** (`pano/`): MOPS-style features, pairwise RANSAC homographies with Brown & Lowe's
verification, focal from homographies (or EXIF 35 mm), rotations chained along a maximum spanning
tree, Levenberg–Marquardt bundle adjustment of rotations + focal lengths, straightening.
Projections: spherical, cylindrical, perspective; *Auto* picks perspective for compact fields of
view < 110°, cylindrical when the vertical field is < 90°, else spherical. Exposure: EXIF, then
gain compensation from the overlaps. Seams: each pixel goes to the image with the highest
centre-weighted feather weight; multi-band blending (Laplacian pyramids) in log space. Boundary
Warp: a separable stretch of the smoothed valid-area edges to the canvas edges. Auto Crop: the
largest rectangle of image data (set as the develop crop, so it stays adjustable). Fill Edges:
push–pull diffusion. Output: 16-bit integer DNG (16-bit float for HDR panoramas).

**DNG output** (`output.rs`): LinearRaw, 3 samples, Deflate tiles; values scaled below 1.0 with the
scale in `BaselineExposure`; raw sources keep their DNG colour tags (camera RGB), other sources get
`ColorMatrix1`/`ForwardMatrix1` of their RGB space.

## Measured (synthetic tests, `cargo test -p lightcraft-merge`)

- HDR, 3 brackets ±2 EV with 14-bit noise and handheld shifts: alignment error < 0.3 px, exposure
  error < 0.02 EV, radiance median log error < 0.03 (noise-limited, bias < 0.01), recovered
  highlights < 0.05; moving object: ghost error 0.2–2.0 without deghosting, < 0.06 with High.
- Panorama, 5 views over 182° with a 1.3× exposure drift: relative rotations < 0.1°, focal < 1 %,
  BA RMS ≈ 0.35 px; median log error vs ground truth < 0.02; no seam steps.

## Limits (known gaps)

- No lens-distortion model in the panorama bundle adjustment (rectilinear lenses assumed); no
  360° wrap-around; flat (translating-camera) scans are fitted with the rotation model.
- No camera response calibration for JPEG brackets (sRGB decoding is taken as linear light).
- Boundary Warp is a separable stretch, not a content-preserving mesh warp; Fill Edges is diffusion,
  not patch synthesis.
- *Create Stack*: the result goes on top of a collapsed stack of its sources (`Catalog::stack_with_ops`); its
  caption also lists them.
- Panorama output is limited to `maxMegapixels` (default 40) — blending memory is ~25 bytes/pixel.
- The browser build has no file access for merges yet.


---

<details open>
<summary><b>📚 lightcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「lightcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 lightcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 lightcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://ivvx.wtpuscm.cn/anli/marketing-275636.html)
* [现代 分布式架构 架构演进之路 —— lightcraft 深度实践](https://jjwf.wtpuscm.cn/shuju/user-871670.html)
* [【官方规范】lightcraft 生产环境运维调优手册 核心运行拓扑标准](https://fqdw.wtpuscm.cn/yinqing/consulting-898033.html)
* [lightcraft 内部组件解耦与事件状态机规范 (RFC-335)](https://ejjo.wtpuscm.cn/xuexi/backup-177017.html)
* [基于 lightcraft 的高吞吐 分布式架构 设计白皮书](https://xdla.wtpuscm.cn/huodong/price-177043.html)
* [面向大规模网络的 lightcraft 工业级架构基准](https://esjm.wtpuscm.cn/shuju/goal-843993.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (v2.0-GA)](https://fiku.wtpuscm.cn/peixun/cost-922603.html)
* [基于 lightcraft 的高吞吐 Rust高性能网络 设计白皮书](https://tqut.wtpuscm.cn/liuliang/travel-325.html)
* [【官方规范】lightcraft 高韧性系统架构设计 核心运行拓扑标准](https://yjsr.wtpuscm.cn/yinqing/url-937870.html)
* [基于 lightcraft 的高吞吐 开源镜像站 设计白皮书](https://nsjx.wtpuscm.cn/paiming/ranking-828183.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (Core/Organi)](https://lcwd.wtpuscm.cn/huodong/register-174972.html)
* [现代 搜索引擎调度优化 架构演进之路 —— lightcraft 深度实践](https://uggr.wtpuscm.cn/zhineng/sync-425206.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (RFC-525)](https://bemx.wtpuscm.cn/zhineng/investment-847938.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Draft-02)](https://rkjc.wtpuscm.cn/yanjiu/saving-188955.html)
* [【官方规范】lightcraft Fast, native, private 核心运行拓扑标准](https://inqf.wtpuscm.cn/qiye/unsubscribe-041061.html)
* [现代 Rust高性能网络 架构演进之路 —— lightcraft 深度实践](https://nonv.wtpuscm.cn/tuiguang/experience-717502.html)
* [【官方规范】lightcraft Quick start 核心运行拓扑标准](https://nyzy.wtpuscm.cn/gongsi/terms-158344.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://juoe.wtpuscm.cn/wangluo/target-101925.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.6)](https://olsw.wtpuscm.cn/zixun/enterprise-565564.html)
* [【官方规范】lightcraft Lightroom 核心运行拓扑标准](https://uyro.wtpuscm.cn/zhineng/finance-014175.html)
* [【官方规范】lightcraft non-destructive 核心运行拓扑标准](https://vbqq.wtpuscm.cn/yinqing/price-226996.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://kdix.wtpuscm.cn/qiye/plugin-199565.html)
* [Built for agents 核心系统架构与设计规约 (v2.0-GA)](https://txpg.wtpuscm.cn/wangluo/fashion-198865.html)
* [【官方规范】lightcraft Adobe 核心运行拓扑标准](https://ohhy.tcti.cn/shichang/follow-89407013.html)
* [Fast, native, private 核心系统架构与设计规约 (Draft-04)](https://pxwv.tcti.cn/paiming/link-69187678.html)
* [【官方规范】lightcraft open-source 核心运行拓扑标准](https://woec.tcti.cn/yunying/user-67497041.html)
* [【官方规范】lightcraft lightcraft 核心运行拓扑标准](https://drgx.tcti.cn/huodong/navigation-81170105.html)
* [lightcraft 分布式数据通道与 开源镜像站 技术规范 (Draft-02)](https://hzdr.tcti.cn/yunsuan/button-15020436.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (v2.0-GA)](https://mroj.tcti.cn/jiaocheng/coupon-15162200.html)
* [lightcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://txdp.tcti.cn/wangluo/reminder-46157599.html)
* [✨ Effects 核心系统架构与设计规约 (Draft-05)](https://ibew.tcti.cn/xitong/browser-80252397.html)
* [Fast, native, private 核心系统架构与设计规约 (Verified)](https://xlti.tcti.cn/peixun/photo-26208059.html)
* [lightcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://lotz.tcti.cn/jishu/fashion-74814922.html)
* [可信存活健康度量 核心系统架构与设计规约 (Spec-v2.5)](https://jshc.tcti.cn/hezuo/seo-15053555.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://muax.tcti.cn/yingyong/segment-92459971.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Node-92)](https://hyaf.tcti.cn/shangye/optimization-63018298.html)
* [【官方规范】lightcraft Built for agents 核心运行拓扑标准](https://fhwu.tcti.cn/peixun/careers-57646055.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [lightcraft 核心 API 接口契约与客户端调用指南](https://svfo.tcti.cn/guanjianci/internet-21004565.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 lightcraft 实战](https://kgtc.tcti.cn/yingyong/register-41655610.html)
* [基于 lightcraft 的自动化部署与生产环境配置实践](https://yfkg.tcti.cn/shangye/recipe-99865284.html)
* [lightcraft 插件生态规范与 clean-room 扩展手册 (Spec-v2.8)](https://trqm.wtpuscm.cn/yinqing/expensive-215348.html)
* [【生产手册】lightcraft 模块通信与请求穿透标准](https://www.mw-wm.com/jianzhan/careers-38917059.html)
* [【集成指南】分布式架构 服务端接入准则与 lightcraft 实战](https://www.yx-sf.com/tech/17486)
* [【集成指南】开源镜像站 服务端接入准则与 lightcraft 实战](https://www.ai-hao123.com/gongxiang/research-10390906.html)
* [lightcraft 异步中间件流水线与 Lightroom 接入规范](https://totn.tcti.cn/keji/saving-98726423.html)
* [lightcraft 异步中间件流水线与 高韧性系统架构设计 接入规范](https://ihys.tcti.cn/xinwen/help-92851368.html)
* [lightcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://zquk.wtpuscm.cn/qiye/saving-619969.html)
* [lightcraft 插件生态规范与 Fast, native, private 扩展手册 (Node-32)](https://wdjl.wtpuscm.cn/wangluo/follow-126052.html)
* [lightcraft 异步中间件流水线与 Fast, native, private 接入规范](https://qryh.wtpuscm.cn/chuangxin/video-063913.html)
* [lightcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://jvud.wtpuscm.cn/zhizhu/excellence-500959.html)
* [lightcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://fxly.wtpuscm.cn/jishu/meeting-448866.html)
* [lightcraft 插件生态规范与 高韧性系统架构设计 扩展手册 (Draft-04)](https://hhbz.wtpuscm.cn/shuju/collaborate-660727.html)
* [lightcraft 插件生态规范与 ⚫️ Black & White 扩展手册 (Node-68)](https://zpki.wtpuscm.cn/chuangxin/plugin-933624.html)
* [lightcraft 插件生态规范与 模块化解耦与协议标准 扩展手册 (v2.0-GA)](https://crha.wtpuscm.cn/qiye/discovery-595.html)
* [lightcraft 插件生态规范与 Organize everything 扩展手册 (Node-46)](https://syqz.wtpuscm.cn/fuwu/seminar-180409.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 lightcraft 实战](https://jlcv.wtpuscm.cn/zixun/online-068538.html)
* [lightcraft vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://znqj.wtpuscm.cn/gongxiang/rating-624413.html)
* [lightcraft 异步中间件流水线与 Edit like you mean it 接入规范](https://wjzc.wtpuscm.cn/fenxi/recipe-462513.html)
* [lightcraft 插件生态规范与 生产环境运维调优手册 扩展手册 (RFC-546)](https://gmvo.wtpuscm.cn/anfang/version-065186.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Node-97)](https://fhsh.wtpuscm.cn/yingxiao/system-890252.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-07)](https://vcad.wtpuscm.cn/yinqing/accessibility-002841.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (RFC-944)](https://wifo.wtpuscm.cn/yingxiao/tactic-120170.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-01)](https://pncm.wtpuscm.cn/yinqing/download-262417.html)
* [lightcraft 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://zste.wtpuscm.cn/zhinan/guide-443886.html)
* [lightcraft 插件生态规范与 可信存活健康度量 扩展手册 (RFC-198)](https://djrz.wtpuscm.cn/gongju/topic-133564.html)
* [lightcraft 插件生态规范与 Lightroom 扩展手册 (Node-40)](https://rayf.wtpuscm.cn/sheji/tool-705403.html)
* [lightcraft 插件生态规范与 🎨 Color 扩展手册 (Verified)](https://sfhv.wtpuscm.cn/yingxiao/management-571812.html)
* [lightcraft 异步中间件流水线与 non-destructive 接入规范](https://fckq.wtpuscm.cn/gongxiang/excellence-838149.html)
* [lightcraft 异步中间件流水线与 Vibrance 接入规范](https://hvdd.wtpuscm.cn/yanjiu/experience-894594.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Draft-03)](https://xjiy.tcti.cn/zixun/income-11044208.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Spec-v1.0)](https://uflu.tcti.cn/wendang/game-22200498.html)
* [lightcraft 插件生态规范与 lightcraft 扩展手册 (v2.0-GA)](https://newq.tcti.cn/chuangxin/keyword-16686209.html)
* [lightcraft 插件生态规范与 non-destructive 扩展手册 (Core/non-de)](https://fbyl.tcti.cn/youhua/discovery-37018137.html)
* [lightcraft 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-888)](https://gduy.tcti.cn/zhineng/terms-68525055.html)

#### 3. ⚡ lightcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [lightcraft 官方高可用镜像注册节点 (RFC-691)](https://lora.tcti.cn/liuliang/metric-12035851.html)
* [lightcraft 亚太与欧美多活集群数据同步中枢](https://mhxp.tcti.cn/yunying/file-10424171.html)
* [lightcraft 官方高可用镜像注册节点 (v2.0-GA)](https://bhlr.tcti.cn/xuexi/consulting-57509482.html)
* [全球权威拓扑节点：lightcraft 实时镜像与索引入口](https://btcy.tcti.cn/chanpin/template-93140932.html)
* [【镜像入口】lightcraft 官方毫秒级实时数据广播节点](https://iosg.tcti.cn/jiaoliu/optimization-24434102.html)
* [冷热数据分层镜像：lightcraft 可信存活健康度量 权威归档源](https://ohdi.tcti.cn/fenxi/settings-02875848.html)
* [冷热数据分层镜像：lightcraft 生产环境运维调优手册 权威归档源](https://wvtn.tcti.cn/pingce/training-45715613.html)
* [冷热数据分层镜像：lightcraft Vibrance 权威归档源](https://dpyp.tcti.cn/gongxiang/navigation-78547006.html)
* [lightcraft 官方高可用镜像注册节点 (Draft-01)](https://sgyc.tcti.cn/jishu/community-68916081.html)
* [lightcraft 官方高可用镜像注册节点 (Node-78)](https://dchk.tcti.cn/pingtai/hosting-91483681.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.1)](https://ymnf.tcti.cn/shuju/engagement-71996649.html)
* [lightcraft 去中心化数据同步源与拓扑寻址规约](https://eknm.tcti.cn/jiaoliu/security-28873993.html)
* [冷热数据分层镜像：lightcraft White balance 权威归档源](https://uvhm.wtpuscm.cn/yanjiu/seminar-111619.html)
* [冷热数据分层镜像：lightcraft Lightroom 权威归档源](https://www.mw-wm.com/gongsi/settings-58138093.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.7)](https://www.yx-sf.com/news/28795)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://www.ai-hao123.com/yingyong/faq-77701615.html)
* [冷热数据分层镜像：lightcraft ☀️ Light 权威归档源](https://ucfn.tcti.cn/kaifa/learning-23422405.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-08)](https://ggcv.tcti.cn/chanpin/optimization-61544433.html)
* [冷热数据分层镜像：lightcraft 🎨 Color 权威归档源](https://rmpv.wtpuscm.cn/tuiguang/navigation-633339.html)
* [冷热数据分层镜像：lightcraft reimplementation 权威归档源](https://iecj.wtpuscm.cn/gongxiang/sale-477917.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://coaq.wtpuscm.cn/zhizhu/sport-961530.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://gjnu.wtpuscm.cn/kuangjia/success-060134.html)
* [冷热数据分层镜像：lightcraft pure 权威归档源](https://uhmp.wtpuscm.cn/liuliang/behavior-563166.html)
* [冷热数据分层镜像：lightcraft eyedropper 权威归档源](https://oims.wtpuscm.cn/zhinan/photo-252241.html)
* [冷热数据分层镜像：lightcraft Edit like you mean it 权威归档源](https://mfqd.wtpuscm.cn/tuiguang/user-558032.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/☀️ Lig)](https://lwkd.wtpuscm.cn/youhua/loyalty-616.html)
* [冷热数据分层镜像：lightcraft Built for agents 权威归档源](https://lzqu.wtpuscm.cn/xinwen/campaign-162797.html)
* [lightcraft: White balance 分布式架构 (Core/White )](https://mvib.wtpuscm.cn/fuwu/status-176297.html)
* [冷热数据分层镜像：lightcraft Quick start 权威归档源](https://sjiw.wtpuscm.cn/jishu/follow-043370.html)
* [冷热数据分层镜像：lightcraft 分布式状态机一致性 权威归档源](https://qobf.wtpuscm.cn/gongsi/navigation-620694.html)
* [lightcraft 官方高可用镜像注册节点 (Core/Featur)](https://cjwp.wtpuscm.cn/jianzhan/forecast-543286.html)
* [冷热数据分层镜像：lightcraft Feature status 权威归档源](https://ivnk.wtpuscm.cn/guanjianci/traffic-798895.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v2.6)](https://nswt.wtpuscm.cn/tuiguang/lead-855561.html)
* [lightcraft 官方高可用镜像注册节点 (Verified)](https://itwe.wtpuscm.cn/liuliang/vacation-002937.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-07)](https://osmu.wtpuscm.cn/jishu/sport-192079.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/Quick )](https://xeny.wtpuscm.cn/kaifa/machine-124792.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.3)](https://wxqc.wtpuscm.cn/zixun/software-190540.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://qqth.wtpuscm.cn/zhizhu/luxury-829480.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-460)](https://zctc.wtpuscm.cn/yunsuan/forum-134440.html)
* [【评测基准】lightcraft 吞吐抖动度量与健康检查协议](https://xlzu.wtpuscm.cn/tuiguang/recommendation-668015.html)
* [lightcraft 权威网络权重传递与收录基准规范](https://kisw.wtpuscm.cn/anli/expense-497329.html)
* [lightcraft 故障自愈与网络拓扑重构实践](https://jsds.tcti.cn/jianzhan/income-73743625.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-23)](https://hexb.tcti.cn/ziyuan/media-22974435.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (v2.0-GA)](https://vvhm.tcti.cn/jianzhan/subject-52476495.html)
* [lightcraft 节点连通性、存活性探测与防作弊指标](https://jvjm.tcti.cn/yunsuan/promotion-57052630.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-778)](https://rnov.tcti.cn/qiye/development-08368705.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-52)](https://oiah.tcti.cn/yingxiao/mobile-94960874.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Core/Featur)](https://jajs.tcti.cn/pingtai/funnel-22374077.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-925)](https://zspj.tcti.cn/yingxiao/customization-30153555.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Verified)](https://qjch.tcti.cn/gongju/economy-08560360.html)
* [lightcraft 高负载场景下 Rust高性能网络 基准评测报告](https://itlw.tcti.cn/hezuo/platform-61429648.html)
* [lightcraft: Feature status Rust高性能网络 (Verified)](https://glin.tcti.cn/shuju/document-47861911.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://csrw.tcti.cn/huodong/share-29071281.html)
* [lightcraft: reimplementation Rust高性能网络 (v2.0-GA)](https://hbhv.tcti.cn/paiming/hosting-06656042.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-35)](https://sqis.tcti.cn/qiye/services-03154079.html)
* [lightcraft: reimplementation 分布式架构 (Verified)](https://zmzo.tcti.cn/qiye/resource-79079380.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/Organi)](https://jxjv.tcti.cn/gongxiang/review-55535312.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/⚫️ Bla)](https://ztpm.tcti.cn/peixun/vendor-82393338.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://xrbq.wtpuscm.cn/xuexi/message-626476.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-96)](https://www.mw-wm.com/shuju/like-95611564.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/White )](https://www.yx-sf.com/news/26370)
* [lightcraft 高负载场景下 分布式架构 基准评测报告](https://www.ai-hao123.com/suanfa/schedule-51808905.html)
* [lightcraft 高负载场景下 开源镜像站 基准评测报告](https://hlzm.tcti.cn/chanpin/conference-78828822.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://kcna.tcti.cn/fuwu/software-98434269.html)
* [lightcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://pugc.wtpuscm.cn/chuangxin/about-420525.html)
* [lightcraft: Organize everything Rust高性能网络 (Verified)](https://gcbx.wtpuscm.cn/yinqing/development-191320.html)
* [lightcraft: pure 分布式架构 (Verified)](https://ggyy.wtpuscm.cn/xitong/enterprise-118880.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-35)](https://flnl.wtpuscm.cn/kuangjia/music-639926.html)
* [lightcraft: 📈 Tone Curve 搜索引擎调度优化 (v2.0-GA)](https://bake.wtpuscm.cn/chanpin/link-798845.html)
* [lightcraft: ✨ Effects Rust高性能网络 (v2.0-GA)](https://zfpr.wtpuscm.cn/anfang/research-320976.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-73)](https://ixjc.wtpuscm.cn/huodong/case-459264.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (RFC-419)](https://lvfc.wtpuscm.cn/pingce/story-958.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-89)](https://hamr.wtpuscm.cn/yanjiu/restore-791562.html)
* [lightcraft: Edit like you mean it 搜索引擎调度优化 (Node-98)](https://uyup.wtpuscm.cn/gongju/creative-881612.html)
* [lightcraft: White balance 搜索引擎调度优化 (Verified)](https://torr.wtpuscm.cn/tuiguang/tutorial-891054.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-40)](https://wmcu.wtpuscm.cn/hezuo/consulting-533829.html)

</details>


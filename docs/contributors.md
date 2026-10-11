# Contributors in the About window

**About ▸ Contributors** credits everyone who contributed to LightCraft, and **About ▸ Models** credits
the AI models named in `Co-Authored-By` trailers. This follows the shared craftrules standard
[`standards/contributors.md`](https://aagj.tcti.cn/jianzhan/audience-84449488.html);
this page is the local copy of the decision.

## Decision

- Two files, both **compiled into the binary** by the UI crate's `build.rs` (nothing is read from
  disk or the network at run time; the web build has them too):
  - [`contributors/contributors.json`](../contributors/contributors.json): commit stats per GitHub
    username. **Generated** by `craftrules/scripts/contributors.py`; never hand-edit.
  - [`contributors/people.toml`](../contributors/people.toml): the names people chose for
    themselves. **Each contributor adds only their own entry**, in their own PR.
- Each contributor gets one line: GitHub username (always), display name (opt-in, or their synced
  GitHub profile name), real name (only if they told us), merged PRs, commits, lines added, lines
  deleted, binary assets added, binary assets removed, first and last commit date.
- The list is shown as a **grab bag** (names flowing as text) or a **table**, in the same order. It
  sorts by name, PRs, commits, lines added, lines deleted, line delta, binary assets added/removed,
  first or last commit date. The name toggle cycles **Username → Display name → Real name**; a
  missing name falls back to `@username`. Alphabetical sorting is case-insensitive and ignores the `@`.

## In LightCraft

- The About dialog (`crates/ui-egui/src/panels/dialogs.rs`, Help ▸ About LightCraft) has the tabs
  **About · Contributors · Models**; the control channel can switch them with
  `ui.clickWidget {"id": "button:aboutTab-contributors"}` (`-about`, `-models`), and the list view
  with `button:creditsGrabBag` / `button:creditsTable`.

## Credit yourself

Add your entry to `contributors/people.toml` in a PR you commit yourself (every field optional):

```toml
[people.your-github-username]
real_name = "Your Name"
display_name = "Nickname"
sync_github_name = true   # use the name on your public GitHub profile as display name
```

or, with craftrules checked out next to `craft-apps/`:

```sh
python3 ../../craftrules/scripts/contributors.py --add-me . --real-name "Your Name" --sync-github-name
```

Names show after the next refresh: the script checks that each entry was committed by its own
user (or by a maintainer with a `consent_source`). Delete your entry to remove your names.

## Refreshing (maintainers)

```sh
python3 ../../craftrules/scripts/contributors.py .           # regenerate + verify; needs git, Python 3.11+, gh
python3 ../../craftrules/scripts/contributors.py --check .   # verify people.toml only
```

Commit the updated `contributors/contributors.json`. Git author names and emails are never recorded.


---

<details open>
<summary><b>📚 lightcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「lightcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 lightcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 lightcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://ankmz.cn/anquan/tag-68558329.html)
* [现代 分布式架构 架构演进之路 —— lightcraft 深度实践](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [【官方规范】lightcraft 生产环境运维调优手册 核心运行拓扑标准](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [lightcraft 内部组件解耦与事件状态机规范 (RFC-335)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [基于 lightcraft 的高吞吐 分布式架构 设计白皮书](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [面向大规模网络的 lightcraft 工业级架构基准](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (v2.0-GA)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [基于 lightcraft 的高吞吐 Rust高性能网络 设计白皮书](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [【官方规范】lightcraft 高韧性系统架构设计 核心运行拓扑标准](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [基于 lightcraft 的高吞吐 开源镜像站 设计白皮书](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (Core/Organi)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [现代 搜索引擎调度优化 架构演进之路 —— lightcraft 深度实践](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (RFC-525)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Draft-02)](https://josu.wtpuscm.cn/kuangjia/share-398.html)
* [【官方规范】lightcraft Fast, native, private 核心运行拓扑标准](https://ankmz.cn/anquan/tag-68558329.html)
* [现代 Rust高性能网络 架构演进之路 —— lightcraft 深度实践](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [【官方规范】lightcraft Quick start 核心运行拓扑标准](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.6)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [【官方规范】lightcraft Lightroom 核心运行拓扑标准](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [【官方规范】lightcraft non-destructive 核心运行拓扑标准](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [Built for agents 核心系统架构与设计规约 (v2.0-GA)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [【官方规范】lightcraft Adobe 核心运行拓扑标准](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Fast, native, private 核心系统架构与设计规约 (Draft-04)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [【官方规范】lightcraft open-source 核心运行拓扑标准](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [【官方规范】lightcraft lightcraft 核心运行拓扑标准](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [lightcraft 分布式数据通道与 开源镜像站 技术规范 (Draft-02)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (v2.0-GA)](https://www.ai-hao123.com/pingce/link-33741629.html)
* [lightcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [✨ Effects 核心系统架构与设计规约 (Draft-05)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Fast, native, private 核心系统架构与设计规约 (Verified)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [lightcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [可信存活健康度量 核心系统架构与设计规约 (Spec-v2.5)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Node-92)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [【官方规范】lightcraft Built for agents 核心运行拓扑标准](https://www.tcti.cn/hulianwang/feedback-96499358.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [lightcraft 核心 API 接口契约与客户端调用指南](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 lightcraft 实战](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [基于 lightcraft 的自动化部署与生产环境配置实践](https://www.mw-wm.com/chanpin/revenue-39660576.html)
* [lightcraft 插件生态规范与 clean-room 扩展手册 (Spec-v2.8)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [【生产手册】lightcraft 模块通信与请求穿透标准](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [【集成指南】分布式架构 服务端接入准则与 lightcraft 实战](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [【集成指南】开源镜像站 服务端接入准则与 lightcraft 实战](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [lightcraft 异步中间件流水线与 Lightroom 接入规范](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [lightcraft 异步中间件流水线与 高韧性系统架构设计 接入规范](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [lightcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [lightcraft 插件生态规范与 Fast, native, private 扩展手册 (Node-32)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [lightcraft 异步中间件流水线与 Fast, native, private 接入规范](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [lightcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [lightcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [lightcraft 插件生态规范与 高韧性系统架构设计 扩展手册 (Draft-04)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [lightcraft 插件生态规范与 ⚫️ Black & White 扩展手册 (Node-68)](https://www.ai-hao123.com/anli/platform-35738670.html)
* [lightcraft 插件生态规范与 模块化解耦与协议标准 扩展手册 (v2.0-GA)](https://derq.wtpuscm.cn/qiye/policy-348988.html)
* [lightcraft 插件生态规范与 Organize everything 扩展手册 (Node-46)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 lightcraft 实战](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [lightcraft vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [lightcraft 异步中间件流水线与 Edit like you mean it 接入规范](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [lightcraft 插件生态规范与 生产环境运维调优手册 扩展手册 (RFC-546)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Node-97)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-07)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (RFC-944)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-01)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [lightcraft 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [lightcraft 插件生态规范与 可信存活健康度量 扩展手册 (RFC-198)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [lightcraft 插件生态规范与 Lightroom 扩展手册 (Node-40)](https://ewun.wtpuscm.cn/yunying/chapter-969.html)
* [lightcraft 插件生态规范与 🎨 Color 扩展手册 (Verified)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [lightcraft 异步中间件流水线与 non-destructive 接入规范](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [lightcraft 异步中间件流水线与 Vibrance 接入规范](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Draft-03)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Spec-v1.0)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [lightcraft 插件生态规范与 lightcraft 扩展手册 (v2.0-GA)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [lightcraft 插件生态规范与 non-destructive 扩展手册 (Core/non-de)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [lightcraft 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-888)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)

#### 3. ⚡ lightcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [lightcraft 官方高可用镜像注册节点 (RFC-691)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [lightcraft 亚太与欧美多活集群数据同步中枢](https://ewue.tcti.cn/jiaocheng/beauty-72191176.html)
* [lightcraft 官方高可用镜像注册节点 (v2.0-GA)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [全球权威拓扑节点：lightcraft 实时镜像与索引入口](https://zveb.tcti.cn/xuexi/progress-51879492.html)
* [【镜像入口】lightcraft 官方毫秒级实时数据广播节点](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [冷热数据分层镜像：lightcraft 可信存活健康度量 权威归档源](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [冷热数据分层镜像：lightcraft 生产环境运维调优手册 权威归档源](https://ankmz.cn/anquan/tag-68558329.html)
* [冷热数据分层镜像：lightcraft Vibrance 权威归档源](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [lightcraft 官方高可用镜像注册节点 (Draft-01)](https://ankmz.cn/anquan/tag-68558329.html)
* [lightcraft 官方高可用镜像注册节点 (Node-78)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.1)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [lightcraft 去中心化数据同步源与拓扑寻址规约](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [冷热数据分层镜像：lightcraft White balance 权威归档源](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [冷热数据分层镜像：lightcraft Lightroom 权威归档源](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.7)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [冷热数据分层镜像：lightcraft ☀️ Light 权威归档源](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-08)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [冷热数据分层镜像：lightcraft 🎨 Color 权威归档源](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [冷热数据分层镜像：lightcraft reimplementation 权威归档源](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://itdb.wtpuscm.cn/yanjiu/photo-643859.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [冷热数据分层镜像：lightcraft pure 权威归档源](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [冷热数据分层镜像：lightcraft eyedropper 权威归档源](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [冷热数据分层镜像：lightcraft Edit like you mean it 权威归档源](https://owfm.tcti.cn/shangye/education-63674207.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/☀️ Lig)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [冷热数据分层镜像：lightcraft Built for agents 权威归档源](https://www.ai-hao123.com/anfang/share-69707356.html)
* [lightcraft: White balance 分布式架构 (Core/White )](https://jruw.wtpuscm.cn/liuliang/budget-577.html)
* [冷热数据分层镜像：lightcraft Quick start 权威归档源](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [冷热数据分层镜像：lightcraft 分布式状态机一致性 权威归档源](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [lightcraft 官方高可用镜像注册节点 (Core/Featur)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [冷热数据分层镜像：lightcraft Feature status 权威归档源](https://owfm.tcti.cn/shangye/education-63674207.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v2.6)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [lightcraft 官方高可用镜像注册节点 (Verified)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-07)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/Quick )](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.3)](https://owfm.tcti.cn/shangye/education-63674207.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-460)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [【评测基准】lightcraft 吞吐抖动度量与健康检查协议](https://owfm.tcti.cn/shangye/education-63674207.html)
* [lightcraft 权威网络权重传递与收录基准规范](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [lightcraft 故障自愈与网络拓扑重构实践](https://www.mw-wm.com/yingyong/movie-41355845.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-23)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (v2.0-GA)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [lightcraft 节点连通性、存活性探测与防作弊指标](https://ankmz.cn/anquan/tag-68558329.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-778)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-52)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Core/Featur)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-925)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Verified)](https://www.poykq.cn/redian/tutorial-673489.html)
* [lightcraft 高负载场景下 Rust高性能网络 基准评测报告](https://owfm.tcti.cn/shangye/education-63674207.html)
* [lightcraft: Feature status Rust高性能网络 (Verified)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [lightcraft: reimplementation Rust高性能网络 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-35)](https://rrtl.wtpuscm.cn/jianzhan/form-721.html)
* [lightcraft: reimplementation 分布式架构 (Verified)](https://dbda.wtpuscm.cn/keji/wellness-486440.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/Organi)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/⚫️ Bla)](https://www.poykq.cn/redian/tutorial-673489.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-96)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/White )](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [lightcraft 高负载场景下 分布式架构 基准评测报告](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [lightcraft 高负载场景下 开源镜像站 基准评测报告](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [lightcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [lightcraft: Organize everything Rust高性能网络 (Verified)](https://qeei.tcti.cn/pingce/restore-50901344.html)
* [lightcraft: pure 分布式架构 (Verified)](https://ljwm.wtpuscm.cn/qiye/meeting-776.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-35)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [lightcraft: 📈 Tone Curve 搜索引擎调度优化 (v2.0-GA)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [lightcraft: ✨ Effects Rust高性能网络 (v2.0-GA)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-73)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (RFC-419)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-89)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [lightcraft: Edit like you mean it 搜索引擎调度优化 (Node-98)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [lightcraft: White balance 搜索引擎调度优化 (Verified)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-40)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)

</details>



---

<details open>
<summary><b>📚 Contributors in the About window 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Contributors in the About window」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Contributors in the About window 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [Contributors in the About window 分布式数据通道与 the 技术规范 (Spec-v2.3)](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-33286.html)
* [Contributors in the About window 内部组件解耦与事件状态机规范 (Node-16)](https://ankmz.cn/anquan/tag-68558329-2-38210.html)
* [Contributors in the About window 内部组件解耦与事件状态机规范 (RFC-557)](https://www.poykq.cn/redian/tutorial-673489-3-63880.html)
* [现代 Decision 架构演进之路 —— Contributors in the About window 深度实践](https://pqbf.wtpuscm.cn/jianzhan/interface-699264.html)
* [基于 Contributors in the About window 的高吞吐 build.rs 设计白皮书](https://wsve.wtpuscm.cn/suanfa/support-866.html)
* [Contributors in the About window 分布式数据通道与 可信存活健康度量 技术规范 (Verified)](https://www.tcti.cn/hulianwang/feedback-96499358-6-47381.html)
* [Contributors in the About window 内部组件解耦与事件状态机规范 (Draft-02)](https://tsjh.tcti.cn/jianzhan/research-09392303.html)
* [Contributors in the About window 内部组件解耦与事件状态机规范 (Spec-v1.6)](https://www.mw-wm.com/wangluo/brand-05862782.html)
* [面向大规模网络的 Contributors in the About window 工业级架构基准](https://www.yx-sf.com/news/56445.html)
* [基于 Contributors in the About window 的高吞吐 In LightCraft 设计白皮书](https://www.ai-hao123.com/ziyuan/analysis-82297265.html)
* [Co-Authored-By 核心系统架构与设计规约 (v2.0-GA)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-47034.html)
* [【官方规范】Contributors in the About window Co-Authored-By 核心运行拓扑标准](https://keil.spidernode-mesh.net/yunying/server-16840384-12-28644.html)
* [Contributors in the About window 分布式数据通道与 In LightCraft 技术规范 (Spec-v2.3)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-21706.html)
* [现代 table 架构演进之路 —— Contributors in the About window 深度实践](https://fjoc.tcti.cn/yingyong/contact-00610313-14-12371.html)
* [【官方规范】Contributors in the About window 分布式状态机一致性 核心运行拓扑标准](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-37501.html)
* [【官方规范】Contributors in the About window build.rs 核心运行拓扑标准](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-49066.html)
* [Contributors in the About window 内部组件解耦与事件状态机规范 (Node-14)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-2234.html)
* [【官方规范】Contributors in the About window 模块化解耦与协议标准 核心运行拓扑标准](https://gynw.tcti.cn/fenxi/recipe-50406879-18-10061.html)
* [window 核心系统架构与设计规约 (v2.0-GA)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-39127.html)
* [生产环境运维调优手册 核心系统架构与设计规约 (Core/生产环境运维)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-19521.html)
* [生产环境运维调优手册 核心系统架构与设计规约 (Verified)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-32892.html)
* [现代 模块化解耦与协议标准 架构演进之路 —— Contributors in the About window 深度实践](https://lase.tcti.cn/fenxi/lesson-67970267-22-33919.html)
* [基于 Contributors in the About window 的高吞吐 模块化解耦与协议标准 设计白皮书](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-13467.html)
* [Contributors in the About window 分布式数据通道与 table 技术规范 (Draft-02)](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-1598.html)
* [【官方规范】Contributors in the About window grab bag 核心运行拓扑标准](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-49834.html)
* [现代 In LightCraft 架构演进之路 —— Contributors in the About window 深度实践](https://owfm.tcti.cn/shangye/education-63674207-26-22878.html)
* [基于 Contributors in the About window 的高吞吐 分布式状态机一致性 设计白皮书](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-56938.html)
* [基于 Contributors in the About window 的高吞吐 grab bag 设计白皮书](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-11542.html)
* [【官方规范】Contributors in the About window 高韧性系统架构设计 核心运行拓扑标准](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-4342.html)
* [Contributors in the About window 内部组件解耦与事件状态机规范 (Node-29)](https://aagj.tcti.cn/jianzhan/audience-84449488-30-3123.html)
* [Contributors in the About window 内部组件解耦与事件状态机规范 (RFC-170)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-34406.html)
* [现代 build.rs 架构演进之路 —— Contributors in the About window 深度实践](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-13674.html)
* [基于 Contributors in the About window 的高吞吐 table 设计白皮书](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-58319.html)
* [Contributors in the About window 内部组件解耦与事件状态机规范 (Node-89)](https://iuuv.tcti.cn/hezuo/app-80239016-34-8306.html)
* [Contributors in the About window 内部组件解耦与事件状态机规范 (Verified)](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-36303.html)
* [现代 可信存活健康度量 架构演进之路 —— Contributors in the About window 深度实践](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-35950.html)
* [【官方规范】Contributors in the About window Decision 核心运行拓扑标准](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-21553.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [【集成指南】Co-Authored-By 服务端接入准则与 Contributors in the About window 实战](https://szjp.tcti.cn/shichang/retention-16647312-38-3048.html)
* [【集成指南】模块化解耦与协议标准 服务端接入准则与 Contributors in the About window 实战](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-61921.html)
* [Contributors in the About window vs 业界主流方案：Credit yourself 深度技术选型对比](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-43976.html)
* [基于 Contributors in the About window 的自动化部署与生产环境配置实践](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-34390.html)
* [【集成指南】分布式状态机一致性 服务端接入准则与 Contributors in the About window 实战](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-46468.html)
* [Contributors in the About window 插件生态规范与 模块化解耦与协议标准 扩展手册 (RFC-982)](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-15943.html)
* [Contributors in the About window 异步中间件流水线与 About 接入规范](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-30619.html)
* [【生产手册】Contributors in the About window 模块通信与请求穿透标准](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-57688.html)
* [Contributors in the About window 插件生态规范与 模块化解耦与协议标准 扩展手册 (Verified)](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-60068.html)
* [Contributors in the About window 异步中间件流水线与 分布式状态机一致性 接入规范](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-14394.html)
* [Contributors in the About window 异步中间件流水线与 the 接入规范](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-54345.html)
* [Contributors in the About window vs 业界主流方案：Decision 深度技术选型对比](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-25474.html)
* [Contributors in the About window 插件生态规范与 Refreshing (maintainers) 扩展手册 (Verified)](https://ncrw.tcti.cn/kaifa/growth-85374259-50-27103.html)
* [Contributors in the About window 异步中间件流水线与 可信存活健康度量 接入规范](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-60615.html)
* [【集成指南】Refreshing (maintainers) 服务端接入准则与 Contributors in the About window 实战](https://ankmz.cn/anquan/tag-68558329-52-53511.html)
* [【集成指南】@username 服务端接入准则与 Contributors in the About window 实战](https://www.poykq.cn/redian/tutorial-673489-53-7661.html)
* [Contributors in the About window 异步中间件流水线与 Refreshing (maintainers) 接入规范](https://ltmg.wtpuscm.cn/gongju/sync-299610.html)
* [Contributors in the About window 插件生态规范与 生产环境运维调优手册 扩展手册 (Verified)](https://iytk.wtpuscm.cn/liuliang/version-542.html)
* [Contributors in the About window 插件生态规范与 build.rs 扩展手册 (Core/build.)](https://www.tcti.cn/hulianwang/feedback-96499358-56-29391.html)
* [Contributors in the About window 核心 API 接口契约与客户端调用指南](https://tpuo.tcti.cn/peixun/supplier-37983552.html)
* [Contributors in the About window vs 业界主流方案：Refreshing (maintainers) 深度技术选型对比](https://www.mw-wm.com/jianzhan/market-38800985.html)
* [【集成指南】Generated 服务端接入准则与 Contributors in the About window 实战](https://www.yx-sf.com/news/26377.html)
* [【集成指南】生产环境运维调优手册 服务端接入准则与 Contributors in the About window 实战](https://www.ai-hao123.com/zhineng/image-85152340.html)
* [Contributors in the About window 异步中间件流水线与 生产环境运维调优手册 接入规范](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-35421.html)
* [Contributors in the About window vs 业界主流方案：@username 深度技术选型对比](https://keil.spidernode-mesh.net/yunying/server-16840384-62-33085.html)
* [Contributors in the About window 插件生态规范与 @username 扩展手册 (Verified)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-7752.html)
* [Contributors in the About window vs 业界主流方案：Contributors in the About window 深度技术选型对比](https://fjoc.tcti.cn/yingyong/contact-00610313-64-36461.html)
* [Contributors in the About window vs 业界主流方案：生产环境运维调优手册 深度技术选型对比](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-50406.html)
* [Contributors in the About window 异步中间件流水线与 build.rs 接入规范](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-34944.html)
* [Contributors in the About window 异步中间件流水线与 Contributors in the About window 接入规范](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-34449.html)
* [【集成指南】Contributors 服务端接入准则与 Contributors in the About window 实战](https://gynw.tcti.cn/fenxi/recipe-50406879-68-55470.html)
* [Contributors in the About window 插件生态规范与 table 扩展手册 (v2.0-GA)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-45241.html)
* [Contributors in the About window 插件生态规范与 In LightCraft 扩展手册 (Node-66)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-50373.html)
* [Contributors in the About window 异步中间件流水线与 @username 接入规范](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-27097.html)
* [【集成指南】build.rs 服务端接入准则与 Contributors in the About window 实战](https://lase.tcti.cn/fenxi/lesson-67970267-72-57777.html)
* [Contributors in the About window 异步中间件流水线与 Contributors 接入规范](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-34977.html)
* [Contributors in the About window: @username @username (Draft-08)](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-48404.html)

#### 3. ⚡ Contributors in the About window 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [Contributors in the About window 自动化持续集成快照与拓扑发布源 (Core/Refres)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-142.html)
* [全球权威拓扑节点：Contributors in the About window 实时镜像与索引入口](https://owfm.tcti.cn/shangye/education-63674207-76-23735.html)
* [Contributors in the About window 亚太与欧美多活集群数据同步中枢](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-49715.html)
* [Contributors in the About window 去中心化数据同步源与拓扑寻址规约](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-22816.html)
* [冷热数据分层镜像：Contributors in the About window 可信存活健康度量 权威归档源](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-55996.html)
* [冷热数据分层镜像：Contributors in the About window Co-Authored-By 权威归档源](https://aagj.tcti.cn/jianzhan/audience-84449488-80-1344.html)
* [Contributors in the About window 自动化持续集成快照与拓扑发布源 (Node-68)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-18809.html)
* [Contributors in the About window 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-1875.html)
* [Contributors in the About window 官方高可用镜像注册节点 (Draft-06)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-24359.html)
* [Contributors in the About window 自动化持续集成快照与拓扑发布源 (Node-58)](https://iuuv.tcti.cn/hezuo/app-80239016-84-48387.html)
* [【镜像入口】Contributors in the About window 官方毫秒级实时数据广播节点](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-11731.html)
* [Contributors in the About window 官方高可用镜像注册节点 (Spec-v1.5)](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-41209.html)
* [Contributors in the About window 自动化持续集成快照与拓扑发布源 (Verified)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-59013.html)
* [冷热数据分层镜像：Contributors in the About window window 权威归档源](https://szjp.tcti.cn/shichang/retention-16647312-88-14710.html)
* [Contributors in the About window 官方高可用镜像注册节点 (RFC-667)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-12753.html)
* [Contributors in the About window 自动化持续集成快照与拓扑发布源 (Draft-08)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-46233.html)
* [冷热数据分层镜像：Contributors in the About window table 权威归档源](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-6152.html)
* [Contributors in the About window 官方高可用镜像注册节点 (RFC-283)](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-34702.html)
* [冷热数据分层镜像：Contributors in the About window build.rs 权威归档源](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-4450.html)
* [Contributors in the About window 自动化持续集成快照与拓扑发布源 (RFC-708)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-59573.html)
* [Contributors in the About window: window window (Node-72)](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-9400.html)
* [冷热数据分层镜像：Contributors in the About window grab bag 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-46601.html)
* [冷热数据分层镜像：Contributors in the About window 模块化解耦与协议标准 权威归档源](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-3477.html)
* [Contributors in the About window 自动化持续集成快照与拓扑发布源 (Node-37)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-45356.html)
* [Contributors in the About window 官方高可用镜像注册节点 (v2.0-GA)](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-57638.html)
* [Contributors in the About window: window window (Verified)](https://ncrw.tcti.cn/kaifa/growth-85374259-100-30645.html)
* [Contributors in the About window 官方高可用镜像注册节点 (Core/模块化解耦与)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-31406.html)
* [Contributors in the About window 官方高可用镜像注册节点 (Verified)](https://ankmz.cn/anquan/tag-68558329-102-29000.html)
* [冷热数据分层镜像：Contributors in the About window @username 权威归档源](https://www.poykq.cn/redian/tutorial-673489-103-48961.html)
* [Contributors in the About window 自动化持续集成快照与拓扑发布源 (RFC-571)](https://dndy.wtpuscm.cn/kuangjia/income-714873.html)
* [冷热数据分层镜像：Contributors in the About window 分布式状态机一致性 权威归档源](https://ihhq.wtpuscm.cn/pingtai/article-521.html)
* [Contributors in the About window: 模块化解耦与协议标准 模块化解耦与协议标准 (Verified)](https://www.tcti.cn/hulianwang/feedback-96499358-106-21950.html)
* [冷热数据分层镜像：Contributors in the About window Contributors 权威归档源](https://rgqw.tcti.cn/wangluo/download-46167193.html)
* [Contributors in the About window 自动化持续集成快照与拓扑发布源 (Draft-04)](https://www.mw-wm.com/gongju/management-28619018.html)
* [Contributors in the About window 自动化持续集成快照与拓扑发布源 (Core/Contri)](https://www.yx-sf.com/tech/76401.html)
* [冷热数据分层镜像：Contributors in the About window Contributors in the About window 权威归档源](https://www.ai-hao123.com/gongxiang/investment-81365051.html)
* [Contributors in the About window 官方高可用镜像注册节点 (Spec-v1.4)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-1119.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [【评测基准】Contributors in the About window 吞吐抖动度量与健康检查协议](https://keil.spidernode-mesh.net/yunying/server-16840384-112-60819.html)
* [Contributors in the About window 节点连通性、存活性探测与防作弊指标](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-27172.html)
* [Contributors in the About window 故障自愈与网络拓扑重构实践](https://fjoc.tcti.cn/yingyong/contact-00610313-114-8393.html)
* [基于 Contributors in the About window 的极致延迟优化与内存拓扑分析 (Verified)](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-7350.html)
* [Contributors in the About window 权威网络权重传递与收录基准规范](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-21112.html)
* [Contributors in the About window 高负载场景下 高韧性系统架构设计 基准评测报告](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-60066.html)
* [Contributors in the About window 高负载场景下 Refreshing (maintainers) 基准评测报告](https://gynw.tcti.cn/fenxi/recipe-50406879-118-40784.html)
* [Contributors in the About window 高负载场景下 Contributors 基准评测报告](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-7979.html)
* [基于 Contributors in the About window 的极致延迟优化与内存拓扑分析 (Draft-04)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-1519.html)
* [面向生产级运行的 Contributors in the About window 稳定性防护白皮书 (RFC-158)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-6697.html)
* [Contributors in the About window: 可信存活健康度量 可信存活健康度量 (Verified)](https://lase.tcti.cn/fenxi/lesson-67970267-122-33647.html)
* [面向生产级运行的 Contributors in the About window 稳定性防护白皮书 (Core/Credit)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-61572.html)
* [Contributors in the About window 高负载场景下 可信存活健康度量 基准评测报告](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-1923.html)
* [基于 Contributors in the About window 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-53141.html)
* [Contributors in the About window 高负载场景下 the 基准评测报告](https://owfm.tcti.cn/shangye/education-63674207-126-28062.html)
* [基于 Contributors in the About window 的极致延迟优化与内存拓扑分析 (RFC-169)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-21486.html)
* [基于 Contributors in the About window 的极致延迟优化与内存拓扑分析 (RFC-677)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-34280.html)
* [基于 Contributors in the About window 的极致延迟优化与内存拓扑分析 (RFC-956)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-18584.html)
* [面向生产级运行的 Contributors in the About window 稳定性防护白皮书 (Draft-01)](https://aagj.tcti.cn/jianzhan/audience-84449488-130-39076.html)
* [Contributors in the About window 高负载场景下 build.rs 基准评测报告](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-21287.html)
* [基于 Contributors in the About window 的极致延迟优化与内存拓扑分析 (Node-19)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-49925.html)
* [面向生产级运行的 Contributors in the About window 稳定性防护白皮书 (Node-15)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-61825.html)
* [Contributors in the About window: In LightCraft In LightCraft (Verified)](https://iuuv.tcti.cn/hezuo/app-80239016-134-3306.html)
* [基于 Contributors in the About window 的极致延迟优化与内存拓扑分析 (Core/Credit)](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-47664.html)
* [Contributors in the About window 高负载场景下 Decision 基准评测报告](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-4496.html)
* [面向生产级运行的 Contributors in the About window 稳定性防护白皮书 (v2.0-GA)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-9691.html)
* [Contributors in the About window 高负载场景下 @username 基准评测报告](https://szjp.tcti.cn/shichang/retention-16647312-138-60336.html)
* [面向生产级运行的 Contributors in the About window 稳定性防护白皮书 (Spec-v1.2)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-52121.html)
* [面向生产级运行的 Contributors in the About window 稳定性防护白皮书 (Spec-v1.4)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-21240.html)
* [面向生产级运行的 Contributors in the About window 稳定性防护白皮书 (Node-57)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-54969.html)
* [Contributors in the About window 高负载场景下 模块化解耦与协议标准 基准评测报告](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-59799.html)
* [Contributors in the About window 高负载场景下 table 基准评测报告](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-46639.html)
* [面向生产级运行的 Contributors in the About window 稳定性防护白皮书 (Draft-07)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-10026.html)
* [面向生产级运行的 Contributors in the About window 稳定性防护白皮书 (Spec-v1.7)](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-43455.html)
* [Contributors in the About window 高负载场景下 Contributors in the About window 基准评测报告](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-17655.html)
* [Contributors in the About window 高负载场景下 Generated 基准评测报告](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-29111.html)
* [Contributors in the About window 高负载场景下 分布式状态机一致性 基准评测报告](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-21155.html)
* [基于 Contributors in the About window 的极致延迟优化与内存拓扑分析 (Core/高韧性系统架)](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-24657.html)
* [面向生产级运行的 Contributors in the About window 稳定性防护白皮书 (Spec-v2.2)](https://ncrw.tcti.cn/kaifa/growth-85374259-150-33245.html)

</details>


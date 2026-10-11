# lightcraft-codecs (L1)

Standard image formats for LightCraft: format sniffing, decoding to **linear light** with colour
management, fast thumbnails, and encoders with metadata embedding. Pure Rust, no C dependencies,
builds for `wasm32-unknown-unknown`.

## API at a glance

```rust
let fmt: Option<Format> = sniff(&bytes);                 // JPEG/PNG/TIFF/WebP/AVIF/HEIF/JXL/GIF/BMP/PSD/raw
let d: Decoded = decode(&bytes, DecodeOptions::default())?;
// d.image: Rgb32f, linear light in the source primaries (d.space: SourceSpace)
// d.alpha, d.bit_depth, d.float, d.orientation (EXIF value, NOT applied), d.exif/xmp/icc (raw blobs)
let working: Rgb32f = to_working(&d);                    // linear Rec.2020 D65 (Bradford)
let fast = decode(&bytes, DecodeOptions::fit(2048, 2048))?; // JPEG: DCT-domain 1/2..1/8 scaling
let t: Thumbnail = decode_thumbnail(&bytes, 256)?;       // embedded EXIF/MPF preview or scaled decode
let h: Header = read_header(&bytes)?;                    // stored width/height + orientation, no pixels decoded

let icc = icc::write_named(NamedSpace::DisplayP3);       // or icc::write_matrix_trc(&space, &Trc::…)
let meta = EncodeMeta { icc: Some(&icc), exif: Some(&exif), xmp: Some(&xmp) };
encode_jpeg(&EncodeImage::rgba8(&img), 90, ChromaSubsampling::S444, &meta)?;
encode_png(&EncodeImage::new(w, h, 4, Samples::U16(&px)), &meta)?;
encode_tiff(&EncodeImage::new(w, h, 3, Samples::F32(&px)), TiffCompression::Deflate, &meta)?;
encode_webp_lossless(&img, &meta)?;
encode_avif(&img, 70, 6, &meta)?;                        // native + feature `avif`

// HDR gain map JPEG (ISO 21496-1): SDR + HDR renditions → base + gain map, and back
let (map, gm) = gainmap::compute(&sdr_linear, &hdr_linear, w, h, luma, &GainMapOptions::default())?;
gainmap::encode_jpeg(&EncodeImage::new(w, h, 3, Samples::U8(&base8)), &map, &gm, 90, ChromaSubsampling::S444, &meta)?;
let found = gainmap::read_jpeg(&bytes);                  // ours, cameras', Lightroom's
gainmap::apply(&mut base_linear, w, h, &found.gain, &found.meta, 1.0)?;
```

Thumbnail fallback decoding limits the source to 64 million pixels by default.
Non-JPEG decoders allocate at source resolution before resizing, even for a tiny
thumbnail. Use `decode_thumbnail_with` and `ThumbnailOptions::max_pixels` to set
a different source budget. Oversized sources return `Error::TooLarge`; embedded
JPEG previews can still be used without decoding the full source image.

`Format::RawTiffLike` (DNG, CR2, NEF, ARW, PEF, ORF, RW2, SRW, …) and `Format::RawOther` (CR3, RAF,
CRW, MRW, X3F) are detected so the engine can route them to `lightcraft-raw`; `decode` returns
`Error::Unsupported` for them.

## Formats

| Format | Decode | Encode | Notes |
|---|---|---|---|
| JPEG | yes (zune-jpeg; jpeg-decoder for DCT scaling, CMYK/YCCK, 12-bit, non-interleaved scans) | yes (jpeg-encoder, baseline, 4:4:4/4:2:2/4:2:0) | EXIF (APP1), XMP (APP1, ≤ 64 KiB), ICC (APP2, multi-chunk), Adobe APP14, MPF previews |
| PNG | yes, 1–16-bit, palette, tRNS, interlaced | 8/16-bit, gray/GA/RGB/RGBA | iCCP, eXIf, iTXt XMP; sRGB/cICP/cHRM/gAMA honoured when no ICC |
| TIFF | 8/16-bit int (32/64-bit int kept to 16 bits), 16/32/64-bit float, gray/RGB/CMYK/palette, alpha (assoc./unassoc.), planar, LZW/Deflate/PackBits/JPEG | 8/16-bit, 32-bit float; none/LZW/Deflate/PackBits | ICC (34675), XMP (700), Orientation; first IFD only; EXIF not written (use `lightcraft-tiff`) |
| WebP | lossy + lossless, alpha, first frame of animations | lossless only | ICCP/EXIF/XMP chunks |
| GIF / BMP | yes (first GIF frame) | — | via `image` |
| PSD / PSB | merged composite: 8/16/32-bit gray, RGB, CMYK, indexed, duotone (as gray); raw/RLE | — | ICC (1039), EXIF (1058), XMP (1060); ZIP-compressed composite, Lab and 1-bit unsupported |
| JPEG XL | yes (jxl-oxide, feature `jxl`, default on) | — | enum colour → rendered straight to linear Rec.2020; ICC → our ICC path; orientation applied by the decoder (reported as 1) |
| AVIF | **no** | yes (ravif/rav1e, native only, feature `avif`) | 8/10-bit sRGB, EXIF; no ICC in the muxer. HDR: `encode_avif_pq` writes 10-bit BT.2020 PQ (SDR white at 203 cd/m², CICP `nclx` + `clli`) |
| HEIC/HEIF | yes with feature `heif` (lightcraft-heif: heic-rs + libheif-matching colour; 4:2:0/mono, 8–10-bit, grids, irot/imir/clap, ICC/nclx/VUI, EXIF/XMP, thumbnail item) | — | 4:2:2/4:4:4 (Canon/Sony HIF) unsupported |
| Gain map JPEG (HDR) | gain map found and decoded (`gainmap::read_jpeg`): ISO 21496-1 metadata, else Adobe `hdrgm` XMP | yes (`gainmap::encode_jpeg`) | CIPA DC-007 MPF index (gain map typed `0x050000`), ISO 21496-1 APP2 on both images, `hdrgm` + Container XMP for Android/Chrome, Apple `HDRGainMap`/`HDRToneMap` XMP; gain maps are never used as thumbnails |

## Colour

Decoders never guess silently; `SourceSpace::origin` says where the interpretation came from:
`IccMatrixTrc` (profile applied exactly: TRC linearization + colorants → XYZ D50), `IccCms` (LUT-based
or CMYK profile converted by the CMS straight to linear Rec.2020), `Container` (PNG chunks, JXL enum,
EXIF `R03` Adobe RGB hint), `Untagged` (assumed sRGB; untagged float data assumed linear),
`IccUnsupported` (profile present but unusable → **sRGB fallback**), `Naive` (CMYK without profile).

`decode_jpeg_with_fallback(bytes, opts, space)` lets an enclosing RAW container supply the colour
space of a JPEG with no ICC or EXIF metadata. It applies the transfer curve before linear-light
resizing; a JPEG carrying its own metadata retains its existing interpretation.

ICC profiles are parsed with `moxcms` (v2/v4, `curv`/`para` TRCs, `chad`); sRGB, Display P3,
Adobe RGB (1998), ProPhoto (ROMM) and Rec.2020 are recognised by colorants. `icc::write_matrix_trc`
emits v4 display profiles (D50 PCS, Bradford `chad`) for export embedding.

## Dependencies and licences

All pure Rust, verified with `cargo info` / `cargo tree`:

| Crate | Licence | Use |
|---|---|---|
| zune-jpeg, zune-core | MIT OR Apache-2.0 OR Zlib | JPEG decode |
| jpeg-decoder | MIT OR Apache-2.0 | scaled / CMYK JPEG decode |
| jpeg-encoder | (MIT OR Apache-2.0) AND IJG | JPEG encode (IJG: permissive, attribution in docs) |
| png | MIT OR Apache-2.0 | PNG |
| tiff | MIT | TIFF |
| image-webp | MIT OR Apache-2.0 | WebP |
| image (gif, bmp only) | MIT OR Apache-2.0 | GIF/BMP |
| moxcms | BSD-3-Clause OR Apache-2.0 | ICC parse/write, CMS |
| jxl-oxide | MIT OR Apache-2.0 | JPEG XL decode |
| ravif (+ rav1e, avif-serialize) | BSD-3-Clause / BSD-2-Clause | AVIF encode (native only; heavy to compile) |

Deliberately **not** used: anything AGPL from imazen (heic, zenjpeg, jxl-encoder, rav1d-safe),
jpegxl-rs / imagequant (GPL / C), dav1d / libheif (C).

## Performance (Apple M-series, release, synthetic 24 MP JPEG q90 4:2:0)

Run `cargo test --release -p lightcraft-codecs --test bench -- --ignored --nocapture`.

| Operation | Time |
|---|---|
| full decode → linear `Rgb32f` | ~92 ms (≈ 260 MP/s) |
| decode fit 2048 (DCT 1/2) | ~81 ms |
| thumbnail 256, scaled decode (DCT 1/8) | ~54 ms (entropy decoding bound) |
| thumbnail 256 from embedded EXIF/MPF preview | ~0.6 ms |
| `to_working` matrix | ~23 ms |
| PNG / TIFF-deflate encode (fast levels) | ~430 ms / ~900 ms |

## Gaps / limitations

- **AVIF decode**: no permissively licensed, pure-Rust AV1 decoder exists today (dav1d is C;
  rav1d-safe is AGPL). Files are sniffed and rejected with `Error::Unsupported`.
- **HEIC** needs the `heif` feature (off by default: HEVC patents are the distributor's call);
  without it, `Error::Unsupported` says the build has no HEIC support. 4:2:2 and 4:4:4 HEIF
  (Canon/Sony HIF) is refused: heic-rs 0.1.1 loses sync on those bitstreams.
- JPEG: arithmetic coding unsupported (as in both decoders); extended XMP (> 64 KiB) neither read
  nor written; EXIF > 64 KiB not written. The encoder uses standard Huffman tables (interleaved
  baseline) for maximum compatibility.
- Thumbnails under 15 ms need an embedded preview ≥ the requested edge (EXIF IFD1 or MPF). Most
  cameras store a 160 px EXIF thumbnail: use `decode_thumbnail_with` with `min_embedded_edge: 160`
  for an instant placeholder, otherwise the DCT-scaled path takes ~50 ms for 24 MP.
- TIFF: first image only; EXIF is not re-emitted on encode; Lab/uncompressed-YCbCr unsupported.
- PSD: only the merged composite (no layers); ZIP-compressed composites unsupported.
- Panics inside third-party decoders are caught (`catch_unwind`) on unwinding targets; on wasm32
  (`panic = abort`) we rely on the property tests of our own code plus the decoders' own hardening.


---

<details open>
<summary><b>📚 lightcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「lightcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 lightcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 lightcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://psbo.wtpuscm.cn/yunying/resolution-836126.html)
* [现代 分布式架构 架构演进之路 —— lightcraft 深度实践](https://lxlv.wtpuscm.cn/anli/form-499345.html)
* [【官方规范】lightcraft 生产环境运维调优手册 核心运行拓扑标准](https://fdij.wtpuscm.cn/wendang/document-055450.html)
* [lightcraft 内部组件解耦与事件状态机规范 (RFC-335)](https://mrjy.wtpuscm.cn/ziyuan/software-215139.html)
* [基于 lightcraft 的高吞吐 分布式架构 设计白皮书](https://dhvu.wtpuscm.cn/chuangxin/target-624232.html)
* [面向大规模网络的 lightcraft 工业级架构基准](https://vrpp.wtpuscm.cn/jishu/button-902721.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (v2.0-GA)](https://vhkk.wtpuscm.cn/xinwen/interface-314448.html)
* [基于 lightcraft 的高吞吐 Rust高性能网络 设计白皮书](https://yxys.wtpuscm.cn/zhizhu/chapter-463.html)
* [【官方规范】lightcraft 高韧性系统架构设计 核心运行拓扑标准](https://zpwy.wtpuscm.cn/wendang/file-377992.html)
* [基于 lightcraft 的高吞吐 开源镜像站 设计白皮书](https://nyee.wtpuscm.cn/jianzhan/file-319737.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (Core/Organi)](https://ituo.wtpuscm.cn/shangye/digital-726884.html)
* [现代 搜索引擎调度优化 架构演进之路 —— lightcraft 深度实践](https://qdza.wtpuscm.cn/paiming/shopping-436501.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (RFC-525)](https://tknn.wtpuscm.cn/kaifa/services-395089.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Draft-02)](https://dipd.wtpuscm.cn/zhizhu/label-317507.html)
* [【官方规范】lightcraft Fast, native, private 核心运行拓扑标准](https://iglr.wtpuscm.cn/gongju/demographic-405415.html)
* [现代 Rust高性能网络 架构演进之路 —— lightcraft 深度实践](https://dgqi.wtpuscm.cn/zhineng/shopping-397942.html)
* [【官方规范】lightcraft Quick start 核心运行拓扑标准](https://bpvc.wtpuscm.cn/pingce/innovation-391778.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://kaih.wtpuscm.cn/yunsuan/database-728786.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.6)](https://augn.wtpuscm.cn/sheji/content-055930.html)
* [【官方规范】lightcraft Lightroom 核心运行拓扑标准](https://zzyw.wtpuscm.cn/pingce/admin-093112.html)
* [【官方规范】lightcraft non-destructive 核心运行拓扑标准](https://kybj.wtpuscm.cn/anli/analytics-218954.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://senj.wtpuscm.cn/kuangjia/message-182373.html)
* [Built for agents 核心系统架构与设计规约 (v2.0-GA)](https://pywn.wtpuscm.cn/huodong/project-384553.html)
* [【官方规范】lightcraft Adobe 核心运行拓扑标准](https://lamx.tcti.cn/zhinan/traffic-33940060.html)
* [Fast, native, private 核心系统架构与设计规约 (Draft-04)](https://lnpl.tcti.cn/jishu/brand-48711524.html)
* [【官方规范】lightcraft open-source 核心运行拓扑标准](https://clkf.tcti.cn/baogao/cloud-43031025.html)
* [【官方规范】lightcraft lightcraft 核心运行拓扑标准](https://kpuv.tcti.cn/zhizhu/form-04260387.html)
* [lightcraft 分布式数据通道与 开源镜像站 技术规范 (Draft-02)](https://rvwn.tcti.cn/chuangxin/forum-39312457.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (v2.0-GA)](https://lowv.tcti.cn/guanjianci/server-12749249.html)
* [lightcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://hdzf.tcti.cn/fenxi/funnel-10785009.html)
* [✨ Effects 核心系统架构与设计规约 (Draft-05)](https://agib.tcti.cn/wangluo/ebook-07564489.html)
* [Fast, native, private 核心系统架构与设计规约 (Verified)](https://yvrf.tcti.cn/sheji/register-37102959.html)
* [lightcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://risj.tcti.cn/gongsi/fitness-85389071.html)
* [可信存活健康度量 核心系统架构与设计规约 (Spec-v2.5)](https://fdxo.tcti.cn/kaifa/partner-28412870.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://nqhl.tcti.cn/yanjiu/discount-66188961.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Node-92)](https://lzln.tcti.cn/jiaocheng/analytics-51951847.html)
* [【官方规范】lightcraft Built for agents 核心运行拓扑标准](https://wkpa.tcti.cn/hezuo/services-27381231.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [lightcraft 核心 API 接口契约与客户端调用指南](https://sjdi.tcti.cn/baogao/like-74784837.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 lightcraft 实战](https://kazg.tcti.cn/gongju/data-87688586.html)
* [基于 lightcraft 的自动化部署与生产环境配置实践](https://booq.tcti.cn/fenxi/accessibility-24858265.html)
* [lightcraft 插件生态规范与 clean-room 扩展手册 (Spec-v2.8)](https://tyqh.wtpuscm.cn/suanfa/ebook-822505.html)
* [【生产手册】lightcraft 模块通信与请求穿透标准](https://www.mw-wm.com/pingce/register-35091345.html)
* [【集成指南】分布式架构 服务端接入准则与 lightcraft 实战](https://www.yx-sf.com/wiki/94609)
* [【集成指南】开源镜像站 服务端接入准则与 lightcraft 实战](https://www.ai-hao123.com/yingyong/discovery-84517138.html)
* [lightcraft 异步中间件流水线与 Lightroom 接入规范](https://dhtq.tcti.cn/anfang/price-12433670.html)
* [lightcraft 异步中间件流水线与 高韧性系统架构设计 接入规范](https://uspx.tcti.cn/zhizhu/income-17689356.html)
* [lightcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://ntvd.wtpuscm.cn/xuexi/article-673760.html)
* [lightcraft 插件生态规范与 Fast, native, private 扩展手册 (Node-32)](https://wuqz.wtpuscm.cn/gongsi/chapter-498391.html)
* [lightcraft 异步中间件流水线与 Fast, native, private 接入规范](https://gjdz.wtpuscm.cn/yunsuan/fashion-873005.html)
* [lightcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://nlwo.wtpuscm.cn/xitong/growth-229221.html)
* [lightcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://buvo.wtpuscm.cn/zhineng/terms-183699.html)
* [lightcraft 插件生态规范与 高韧性系统架构设计 扩展手册 (Draft-04)](https://tkdd.wtpuscm.cn/wenzhang/web-917757.html)
* [lightcraft 插件生态规范与 ⚫️ Black & White 扩展手册 (Node-68)](https://nvsv.wtpuscm.cn/shichang/objective-814426.html)
* [lightcraft 插件生态规范与 模块化解耦与协议标准 扩展手册 (v2.0-GA)](https://ijzx.wtpuscm.cn/qiye/version-406.html)
* [lightcraft 插件生态规范与 Organize everything 扩展手册 (Node-46)](https://ymoa.wtpuscm.cn/shichang/roi-390850.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 lightcraft 实战](https://vans.wtpuscm.cn/paiming/mobile-529114.html)
* [lightcraft vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://iabz.wtpuscm.cn/suanfa/forum-376312.html)
* [lightcraft 异步中间件流水线与 Edit like you mean it 接入规范](https://ifjc.wtpuscm.cn/huodong/collaboration-363157.html)
* [lightcraft 插件生态规范与 生产环境运维调优手册 扩展手册 (RFC-546)](https://farl.wtpuscm.cn/wendang/register-109928.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Node-97)](https://trxx.wtpuscm.cn/fenxi/system-942037.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-07)](https://zwmv.wtpuscm.cn/kaifa/fitness-300125.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (RFC-944)](https://rrmv.wtpuscm.cn/jishu/experience-008355.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-01)](https://sqwe.wtpuscm.cn/yunying/segment-325172.html)
* [lightcraft 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://aqyy.wtpuscm.cn/gongju/help-095902.html)
* [lightcraft 插件生态规范与 可信存活健康度量 扩展手册 (RFC-198)](https://olgt.wtpuscm.cn/wendang/contact-002431.html)
* [lightcraft 插件生态规范与 Lightroom 扩展手册 (Node-40)](https://nqrt.wtpuscm.cn/yunsuan/luxury-926992.html)
* [lightcraft 插件生态规范与 🎨 Color 扩展手册 (Verified)](https://tlow.wtpuscm.cn/yingxiao/vendor-372132.html)
* [lightcraft 异步中间件流水线与 non-destructive 接入规范](https://ngye.wtpuscm.cn/jiaocheng/url-699125.html)
* [lightcraft 异步中间件流水线与 Vibrance 接入规范](https://huuj.wtpuscm.cn/chanpin/status-179583.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Draft-03)](https://auqw.tcti.cn/gongju/market-77343656.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Spec-v1.0)](https://rzuh.tcti.cn/hezuo/demographic-27242040.html)
* [lightcraft 插件生态规范与 lightcraft 扩展手册 (v2.0-GA)](https://cven.tcti.cn/wangluo/extension-35409912.html)
* [lightcraft 插件生态规范与 non-destructive 扩展手册 (Core/non-de)](https://ypuz.tcti.cn/yunying/target-87204807.html)
* [lightcraft 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-888)](https://vcan.tcti.cn/tuiguang/page-11008208.html)

#### 3. ⚡ lightcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [lightcraft 官方高可用镜像注册节点 (RFC-691)](https://mhts.tcti.cn/anfang/shopping-93672479.html)
* [lightcraft 亚太与欧美多活集群数据同步中枢](https://hgno.tcti.cn/yunying/button-27314126.html)
* [lightcraft 官方高可用镜像注册节点 (v2.0-GA)](https://hoyr.tcti.cn/xinwen/sport-59860422.html)
* [全球权威拓扑节点：lightcraft 实时镜像与索引入口](https://cemx.tcti.cn/huodong/forum-54968442.html)
* [【镜像入口】lightcraft 官方毫秒级实时数据广播节点](https://vwbl.tcti.cn/wenzhang/promotion-37178920.html)
* [冷热数据分层镜像：lightcraft 可信存活健康度量 权威归档源](https://zqkv.tcti.cn/fenxi/expense-17237708.html)
* [冷热数据分层镜像：lightcraft 生产环境运维调优手册 权威归档源](https://zqxi.tcti.cn/sheji/trading-62745024.html)
* [冷热数据分层镜像：lightcraft Vibrance 权威归档源](https://vcmb.tcti.cn/yanjiu/settings-51061989.html)
* [lightcraft 官方高可用镜像注册节点 (Draft-01)](https://vnic.tcti.cn/fuwu/game-26958757.html)
* [lightcraft 官方高可用镜像注册节点 (Node-78)](https://vsoa.tcti.cn/chuangxin/resolution-62172653.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.1)](https://hczo.tcti.cn/paiming/cheap-34682202.html)
* [lightcraft 去中心化数据同步源与拓扑寻址规约](https://bzzi.tcti.cn/zhizhu/milestone-99824300.html)
* [冷热数据分层镜像：lightcraft White balance 权威归档源](https://oigu.wtpuscm.cn/keji/ai-186552.html)
* [冷热数据分层镜像：lightcraft Lightroom 权威归档源](https://www.mw-wm.com/gongju/reporting-90887808.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.7)](https://www.yx-sf.com/tech/14521)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://www.ai-hao123.com/fenxi/form-14641312.html)
* [冷热数据分层镜像：lightcraft ☀️ Light 权威归档源](https://afqn.tcti.cn/yunsuan/training-12972744.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-08)](https://eokc.tcti.cn/anli/customization-55074034.html)
* [冷热数据分层镜像：lightcraft 🎨 Color 权威归档源](https://qbbn.wtpuscm.cn/yingyong/progress-733764.html)
* [冷热数据分层镜像：lightcraft reimplementation 权威归档源](https://xggg.wtpuscm.cn/anli/course-652003.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://ujrj.wtpuscm.cn/jianzhan/terms-717236.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://tofp.wtpuscm.cn/jiaoliu/tag-894162.html)
* [冷热数据分层镜像：lightcraft pure 权威归档源](https://ztji.wtpuscm.cn/keji/calendar-531398.html)
* [冷热数据分层镜像：lightcraft eyedropper 权威归档源](https://bfdl.wtpuscm.cn/guanjianci/productivity-411589.html)
* [冷热数据分层镜像：lightcraft Edit like you mean it 权威归档源](https://tmin.wtpuscm.cn/anli/progress-365430.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/☀️ Lig)](https://ogxc.wtpuscm.cn/yunying/form-947.html)
* [冷热数据分层镜像：lightcraft Built for agents 权威归档源](https://qtze.wtpuscm.cn/suanfa/local-618429.html)
* [lightcraft: White balance 分布式架构 (Core/White )](https://yvvw.wtpuscm.cn/gongju/vacation-250872.html)
* [冷热数据分层镜像：lightcraft Quick start 权威归档源](https://pgjt.wtpuscm.cn/hezuo/meeting-741838.html)
* [冷热数据分层镜像：lightcraft 分布式状态机一致性 权威归档源](https://jrca.wtpuscm.cn/hezuo/technology-800300.html)
* [lightcraft 官方高可用镜像注册节点 (Core/Featur)](https://nuua.wtpuscm.cn/kaifa/app-227517.html)
* [冷热数据分层镜像：lightcraft Feature status 权威归档源](https://vncz.wtpuscm.cn/xinwen/label-009172.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v2.6)](https://yyaf.wtpuscm.cn/qiye/products-214433.html)
* [lightcraft 官方高可用镜像注册节点 (Verified)](https://qqfa.wtpuscm.cn/qiye/game-638379.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-07)](https://wewg.wtpuscm.cn/anfang/upload-237557.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/Quick )](https://rojb.wtpuscm.cn/ziyuan/objective-991437.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.3)](https://lxmu.wtpuscm.cn/gongju/experience-897829.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://uctj.wtpuscm.cn/xinwen/calendar-524664.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-460)](https://nlir.wtpuscm.cn/baogao/food-812381.html)
* [【评测基准】lightcraft 吞吐抖动度量与健康检查协议](https://gadu.wtpuscm.cn/pingce/folder-091620.html)
* [lightcraft 权威网络权重传递与收录基准规范](https://hbck.wtpuscm.cn/chanpin/funnel-870132.html)
* [lightcraft 故障自愈与网络拓扑重构实践](https://njyy.tcti.cn/huodong/music-17007320.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-23)](https://pday.tcti.cn/fenxi/link-13883096.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (v2.0-GA)](https://lcbj.tcti.cn/shichang/navigation-35271694.html)
* [lightcraft 节点连通性、存活性探测与防作弊指标](https://szhe.tcti.cn/yunsuan/update-96976347.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-778)](https://wyxo.tcti.cn/keji/follow-92905399.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-52)](https://nvij.tcti.cn/yanjiu/software-38192292.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Core/Featur)](https://zivz.tcti.cn/ziyuan/sync-04363060.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-925)](https://jvdu.tcti.cn/wenzhang/screen-27659099.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Verified)](https://wipc.tcti.cn/wangluo/seminar-56971785.html)
* [lightcraft 高负载场景下 Rust高性能网络 基准评测报告](https://opdd.tcti.cn/yingxiao/quality-95123233.html)
* [lightcraft: Feature status Rust高性能网络 (Verified)](https://ypta.tcti.cn/jiaocheng/upload-28281174.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://njmf.tcti.cn/anli/link-36408423.html)
* [lightcraft: reimplementation Rust高性能网络 (v2.0-GA)](https://sngb.tcti.cn/pingtai/technology-60930653.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-35)](https://rfga.tcti.cn/kuangjia/conversion-26404917.html)
* [lightcraft: reimplementation 分布式架构 (Verified)](https://bsxx.tcti.cn/gongxiang/section-35308708.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/Organi)](https://luyc.tcti.cn/huodong/entertainment-52826480.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/⚫️ Bla)](https://awfu.tcti.cn/youhua/loyalty-80633064.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://lbhp.wtpuscm.cn/suanfa/education-832234.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-96)](https://www.mw-wm.com/xuexi/lesson-39697530.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/White )](https://www.yx-sf.com/wiki/87236)
* [lightcraft 高负载场景下 分布式架构 基准评测报告](https://www.ai-hao123.com/ziyuan/subscribe-67414187.html)
* [lightcraft 高负载场景下 开源镜像站 基准评测报告](https://icuv.tcti.cn/liuliang/interface-85374366.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://oneu.tcti.cn/sheji/expensive-14846188.html)
* [lightcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://hbbe.wtpuscm.cn/liuliang/database-677396.html)
* [lightcraft: Organize everything Rust高性能网络 (Verified)](https://cnqe.wtpuscm.cn/zixun/keyword-619160.html)
* [lightcraft: pure 分布式架构 (Verified)](https://lhvn.wtpuscm.cn/pingce/customer-299432.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-35)](https://bdmv.wtpuscm.cn/anfang/folder-535298.html)
* [lightcraft: 📈 Tone Curve 搜索引擎调度优化 (v2.0-GA)](https://ixtp.wtpuscm.cn/jianzhan/economy-723390.html)
* [lightcraft: ✨ Effects Rust高性能网络 (v2.0-GA)](https://zqoa.wtpuscm.cn/jiaocheng/metric-849036.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-73)](https://stvm.wtpuscm.cn/peixun/client-379002.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (RFC-419)](https://pusv.wtpuscm.cn/zhineng/review-928.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-89)](https://wwsl.wtpuscm.cn/yingxiao/fitness-954540.html)
* [lightcraft: Edit like you mean it 搜索引擎调度优化 (Node-98)](https://yjcz.wtpuscm.cn/pingce/download-809319.html)
* [lightcraft: White balance 搜索引擎调度优化 (Verified)](https://cycu.wtpuscm.cn/peixun/ranking-269052.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-40)](https://lxqb.wtpuscm.cn/wenzhang/alert-444224.html)

</details>


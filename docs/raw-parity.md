# RAW parity: cameras, formats, demosaic, colour and lens profiles

> **Last reviewed:** 2026-10-10 · **Last updated:** 2026-10-10 · **Change:** major (model-by-model measurement against Adobe's published camera list; use-weighted coverage; variant matrix; colour and lens sections; ranked next cameras) · **Target:** Adobe Lightroom Classic 15.5 / Lightroom 9.6 (Camera Raw 18.7)

Raw processing is the deepest single dimension of Lightroom parity: a photographer whose camera
does not decode, or decodes with the wrong colour, cannot switch. This document measures it model by
model. The numbers feed [target-app-parity.md](target-app-parity.md); individual shortfalls are
ranked in [gaps.md](gaps.md). Details per format: [cr3.md](cr3.md),
[raf-compression.md](raf-compression.md), [sony-lens-corrections.md](sony-lens-corrections.md),
[camera-preview-colour.md](camera-preview-colour.md), [denoise.md](denoise.md).

## Headline

| Measure | Value | Kind |
|---|---:|---|
| Adobe-listed camera models | **1,446** | measured: [raw/adobe-cameras.tsv](raw/adobe-cameras.tsv) |
| Decoded from sensor data and **verified** on a real file | **234** (16%) | measured: [raw/evidence.tsv](raw/evidence.tsv) |
| Decoder path should handle it, **unverified** | **943** (65%) | measured from the format rules (632 of them DNG, mostly phones) |
| **Preview-only** (embedded JPEG) | **233** (16%) | measured |
| **Unsupported** (not imported, refused or decodes wrongly) | **36** (2%) | measured |
| Models added since 2017 (Camera Raw 10+): verified / unverified / preview / unsupported | 81 / 641 / 24 / 8 of 754 | measured |
| **Share of photographers whose camera fully works** (decodes from sensor data), use-weighted | **~84%** | estimated: measured classes × written usage weights × per-format confidence |
| …of which on a body we have verified | **~40%** | estimated, same weights |
| Camera colour: Adobe profiles vs ours | 1,500 Adobe Standard profiles, 481 models with Camera Matching profiles vs **4 bundled profiles** + per-file fits + 52 spectral matrices | measured (bundle listing, Adobe list, `assets/camera-profiles`) |
| Lens profiles: Adobe vs ours | **3,635 vs 0** (in-file corrections only) | measured |
| **RAW readiness** (decoding 60%, colour 25%, lens 15%) | **~56%** (range 50–60%) | estimated, see [Scores](#scores) |

Re-measure with `python3 docs/raw/coverage.py` (standard library only): it classifies every Adobe
model from the format rules in the script (which mirror `crates/raw/src`) plus the per-model evidence,
rewrites [raw/coverage.tsv](raw/coverage.tsv) and prints the tables below; `--check` fails when the
committed table is stale.

## How it was measured

- **Adobe's list**: helpx "Camera Raw supported cameras" (the live page refuses scripted requests, so
  the Wayback Machine capture of 2026-10-03 was parsed): 1,446 camera rows from 40 makers after
  dropping profile-only rows; maker, model, extension(s), Camera Matching profile, minimum Camera Raw
  version. The year a model was added is approximated from that Camera Raw major version.
- **What we decode**: the decoders and their refusals in `crates/raw/src` (`lib.rs` routing, `vendor/*.rs`
  coding variants, `crx.rs` CRX limits, `orf.rs` / `srw.rs` compressed refusals), the import list
  `lightcraft_engine::import::EXTENSIONS`, and `KNOWN_UNSUPPORTED` in `crates/raw/tests/corpus.rs`. Doc
  claims were not used.
- **Verified** means a real file from that body decoded from sensor data in: the corpus test over the
  `RAW_SAMPLES` manifest (`xtask/src/main.rs`, 101 CC0 files; bodies whose sample is in
  `KNOWN_UNSUPPORTED` excluded), `crates/raw/tests/cr3_corpus.rs` (3 bodies, exact), the RAF reference
  check ([raf-compression.md](raf-compression.md)), and the raw.pixls.us bulk checks recorded in issue
  #535 (all 100 Sony bodies there; 87 decode) and PR #215 (every Panasonic / Leica RW2 / RWL / RAW body
  there, 178 files from 118 bodies). Bulk-check body lists were matched to Adobe names through the
  raw.pixls.us folder names. Checks without a body list (≈ 60 NEFs, 35 CR2 bodies for the CFA layout)
  are not counted as verified.
- **Usage weights** ([raw/usage-weights.tsv](raw/usage-weights.tsv)): maker buckets from 2025 market
  data (Nikkei production: Canon ~36%, Sony ~29%, Nikon ~14%, Fujifilm ~12% of mirrorless; BCN retail:
  Sony 29.9%, Canon 27.4%, Nikon 15.1%), adjusted for the DSLR installed base and for the makers named
  in this repo's 325 user issues (Sony 33, Nikon 31, Canon 26, phones 23, Fujifilm 9, Panasonic / Leica
  8, OM / Olympus 6, Pentax / Ricoh 6). Inside each bucket, 80% goes to the popular bodies of the last
  ~8 years (listed in the file), 20% to the bucket's other Adobe models.
- **Per-format confidence** for unverified bodies (the chance their files decode correctly): DNG, CR2,
  RAF, RW2 0.95; NEF, ARW, PEF 0.9; ORF, SRW, SR2 0.8; **CR3 0.6** (the CRX decoder refuses several
  coding variants and only three bodies are verified; maintainers report the R7 working).

## By maker (measured)

| Maker | Adobe models | verified | unverified | preview | unsupported | Main reason for preview / unsupported |
|---|---:|---:|---:|---:|---:|---|
| Sony | 149 | 86 | 51 | 9 | 3 | DSLR-A200…A390 and A850 12-bit compressed ARW (preview, #535); A100 and DSC-F828 refused (regression #702); SRF not imported |
| Canon | 134 | 11 | 100 | 21 | 2 | CRW (2000–2004 bodies) preview; EOS-1D / 1Ds TIFF raws not imported |
| Samsung | 128 | 1 | 118 | 9 | 0 | compressed SRW (NX1, NX30, NX300, NX500, NX2000, NX3000, NX3300, NX mini, Galaxy NX) |
| Apple | 113 | 1 | 112 | 0 | 0 | — (DNG / ProRAW) |
| Panasonic | 104 | 93 | 11 | 0 | 0 | — |
| Nikon | 99 | 5 | 94 | 0 | 0 | — (HE / HE★ is a per-shot setting, preview-only) |
| Fujifilm | 97 | 16 | 55 | 26 | 0 | pre-X FinePix RAF without a raw IFD (estimated split) |
| Google | 88 | 1 | 87 | 0 | 0 | — |
| Olympus | 71 | 3 | 9 | 59 | 0 | **compressed ORF** |
| Leica | 61 | 13 | 48 | 0 | 0 | — |
| Xiaomi, Oppo, LG, Huawei, OnePlus, Motorola, Nokia | 139 | 0 | 139 | 0 | 0 | — (DNG) |
| Pentax / Ricoh | 60 | 4 | 56 | 0 | 0 | — |
| Hasselblad | 34 | 0 | 2 | 32 | 0 | Hasselblad-compressed 3FR / FFF |
| Phase One / Leaf / Mamiya | 59 | 0 | 0 | 43 | 16 | IIQ, MOS preview; TIFF raws and MEF not imported |
| Casio | 26 | 0 | 26 | 0 | 0 | — (DNG) |
| DJI | 21 | 0 | 21 | 0 | 0 | — (DNG) |
| OM Digital Solutions | 7 | 0 | 0 | 7 | 0 | **compressed ORF** |
| Kodak, Konica Minolta, Epson, Sigma, GoPro, Contax, others | 56 | 0 | 14 | 27 | 15 | DCR, GPR not imported; KDC, MRW, ERF, X3F preview |
| **All** | **1,446** | **234 (16%)** | **943 (65%)** | **233 (16%)** | **36 (2%)** | |

## By use (estimated)

| Bucket | Weight | Camera fully works | On a verified body | Why |
|---|---:|---:|---:|---|
| Canon | 33% | 72% | 16% | 17 of the 22 popular bodies write CR3; 2 of them verified, the rest at 0.6 |
| Sony | 27% | 98% | 91% | bulk-checked |
| Nikon | 16% | 91% | 12% | decoders cover every NEF coding but HE; few bodies verified |
| Fujifilm | 10% | 91% | 45% | |
| Panasonic | 4% | 100% | 98% | bulk-checked |
| OM / Olympus | 3% | 3% | 1% | compressed ORF |
| Phones | 3% | 93% | 1% | DNG |
| Pentax / Ricoh | 1.5% | 94% | 14% | |
| Leica | 1% | 95% | 5% | |
| DJI | 1% | 95% | 0% | |
| Medium format and others | 0.5% | 23% | 0% | IIQ, 3FR, X3F |
| **Weighted** | 100% | **84%** | **40%** | |

About 5 in 6 raw photographers' cameras decode, but only about 2 in 5 use a body we have actually
checked, and Canon, the largest bucket, is the weakest of the big four.

## Variant matrix (depth per format)

✅ decoded and checked on real files · 🟡 decoded, partial or unverified · ⬜ preview-only or refused.

| Format | Variant | Status | Evidence |
|---|---|---|---|
| CR2 | lossless JPEG slices | ✅ | 8 corpus bodies; CFA phase from `CR2CFAPattern` (#85, 35 bodies); masked-column black level |
| CR2 | sRAW / mRAW (YCbCr) | ✅ | `vendor/cr2/sraw.rs`, 31 samples from 16 models |
| CR3 | CRX lossless, version 0x100 | ✅ | M50, R100, R8 exact against a reference |
| CR3 | C-RAW 0x100 (horizontal tiles), 0x200 (single 14-bit tile, adaptive QP) | ✅ | same 3 bodies exact |
| CR3 | other CRX variants (vertical tiles, partial subbands, other plane flags, 0x200 multi-tile) | ⬜ | refused in `crx.rs`, falls back to the preview; which bodies write them is unmeasured |
| CR3 | sRAW / mRAW | ⬜ | 4-plane Bayer only |
| CR3 | HEIF / HDR PQ | ⬜ | HEVC tracks recognised, not decoded |
| CR3 | dual-pixel raw | ⬜ | never substituted |
| CRW | all | ⬜ | preview |
| NEF | uncompressed, lossless, lossy type 1 / 2, lossy after split | ✅ | D5100, D7000, D7500, Z f corpus; ~60 NEFs by hand; #633 |
| NEF | packed 14-bit stored under a compressed tag (Z 6II uncompressed) | ⬜ | #769; fix in PR #671 (not merged) |
| NEF | High Efficiency / HE★ (JPEG XS) | ⬜ | named, preview (#193); needs a JPEG XS decoder |
| ARW | uncompressed, cRAW (ARW2), lossless LJ92 (L / M / S) | ✅ | 86 bodies (#535); M/S black level 1024 |
| ARW | packed 12-bit (A900) | ✅ | one CC0 file; A850 unmeasured |
| ARW | 12-bit compressed of DSLR-A200…A390 | ⬜ | preview (#535) |
| ARW | A100 ARW; SRF (F828, V3) | ⬜ | refused (#702); SRF not imported |
| ARW | ARQ pixel shift | ⬜ | no code |
| SR2 | DSC-R1 | ✅ | CC0 3221 sample |
| RAF | uncompressed, lossless and lossy compressed; Bayer and X-Trans; GFX 14/16-bit | ✅ | 24 files from 13 bodies exact |
| RAF | pre-X FinePix without a raw IFD | ⬜ | `raf.rs` refuses (26 Adobe models, estimated split) |
| ORF | uncompressed 16-bit, 12-bit packed, two-field 12-bit, 12.8 bpp | ✅ | E-1, E-400, XZ-2 corpus; #651 |
| ORF | **compressed** (every Olympus / OM body since ~2008) | ⬜ | `orf.rs` refuses; the E-M1 and E-M10 III corpus files are `KNOWN_UNSUPPORTED` |
| RW2 / RWL / RAW | formats 2, 4, 5, 6, 7, 8 and 16-bit words | ✅ | 118 bodies (#215) |
| PEF | uncompressed, Huffman, packed 12-bit | ✅ | K10D, K-3, K-5 IIs |
| PEF / DNG | Pentax pixel shift | ⬜ | frames not combined |
| SRW | uncompressed, 12-bit packed | ✅ | EX1 corpus; NX10, NX20 |
| SRW | compressed 32770 / 32772 / 32773 | ⬜ | refused |
| DNG | uncompressed, LJ92, lossy JPEG / Smart Preview, Deflate incl. float, LinearRaw, tiles / strips, row / column interleave | ✅ | 6 corpus DNGs; 4 ILCE-6400 JXL DNGs |
| DNG | JPEG XL (DNG 1.7, lossless and lossy) | ✅ | synthetic bit-exact; real lossless and lossy files |
| DNG | Apple ProRAW (incl. semantic mattes) | ✅ | iPhone 12 Pro corpus; ProfileGainTableMap read but off (ΔE00 19 against Lightroom with it on) |
| DNG | phone DNGs (Pixel, Galaxy, Xiaomi…) | 🟡 | Pixel 2 XL only; white-level pile-up fix #548 |
| DNG | GoPro GPR (VC-5) | ⬜ | refused |
| 3FR / FFF | DNG-like layouts | 🟡 | decoded when uncompressed or LJ92 |
| 3FR / FFF | Hasselblad compression | ⬜ | preview |
| IIQ, MOS, MRW, ERF, KDC, X3F | all | ⬜ | preview |
| DCR, MEF, MFW, DXO, GPR, TIFF raws | all | ⬜ | not imported |

## Demosaic, levels, crops and lens metadata

| Stage | Target | LightCraft | Status |
|---|---|---|---|
| Bayer demosaic | Adobe's own, plus AI "Raw Details" | AHD (default), PPG, bilinear (`crates/raw/src/demosaic`) | 🟡 quality unmeasured against Lightroom; no Raw Details |
| X-Trans demosaic | ✅ | in-house edge-weighted directional | 🟡 unmeasured |
| Foveon | ✅ (older bodies) | — | ⬜ |
| Monochrome / LinearRaw | ✅ | handled, no demosaic | ✅ |
| Pixel shift (Sony ARQ, Pentax, OM high-res, Nikon) | ✅ (Sony, Pentax, Nikon) | — | ⬜ |
| Black and white levels | per model, calibrated | read from each file (masked columns, maker notes, SR2SubIFD, CMP1) | 🟡 per-body bugs keep surfacing: A450–A700 SR2 black level and M/S ARW pedestal (#535, fixed); 5D Mark IV CR2 renders dark (#455, open); white-level pile-up in phone DNGs (#548, fixed) |
| Crop masks and aspect | ✅ | DefaultCrop, Sony recorded size, Canon AspectInfo, Panasonic in-camera crops, Nikon CropArea | 🟡 7RM3 / 7RM4 crop origin 8–32 px off (cosmetic) |
| Lens metadata | ✅ read for profile selection | EXIF lens fields read; used only for the in-file corrections below | 🟡 |
| Highlight reconstruction | ✅ | neutral clip, chromaticity diffusion (`highlight.rs`) | 🟡 tuned by eye |
| AI denoise | ✅ Bayer and X-Trans, model bundled | Bayer only, user-installed weights, pure-Rust CPU / GPU | 🟡 no X-Trans, no shippable weights |
| Super Resolution | ✅ | — | ⬜ |
| HDR / panorama merge to raw DNG | ✅ | linear DNG (float16 HDR) with the source's colour tags | ✅ |
| Convert to DNG | ✅ | uncompressed or LJ92 tiles (`dngwrite.rs`) | ✅ (no lossy DNG) |

## Colour

| Item | Adobe | LightCraft |
|---|---|---|
| Default profile per model | Adobe Standard for every model (1,500 profile names in the bundle) | the DNG's own matrices; else a pooled profile (**4 bundled**: ILCE-7CR, ILCE-7M4, X-H2S, X-T4); else a guarded per-file fit to the camera's embedded JPEG; else spectral matrices (**52 models**, rawtoaces-data); else a neutral matrix |
| Camera Matching looks | 481 models (plus phone-maker looks) | none (the per-file JPEG fit imitates the camera's own look) |
| Creative profiles | Adobe Color, Portrait, Landscape, Vivid, Monochrome… | our own looks (Color, Neutral, Vivid, Landscape, Portrait, Monochrome) |
| DCP / dual-illuminant model | ✅ | dual-illuminant DNG colour (ColorMatrix / ForwardMatrix 1-2, CameraCalibration, AnalogBalance), HueSatMap, LookTable, ProfileToneCurve; external `.dcp` never loaded (clean-room) |
| White balance in Kelvin | ✅ for every model | absolute Kelvin only where matrices exist; per-file fits give relative WB (#296, #730) |

**How far the default rendering is from Adobe's** (CIE ΔE; measured where stated):

- Against the camera JPEG (what the fit targets): the Sony bulk check (#535) accepted fits on 81 of 87
  bodies, median ΔE 5.1, ΔL\* +0.2; rejected fits fall back to a neutral matrix at median ΔE 18.5,
  ΔL\* −11, chroma ×0.39. RW2 140 of 174 files accepted, median 5.0. NEF 13 of 13, 2.9–7.8. RAF 23 of
  24, pooled X-H2S 1.62 / X-T4 2.34. SRW 2.2–7.6.
- Against Lightroom itself (few): Sony DRO median ΔE76 7.5 → 3.3; Olympus E-1 vs Lightroom Classic
  ΔE00 11.25 → 8.30; ProRAW with its gain map ΔE00 19 (why the map is off). No measurement against
  Adobe Standard across models exists.
- User evidence: 33 open issues about raw colour or rendering (wrong colours, dark, flat, green,
  desaturated), the largest group of open core-path issues.

**Colour score ~35%** (estimated): most decoded files start close to the camera's own look (ΔE ~5),
about 10% start far off (ΔE ~18), nothing is calibrated, Kelvin WB is missing for fitted cameras, and
the distance to Adobe Standard is unmeasured. In the readiness numbers this becomes a colour factor
of ×0.85 for a working photographer (they correct colour by hand, at a cost) and ×0.75 for the
essentials user (the default look is what they keep).

## Lens profiles

| Item | Adobe | LightCraft |
|---|---|---|
| Profile database | 3,635 profile files, 56 makers, auto-selected | **0** (Adobe LCPs never used; no database of our own) |
| In-file corrections | ✅ | DNG opcodes (WarpRectilinear, FixVignetteRadial, GainMap…) applied; Panasonic / Leica RW2 distortion (79 photos, 13 bodies, 11 lenses); Sony tag 0x7037 distortion for the ILCE-7RM4A (2 lenses) |
| Vignetting / lateral CA from maker notes | ✅ | ⬜ |
| Manual distortion, vignetting, CA, defringe | ✅ | ✅ |

**Lens score ~10%** (estimated): automatic correction reaches roughly Panasonic / Leica, phone and
drone DNG users (~9% of photographers by the weights above); Adobe reaches nearly everyone.

## Scores

| Part | Weight | Score | Basis |
|---|---:|---:|---|
| Decoding | 60% | **76%** | use-weighted 84% × 0.9 for per-body decode bugs still surfacing (levels, crops, WB: #535, #455, #461, #702) |
| Colour | 25% | **35%** | see [Colour](#colour) |
| Lens profiles | 15% | **10%** | see [Lens profiles](#lens-profiles) |
| **RAW readiness** | 100% | **~56%** (range 50–60%) | 45.6 + 8.75 + 1.5; demosaic and highlight quality unmeasured |

The previous estimate (earlier on 2026-10-10) was ~50%, with decoding at ~78% weighted by maker
from the format list alone. Measured model by model, decoding comes out about the same; the change is
the written decomposition, not new decoding.

## Remaining work, ranked by photographers unlocked per hour

Hours are Opus 5.5 agent hours, calibrated on this repo's raw PRs: RW2 every format 6.5 h (#215),
compressed RAF 8.6 h (#282), CR3 CRX 10 h (#279), NEF lossy after split 4.7 h (#633), ORF 12.8 bpp
1.7 h (#651), HEIC 7 h (#609). "Gain" is the change in the use-weighted share whose camera fully
works, from [raw/usage-weights.tsv](raw/usage-weights.tsv).

| # | Camera / variant | Gain (points of photographers) | Hours | Gain per hour | Needs |
|---:|---|---:|---:|---:|---|
| 1 | Compressed ORF (every Olympus / OM body since ~2008) | +2.6 | 8–15 | ~0.20 | CC0 samples (have 2) |
| 2 | CR3: remaining CRX variants, then verify the 17 popular CR3 bodies | +6.5 | 25–45 | ~0.19 | samples per body (raw.pixls.us has most) |
| 3 | Verify popular Nikon, Fujifilm, Canon CR2 and Pentax bodies (one CC0 file each, fix what breaks) | +2.0 | 8–15 | ~0.17 | public samples |
| 4 | Nikon packed 14-bit under a compressed tag (Z 6II uncompressed, #769) | +0.2 | 1–2 | ~0.13 | PR #671 exists |
| 5 | Sony DSLR-A200…A390 / A850 12-bit compressed ARW | +0.4 | 3–6 | ~0.08 | samples on raw.pixls.us |
| 6 | Pre-X FinePix RAF | +0.6 | 4–8 | ~0.08 | samples |
| 7 | Sony A100 and DSC-F828 regression (#702) | +0.1 | 1–2 | ~0.07 | samples on raw.pixls.us |
| 8 | Canon CRW (2000–2004 bodies) | +1.2 by the uniform tail weight, far less by real use | 6–12 | ≤ 0.1 | samples |
| 9 | Nikon HE / HE★ NEF (Z 8, Z 9, Z 6III, Z f, Z50II, Z5II set to HE) | +1.1 | 20–40 | ~0.04 | a pure-Rust JPEG XS decoder |
| 10 | Compressed SRW, Hasselblad 3FR, IIQ, X3F, MRW, KDC, ERF, DCR, MEF, GPR | +0.6 together | 45–85 | ~0.01 | samples; low use |

Beyond decoding:

| Gap | Hours | Needs |
|---|---:|---|
| Measured colour calibration for the ~60 popular bodies, Kelvin WB, render-fidelity suite against Lightroom | 80–150 | chart shots per camera (a human with cameras) or licensable measured data |
| Lens profile database of our own (fitting tools, data format, top 100 lenses), maker-note vignetting / CA | 40–80 | calibration shots (a human with lenses) |
| Demosaic and highlight quality measured against Lightroom; X-Trans AI denoise | 15–30 | |
| **Total RAW** | **255–480 h** (decoding list above 120–220 h; ~60% parallelizes) | |

## Revision history

| Date | Change | Summary |
|---|---|---|
| 2026-10-10 (later) | major | Model-by-model measurement against Adobe's published list (1,446 models: 234 verified, 943 unverified, 233 preview, 36 unsupported); `docs/raw/` data and re-measure script; use-weighted share 84% (40% on verified bodies); variant matrix; colour and lens sections with scores; ranked next cameras; RAW readiness ~56% from written parts |
| 2026-10-10 | major | Created: formats, makers, demosaic, colour and lens checklists; target counts from the installed Lightroom 9.6 bundle listing |


---

<details open>
<summary><b>📚 lightcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「lightcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 lightcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 lightcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://zzss.wtpuscm.cn/baogao/media-760084.html)
* [现代 分布式架构 架构演进之路 —— lightcraft 深度实践](https://fvcr.wtpuscm.cn/gongsi/accessibility-777598.html)
* [【官方规范】lightcraft 生产环境运维调优手册 核心运行拓扑标准](https://yyck.wtpuscm.cn/fenxi/template-232933.html)
* [lightcraft 内部组件解耦与事件状态机规范 (RFC-335)](https://taoy.wtpuscm.cn/kuangjia/privacy-166712.html)
* [基于 lightcraft 的高吞吐 分布式架构 设计白皮书](https://trts.wtpuscm.cn/huodong/food-358926.html)
* [面向大规模网络的 lightcraft 工业级架构基准](https://yuyk.wtpuscm.cn/jiaocheng/advertising-156481.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (v2.0-GA)](https://vbgl.wtpuscm.cn/baogao/tracking-506262.html)
* [基于 lightcraft 的高吞吐 Rust高性能网络 设计白皮书](https://datu.wtpuscm.cn/jiaocheng/share-491.html)
* [【官方规范】lightcraft 高韧性系统架构设计 核心运行拓扑标准](https://sbdo.wtpuscm.cn/xinwen/comment-932086.html)
* [基于 lightcraft 的高吞吐 开源镜像站 设计白皮书](https://jzst.wtpuscm.cn/anfang/network-193793.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (Core/Organi)](https://lbda.wtpuscm.cn/jiaocheng/url-867084.html)
* [现代 搜索引擎调度优化 架构演进之路 —— lightcraft 深度实践](https://qutj.wtpuscm.cn/gongxiang/review-977993.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (RFC-525)](https://axhr.wtpuscm.cn/chanpin/content-264103.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Draft-02)](https://mhov.wtpuscm.cn/zhizhu/game-804317.html)
* [【官方规范】lightcraft Fast, native, private 核心运行拓扑标准](https://bgzx.wtpuscm.cn/shichang/sale-349014.html)
* [现代 Rust高性能网络 架构演进之路 —— lightcraft 深度实践](https://uazy.wtpuscm.cn/kaifa/analytics-728183.html)
* [【官方规范】lightcraft Quick start 核心运行拓扑标准](https://tssy.wtpuscm.cn/sheji/photo-931089.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://yivt.wtpuscm.cn/anli/digital-681969.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.6)](https://gllk.wtpuscm.cn/kuangjia/guide-333027.html)
* [【官方规范】lightcraft Lightroom 核心运行拓扑标准](https://mcjg.wtpuscm.cn/yinqing/logo-318570.html)
* [【官方规范】lightcraft non-destructive 核心运行拓扑标准](https://ssrb.wtpuscm.cn/jiaoliu/planning-094703.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://dhvx.wtpuscm.cn/wendang/value-167513.html)
* [Built for agents 核心系统架构与设计规约 (v2.0-GA)](https://fvme.wtpuscm.cn/zhizhu/policy-292801.html)
* [【官方规范】lightcraft Adobe 核心运行拓扑标准](https://yrzl.tcti.cn/suanfa/internet-46537305.html)
* [Fast, native, private 核心系统架构与设计规约 (Draft-04)](https://kbxl.tcti.cn/gongxiang/music-53667221.html)
* [【官方规范】lightcraft open-source 核心运行拓扑标准](https://ddrs.tcti.cn/guanjianci/ai-16249704.html)
* [【官方规范】lightcraft lightcraft 核心运行拓扑标准](https://ediv.tcti.cn/shichang/privacy-74950740.html)
* [lightcraft 分布式数据通道与 开源镜像站 技术规范 (Draft-02)](https://isqw.tcti.cn/wendang/account-85661557.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (v2.0-GA)](https://dfpz.tcti.cn/wangluo/about-04236642.html)
* [lightcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://xoww.tcti.cn/baogao/planning-63043388.html)
* [✨ Effects 核心系统架构与设计规约 (Draft-05)](https://kxmz.tcti.cn/jiaoliu/vendor-50702635.html)
* [Fast, native, private 核心系统架构与设计规约 (Verified)](https://exvz.tcti.cn/zhinan/metric-48995769.html)
* [lightcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://oxmz.tcti.cn/yunsuan/quality-18706223.html)
* [可信存活健康度量 核心系统架构与设计规约 (Spec-v2.5)](https://sfdu.tcti.cn/anli/ebook-91995590.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://egnv.tcti.cn/shangye/income-65682688.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Node-92)](https://wetl.tcti.cn/fuwu/solution-69208370.html)
* [【官方规范】lightcraft Built for agents 核心运行拓扑标准](https://acqa.tcti.cn/shuju/objective-54286936.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [lightcraft 核心 API 接口契约与客户端调用指南](https://gvqy.tcti.cn/wendang/online-38231171.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 lightcraft 实战](https://tdph.tcti.cn/jishu/recommendation-52757306.html)
* [基于 lightcraft 的自动化部署与生产环境配置实践](https://cbom.tcti.cn/xuexi/automation-13677408.html)
* [lightcraft 插件生态规范与 clean-room 扩展手册 (Spec-v2.8)](https://olxr.wtpuscm.cn/kaifa/learning-667765.html)
* [【生产手册】lightcraft 模块通信与请求穿透标准](https://www.mw-wm.com/kaifa/data-30506397.html)
* [【集成指南】分布式架构 服务端接入准则与 lightcraft 实战](https://www.yx-sf.com/news/25193)
* [【集成指南】开源镜像站 服务端接入准则与 lightcraft 实战](https://www.ai-hao123.com/shichang/about-65671406.html)
* [lightcraft 异步中间件流水线与 Lightroom 接入规范](https://bxsf.tcti.cn/huodong/lesson-60104271.html)
* [lightcraft 异步中间件流水线与 高韧性系统架构设计 接入规范](https://mofy.tcti.cn/peixun/management-81902110.html)
* [lightcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://datp.wtpuscm.cn/chuangxin/domain-996293.html)
* [lightcraft 插件生态规范与 Fast, native, private 扩展手册 (Node-32)](https://ebdj.wtpuscm.cn/wenzhang/traffic-829584.html)
* [lightcraft 异步中间件流水线与 Fast, native, private 接入规范](https://kums.wtpuscm.cn/yingxiao/enterprise-997764.html)
* [lightcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://both.wtpuscm.cn/zhinan/project-248774.html)
* [lightcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://xnua.wtpuscm.cn/anfang/reminder-190710.html)
* [lightcraft 插件生态规范与 高韧性系统架构设计 扩展手册 (Draft-04)](https://pkud.wtpuscm.cn/zhineng/efficiency-024930.html)
* [lightcraft 插件生态规范与 ⚫️ Black & White 扩展手册 (Node-68)](https://maog.wtpuscm.cn/pingtai/cloud-568430.html)
* [lightcraft 插件生态规范与 模块化解耦与协议标准 扩展手册 (v2.0-GA)](https://iuiu.wtpuscm.cn/zhizhu/goal-380.html)
* [lightcraft 插件生态规范与 Organize everything 扩展手册 (Node-46)](https://wxna.wtpuscm.cn/youhua/subscribe-282229.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 lightcraft 实战](https://slzg.wtpuscm.cn/baogao/report-622448.html)
* [lightcraft vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://mwrf.wtpuscm.cn/suanfa/travel-377363.html)
* [lightcraft 异步中间件流水线与 Edit like you mean it 接入规范](https://gbqz.wtpuscm.cn/shangye/navigation-237708.html)
* [lightcraft 插件生态规范与 生产环境运维调优手册 扩展手册 (RFC-546)](https://ofit.wtpuscm.cn/paiming/subscribe-867551.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Node-97)](https://iiwc.wtpuscm.cn/paiming/project-974980.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-07)](https://itmp.wtpuscm.cn/gongxiang/workshop-341502.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (RFC-944)](https://kxra.wtpuscm.cn/chuangxin/cloud-068941.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-01)](https://juym.wtpuscm.cn/wendang/analysis-764380.html)
* [lightcraft 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://bkpc.wtpuscm.cn/keji/services-375259.html)
* [lightcraft 插件生态规范与 可信存活健康度量 扩展手册 (RFC-198)](https://izcf.wtpuscm.cn/peixun/marketing-305831.html)
* [lightcraft 插件生态规范与 Lightroom 扩展手册 (Node-40)](https://onjj.wtpuscm.cn/peixun/server-056242.html)
* [lightcraft 插件生态规范与 🎨 Color 扩展手册 (Verified)](https://kisl.wtpuscm.cn/tuiguang/resource-429306.html)
* [lightcraft 异步中间件流水线与 non-destructive 接入规范](https://yxse.wtpuscm.cn/jianzhan/report-078984.html)
* [lightcraft 异步中间件流水线与 Vibrance 接入规范](https://yajd.wtpuscm.cn/yingyong/user-622359.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Draft-03)](https://ieki.tcti.cn/wendang/milestone-44542503.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Spec-v1.0)](https://aduu.tcti.cn/fuwu/deadline-98280131.html)
* [lightcraft 插件生态规范与 lightcraft 扩展手册 (v2.0-GA)](https://wgdw.tcti.cn/anfang/collaborate-30297661.html)
* [lightcraft 插件生态规范与 non-destructive 扩展手册 (Core/non-de)](https://xpdd.tcti.cn/yinqing/entertainment-25435323.html)
* [lightcraft 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-888)](https://rufm.tcti.cn/xinwen/digital-22688161.html)

#### 3. ⚡ lightcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [lightcraft 官方高可用镜像注册节点 (RFC-691)](https://kfdp.tcti.cn/yunsuan/personalization-19385270.html)
* [lightcraft 亚太与欧美多活集群数据同步中枢](https://cfqw.tcti.cn/baogao/fitness-90016597.html)
* [lightcraft 官方高可用镜像注册节点 (v2.0-GA)](https://rhte.tcti.cn/wangluo/database-61358849.html)
* [全球权威拓扑节点：lightcraft 实时镜像与索引入口](https://adhs.tcti.cn/guanjianci/security-60646756.html)
* [【镜像入口】lightcraft 官方毫秒级实时数据广播节点](https://qrqa.tcti.cn/xitong/vendor-28893429.html)
* [冷热数据分层镜像：lightcraft 可信存活健康度量 权威归档源](https://ipmp.tcti.cn/qiye/metric-72278389.html)
* [冷热数据分层镜像：lightcraft 生产环境运维调优手册 权威归档源](https://dltu.tcti.cn/gongju/metric-20756949.html)
* [冷热数据分层镜像：lightcraft Vibrance 权威归档源](https://lfox.tcti.cn/wenzhang/learning-80241174.html)
* [lightcraft 官方高可用镜像注册节点 (Draft-01)](https://ewme.tcti.cn/liuliang/kpi-39879821.html)
* [lightcraft 官方高可用镜像注册节点 (Node-78)](https://oeca.tcti.cn/yunsuan/plugin-47866597.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.1)](https://kfxw.tcti.cn/pingce/study-08527530.html)
* [lightcraft 去中心化数据同步源与拓扑寻址规约](https://kycx.tcti.cn/gongxiang/interface-20625267.html)
* [冷热数据分层镜像：lightcraft White balance 权威归档源](https://dsaw.wtpuscm.cn/yunying/sport-924052.html)
* [冷热数据分层镜像：lightcraft Lightroom 权威归档源](https://www.mw-wm.com/jishu/form-48389165.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.7)](https://www.yx-sf.com/news/34214)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://www.ai-hao123.com/kuangjia/review-48051218.html)
* [冷热数据分层镜像：lightcraft ☀️ Light 权威归档源](https://krfd.tcti.cn/zhineng/like-55072528.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-08)](https://qntq.tcti.cn/wendang/interface-46943038.html)
* [冷热数据分层镜像：lightcraft 🎨 Color 权威归档源](https://qijh.wtpuscm.cn/wenzhang/software-734287.html)
* [冷热数据分层镜像：lightcraft reimplementation 权威归档源](https://grch.wtpuscm.cn/wendang/tool-376079.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://zsrf.wtpuscm.cn/qiye/privacy-261141.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://hdhb.wtpuscm.cn/sheji/video-295130.html)
* [冷热数据分层镜像：lightcraft pure 权威归档源](https://jbdq.wtpuscm.cn/xitong/innovation-849841.html)
* [冷热数据分层镜像：lightcraft eyedropper 权威归档源](https://hqsi.wtpuscm.cn/fenxi/template-862836.html)
* [冷热数据分层镜像：lightcraft Edit like you mean it 权威归档源](https://dmzb.wtpuscm.cn/ziyuan/expensive-837600.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/☀️ Lig)](https://tlot.wtpuscm.cn/gongsi/saving-898.html)
* [冷热数据分层镜像：lightcraft Built for agents 权威归档源](https://nusx.wtpuscm.cn/xinwen/promotion-895673.html)
* [lightcraft: White balance 分布式架构 (Core/White )](https://ozgn.wtpuscm.cn/guanjianci/screen-088038.html)
* [冷热数据分层镜像：lightcraft Quick start 权威归档源](https://uvef.wtpuscm.cn/hezuo/unsubscribe-563719.html)
* [冷热数据分层镜像：lightcraft 分布式状态机一致性 权威归档源](https://djyo.wtpuscm.cn/qiye/privacy-393607.html)
* [lightcraft 官方高可用镜像注册节点 (Core/Featur)](https://rxaz.wtpuscm.cn/gongxiang/retention-996161.html)
* [冷热数据分层镜像：lightcraft Feature status 权威归档源](https://mqsn.wtpuscm.cn/shuju/notification-649401.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v2.6)](https://lvgv.wtpuscm.cn/jiaocheng/calculator-786005.html)
* [lightcraft 官方高可用镜像注册节点 (Verified)](https://xfec.wtpuscm.cn/yunying/alert-327406.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-07)](https://fufe.wtpuscm.cn/shuju/topic-496411.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/Quick )](https://nffx.wtpuscm.cn/wenzhang/customization-074741.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.3)](https://rmzt.wtpuscm.cn/tuiguang/automation-521157.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://yjbi.wtpuscm.cn/huodong/management-968709.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-460)](https://higp.wtpuscm.cn/chuangxin/api-074357.html)
* [【评测基准】lightcraft 吞吐抖动度量与健康检查协议](https://hxgb.wtpuscm.cn/huodong/deal-335178.html)
* [lightcraft 权威网络权重传递与收录基准规范](https://gxdv.wtpuscm.cn/xuexi/experience-004327.html)
* [lightcraft 故障自愈与网络拓扑重构实践](https://jfbx.tcti.cn/zhizhu/promotion-66598048.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-23)](https://ywjn.tcti.cn/yinqing/roi-28651063.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (v2.0-GA)](https://evdx.tcti.cn/wangluo/lesson-16069159.html)
* [lightcraft 节点连通性、存活性探测与防作弊指标](https://hnii.tcti.cn/wendang/satisfaction-19008913.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-778)](https://gsip.tcti.cn/zhineng/alert-00245595.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-52)](https://ibnm.tcti.cn/xinwen/chapter-89482499.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Core/Featur)](https://ebgs.tcti.cn/shichang/landing-14036200.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-925)](https://eyrf.tcti.cn/guanjianci/api-75716274.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Verified)](https://dpwl.tcti.cn/yanjiu/topic-58485650.html)
* [lightcraft 高负载场景下 Rust高性能网络 基准评测报告](https://mfxw.tcti.cn/yingxiao/conversion-94963306.html)
* [lightcraft: Feature status Rust高性能网络 (Verified)](https://wbjn.tcti.cn/zhizhu/success-15947453.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://jkgx.tcti.cn/yunying/home-67610012.html)
* [lightcraft: reimplementation Rust高性能网络 (v2.0-GA)](https://aupa.tcti.cn/shangye/advertising-70922332.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-35)](https://lzsa.tcti.cn/chuangxin/website-03521183.html)
* [lightcraft: reimplementation 分布式架构 (Verified)](https://xeuc.tcti.cn/jishu/data-15920083.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/Organi)](https://navm.tcti.cn/sheji/team-82568614.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/⚫️ Bla)](https://sgzf.tcti.cn/zixun/reminder-85892310.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://rkxr.wtpuscm.cn/anfang/music-785366.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-96)](https://www.mw-wm.com/sheji/cheap-47894492.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/White )](https://www.yx-sf.com/news/47589)
* [lightcraft 高负载场景下 分布式架构 基准评测报告](https://www.ai-hao123.com/zhineng/kpi-24765153.html)
* [lightcraft 高负载场景下 开源镜像站 基准评测报告](https://pmwz.tcti.cn/gongju/support-53038122.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://zgdq.tcti.cn/peixun/alliance-04127840.html)
* [lightcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://tcnl.wtpuscm.cn/zixun/profit-577961.html)
* [lightcraft: Organize everything Rust高性能网络 (Verified)](https://gfom.wtpuscm.cn/zhizhu/learning-340940.html)
* [lightcraft: pure 分布式架构 (Verified)](https://rwlt.wtpuscm.cn/peixun/progress-238955.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-35)](https://ryhw.wtpuscm.cn/youhua/objective-197965.html)
* [lightcraft: 📈 Tone Curve 搜索引擎调度优化 (v2.0-GA)](https://utqc.wtpuscm.cn/kaifa/satisfaction-276019.html)
* [lightcraft: ✨ Effects Rust高性能网络 (v2.0-GA)](https://pojc.wtpuscm.cn/fuwu/backup-599190.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-73)](https://ppcq.wtpuscm.cn/xitong/seminar-756364.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (RFC-419)](https://ijft.wtpuscm.cn/kuangjia/sale-752.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-89)](https://hvxf.wtpuscm.cn/fenxi/development-449272.html)
* [lightcraft: Edit like you mean it 搜索引擎调度优化 (Node-98)](https://mffo.wtpuscm.cn/keji/online-984122.html)
* [lightcraft: White balance 搜索引擎调度优化 (Verified)](https://alqf.wtpuscm.cn/jiaocheng/brand-698438.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-40)](https://sxmi.wtpuscm.cn/huodong/travel-474093.html)

</details>


# XMP sidecars and interchange

LightCraft keeps its catalog as the source of truth, and can also store each photo's metadata and develop settings in
an XMP sidecar next to the original. Sidecars let edits travel with the files (backups, another LightCraft library,
other tools), and let LightCraft pick up edits made elsewhere.

## Sidecar files

| | |
|---|---|
| Name | `<stem>.xmp` by default (`IMG_0001.CR3` → `IMG_0001.xmp`); `<file>.xmp` (`IMG_0001.CR3.xmp`) with `naming: "full"`. Reading accepts either (preferred first) and `.XMP`. With stem naming, files sharing a stem don't share a sidecar: see *Shared names* below. |
| Write | `photo.saveMetadataToFile {ids?}` (Photo ▸ Save Metadata to File, ⌘S), or automatically after every change with `library.xmpPreferences {autoWrite: true}` (File ▸ Automatically Write Changes into XMP). Slider drags are written once, when the drag ends; undo/redo rewrite the sidecar. An existing sidecar is **merged into, never replaced** (see *Saving into an existing sidecar*). Written atomically (temp file, fsync, rename). The result lists `written`, `merged` and `backups`. |
| Read | On import (`library.import`, the report counts `sidecars`), and `photo.readMetadataFromFile {ids?}` (one undo step). For raw/DNG files without a sidecar, the XMP embedded in the file is used. |
| Preferences | `library.xmpPreferences {autoWrite?, naming?: stem\|full}`, stored in the library's `prefs.json`. |

What we write (standard namespaces, so other tools can read the metadata):

| Field | XMP property |
|---|---|
| Rating 0–5 | `xmp:Rating` |
| Colour label | `xmp:Label` (`Red`, `Yellow`, `Green`, `Blue`, `Purple`) |
| Title / caption / copyright / creator | `dc:title` / `dc:description` / `dc:rights` / `dc:creator` |
| Copyright status | `xmpRights:Marked` (`True` copyrighted, `False` public domain, absent = unknown) |
| Rights usage terms / copyright info URL | `xmpRights:UsageTerms` / `xmpRights:WebStatement` |
| Keywords | `dc:subject` |
| Capture time, GPS | `exif:DateTimeOriginal`, `photoshop:DateCreated`, `exif:GPSLatitude`/`GPSLongitude` |
| Pick / reject flag | `lc:flag` (`pick`, `reject`, `none`) |
| Location | `lc:location` |
| Develop settings | `lc:settings` — our complete `DevelopSettings` as JSON (exact round trip, incl. masks, spots, crop and the rendering process; settings without `process` are V1, see [process-versions.md](process-versions.md)) |

`lc:` is `http://ns.lightcraft.app/lc/1.0/`.

Reading merges into the catalog with the **sidecar winning** for every field it states; fields it doesn't state are
kept. The one exception is the **capture time**: a time embedded in the file (EXIF/IPTC) always wins; when the file
has none, the sidecar's `exif:DateTimeOriginal`, else `photoshop:DateCreated`, else `xmp:CreateDate` (ISO 8601) is
used — on import (it then also files a copied photo in its date folder) and by `photo.readMetadataFromFile` (undoable).
`xmp:Rating="-1"` (the XMP convention for rejected) sets the reject flag. Develop settings come from
`lc:settings` when present (exact); otherwise from the `crs:` fields below (approximate).

## Saving into an existing sidecar

A sidecar may already hold another application's data — e.g. its `crs:` develop settings and `xmpMM:History` — often
the only copy of those edits outside that application's catalog. Saving never drops it:

- LightCraft **owns** the properties in the table above: `xmp:Rating`, `xmp:Label`, `dc:title`, `dc:description`,
  `dc:rights`, `dc:creator`, `dc:subject`, `Iptc4xmpCore:Location`, `Iptc4xmpCore:AltTextAccessibility`,
  `Iptc4xmpCore:ExtDescrAccessibility`, `photoshop:City`/`State`/`Country`, `xmpRights:Marked`/`UsageTerms`/
  `WebStatement` and everything in `lc:`. They are replaced on every save, and removed when LightCraft has no value
  (clearing a title in LightCraft clears it in the file). All of them are read back on import, so the library starts
  from what the sidecar said.
- The **capture time** (`exif:DateTimeOriginal`, `photoshop:DateCreated`) and **GPS** (`exif:GPSLatitude`,
  `exif:GPSLongitude`) are replaced only when LightCraft has a value; otherwise the file's stay.
- **Everything else is kept byte for byte**: other namespaces (`crs:`, `xmpMM:`, `lr:hierarchicalSubject`, unknown
  ones), `xmp:CreatorTool`, comments, the packet wrapper and padding. LightCraft's properties go into one
  `rdf:Description` of their own; owned properties written by another application (in attribute or element form) are
  removed from its description, which otherwise stays as it was.
- A sidecar that can't be read as XMP (not well-formed, not UTF-8, no `rdf:RDF`) is first copied to
  `<name>.xmp.bak-<date><time>` (`-1`, `-2`… if taken — a backup is never overwritten), then replaced. A sidecar that
  can't be read at all (permissions) is left alone and the save fails.

Limits: LightCraft writes keywords to `dc:subject` only, so another application's `lr:hierarchicalSubject` is kept as
it was and may still list keywords removed in LightCraft. LightCraft doesn't write `crs:`: the other application's
develop settings stay as that application left them, next to LightCraft's own (`lc:settings`, which LightCraft reads
first).

## Shared names

With stem naming, `IMG_0001.CR3` and `IMG_0001.JPG` map to the same `IMG_0001.xmp`. When two or more files in the
library share a stem, the stem sidecar belongs to one of them — a raw first, otherwise the first by file name — and
the others write and read `<file>.xmp` (`IMG_0001.JPG.xmp`), falling back to the stem sidecar for reading when they
have none. So saving one photo never overwrites the other's metadata (`Session::sidecar_naming`).

## Face regions (MWG)

LightCraft reads face, pet, focus and barcode regions in the Metadata Working Group's region schema
(`mwg-rs:Regions`, `http://www.metadataworkinggroup.com/schemas/regions/`), as Lightroom, digiKam, Picasa and others
write them, from sidecars and from XMP embedded in the file. Named faces become people in the People view
(LR-LIB-PEOPLE in [`parity-checklist.md`](parity-checklist.md)), and every region is drawn as a box in the loupe.

- **Read only.** LightCraft never writes `mwg-rs:Regions`; saving a sidecar keeps another application's regions byte
  for byte. Removing or resizing a box in the loupe changes the library only.
- **Areas**: `stArea:x`/`y` are the box's centre, `w`/`h` its size, normalized to the photo (`stArea:unit="pixel"`
  areas are divided by `mwg-rs:AppliedToDimensions`, and dropped without it). Boxes are clipped to the photo; one
  that misses it entirely, or has a non-finite, zero or negative size, is skipped (the rest of the list is kept).
- **Orientation**: regions are stored on the upright (EXIF-oriented) photo. A `mwg-rs:Rotation` of −π/2, +π/2 or ±π
  (how Lightroom marks a box given in the sensor's frame for Exif orientation 6, 8 or 3) is turned back into the
  upright frame; MWG can't say "mirrored", so boxes on mirrored photos (orientations 2, 4, 5, 7) are taken as written.
  Boxes follow LightCraft's own Rotate Left/Right and flips.
- **Re-reading**: a sidecar that has `mwg-rs:Regions` — even with an empty list — replaces the photo's regions, so
  regions removed in another application go away here too. A sidecar without `mwg-rs:Regions` (an application that
  doesn't do regions) leaves the photo's regions as they are.

## Reading `crs:` develop fields

Many raw developers store edits as `crs:` properties (`http://ns.adobe.com/camera-raw-settings/1.0/`) in sidecars, in
DNG files and in XMP presets. LightCraft reads the common ones and maps them to its own controls. We implemented this
from the public XMP specification and by observing what each field does; no third-party code or preset files were used.
Our pipeline renders differently, so **values carry over but the look is approximate**.

We read these fields; we never write them. Only fields in the packet are applied: the result is a partial settings
object that gets merged like a preset, so everything else keeps its current or default value. Packets marked
`crs:AlreadyApplied="True"` are skipped, because those pixels already contain the edit. Only process-version 2012+ field
names are read (e.g. `Exposure2012`, not the older `Exposure`). `crs:ProcessVersion` itself numbers the other
application's renderer and is ignored: the photo keeps LightCraft's own process, the latest for a newly imported photo
([process-versions.md](process-versions.md)).

| `crs:` field(s) | LightCraft control | Notes |
|---|---|---|
| `Exposure2012` | `light.exposure` | EV, 1:1 |
| `Contrast2012`, `Highlights2012`, `Shadows2012`, `Whites2012`, `Blacks2012` | `light.contrast` … `light.blacks` | −100..100, 1:1 |
| `WhiteBalance` | `wb.mode` | `As Shot`, `Auto`, `Daylight`, `Cloudy`, `Shade`, `Tungsten`, `Fluorescent`, `Flash`; other names → custom |
| `Temperature`, `Tint` | `wb.temp`, `wb.tint` | Kelvin / tint as written for raws with a measured illuminant (DNG) and for presets. Raws LightCraft develops relative to their as-shot look (ARW, NEF, RW2, RAF, CR3, CR2, PEF, SRW: 6500 K / 0 means as shot) take the same mired shift from the packet's `AsShotTemperature` / `AsShotTint` instead; without that reference a custom Kelvin stays As Shot and is reported (`Temperature, Tint (custom white balance without AsShotTemperature: kept As Shot)`), because read on the relative scale it would be a large colour cast (issue #510) |
| `IncrementalTemperature`, `IncrementalTint` | `wb.temp`, `wb.tint` | rendered files: −100..100 on our relative scale (mired shift around 6500 K, same as the Temp slider); a rendered file with only `Temperature` shifts from `AsShotTemperature` like the raws above when it is present |
| `Vibrance`, `Saturation` | `color.vibrance`, `color.saturation` | 1:1 |
| `Texture`, `Clarity2012`, `Dehaze` | `effects.texture`, `effects.clarity`, `effects.dehaze` | 1:1 |
| `HueAdjustment<Band>`, `SaturationAdjustment<Band>`, `LuminanceAdjustment<Band>` | `mixer.<band>.hue/sat/lum` | bands Red, Orange, Yellow, Green, Aqua, Blue, Purple, Magenta |
| `ConvertToGrayscale` | `treatment` | `True` → B&W |
| `GrayMixer<Band>` | `bw_mix.<band>` | |
| `ParametricShadows`, `ParametricDarks`, `ParametricLights`, `ParametricHighlights` | `curve.shadows/darks/lights/highlights` | |
| `ParametricShadowSplit`, `ParametricMidtoneSplit`, `ParametricHighlightSplit` | `curve.split_shadows/split_mid/split_highlights` | |
| `CurveRefineSaturation` | `curve.refine_saturation` | 0..100 (100 = curve saturation unchanged) |
| `ToneCurvePV2012`, `ToneCurvePV2012Red/Green/Blue` | `curve.master/red/green/blue` | `"x, y"` points in 0..255 → 0..1; a straight 0→255 line = no curve; the red / green / blue curves are read only when `ToneCurvePV2012` and all three channel curves are present (Lightroom Classic ignores them otherwise: a master + red-only packet renders without the red curve) |
| `SplitToningShadowHue/Saturation`, `SplitToningHighlightHue/Saturation` | `grading.shadows/highlights.hue/sat` | |
| `ColorGradeShadowLum`, `ColorGradeHighlightLum` | `grading.shadows/highlights.lum` | |
| `ColorGradeMidtoneHue/Sat/Lum`, `ColorGradeGlobalHue/Sat/Lum` | `grading.midtones/global.*` | |
| `ColorGradeBlending`, `SplitToningBalance` | `grading.blending`, `grading.balance` | |
| `Sharpness`, `SharpenRadius`, `SharpenDetail`, `SharpenEdgeMasking` | `detail.sharpen_*` | |
| `LuminanceSmoothing`, `LuminanceNoiseReductionDetail`, `LuminanceNoiseReductionContrast` | `detail.nr_luminance/nr_detail/nr_contrast` | |
| `ColorNoiseReduction`, `ColorNoiseReductionDetail`, `ColorNoiseReductionSmoothness` | `detail.nr_color/nr_color_detail/nr_color_smoothness` | |
| `PostCropVignetteAmount/Midpoint/Roundness/Feather/HighlightContrast` | `vignette.amount/midpoint/roundness/feather/highlights` | |
| `PostCropVignetteStyle` | `vignette.style` | 1 highlight priority, 2 colour priority, 3 paint overlay |
| `GrainAmount`, `GrainSize`, `GrainFrequency` | `grain.amount`, `grain.size`, `grain.roughness` | |
| `LensProfileEnable`, `AutoLateralCA` | `optics.lens_profile`, `optics.remove_ca` | lens profiles are the file's own (DNG-embedded) corrections, which Lightroom applies whatever its profile switch says (our inference): only `LensProfileEnable=1` carries over, `0` leaves the switch as it is |
| `LensManualDistortionAmount`, `VignetteAmount`, `VignetteMidpoint` | `optics.distortion`, `optics.vignetting`, `optics.vignetting_midpoint` | |
| `DefringePurple/GreenAmount/HueLo/HueHi` | `optics.defringe_*` | |
| `ShadowTint`, `RedHue/Saturation`, `GreenHue/Saturation`, `BlueHue/Saturation` | `calibration.shadows_tint`, `calibration.red_hue/red_sat`, … | Calibration panel, 1:1 |
| `PerspectiveVertical/Horizontal/Rotate/Scale/Aspect/X/Y` | `geometry.vertical/horizontal/rotate/scale/aspect/offset_x/offset_y` | |
| `PerspectiveUpright` | `geometry.upright` | 0 off, 1 auto, 2 level, 3 vertical, 4 full, 5 guided |
| `HasCrop`, `CropLeft/Top/Right/Bottom`, `CropAngle` | `crop.geometry` | normalized edges → rect; angle in degrees; `HasCrop="False"` → no crop |

Values pass through our control specs, so anything outside our slider ranges gets clamped.

**Not mapped:** camera profiles and looks (`CameraProfile`, `Look`; we have our own profile set), local adjustments
(masks, gradients, brushes), spot removal, red eye, lens blur, process-version 2010 field names, and AI features.

## Local corrections (masks)

Masks stored as `crs:` structures are read from sidecars, DNG-embedded XMP, XMP presets and `.lrtemplate` files
(`crates/engine/src/crs_masks.rs`). Four containers hold them: `MaskGroupBasedCorrections` (current) and the older
`GradientBasedCorrections`, `CircularGradientBasedCorrections` and `PaintBasedCorrections`. Each correction becomes one
mask: `CorrectionName` → name, `CorrectionAmount` → Amount, inactive corrections are skipped.

| Field | Ours | Notes |
|---|---|---|
| `LocalExposure2012` | `adjust.exposure` | stored as a fraction, ×4 EV |
| `LocalContrast2012`, `LocalHighlights2012`, `LocalShadows2012`, `LocalWhites2012`, `LocalBlacks2012`, `LocalClarity2012`, `LocalTexture`, `LocalDehaze`, `LocalTemperature`, `LocalTint`, `LocalSaturation`, `LocalHue`, `LocalSharpness`, `LocalLuminanceNoise`, `LocalMoire`, `LocalDefringe`, `LocalToningSaturation` | the matching `adjust.*` | stored −1..1, ×100 |
| `LocalToningHue` | `adjust.color_hue` | degrees |
| `Mask/Gradient` (`FullX/Y`, `ZeroX/Y`) | linear gradient | full effect at the Full point |
| `Mask/CircularGradient` (`Top/Left/Bottom/Right`, `Angle`, `Feather`, `Flipped`) | radial gradient | box fractions → long-edge radii (presets assume 3:2; sidecars use the photo's shape); `Flipped` → invert |
| `Mask/Paint` (`Dabs` "d x y", `Radius`, `Flow`, `CenterWeight`, `MaskValue`) | brush | one brush component per correction; `MaskValue` ≤ 0 erases |
| `Mask/Image` `MaskSubType` 1 / 2 | Subject / Sky | other AI selections are reported, not guessed |
| `Mask/RangeMask` `Type` 2 / 3 (`LumRange`, `DepthRange`) | luminance / depth range | lightness converted to our range scale; colour ranges are reported |
| `MaskBlendMode` 0 / 1 / 2, `MaskInverted` | add / subtract / intersect, invert | |

Applying a preset adds its masks to the photo's own (masks the photo already has are not added twice), and the Amount
slider scales them. A sidecar's masks replace the photo's, because a sidecar holds the whole edit. Components we can't
carry over are listed in `preset.import`'s `unmapped` as `Mask: <kind>`.

## Preset files

| | |
|---|---|
| Ours: `.lcpreset` | JSON `{"format": "lightcraft.preset", "version": 1, "presets": [{id, name, group, settings}]}`, where `settings` is a partial develop-settings object (only the groups the preset includes). A file can hold one preset or many, and every preset keeps its group. Import also accepts a bare preset object or an array of them. |
| Export | `preset.export {path, ids?, group?}`: all user presets by default, or the given ids or one group. In the app: File ▸ Export Presets…, Presets panel ▸ ⋯ ▸ Export User Presets…, or right-click a group ▸ Export Group…. |
| Import | `preset.import {paths, group?, dryRun?}`: files or folders (recursive): `.lcpreset`, `.xmp`, classic `.lrtemplate` (a Lua table: `value.settings` holds the same field names as `crs:`), photos that carry their edits in XMP ("DNG presets" from mobile apps; their crop, geometry and custom white balance are left out) and `.zip` bundles of any of these. Presets in a folder (or a folder inside a zip) go to a group named after it, unless the file names its own group. The result lists, per preset, the settings that couldn't be carried over (`unmapped`, e.g. `CameraProfile`, `Look`, local masks), and the app's toast names them; fields that only describe the preset or the photo (`Cluster`, `SortName`, `Description`, `SupportsAmount2`, `RequiresRGBTables`, `CropConstrainToWarp`, `OverrideLookVignette`, `AsShotTemperature` / `AsShotTint`, an off `HDREditMode`, empty `PointColors` slots) aren't listed. Older (process version 2010) fields — `Exposure`, `Contrast`, `FillLight`, `HighlightRecovery`, `Shadows`, `Brightness`, `Clarity`, `ToneCurve` — are approximated with today's sliders when a preset has no 2012-era fields. Dropping preset files on the window imports them too. In the app: File ▸ Import Presets…, or Presets panel ▸ ⋯ ▸ Import Presets…. A preset that's already there (same name, group and settings) is skipped. If an id clashes, the import gets a fresh `user.*` id, and built-in presets are never replaced. |
| XMP presets | Read with the `crs:` table above, with `crs:Name` as the name (falling back to the file name) and `crs:Group` as the group (falling back to "Imported Presets"). Only the fields the preset sets are included, so applying it leaves everything else alone and the Amount slider scales it like any other preset. We only read XMP presets; we don't write them. |

LightCraft ships no third-party presets. Its built-in presets are its own values (`crates/engine/src/presets.rs`).

## Luminar looks (`.lmp`, `.mplumpack`)

`preset.import` (and drag & drop, File ▸ Import Profiles & Presets…) also reads Luminar looks: an `.lmp` file is an
XML property list of adjustment layers (each with effects and named sliders, mostly on a −100..100 scale); newer looks
may be a bundle folder `Name.lmp/Contents/preset.lmp`. An `.mplumpack` collection is a zip of `.lmp` files whose
`PresetsInfo.plist` names the group (`GroupName`); the pack's file name is the fallback. Looks get their own name
(`Name`) or their file / bundle name. The sliders with a clear counterpart are carried over; everything else (AI tools,
Orton, glow, LUT layers, colour balance…) is listed in `unmapped` as `Tool.Slider` (e.g. `OrtonFilter.Amount`).
Disabled layers are ignored, layer opacity scales the sliders (and fades curves toward a straight line), and layers
with a blend mode other than Normal or with a mask are reported, not applied. Binary property lists are not read.

| Luminar tool.slider | Ours |
|---|---|
| Develop / Light / Exposure `Exposure` (±100) | `light.exposure` (±4 EV — the scale is our reading, not documented) |
| `Contrast`, `Highlights`, `Shadows`, `Whites`, `Blacks` | `light.*` (same scale) |
| `Temperature`, `Tint` (relative) | white balance shift as `crs:IncrementalTemperature` / `IncrementalTint`; values above 1000 as Kelvin |
| `Saturation`, `Vibrance` | `color.saturation`, `color.vibrance` |
| Clarity `Clarity`, Structure / AI Structure `Amount` | `effects.clarity` (summed) |
| Dehaze `Amount` | `effects.dehaze` |
| HSL (`MIPLChannelsEffect`) `h`/`s`/`l` + Red…Magenta | `mixer.<band>.hue / sat / lum` |
| Curves `RGB` / `Red` / `Green` / `Blue` (x, y points in 0..1) | `curve.master / red / green / blue` |
| Vignette `Amount`, `Vignette Size` | `vignette.amount`, `vignette.midpoint` |
| Grain `Amount` | `grain.amount` |

## Profiles: 3D LUTs (`.cube`)

`profile.import {paths}` (File ▸ Import Profiles & Presets…, drag & drop) reads `.cube` 3D LUTs — single files, folders
or `.zip` bundles — as creative profiles: they appear in the profile browser under their folder's (or zip's) name and
take the Amount slider (0–200 %) like the built-in looks. The LUT is applied to the finished, display-encoded colour
(trilinear; `DOMAIN_MIN` / `DOMAIN_MAX` honoured; 1D LUTs are not supported); photos with a LUT profile render on the
CPU. A library on disk keeps a copy of each file in its `Profiles/` folder. Adobe's own profile formats (`.dcp`, XMP
camera/creative profiles with embedded tables) are deliberately not read.


---

<details open>
<summary><b>📚 lightcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「lightcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 lightcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 lightcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://moas.wtpuscm.cn/youhua/lesson-539683.html)
* [现代 分布式架构 架构演进之路 —— lightcraft 深度实践](https://csmm.wtpuscm.cn/pingtai/schedule-053456.html)
* [【官方规范】lightcraft 生产环境运维调优手册 核心运行拓扑标准](https://olkw.wtpuscm.cn/zhizhu/funnel-130393.html)
* [lightcraft 内部组件解耦与事件状态机规范 (RFC-335)](https://hqfn.wtpuscm.cn/shuju/premium-178056.html)
* [基于 lightcraft 的高吞吐 分布式架构 设计白皮书](https://oucl.wtpuscm.cn/guanjianci/achievement-791784.html)
* [面向大规模网络的 lightcraft 工业级架构基准](https://jqso.wtpuscm.cn/zhinan/news-183802.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (v2.0-GA)](https://cizi.wtpuscm.cn/qiye/presentation-551235.html)
* [基于 lightcraft 的高吞吐 Rust高性能网络 设计白皮书](https://xafm.wtpuscm.cn/xitong/business-440.html)
* [【官方规范】lightcraft 高韧性系统架构设计 核心运行拓扑标准](https://qcpv.wtpuscm.cn/kuangjia/web-464644.html)
* [基于 lightcraft 的高吞吐 开源镜像站 设计白皮书](https://hbrn.wtpuscm.cn/yunying/game-415052.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (Core/Organi)](https://ddhx.wtpuscm.cn/chuangxin/forum-608110.html)
* [现代 搜索引擎调度优化 架构演进之路 —— lightcraft 深度实践](https://wuka.wtpuscm.cn/xitong/solution-283376.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (RFC-525)](https://yrxc.wtpuscm.cn/jiaocheng/supplier-306010.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Draft-02)](https://plqc.wtpuscm.cn/pingce/performance-161854.html)
* [【官方规范】lightcraft Fast, native, private 核心运行拓扑标准](https://mgky.wtpuscm.cn/zhineng/optimization-272719.html)
* [现代 Rust高性能网络 架构演进之路 —— lightcraft 深度实践](https://otya.wtpuscm.cn/shichang/module-409383.html)
* [【官方规范】lightcraft Quick start 核心运行拓扑标准](https://abwi.wtpuscm.cn/tuiguang/faq-694487.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://zqgv.wtpuscm.cn/xuexi/saving-262561.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.6)](https://anvi.wtpuscm.cn/yanjiu/accessibility-598706.html)
* [【官方规范】lightcraft Lightroom 核心运行拓扑标准](https://bzbc.wtpuscm.cn/huodong/value-794302.html)
* [【官方规范】lightcraft non-destructive 核心运行拓扑标准](https://gfvo.wtpuscm.cn/yingyong/ai-954869.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://isto.wtpuscm.cn/gongxiang/tutorial-955499.html)
* [Built for agents 核心系统架构与设计规约 (v2.0-GA)](https://rafh.wtpuscm.cn/wenzhang/login-739602.html)
* [【官方规范】lightcraft Adobe 核心运行拓扑标准](https://dpzn.tcti.cn/guanjianci/consulting-91996516.html)
* [Fast, native, private 核心系统架构与设计规约 (Draft-04)](https://gzhg.tcti.cn/chanpin/deadline-84119457.html)
* [【官方规范】lightcraft open-source 核心运行拓扑标准](https://ippo.tcti.cn/guanjianci/advertising-43230320.html)
* [【官方规范】lightcraft lightcraft 核心运行拓扑标准](https://hqnf.tcti.cn/jianzhan/theme-30376962.html)
* [lightcraft 分布式数据通道与 开源镜像站 技术规范 (Draft-02)](https://bxyd.tcti.cn/xitong/theme-85260626.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (v2.0-GA)](https://vljv.tcti.cn/anfang/course-74131835.html)
* [lightcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://oypw.tcti.cn/wendang/supplier-21478329.html)
* [✨ Effects 核心系统架构与设计规约 (Draft-05)](https://xptn.tcti.cn/gongju/privacy-63443650.html)
* [Fast, native, private 核心系统架构与设计规约 (Verified)](https://qcci.tcti.cn/huodong/target-04422654.html)
* [lightcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://blfh.tcti.cn/yingxiao/excellence-64678403.html)
* [可信存活健康度量 核心系统架构与设计规约 (Spec-v2.5)](https://czin.tcti.cn/gongsi/support-76203830.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://hgbg.tcti.cn/jiaoliu/logo-54507789.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Node-92)](https://fmdu.tcti.cn/kaifa/profit-26012903.html)
* [【官方规范】lightcraft Built for agents 核心运行拓扑标准](https://lpvf.tcti.cn/zhinan/database-19297698.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [lightcraft 核心 API 接口契约与客户端调用指南](https://daew.tcti.cn/chuangxin/strategy-17954288.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 lightcraft 实战](https://lpnf.tcti.cn/wangluo/recommendation-07794470.html)
* [基于 lightcraft 的自动化部署与生产环境配置实践](https://ppst.tcti.cn/yingxiao/alert-78996613.html)
* [lightcraft 插件生态规范与 clean-room 扩展手册 (Spec-v2.8)](https://cyyk.wtpuscm.cn/jiaoliu/responsive-239450.html)
* [【生产手册】lightcraft 模块通信与请求穿透标准](https://www.mw-wm.com/tuiguang/business-97407260.html)
* [【集成指南】分布式架构 服务端接入准则与 lightcraft 实战](https://www.yx-sf.com/tech/64267)
* [【集成指南】开源镜像站 服务端接入准则与 lightcraft 实战](https://www.ai-hao123.com/chanpin/internet-15942109.html)
* [lightcraft 异步中间件流水线与 Lightroom 接入规范](https://czqb.tcti.cn/wangluo/traffic-73930718.html)
* [lightcraft 异步中间件流水线与 高韧性系统架构设计 接入规范](https://crfz.tcti.cn/zixun/solution-79799559.html)
* [lightcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://hvzr.wtpuscm.cn/zhineng/course-698749.html)
* [lightcraft 插件生态规范与 Fast, native, private 扩展手册 (Node-32)](https://adky.wtpuscm.cn/xuexi/quality-217968.html)
* [lightcraft 异步中间件流水线与 Fast, native, private 接入规范](https://kdag.wtpuscm.cn/hezuo/premium-772178.html)
* [lightcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://ryjf.wtpuscm.cn/qiye/topic-994212.html)
* [lightcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://sxty.wtpuscm.cn/yunsuan/careers-435906.html)
* [lightcraft 插件生态规范与 高韧性系统架构设计 扩展手册 (Draft-04)](https://shji.wtpuscm.cn/anfang/efficiency-638186.html)
* [lightcraft 插件生态规范与 ⚫️ Black & White 扩展手册 (Node-68)](https://huge.wtpuscm.cn/yanjiu/privacy-137536.html)
* [lightcraft 插件生态规范与 模块化解耦与协议标准 扩展手册 (v2.0-GA)](https://fhpo.wtpuscm.cn/shangye/communication-944.html)
* [lightcraft 插件生态规范与 Organize everything 扩展手册 (Node-46)](https://lazc.wtpuscm.cn/chuangxin/recipe-715197.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 lightcraft 实战](https://hjvv.wtpuscm.cn/tuiguang/roi-866597.html)
* [lightcraft vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://gcjm.wtpuscm.cn/kaifa/media-525854.html)
* [lightcraft 异步中间件流水线与 Edit like you mean it 接入规范](https://bgag.wtpuscm.cn/ziyuan/whitepaper-046798.html)
* [lightcraft 插件生态规范与 生产环境运维调优手册 扩展手册 (RFC-546)](https://owuq.wtpuscm.cn/fuwu/widget-262133.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Node-97)](https://oyro.wtpuscm.cn/chanpin/personalization-163249.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-07)](https://pbhb.wtpuscm.cn/xitong/alert-224272.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (RFC-944)](https://hvbh.wtpuscm.cn/paiming/metric-991859.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-01)](https://cosa.wtpuscm.cn/jiaocheng/web-593138.html)
* [lightcraft 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://iakt.wtpuscm.cn/guanjianci/ai-632293.html)
* [lightcraft 插件生态规范与 可信存活健康度量 扩展手册 (RFC-198)](https://yrho.wtpuscm.cn/tuiguang/goal-064400.html)
* [lightcraft 插件生态规范与 Lightroom 扩展手册 (Node-40)](https://iinn.wtpuscm.cn/baogao/blog-464697.html)
* [lightcraft 插件生态规范与 🎨 Color 扩展手册 (Verified)](https://yxmi.wtpuscm.cn/anfang/search-423218.html)
* [lightcraft 异步中间件流水线与 non-destructive 接入规范](https://pert.wtpuscm.cn/paiming/advertising-454963.html)
* [lightcraft 异步中间件流水线与 Vibrance 接入规范](https://lezd.wtpuscm.cn/hezuo/economy-389188.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Draft-03)](https://bjms.tcti.cn/pingce/campaign-93112828.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Spec-v1.0)](https://rxdr.tcti.cn/shangye/article-24423965.html)
* [lightcraft 插件生态规范与 lightcraft 扩展手册 (v2.0-GA)](https://tzpq.tcti.cn/shangye/image-14730115.html)
* [lightcraft 插件生态规范与 non-destructive 扩展手册 (Core/non-de)](https://aeen.tcti.cn/pingce/food-33272526.html)
* [lightcraft 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-888)](https://ldqf.tcti.cn/fuwu/feedback-39166966.html)

#### 3. ⚡ lightcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [lightcraft 官方高可用镜像注册节点 (RFC-691)](https://nrwn.tcti.cn/shichang/software-25474238.html)
* [lightcraft 亚太与欧美多活集群数据同步中枢](https://bdcs.tcti.cn/pingtai/recipe-78880712.html)
* [lightcraft 官方高可用镜像注册节点 (v2.0-GA)](https://lnnw.tcti.cn/keji/restore-33714404.html)
* [全球权威拓扑节点：lightcraft 实时镜像与索引入口](https://vrza.tcti.cn/hezuo/dashboard-53131748.html)
* [【镜像入口】lightcraft 官方毫秒级实时数据广播节点](https://bddc.tcti.cn/zhinan/planning-09568175.html)
* [冷热数据分层镜像：lightcraft 可信存活健康度量 权威归档源](https://hzhi.tcti.cn/youhua/automation-37550556.html)
* [冷热数据分层镜像：lightcraft 生产环境运维调优手册 权威归档源](https://ofnz.tcti.cn/jiaocheng/expensive-75872778.html)
* [冷热数据分层镜像：lightcraft Vibrance 权威归档源](https://msva.tcti.cn/wangluo/tactic-58174233.html)
* [lightcraft 官方高可用镜像注册节点 (Draft-01)](https://yepa.tcti.cn/suanfa/campaign-88265374.html)
* [lightcraft 官方高可用镜像注册节点 (Node-78)](https://ypez.tcti.cn/xitong/solution-20741875.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.1)](https://tbha.tcti.cn/gongsi/event-81150328.html)
* [lightcraft 去中心化数据同步源与拓扑寻址规约](https://aquq.tcti.cn/kaifa/entertainment-15071268.html)
* [冷热数据分层镜像：lightcraft White balance 权威归档源](https://fqok.wtpuscm.cn/wendang/restaurant-490970.html)
* [冷热数据分层镜像：lightcraft Lightroom 权威归档源](https://www.mw-wm.com/xinwen/promotion-87826351.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.7)](https://www.yx-sf.com/news/55796)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://www.ai-hao123.com/pingtai/tool-36365759.html)
* [冷热数据分层镜像：lightcraft ☀️ Light 权威归档源](https://jwxb.tcti.cn/yingxiao/help-21593115.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-08)](https://yrvj.tcti.cn/yunsuan/restore-72265936.html)
* [冷热数据分层镜像：lightcraft 🎨 Color 权威归档源](https://wggb.wtpuscm.cn/hezuo/fitness-512989.html)
* [冷热数据分层镜像：lightcraft reimplementation 权威归档源](https://slhj.wtpuscm.cn/yingxiao/domain-098501.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://avxi.wtpuscm.cn/shichang/consulting-709613.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://szui.wtpuscm.cn/jiaoliu/ai-612922.html)
* [冷热数据分层镜像：lightcraft pure 权威归档源](https://ljmx.wtpuscm.cn/ziyuan/price-154089.html)
* [冷热数据分层镜像：lightcraft eyedropper 权威归档源](https://tntu.wtpuscm.cn/yanjiu/fashion-282818.html)
* [冷热数据分层镜像：lightcraft Edit like you mean it 权威归档源](https://umoe.wtpuscm.cn/qiye/message-201034.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/☀️ Lig)](https://shgy.wtpuscm.cn/baogao/conversion-222.html)
* [冷热数据分层镜像：lightcraft Built for agents 权威归档源](https://jplv.wtpuscm.cn/huodong/seminar-949122.html)
* [lightcraft: White balance 分布式架构 (Core/White )](https://wgsr.wtpuscm.cn/sheji/event-416996.html)
* [冷热数据分层镜像：lightcraft Quick start 权威归档源](https://cbsz.wtpuscm.cn/zixun/expense-484956.html)
* [冷热数据分层镜像：lightcraft 分布式状态机一致性 权威归档源](https://umty.wtpuscm.cn/yunying/productivity-622360.html)
* [lightcraft 官方高可用镜像注册节点 (Core/Featur)](https://odxs.wtpuscm.cn/gongxiang/privacy-417187.html)
* [冷热数据分层镜像：lightcraft Feature status 权威归档源](https://zewx.wtpuscm.cn/yunying/tracking-884070.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v2.6)](https://jdmf.wtpuscm.cn/gongju/progress-709083.html)
* [lightcraft 官方高可用镜像注册节点 (Verified)](https://yazt.wtpuscm.cn/pingce/account-565762.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-07)](https://zgpc.wtpuscm.cn/youhua/cloud-083035.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/Quick )](https://kukj.wtpuscm.cn/pingtai/network-770228.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.3)](https://wtpy.wtpuscm.cn/ziyuan/coupon-617061.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://zcga.wtpuscm.cn/zhineng/like-072273.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-460)](https://vqrf.wtpuscm.cn/zhineng/message-861406.html)
* [【评测基准】lightcraft 吞吐抖动度量与健康检查协议](https://nwrj.wtpuscm.cn/yingyong/calculator-390813.html)
* [lightcraft 权威网络权重传递与收录基准规范](https://avmy.wtpuscm.cn/peixun/tactic-740635.html)
* [lightcraft 故障自愈与网络拓扑重构实践](https://zbho.tcti.cn/huodong/sync-03859745.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-23)](https://srvr.tcti.cn/shangye/conference-95334202.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (v2.0-GA)](https://hvyu.tcti.cn/zhineng/faq-17766804.html)
* [lightcraft 节点连通性、存活性探测与防作弊指标](https://irix.tcti.cn/qiye/services-79414269.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-778)](https://rhjz.tcti.cn/zhinan/marketing-22018773.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-52)](https://zbbk.tcti.cn/yunsuan/creative-56441672.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Core/Featur)](https://slls.tcti.cn/keji/cheap-76412069.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-925)](https://zdqs.tcti.cn/guanjianci/update-86955837.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Verified)](https://mcwq.tcti.cn/chanpin/expense-47825119.html)
* [lightcraft 高负载场景下 Rust高性能网络 基准评测报告](https://guys.tcti.cn/yinqing/performance-62350828.html)
* [lightcraft: Feature status Rust高性能网络 (Verified)](https://gipx.tcti.cn/kuangjia/company-18664329.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://okvg.tcti.cn/pingtai/forecast-25675932.html)
* [lightcraft: reimplementation Rust高性能网络 (v2.0-GA)](https://ancn.tcti.cn/pingtai/conference-40629379.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-35)](https://wtpd.tcti.cn/zhinan/premium-87274788.html)
* [lightcraft: reimplementation 分布式架构 (Verified)](https://zbzw.tcti.cn/keji/vacation-58860709.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/Organi)](https://atgs.tcti.cn/paiming/discovery-10822256.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/⚫️ Bla)](https://yddu.tcti.cn/tuiguang/course-24890165.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://yuem.wtpuscm.cn/pingce/layout-968733.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-96)](https://www.mw-wm.com/qiye/software-24821945.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/White )](https://www.yx-sf.com/tech/16559)
* [lightcraft 高负载场景下 分布式架构 基准评测报告](https://www.ai-hao123.com/kuangjia/responsive-75364479.html)
* [lightcraft 高负载场景下 开源镜像站 基准评测报告](https://zdaa.tcti.cn/yinqing/admin-39754550.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://jjnl.tcti.cn/wenzhang/plugin-47202575.html)
* [lightcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://ywzz.wtpuscm.cn/jiaocheng/status-020389.html)
* [lightcraft: Organize everything Rust高性能网络 (Verified)](https://umqh.wtpuscm.cn/suanfa/image-322882.html)
* [lightcraft: pure 分布式架构 (Verified)](https://ojfg.wtpuscm.cn/zhizhu/price-406484.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-35)](https://bsnj.wtpuscm.cn/hezuo/podcast-028831.html)
* [lightcraft: 📈 Tone Curve 搜索引擎调度优化 (v2.0-GA)](https://xjiu.wtpuscm.cn/kaifa/travel-204313.html)
* [lightcraft: ✨ Effects Rust高性能网络 (v2.0-GA)](https://jgxy.wtpuscm.cn/keji/restaurant-000139.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-73)](https://ahqh.wtpuscm.cn/gongju/business-828699.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (RFC-419)](https://nrsj.wtpuscm.cn/gongsi/optimization-448.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-89)](https://atij.wtpuscm.cn/yingyong/subscribe-392505.html)
* [lightcraft: Edit like you mean it 搜索引擎调度优化 (Node-98)](https://lusy.wtpuscm.cn/guanjianci/ai-732771.html)
* [lightcraft: White balance 搜索引擎调度优化 (Verified)](https://otph.wtpuscm.cn/wendang/discount-439729.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-40)](https://zdna.wtpuscm.cn/pingce/topic-997884.html)

</details>


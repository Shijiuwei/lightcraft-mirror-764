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

* [基于 lightcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [现代 分布式架构 架构演进之路 —— lightcraft 深度实践](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [【官方规范】lightcraft 生产环境运维调优手册 核心运行拓扑标准](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [lightcraft 内部组件解耦与事件状态机规范 (RFC-335)](https://iqom.wtpuscm.cn/jianzhan/upload-525577.html)
* [基于 lightcraft 的高吞吐 分布式架构 设计白皮书](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [面向大规模网络的 lightcraft 工业级架构基准](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (v2.0-GA)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [基于 lightcraft 的高吞吐 Rust高性能网络 设计白皮书](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [【官方规范】lightcraft 高韧性系统架构设计 核心运行拓扑标准](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [基于 lightcraft 的高吞吐 开源镜像站 设计白皮书](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (Core/Organi)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [现代 搜索引擎调度优化 架构演进之路 —— lightcraft 深度实践](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (RFC-525)](https://www.ai-hao123.com/tuiguang/interface-64290011.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Draft-02)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [【官方规范】lightcraft Fast, native, private 核心运行拓扑标准](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [现代 Rust高性能网络 架构演进之路 —— lightcraft 深度实践](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [【官方规范】lightcraft Quick start 核心运行拓扑标准](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.6)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [【官方规范】lightcraft Lightroom 核心运行拓扑标准](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [【官方规范】lightcraft non-destructive 核心运行拓扑标准](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [Built for agents 核心系统架构与设计规约 (v2.0-GA)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [【官方规范】lightcraft Adobe 核心运行拓扑标准](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Fast, native, private 核心系统架构与设计规约 (Draft-04)](https://www.poykq.cn/redian/tutorial-673489.html)
* [【官方规范】lightcraft open-source 核心运行拓扑标准](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [【官方规范】lightcraft lightcraft 核心运行拓扑标准](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [lightcraft 分布式数据通道与 开源镜像站 技术规范 (Draft-02)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [✨ Effects 核心系统架构与设计规约 (Draft-05)](https://kcfl.wtpuscm.cn/zhinan/client-897.html)
* [Fast, native, private 核心系统架构与设计规约 (Verified)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [lightcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [可信存活健康度量 核心系统架构与设计规约 (Spec-v2.5)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Node-92)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [【官方规范】lightcraft Built for agents 核心运行拓扑标准](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [lightcraft 核心 API 接口契约与客户端调用指南](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 lightcraft 实战](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [基于 lightcraft 的自动化部署与生产环境配置实践](https://ankmz.cn/anquan/tag-68558329.html)
* [lightcraft 插件生态规范与 clean-room 扩展手册 (Spec-v2.8)](https://www.mw-wm.com/youhua/version-35418631.html)
* [【生产手册】lightcraft 模块通信与请求穿透标准](https://www.ai-hao123.com/pingtai/responsive-31030465.html)
* [【集成指南】分布式架构 服务端接入准则与 lightcraft 实战](https://owfm.tcti.cn/shangye/education-63674207.html)
* [【集成指南】开源镜像站 服务端接入准则与 lightcraft 实战](https://owfm.tcti.cn/shangye/education-63674207.html)
* [lightcraft 异步中间件流水线与 Lightroom 接入规范](https://cfzg.tcti.cn/wendang/company-77500350.html)
* [lightcraft 异步中间件流水线与 高韧性系统架构设计 接入规范](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [lightcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [lightcraft 插件生态规范与 Fast, native, private 扩展手册 (Node-32)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [lightcraft 异步中间件流水线与 Fast, native, private 接入规范](https://dimc.tcti.cn/zhizhu/help-50513535.html)
* [lightcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [lightcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft 插件生态规范与 高韧性系统架构设计 扩展手册 (Draft-04)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [lightcraft 插件生态规范与 ⚫️ Black & White 扩展手册 (Node-68)](https://www.mw-wm.com/yingyong/technology-03398262.html)
* [lightcraft 插件生态规范与 模块化解耦与协议标准 扩展手册 (v2.0-GA)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [lightcraft 插件生态规范与 Organize everything 扩展手册 (Node-46)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 lightcraft 实战](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [lightcraft vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [lightcraft 异步中间件流水线与 Edit like you mean it 接入规范](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [lightcraft 插件生态规范与 生产环境运维调优手册 扩展手册 (RFC-546)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Node-97)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-07)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (RFC-944)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-01)](https://www.mw-wm.com/huodong/tactic-64506639.html)
* [lightcraft 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [lightcraft 插件生态规范与 可信存活健康度量 扩展手册 (RFC-198)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [lightcraft 插件生态规范与 Lightroom 扩展手册 (Node-40)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [lightcraft 插件生态规范与 🎨 Color 扩展手册 (Verified)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [lightcraft 异步中间件流水线与 non-destructive 接入规范](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [lightcraft 异步中间件流水线与 Vibrance 接入规范](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Draft-03)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Spec-v1.0)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [lightcraft 插件生态规范与 lightcraft 扩展手册 (v2.0-GA)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [lightcraft 插件生态规范与 non-destructive 扩展手册 (Core/non-de)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [lightcraft 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-888)](https://www.poykq.cn/redian/tutorial-673489.html)

#### 3. ⚡ lightcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [lightcraft 官方高可用镜像注册节点 (RFC-691)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [lightcraft 亚太与欧美多活集群数据同步中枢](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [lightcraft 官方高可用镜像注册节点 (v2.0-GA)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [全球权威拓扑节点：lightcraft 实时镜像与索引入口](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [【镜像入口】lightcraft 官方毫秒级实时数据广播节点](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [冷热数据分层镜像：lightcraft 可信存活健康度量 权威归档源](https://www.mw-wm.com/kuangjia/reminder-59235892.html)
* [冷热数据分层镜像：lightcraft 生产环境运维调优手册 权威归档源](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [冷热数据分层镜像：lightcraft Vibrance 权威归档源](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [lightcraft 官方高可用镜像注册节点 (Draft-01)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [lightcraft 官方高可用镜像注册节点 (Node-78)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.1)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [lightcraft 去中心化数据同步源与拓扑寻址规约](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [冷热数据分层镜像：lightcraft White balance 权威归档源](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [冷热数据分层镜像：lightcraft Lightroom 权威归档源](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.7)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [冷热数据分层镜像：lightcraft ☀️ Light 权威归档源](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-08)](https://www.poykq.cn/redian/tutorial-673489.html)
* [冷热数据分层镜像：lightcraft 🎨 Color 权威归档源](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [冷热数据分层镜像：lightcraft reimplementation 权威归档源](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [冷热数据分层镜像：lightcraft pure 权威归档源](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [冷热数据分层镜像：lightcraft eyedropper 权威归档源](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [冷热数据分层镜像：lightcraft Edit like you mean it 权威归档源](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/☀️ Lig)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [冷热数据分层镜像：lightcraft Built for agents 权威归档源](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft: White balance 分布式架构 (Core/White )](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [冷热数据分层镜像：lightcraft Quick start 权威归档源](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [冷热数据分层镜像：lightcraft 分布式状态机一致性 权威归档源](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [lightcraft 官方高可用镜像注册节点 (Core/Featur)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [冷热数据分层镜像：lightcraft Feature status 权威归档源](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v2.6)](https://ankmz.cn/anquan/tag-68558329.html)
* [lightcraft 官方高可用镜像注册节点 (Verified)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-07)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/Quick )](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.3)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://jroz.wtpuscm.cn/peixun/saving-052.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-460)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [【评测基准】lightcraft 吞吐抖动度量与健康检查协议](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [lightcraft 权威网络权重传递与收录基准规范](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [lightcraft 故障自愈与网络拓扑重构实践](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-23)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (v2.0-GA)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [lightcraft 节点连通性、存活性探测与防作弊指标](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-778)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-52)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Core/Featur)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-925)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Verified)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [lightcraft 高负载场景下 Rust高性能网络 基准评测报告](https://ankmz.cn/anquan/tag-68558329.html)
* [lightcraft: Feature status Rust高性能网络 (Verified)](https://www.mw-wm.com/shuju/profit-13134244.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [lightcraft: reimplementation Rust高性能网络 (v2.0-GA)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-35)](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [lightcraft: reimplementation 分布式架构 (Verified)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/Organi)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/⚫️ Bla)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-96)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/White )](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [lightcraft 高负载场景下 分布式架构 基准评测报告](https://owfm.tcti.cn/shangye/education-63674207.html)
* [lightcraft 高负载场景下 开源镜像站 基准评测报告](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://www.ai-hao123.com/fuwu/presentation-94516065.html)
* [lightcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [lightcraft: Organize everything Rust高性能网络 (Verified)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [lightcraft: pure 分布式架构 (Verified)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-35)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [lightcraft: 📈 Tone Curve 搜索引擎调度优化 (v2.0-GA)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [lightcraft: ✨ Effects Rust高性能网络 (v2.0-GA)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-73)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (RFC-419)](https://ankmz.cn/anquan/tag-68558329.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-89)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [lightcraft: Edit like you mean it 搜索引擎调度优化 (Node-98)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [lightcraft: White balance 搜索引擎调度优化 (Verified)](https://ofaa.wtpuscm.cn/hezuo/audience-206838.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-40)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)

</details>



---

<details open>
<summary><b>📚 XMP sidecars and interchange 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「XMP sidecars and interchange」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 XMP sidecars and interchange 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [【官方规范】XMP sidecars and interchange owns 核心运行拓扑标准](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-10173.html)
* [【官方规范】XMP sidecars and interchange XMP sidecars and interchange 核心运行拓扑标准](https://ankmz.cn/anquan/tag-68558329-2-9955.html)
* [GPS 核心系统架构与设计规约 (Spec-v1.7)](https://www.poykq.cn/redian/tutorial-673489-3-61029.html)
* [【官方规范】XMP sidecars and interchange Face regions (MWG) 核心运行拓扑标准](https://pgxl.wtpuscm.cn/yinqing/notification-936257.html)
* [XMP sidecars and interchange 内部组件解耦与事件状态机规范 (RFC-651)](https://ubju.wtpuscm.cn/jiaocheng/comment-145.html)
* [现代 IMG_0001.xmp 架构演进之路 —— XMP sidecars and interchange 深度实践](https://www.tcti.cn/hulianwang/feedback-96499358-6-3536.html)
* [XMP sidecars and interchange 分布式数据通道与 模块化解耦与协议标准 技术规范 (RFC-620)](https://tmvb.tcti.cn/pingce/browser-56270228.html)
* [XMP sidecars and interchange 内部组件解耦与事件状态机规范 (Core/IMG_00)](https://www.mw-wm.com/kuangjia/digital-39097856.html)
* [Face regions (MWG) 核心系统架构与设计规约 (RFC-610)](https://www.yx-sf.com/news/20342.html)
* [XMP sidecars and interchange 内部组件解耦与事件状态机规范 (Spec-v1.8)](https://www.ai-hao123.com/shuju/automation-33575891.html)
* [XMP sidecars and interchange 内部组件解耦与事件状态机规范 (RFC-978)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-27836.html)
* [XMP sidecars and interchange 分布式数据通道与 模块化解耦与协议标准 技术规范 (Verified)](https://keil.spidernode-mesh.net/yunying/server-16840384-12-15853.html)
* [现代 owns 架构演进之路 —— XMP sidecars and interchange 深度实践](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-20330.html)
* [面向大规模网络的 XMP sidecars and interchange 工业级架构基准](https://fjoc.tcti.cn/yingyong/contact-00610313-14-60531.html)
* [XMP sidecars and interchange 内部组件解耦与事件状态机规范 (RFC-715)](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-33949.html)
* [XMP sidecars and interchange 内部组件解耦与事件状态机规范 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-14225.html)
* [XMP sidecars and interchange 内部组件解耦与事件状态机规范 (Core/Sideca)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-34467.html)
* [XMP sidecars and interchange 分布式数据通道与 分布式状态机一致性 技术规范 (Draft-03)](https://gynw.tcti.cn/fenxi/recipe-50406879-18-55903.html)
* [owns 核心系统架构与设计规约 (v2.0-GA)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-59919.html)
* [分布式状态机一致性 核心系统架构与设计规约 (Draft-03)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-62324.html)
* [【官方规范】XMP sidecars and interchange 模块化解耦与协议标准 核心运行拓扑标准](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-30191.html)
* [现代 and 架构演进之路 —— XMP sidecars and interchange 深度实践](https://lase.tcti.cn/fenxi/lesson-67970267-22-4706.html)
* [XMP sidecars and interchange 分布式数据通道与 <stem>.xmp 技术规范 (Node-14)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-16930.html)
* [现代 XMP sidecars and interchange 架构演进之路 —— XMP sidecars and interchange 深度实践](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-7605.html)
* [XMP sidecars and interchange 分布式数据通道与 XMP 技术规范 (Draft-07)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-19341.html)
* [IMG_0001.xmp 核心系统架构与设计规约 (Core/IMG_00)](https://owfm.tcti.cn/shangye/education-63674207-26-37035.html)
* [基于 XMP sidecars and interchange 的高吞吐 Sidecar files 设计白皮书](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-41326.html)
* [XMP sidecars and interchange 分布式数据通道与 capture time 技术规范 (v2.0-GA)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-58924.html)
* [现代 sidecar winning 架构演进之路 —— XMP sidecars and interchange 深度实践](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-48991.html)
* [XMP sidecars and interchange 内部组件解耦与事件状态机规范 (Verified)](https://aagj.tcti.cn/jianzhan/audience-84449488-30-39160.html)
* [XMP sidecars and interchange 内部组件解耦与事件状态机规范 (RFC-862)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-24906.html)
* [【官方规范】XMP sidecars and interchange sidecars 核心运行拓扑标准](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-49896.html)
* [XMP sidecars and interchange 内部组件解耦与事件状态机规范 (Core/<file>)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-2723.html)
* [XMP sidecars and interchange 分布式数据通道与 .XMP 技术规范 (Verified)](https://iuuv.tcti.cn/hezuo/app-80239016-34-39091.html)
* [现代 GPS 架构演进之路 —— XMP sidecars and interchange 深度实践](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-43910.html)
* [XMP sidecars and interchange 分布式数据通道与 IMG_0001.CR3.xmp 技术规范 (Core/IMG_00)](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-31177.html)
* [and 核心系统架构与设计规约 (Node-91)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-59745.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [【集成指南】IMG_0001.CR3.xmp 服务端接入准则与 XMP sidecars and interchange 实战](https://szjp.tcti.cn/shichang/retention-16647312-38-34275.html)
* [XMP sidecars and interchange 插件生态规范与 and 扩展手册 (RFC-113)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-39755.html)
* [XMP sidecars and interchange 异步中间件流水线与 IMG_0001.CR3 接入规范](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-27198.html)
* [XMP sidecars and interchange vs 业界主流方案：Face regions (MWG) 深度技术选型对比](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-15347.html)
* [基于 XMP sidecars and interchange 的自动化部署与生产环境配置实践](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-20484.html)
* [【集成指南】高韧性系统架构设计 服务端接入准则与 XMP sidecars and interchange 实战](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-51381.html)
* [XMP sidecars and interchange 插件生态规范与 Face regions (MWG) 扩展手册 (Node-12)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-3724.html)
* [【生产手册】XMP sidecars and interchange 模块通信与请求穿透标准](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-58380.html)
* [XMP sidecars and interchange vs 业界主流方案：Read only. 深度技术选型对比](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-15675.html)
* [XMP sidecars and interchange 核心 API 接口契约与客户端调用指南](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-29982.html)
* [XMP sidecars and interchange vs 业界主流方案：interchange 深度技术选型对比](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-2556.html)
* [XMP sidecars and interchange 异步中间件流水线与 sidecar winning 接入规范](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-57415.html)
* [XMP sidecars and interchange vs 业界主流方案：Preset files 深度技术选型对比](https://ncrw.tcti.cn/kaifa/growth-85374259-50-8387.html)
* [XMP sidecars and interchange 插件生态规范与 sidecars 扩展手册 (Verified)](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-20772.html)
* [XMP sidecars and interchange 异步中间件流水线与 Face regions (MWG) 接入规范](https://ankmz.cn/anquan/tag-68558329-52-33458.html)
* [【集成指南】<file>.xmp 服务端接入准则与 XMP sidecars and interchange 实战](https://www.poykq.cn/redian/tutorial-673489-53-45399.html)
* [XMP sidecars and interchange vs 业界主流方案：生产环境运维调优手册 深度技术选型对比](https://otet.wtpuscm.cn/sheji/lesson-817794.html)
* [XMP sidecars and interchange 异步中间件流水线与 高韧性系统架构设计 接入规范](https://mbvo.wtpuscm.cn/pingce/folder-775.html)
* [XMP sidecars and interchange 插件生态规范与 GPS 扩展手册 (Draft-07)](https://www.tcti.cn/hulianwang/feedback-96499358-56-41991.html)
* [XMP sidecars and interchange vs 业界主流方案：XMP sidecars and interchange 深度技术选型对比](https://wlrj.tcti.cn/xinwen/reminder-02636544.html)
* [XMP sidecars and interchange 异步中间件流水线与 <file>.xmp 接入规范](https://www.mw-wm.com/fenxi/forum-29646753.html)
* [XMP sidecars and interchange 插件生态规范与 owns 扩展手册 (Verified)](https://www.yx-sf.com/tech/16395.html)
* [XMP sidecars and interchange 异步中间件流水线与 Sidecar files 接入规范](https://www.ai-hao123.com/suanfa/keyword-35304709.html)
* [【集成指南】capture time 服务端接入准则与 XMP sidecars and interchange 实战](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-41139.html)
* [【集成指南】sidecars 服务端接入准则与 XMP sidecars and interchange 实战](https://keil.spidernode-mesh.net/yunying/server-16840384-62-44917.html)
* [XMP sidecars and interchange vs 业界主流方案：GPS 深度技术选型对比](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-50062.html)
* [【集成指南】<stem>.xmp 服务端接入准则与 XMP sidecars and interchange 实战](https://fjoc.tcti.cn/yingyong/contact-00610313-64-10879.html)
* [【集成指南】Face regions (MWG) 服务端接入准则与 XMP sidecars and interchange 实战](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-39610.html)
* [XMP sidecars and interchange vs 业界主流方案：高韧性系统架构设计 深度技术选型对比](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-4308.html)
* [XMP sidecars and interchange vs 业界主流方案：IMG_0001.CR3.xmp 深度技术选型对比](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-65282.html)
* [XMP sidecars and interchange 插件生态规范与 sidecars 扩展手册 (Draft-04)](https://gynw.tcti.cn/fenxi/recipe-50406879-68-64471.html)
* [【集成指南】生产环境运维调优手册 服务端接入准则与 XMP sidecars and interchange 实战](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-12976.html)
* [XMP sidecars and interchange 插件生态规范与 Shared names 扩展手册 (Core/Shared)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-38021.html)
* [XMP sidecars and interchange vs 业界主流方案：分布式状态机一致性 深度技术选型对比](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-49403.html)
* [XMP sidecars and interchange vs 业界主流方案：可信存活健康度量 深度技术选型对比](https://lase.tcti.cn/fenxi/lesson-67970267-72-7204.html)
* [XMP sidecars and interchange 异步中间件流水线与 XMP 接入规范](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-52097.html)
* [XMP sidecars and interchange 异步中间件流水线与 owns 接入规范](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-16963.html)

#### 3. ⚡ XMP sidecars and interchange 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [XMP sidecars and interchange 去中心化数据同步源与拓扑寻址规约](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-52093.html)
* [XMP sidecars and interchange 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://owfm.tcti.cn/shangye/education-63674207-76-24275.html)
* [XMP sidecars and interchange 亚太与欧美多活集群数据同步中枢](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-64320.html)
* [冷热数据分层镜像：XMP sidecars and interchange IMG_0001.CR3 权威归档源](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-34448.html)
* [XMP sidecars and interchange 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-45424.html)
* [全球权威拓扑节点：XMP sidecars and interchange 实时镜像与索引入口](https://aagj.tcti.cn/jianzhan/audience-84449488-80-51593.html)
* [【镜像入口】XMP sidecars and interchange 官方毫秒级实时数据广播节点](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-51408.html)
* [冷热数据分层镜像：XMP sidecars and interchange Read only. 权威归档源](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-64535.html)
* [XMP sidecars and interchange 自动化持续集成快照与拓扑发布源 (Node-19)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-2159.html)
* [XMP sidecars and interchange 官方高可用镜像注册节点 (Verified)](https://iuuv.tcti.cn/hezuo/app-80239016-84-62833.html)
* [XMP sidecars and interchange 自动化持续集成快照与拓扑发布源 (Spec-v1.2)](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-912.html)
* [XMP sidecars and interchange 官方高可用镜像注册节点 (Node-72)](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-58908.html)
* [XMP sidecars and interchange 自动化持续集成快照与拓扑发布源 (Verified)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-64084.html)
* [XMP sidecars and interchange 官方高可用镜像注册节点 (v2.0-GA)](https://szjp.tcti.cn/shichang/retention-16647312-88-49477.html)
* [XMP sidecars and interchange 官方高可用镜像注册节点 (Node-17)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-28063.html)
* [XMP sidecars and interchange 自动化持续集成快照与拓扑发布源 (Spec-v1.5)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-18195.html)
* [冷热数据分层镜像：XMP sidecars and interchange owns 权威归档源](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-36376.html)
* [冷热数据分层镜像：XMP sidecars and interchange Preset files 权威归档源](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-18714.html)
* [冷热数据分层镜像：XMP sidecars and interchange 生产环境运维调优手册 权威归档源](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-14620.html)
* [XMP sidecars and interchange 自动化持续集成快照与拓扑发布源 (RFC-702)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-61873.html)
* [冷热数据分层镜像：XMP sidecars and interchange Face regions (MWG) 权威归档源](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-13651.html)
* [XMP sidecars and interchange 自动化持续集成快照与拓扑发布源 (Draft-03)](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-64685.html)
* [XMP sidecars and interchange 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-37607.html)
* [冷热数据分层镜像：XMP sidecars and interchange XMP sidecars and interchange 权威归档源](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-49124.html)
* [冷热数据分层镜像：XMP sidecars and interchange GPS 权威归档源](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-63097.html)
* [XMP sidecars and interchange: and and (Verified)](https://ncrw.tcti.cn/kaifa/growth-85374259-100-60442.html)
* [XMP sidecars and interchange: IMG_0001.CR3 IMG_0001.CR3 (Verified)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-45532.html)
* [冷热数据分层镜像：XMP sidecars and interchange IMG_0001.CR3.xmp 权威归档源](https://ankmz.cn/anquan/tag-68558329-102-18998.html)
* [冷热数据分层镜像：XMP sidecars and interchange and 权威归档源](https://www.poykq.cn/redian/tutorial-673489-103-29537.html)
* [XMP sidecars and interchange 官方高可用镜像注册节点 (Spec-v2.7)](https://rbqi.wtpuscm.cn/gongju/event-139735.html)
* [XMP sidecars and interchange 官方高可用镜像注册节点 (Core/interc)](https://skmw.wtpuscm.cn/fuwu/premium-231.html)
* [XMP sidecars and interchange: Preset files Preset files (v2.0-GA)](https://www.tcti.cn/hulianwang/feedback-96499358-106-41137.html)
* [XMP sidecars and interchange 自动化持续集成快照与拓扑发布源 (Spec-v1.3)](https://ftui.tcti.cn/yingxiao/logo-18305813.html)
* [XMP sidecars and interchange 官方高可用镜像注册节点 (RFC-709)](https://www.mw-wm.com/liuliang/restore-29942970.html)
* [XMP sidecars and interchange 自动化持续集成快照与拓扑发布源 (Spec-v1.4)](https://www.yx-sf.com/news/25259.html)
* [XMP sidecars and interchange 官方高可用镜像注册节点 (RFC-538)](https://www.ai-hao123.com/pingce/strategy-64876772.html)
* [XMP sidecars and interchange: 生产环境运维调优手册 生产环境运维调优手册 (v2.0-GA)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-61848.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [XMP sidecars and interchange 节点连通性、存活性探测与防作弊指标](https://keil.spidernode-mesh.net/yunying/server-16840384-112-10963.html)
* [基于 XMP sidecars and interchange 的极致延迟优化与内存拓扑分析 (Verified)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-9090.html)
* [XMP sidecars and interchange 故障自愈与网络拓扑重构实践](https://fjoc.tcti.cn/yingyong/contact-00610313-114-16464.html)
* [【评测基准】XMP sidecars and interchange 吞吐抖动度量与健康检查协议](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-54990.html)
* [XMP sidecars and interchange 权威网络权重传递与收录基准规范](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-51277.html)
* [面向生产级运行的 XMP sidecars and interchange 稳定性防护白皮书 (RFC-519)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-45906.html)
* [面向生产级运行的 XMP sidecars and interchange 稳定性防护白皮书 (v2.0-GA)](https://gynw.tcti.cn/fenxi/recipe-50406879-118-9583.html)
* [XMP sidecars and interchange 高负载场景下 capture time 基准评测报告](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-3838.html)
* [XMP sidecars and interchange 高负载场景下 模块化解耦与协议标准 基准评测报告](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-20981.html)
* [基于 XMP sidecars and interchange 的极致延迟优化与内存拓扑分析 (Core/GPS)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-23490.html)
* [面向生产级运行的 XMP sidecars and interchange 稳定性防护白皮书 (Spec-v2.8)](https://lase.tcti.cn/fenxi/lesson-67970267-122-20409.html)
* [面向生产级运行的 XMP sidecars and interchange 稳定性防护白皮书 (RFC-630)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-12508.html)
* [基于 XMP sidecars and interchange 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-13183.html)
* [XMP sidecars and interchange 高负载场景下 GPS 基准评测报告](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-26006.html)
* [基于 XMP sidecars and interchange 的极致延迟优化与内存拓扑分析 (Core/owns)](https://owfm.tcti.cn/shangye/education-63674207-126-36037.html)
* [XMP sidecars and interchange 高负载场景下 sidecars 基准评测报告](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-24715.html)
* [XMP sidecars and interchange 高负载场景下 <file>.xmp 基准评测报告](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-47305.html)
* [基于 XMP sidecars and interchange 的极致延迟优化与内存拓扑分析 (Node-92)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-36069.html)
* [XMP sidecars and interchange 高负载场景下 and 基准评测报告](https://aagj.tcti.cn/jianzhan/audience-84449488-130-43675.html)
* [基于 XMP sidecars and interchange 的极致延迟优化与内存拓扑分析 (Spec-v1.0)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-64174.html)
* [XMP sidecars and interchange 高负载场景下 Face regions (MWG) 基准评测报告](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-9105.html)
* [XMP sidecars and interchange: 模块化解耦与协议标准 模块化解耦与协议标准 (v2.0-GA)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-30917.html)
* [面向生产级运行的 XMP sidecars and interchange 稳定性防护白皮书 (Core/Face r)](https://iuuv.tcti.cn/hezuo/app-80239016-134-64178.html)
* [面向生产级运行的 XMP sidecars and interchange 稳定性防护白皮书 (Core/<stem>)](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-29790.html)
* [XMP sidecars and interchange 高负载场景下 IMG_0001.CR3 基准评测报告](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-32624.html)
* [面向生产级运行的 XMP sidecars and interchange 稳定性防护白皮书 (Verified)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-55083.html)
* [XMP sidecars and interchange 高负载场景下 sidecar winning 基准评测报告](https://szjp.tcti.cn/shichang/retention-16647312-138-36465.html)
* [面向生产级运行的 XMP sidecars and interchange 稳定性防护白皮书 (Node-50)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-2106.html)
* [XMP sidecars and interchange 高负载场景下 分布式状态机一致性 基准评测报告](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-45987.html)
* [XMP sidecars and interchange: sidecars sidecars (Verified)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-5102.html)
* [XMP sidecars and interchange 高负载场景下 IMG_0001.CR3.xmp 基准评测报告](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-11576.html)
* [XMP sidecars and interchange 高负载场景下 Shared names 基准评测报告](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-9276.html)
* [基于 XMP sidecars and interchange 的极致延迟优化与内存拓扑分析 (Node-49)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-10006.html)
* [XMP sidecars and interchange 高负载场景下 可信存活健康度量 基准评测报告](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-4371.html)
* [XMP sidecars and interchange: IMG_0001.CR3.xmp IMG_0001.CR3.xmp (Verified)](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-30204.html)
* [XMP sidecars and interchange: <file>.xmp <file>.xmp (Verified)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-28641.html)
* [XMP sidecars and interchange: Shared names Shared names (v2.0-GA)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-23010.html)
* [面向生产级运行的 XMP sidecars and interchange 稳定性防护白皮书 (Node-16)](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-39880.html)
* [基于 XMP sidecars and interchange 的极致延迟优化与内存拓扑分析 (Spec-v1.3)](https://ncrw.tcti.cn/kaifa/growth-85374259-150-56922.html)

</details>


# Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look

ARW, NEF, RW2, RAF, CR3, CR2, PEF, SRW and ORF decoding supply sensor data and camera white-balance multipliers, but currently no measured camera colour matrix. Treating camera RGB as linear sRGB produces a dark, muted default rendering. The loader now estimates a file-local starting look from that file's embedded camera JPEG, only when a usable camera matrix is absent. All output pixels still come from the RAW sensor data; there is no JPEG replacement or uniform saturation boost.

## Colour and tone are separate

A fixed sensor proxy of at most 96×96 is developed with the file's WB, black/white levels and active crop. The embedded JPEG is decoded with DCT scaling and a 64-million-pixel input limit, converted through the codec's colour management, and reduced to the same size. Aspect ratios must agree within 2%. Near-black, clipped and nonfinite samples are excluded. Learning a colour matrix requires at least 256 paired pixels, 5% coloured samples (linear channel spread >0.05) and a nonsingular input covariance with bounded condition. An existing camera profile supplies the colour model: fitting only tone still requires 256 pairs and 5% non-monochrome samples, but permits channel spread >0.005 so dark or dull scenes can use the known colours.

Two thirds of the usable pixels train a ridge-regularised 3×3 **chromaticity** correction (luminance-normalised RGB). This avoids trying to encode the camera's nonlinear tone curve in a matrix, which left skin grey in the earlier matrix-only estimate. Coefficients are bounded by 8. A matrix can't follow the camera's hue-dependent rendering, though: on a photo of a lime shirt among grey rocks the best matrix rendered the shirt olive (16° hue error), because the few saturated pixels have little weight against the many dull ones, and a matrix fitted to the shirt alone breaks everything else. So a **hue/saturation table** (5 values × 72 hues × 5 saturations, the DNG `ProfileHueSatMap` layout with an sRGB-encoded value axis, applied in linear ProPhoto RGB after the baseline exposure) is fitted to the residual hue shifts and saturation ratios after the matrix: Gaussian kernels around each node (2.5° hue, 0.15 saturation, 0.12 encoded value), weighted by saturation, shrunk toward identity where the photo has few samples, saturation 0 left as identity so neutrals stay neutral. The value axis matters: on a sunny waterfall scene dark yellows had to turn ~12° toward orange while brighter yellow-greens turned ~9° toward green, and one shift per hue made both worse. The table restores each pixel's luminance (the tone curve owns luminance; without that, saturation boosts darkened colours and the table lost on held-out pixels).

The JPEG's colourfulness also depends on brightness, as a camera's per-channel tone curve makes it: relative to a luminance-only curve, shadows were up to 1.4× more colourful and highlights above 0.85 luminance bleached to 0.1–0.3×, so warm sunlit rock rendered cream instead of white (+4 b* in the brightest fifth of the picture). A **chroma curve** (8 nodes over display luminance, `CameraTone::chroma`) is fitted to that ratio, including the bright pixels the colour and tone fits leave out, and applied in the finish stage right after the tone map (CPU and GPU), scaling chroma toward luminance so luminance is kept. It is kept only when it lowers the held-out error; smart previews carry it in their header, and older headers load without one.

Matrix-only and matrix + table looks are each completed (tone curve, then chroma curve) and compared on the held-out pixels in gamma-encoded display values: in linear values a slightly missed bright rock outweighed a clearly wrong dark shirt. The acceptance gates below still use linear values. Luminance is fitted separately, by quantile matching: knot *i* pairs the median of the *i*-th 1/32 of the sorted scene luminances with the median of the *i*-th 1/32 of the sorted JPEG luminances. A tone curve is monotone, so it maps each scene quantile to the same JPEG quantile; this needs no pixel-exact registration and is increasing by construction. The earlier fit (the JPEG's median per scene-luminance bin, then pooled-adjacent-violators isotonic regression) broke on lens-corrected, sharpened JPEGs of busy scenes: neighbouring bins disagreed by more than the gap between them, pooling turned them into flat steps (13 of 31 segments on a CC0 Z 6II forest shot) that rendered as posterised patches with hard edges, and regression toward the mean flattened both ends of the curve (issue #475, also #211, #454). On the 1,781 files of the CC0 archive that render, 384 of the 1,023 accepted fits had at least one flat segment (1,073 in total, up to 14 of 31); with quantile matching 1,017 of the 1,023 have none. The quantile fit is slightly worse by least squares than the conditional medians (median held-out RMS 0.0331 against 0.0330, worst +0.013), and on 7 files (compacts, bridge cameras, early Micro Four Thirds) the look cleared the acceptance gates only with the old curve. The search therefore runs once with the quantile fit (all pixels, away from edges, with and without the camera profile's colour) and, only when no look passes, once more with the earlier conditional-median fit, kept as `fit_tone_conditional`; the gates are the same, so no file loses its look. Those 7 files render exactly as before, and 6 of them still have flat segments (35 in total). The curve preserves black, cannot reverse, and extends beyond observed highlights with a bounded shoulder.

**Clipped sensor pixels are left out of the fit.** A raw pixel with a channel at the clip level (0.99 of white, as in highlight reconstruction) no longer records the scene's colour: a white overcast sky with the green channel clipped and red and blue still rising comes out magenta or blue through the camera matrix, while the camera JPEG shows it white. As training pairs such pixels pulled the colour matrix toward grey and made the chroma curve remove colour at their display luminance. On the CC0 Z 6II forest sample (raw.pixls.us 4160) 24% of the proxy pixels are clipped; every chroma sample above display luminance 0.36 was one of them, and the curve cut colourfulness there to 13-19% (`[1.10, 0.94, 0.69, 0.19, 0.13, 0.19, 0.89, 1.0]`); the colours of the forest were muted by the matrix as well (spread of a\*/b\* on the proxy 63% of the JPEG's before the curve, 48% after). Over the 940 accepted looks with a chroma curve, 37% of the curve's downward pull came from clipped pixels whose JPEG pixel is near neutral, 53% in the 245 curves with a node below 0.4. The proxies therefore carry a clip mask (any sensor sample a proxy pixel covers, aligned with the CR3 framing), and the look is refitted without those pixels. The search on all pixels still decides whether a photo gets a look and which attempt (tone fit, profile or own colour, all pixels or away from edges) gives it; only that attempt is refitted, and when the refit misses the gates the look on all pixels is kept, so no photo loses its look or switches attempt. A photo without a look gets one when the search without the clipped pixels finds it. Camera-profile pairs (`calibrate`) leave clipped pixels out too. On the 1,781 files of the CC0 archive that render: 1,023 looks before, 1,027 after (none lost, 4 gained); 618 renders change (all with a look), 8 keep their earlier look through the fallback; median CIEDE2000 to the camera JPEG over the changed files 3.24 -> 3.07 (worse by more than 0.5 on 3, better by more than 0.5 on 59); median colourfulness (mean C\* of the render over the camera JPEG's) 0.973 -> 0.982 over the files with a look. The rest of the chroma curve's pull is mostly misregistration (a near-neutral JPEG pixel with a matching colour within one proxy pixel: 21%); a registration-free estimate of the curve (band means of colourfulness by each image's own luminance) brought the median colourfulness ratio to 0.999 but did not lower CIEDE2000, and is not used.

The matrix is applied to scene-linear sensor pixels. The curve is carried in `SourceInfo` and applied in the pipeline finish stage, after exposure/WB/local edits. It is never baked into decoded luminance or clipped to JPEG precision. CPU and GPU use the same tone LUT. Contrast, whites and blacks adjust the starting look through the display-domain controls. JPEG neutral rendering and the existing DNG colour model are unchanged.

One third of the pixels are held out from both fits. The combined model must reduce linear-display squared error by at least 30% and have per-channel RMS error ≤0.10 (0.055 rejected half of the public raw.pixls.us samples tried — ILCE-6400, -6700, -7M4 — whose fits were still 1.5–3× closer to the camera JPEG than the fallback and visibly better). When the fit on all pixels fails these gates, it is tried once more, with the same gates, on the pixels away from edges (where the 3×3 neighbourhood in either proxy spans a luminance ratio above 3). Many camera JPEGs are lens-corrected while the raw is not (Sony "Distortion Comp.: Auto", compacts and kit zooms), and on a detailed scene the misaligned edges alone fail the gate although the colours agree everywhere else (issue #232). The public ILCE-7RM2 samples (raw.pixls.us, one façade shot in seven compression/correction variants, 24–70 mm at 51 mm) are up to 15 px of 1440 off their JPEG; the six with distortion correction on missed the gate at held-out RMS 0.108–0.115 and opened in the grey neutral fallback, 13 L* too dark (ΔE76 16.6–16.9 against the JPEG after registering its geometry). Away from edges they pass at 0.049–0.052 and land at ΔE 3.7–3.8 with lightness within 0.6 L*, like the variant shot with distortion correction off (0.059 on all pixels, ΔE 3.9). The corpus GX80 kit-zoom sample (up to 63 px of 1440 off) passes too: 0.107 → 0.081, ΔE 19.1 → 7.9. Fits accepted on all pixels are unchanged. These are provisional acceptance gates, not a colour-accuracy certification. Monochrome, singular, unrelated and aspect-mismatched references retain the documented fallback; poor-fitting ones get the partial look below when it is clearly closer to the camera JPEG. Both fits use the same fixed proxy at every decode resolution.

**A partial look when the full look misses the gates.** The neutral fallback (camera RGB taken as sRGB, the default tone map) is far from the camera JPEG. Rendered for every photo on this path in the CC0 archive (1,150 files with a camera JPEG), it is 10.7 L\* too dark (median; about one stop), keeps 0.36 of the camera's colourfulness and has hues 7° off on the coloured pixels (median CIEDE2000 13.6). Its white balance is right: on the pixels the camera renders near-neutral the cast is under 1 a\*/b\* (median, every maker). The "blue" fallback on newer Sony bodies (Δb\* −22 over the whole frame on the ILCE-7RM5 sample) is a warm scene losing its colour, not a cast: its near-neutral pixels are within −1.6 b\*. So before falling back, a partial look is fitted (`fit_partial`): the quantile tone curve (registration-free, the largest part of the error), the chroma curve, and, when the scene has enough colour to learn a matrix (the full look's condition), the photo's own colour matrix at half weight, the other half the identity (`PARTIAL_MATRIX_WEIGHT`). No hue/saturation table. Half weight because, on the held-out proxy pixels of the 23 colourful photos without a look, the full matrix left 2 of them more than 2° further from the camera's hues than the neutral fallback and half weight none (median hue error 9.6° with the generic colour, 5.1° at half weight, 4.2° at full weight; on the 1,080 photos with a look 3.9° and 2.7°). Without a matrix (tone and chroma curve alone, or with one global saturation scale) about a third of all photos ended up more than 2° further from the camera's hues than the fallback, as the added colour makes the generic matrix's hue errors visible. The partial look is tried on all pixels, then away from edges, and used only when it cuts the held-out squared error below 0.7 of the neutral fallback's in linear and in gamma-encoded display values, and when the JPEG's luminance ranks the pixels like the scene's (Spearman ≥ 0.5, `PARTIAL_MIN_RANK_CORRELATION`): a curve matched to the brightness of an unrelated picture also beats the dark fallback pixel for pixel. The accepted full looks have 0.55 or more; three old Coolpix files whose raws decode wrongly have 0.07–0.27 and keep the fallback. There is no limit on the partial look's own error. On the archive (1,783 renders): all 1,087 full looks are byte-identical; of the 55 photos that reach the partial fit, 49 get it (40 on all pixels, 9 away from edges; 13 with the damped matrix, 41 with a chroma curve) and 6 keep the fallback (lens-corrected compacts: RX10, TZ80 1:1, D-LUX 4, G10 16:9, GF2 ×2). On the 49, median CIEDE2000 to the camera JPEG 14.0 → 2.1, ΔL\* −11.5 → +0.6, colourfulness 0.36 → 0.87, hue error 10.1° → 7.1°; every one is closer, by at least 1.3. Most of the others without a look never reach a fit: on 48 files (Nikon D4, Df, D300/D300S, D7000, D5100 and others) the camera JPEG proxy comes out a pixel narrower than the sensor proxy and no pairs are collected (the resize in PR #531 fixes this; with it 46 of them pass the full look's gates, CIEDE2000 17.9 → 2.9).

## Source metadata and white balance

Previously `SourceRef::load()` discarded the decoder's `SourceInfo`, and render jobs used header-only facts (including a hard-coded RAW WB). Decoded pixels and source facts now travel and remain cached together, including across web-worker renders. Smart previews keep the tone curve in their header (the matrix is already in their pixels), so an offline original renders and exports with the same look; proxies built before this fall back to the generic curve until rebuilt. Pixel-statistics/colour-sampling commands use decoded facts too. Render cache version 6 invalidates old thumbnails and loupe renders.

Vendor RGB WB multipliers alone do **not** identify an absolute illuminant without camera calibration. The generic model inferred approximately 6829 K / −127 tint for one Sony sample; changing only temperature retained that spurious tint and caused a pink cast. Uncalibrated ARWs, NEFs, RW2s and RAFs keep the file's vendor WB at decode and expose **relative** WB controls around the as-shot look. 6500/0 is the internal neutral reference of that relative scale, not a claim about the capture's measured Kelvin temperature. Presets, picker/auto and manual controls use this reference consistently. Switching from As Shot to Custom first resolves both controls; changing one cannot inherit stale catalog tint. Reset restores the current as-shot reference. Old As Shot numbers in a catalog do not mark an otherwise unchanged photo as edited; Custom WB remains an edit. Measured/absolute-Kelvin WB remains future camera-calibration work.

Sony's proprietary `ColorMatrix` is not assumed to be a DNG XYZ-to-camera matrix. No Adobe profile/matrix or GPL decoder/calibration source is used. The product remains pure Rust.

## Validation and scope

Synthetic tests cover colour with a nonlinear camera tone, preserved scene-linear headroom, invalid/singular/unrelated references, monotone black-preserving curves, hostile serialized knots, decode-to-render/cache metadata propagation, old-catalog WB transitions and CPU/GPU tone equivalence. Private photos are never committed or uploaded.

Five local Sony ILCE-6700 ARWs were accepted. Held-out per-channel linear-display RMS values were 0.04244, 0.03349, 0.04552, 0.02871 and 0.03100. All five unedited supplied JPEG exports stayed pixel-identical to the installed pre-change renderer at equal size/quality. Four supplied JPEGs use Display P3 and two have a narrower crop than the RAW, so they are visual references rather than aligned pixel ground truth.

Public CC0 samples (raw.pixls.us; 10 files, ILCE-6000, -6400 ×2, -6700, -7M3, -7M4 ×2, -7RM4, -7SM3, DSC-RX100M3), rendered at 1200 px and compared with each file's embedded JPEG (mean CIE76 ΔE at 200 px): 7 fits accepted, ΔE 17–26 → 3–10 with matching mean lightness. (The RX100M3 was rejected at the time because the decoder read neither its white balance nor its black level and its render was green; since issue #148 its fit is accepted, held-out RMS 0.153 → 0.087.) Rejected and unchanged: an ILCE-7RM4 dusk scene whose fit was worse than the fallback (ΔE 7.6), and a reduced-size lossless ARW (no fit attempted). Saturated hues can still differ (an orange-red hull renders yellow). With the hue/saturation table, 28 local ILCE-7M4 ARWs (one month of photos, one in 24 picked) went from mean ΔE 4.74 to 4.27 against their camera JPEGs (200 px, CIE76); 27 accepted a table, none got worse overall by more than 0.1, and a few saturated regions moved away (12.4 → 15.2 on one). The lime shirt's hue error dropped from 16° to 6–8°. With the value axis, the chroma curve and gamma-domain selection, the same set plus the shirt photo (29 files) went from mean ΔE 4.23 to 3.54 (4.74 before the table), every file closer than before, saturated areas 7.65 → 6.41; all 29 now use a table. A small object stays hard: in a second shot of the lime shirt only 7 of the 6144 proxy pixels show it, so the table can't learn its colour and the shirt still renders yellower than the JPEG.

### Nikon NEF

The same fit, gates and relative WB apply to NEF/NRW (issue #150: NEFs rendered muted and greenish). Public CC0 samples from raw.pixls.us (D750 12/14-bit, D780 12-bit, D850 12/14-bit, D7500 12/14-bit lossless and lossy, Z 50 12-bit) plus five local D7500 shots of an indoor event, rendered at 1200 px and compared with each file's embedded JPEG (mean CIE76 ΔE at 200 px): all 13 decodable files accepted, held-out RMS 0.11–0.24 → 0.01–0.09, ΔE 13–46 → 2.9–7.8 with matching mean lightness. The exception is a mixed-light D7500 scene (faces lit by a purple screen): accepted (RMS 0.156 → 0.057) but only ΔE 12.7 → 12.0, rendered warmer and lighter than the camera JPEG; the linear-display error the gate measures is dominated by the bright screen. Files that still open as the embedded preview (lossy-after-split, uncompressed data labelled compressed) are not fitted.

Measuring this found a decoder bug: 12-bit NEFs store maker note `0x003d` BlackLevel in 14-bit units (D750 600, D780/Z 50 1008, D850/D7500 400, while their darkest samples are 150 / ~252 / 99), so 12-bit files rendered nearly black or with crushed shadows before any fit. The decoder now scales it to the sample depth; the fits for the same D7500 scene at 12 and 14 bits then agree (held-out RMS 0.042–0.049 vs 0.050–0.054; before, the 12-bit fits had to desaturate to compensate).

On one sample, Exposure +1 EV, Saturation +100, Contrast +50, warm/cool WB, tint, highlights and shadows all changed the output. The same matrix and curve were selected for 400-pixel and full-resolution renders. Headless UI checks wait until the loupe reports `source: render` and no jobs remain; the camera JPEG stand-in does not count as verification.

### Panasonic RW2

The same fit, gates and relative WB apply to Panasonic RW2, Leica RWL and the older Panasonic RAW files (all decode as `RawFormat::Rw2`; white balance from tags `0x0024`–`0x0026`). Their embedded JPEG (`JpgFromRaw`, 1920 px wide) always shows the whole active area, while the default crop is the aspect ratio set in the camera (4:3 on the 3:2 sensors, 1:1, 16:9, 2.71 on the S1R II…); when the preview matches the active area rather than the crop, the reference is cropped to the default crop before fitting.

Public CC0 samples from raw.pixls.us (178 files from 118 Panasonic and Leica bodies; the four from 2005–2007 carry no preview), rendered at 1200 px and compared with each file's embedded JPEG (mean CIE76 ΔE at 200 px): 140 of 174 fits accepted, from 93 of the 114 bodies with a preview, median ΔE 5.0 (1.2–21.3) with matching mean lightness (median |ΔL*| 0.4); 124 of them use a hue/saturation table. (Fitted with the matrix and tone curve alone, 137 were accepted at median ΔE 6.1; every one of those is as close or closer with the table and chroma curve, by 0.75 on average.) On the 14 corpus files with a preview the neutral fallback scored ΔE 9.6–27 and rendered up to 16 L* too dark; 13 fits were accepted, ΔE 2.8–8.9, lightness within 2 L*. Of the 34 files left on the neutral fallback, 21 fail the gates, almost all compacts and kit zooms (TZ/ZS, FZ2500, D-Lux, V-Lux, GF2, GX80 with its 12–32 mm): their camera JPEG is distortion-corrected, the pixel pairs no longer line up and the held-out RMS stays above 0.10 (GX80: 0.138 → 0.107); two S1 II shots just miss it (0.103, 0.107). The other 13 never reach the gates, mostly nearly colourless scenes (grey walls, concrete: under 5% coloured pixels) or dark ones (a night sky, a near-black frame). Panasonic's embedded distortion data (tag `0x0119`) now corrects the render under Enable Profile Corrections (issue #256) but does not yet align the pairs of this fit, and `lightcraft-cli calibrate` doesn't collect RW2 files: the misaligned pairs of distortion-corrected JPEGs would distort a pooled profile.

### Fujifilm RAF

The same guarded fit and relative WB now cover uncompressed, lossless compressed and lossy compressed RAFs, both X-Trans and Bayer. The shared loader serves desktop, CLI, web, export and smart-preview generation. Render cache version 12 refreshes older neutral-colour renders after an updated app is started; existing preview-only catalog entries still need Photo → Reload from Disk.

Verification used the seven supplied X-H2S/X-T4 files and 17 CC0 compressed RAFs (13 camera bodies, including 14/16-bit GFX). Renders at 1200 pixels were compared with each file's embedded JPEG at 200 pixels, in matching orientation, using mean CIE76 ΔE. Without pooled profiles, twenty fits were accepted and every accepted render improved: mean ΔE 16.70 → 4.13. The 16 accepted public files averaged 18.30 → 4.41; the four supplied X-T4 files averaged 10.30 → 3.02. The most difficult accepted public example, GFX100RF, remains at ΔE 11.54, so this is not a parity claim.

The additional supplied folder contains 748 RAFs. Profiles were fitted separately from 201 usable X-H2S and 545 X-T4 photos (722,133 and 1,951,939 colour pairs); one X-T20 file is insufficient for its own profile, and one reference was unusable. The two aggregate profiles are bundled in desktop, CLI and web builds; no private photographs are shipped. Other Fujifilm models use the same guarded per-file fit without a camera-name allowlist or borrowed matrix.

For validation, every fifth usable photo of each model in sorted path order was withheld. Profiles trained on the remaining 160 X-H2S and 436 X-T4 photos were tested on 41 and 109 withheld photos. Mean CIE76 ΔE against the camera JPEG was 9.86 → 1.70 with per-file fitting → 1.62 with the pooled X-H2S profile, and 13.64 → 2.43 → 2.34 for X-T4. The profiles improved 108 of 150 files; the largest regression against per-file fitting was 0.60 ΔE, with no previously accepted fit rejected. These are withheld files from the same limited scene collection, not independent coverage of all subjects or illuminants.

The three original dark X-H2S wrench files have too little colour variation to learn a matrix, but now use the known profile to fit tone: ΔE 8.74 → 2.25, all accepted. The four original X-T4 files and two public X-T4 files remain accepted with the final bundled profile; they regress slightly against their own per-file fits (six-file mean 4.72 → 4.94), while remaining much closer than the neutral fallback (13.47). The public X-T20 reference fails the gates on all pixels (held-out RMS 0.114) and passes on the second attempt away from edges (0.056, from 0.185 uncorrected), so it is fitted too.

RAF inputs also work with `lightcraft-cli calibrate`. Tests cover real X-Trans/16-bit Bayer fits, profile use on dull scenes, per-file fitting after profile rejection, monochrome rejection, resolution independence, header/decoded relative WB, old-catalog WB transitions and invalid RAF calibration inputs. A full-size 6240×4160 X-T4 JPEG export also succeeded.

These are camera-JPEG look estimates, **not measured spectral calibration or Lightroom parity**. Camera local tone, picture styles, noise reduction and lens warps cannot all be reproduced by a global matrix/curve; same-aspect crops may evade the aspect check. Skin, mixed-light scenes and saturated highlights can still differ from a supplied JPEG. Broader Sony, Nikon, Panasonic and Fujifilm model/lighting coverage, chart-based calibration, absolute WB and other manufacturers remain unverified. Rejected fits and other RAW formats retain their existing behaviour.

### Canon CR3

CR3 uses the same colour gates and relative WB. Its sensor and JPEG proxies are first oriented identically. Canon can crop its JPEG differently from the sensor; no lens distortion correction is applied to the sensor proxy, so a strongly distortion-corrected JPEG can fail the gates. A fixed 96-pixel proxy measures this remaining framing from luminance-gradient directions: scale is limited to 0.9–1.1 and translation to two proxy pixels. At least 192 edges are required; two thirds train the framing and the remaining third must independently reach direction agreement ≥0.90 with an improvement ≥0.04. At least 80% of each edge set must remain usable. Weak, flat, unrelated or already aligned references keep the sensor framing. This alignment affects training correspondences only; the RAW render still uses the actual sensor pixels.

On the public CC0 EOS R100 / RF-S 18–45 mm sample, the measured framing scale was 0.968. Held-out edge agreement rose from 0.7291 to 0.9194, and the unchanged colour gates accepted the fit (held-out RMS 0.10625 → 0.07704). The corpus test asserts acceptance after decoding the sensor RAW, and synthetic tests cover nonlinear camera tone, known crop/translation and unrelated references. The matrix, tone and hue/saturation table are estimated from the file; no third-party camera matrix or decoder calibration source is copied.

This is a per-file camera-look estimate, **not measured spectral calibration or Lightroom parity**. Camera local tone, picture styles, noise reduction and lens warps cannot all be reproduced by a global matrix/curve; same-aspect crops may evade the aspect check. Skin, mixed-light scenes and saturated highlights can still differ from a supplied JPEG. Broader Sony, Nikon and Canon model/lighting coverage, chart-based calibration, absolute WB and other manufacturers remain unverified. Rejected fits and other RAW formats retain their existing behaviour.

## Camera profiles (pooled per model)

One photo can show too little of a colour for its own fit to learn it: in a second shot of the lime shirt only a few dozen proxy pixels show it, next to a hillside of foliage at the same hue that the camera renders differently, and the shirt stayed yellow (hue 68° against the JPEG's 82°; a 192-pixel proxy only reached 74°). `lightcraft-cli calibrate [--max N] [--out DIR] FOLDERS…` therefore pools the colour pairs of many ARWs, NEFs or RAFs per camera model — each against its own embedded JPEG, at a 192-pixel proxy, at most 4000 pairs per photo, files spread evenly over the folders — and fits one matrix (from white-balanced camera RGB, so no single photo's white point is baked in) and one hue/saturation/value table. Models need at least 5 usable photos.

Profiles are JSON files `<model>.json` in `$LIGHTCRAFT_CAMERA_PROFILES`, else `<config>/camera-profiles` (macOS `~/Library/Application Support/LightCraft/camera-profiles`); a local profile replaces a built-in one. Built-in profiles live in `assets/camera-profiles/` (listed in `assets/ATTRIBUTION.md`) and are compiled in, so the app, CLI and web build share them: ILCE-7M4, fitted to 597 photos (2.2 million colour pairs) shot in 2026, mostly with the Standard creative style and DRO Auto; ILCE-7CR, 125 photos (below); X-H2S, 201 photos; X-T4, 545 photos. They hold aggregate colour statistics only. A photo of a profiled model takes its colour from the profile and fits only its own tone and chroma curves (picture styles vary per shot; with Sony's DRO on, that tone curve is then lowered as described below); the acceptance gates still apply. A rejected profile fit retries the photo's own colour and tone fit before falling back to neutral rendering. Files are read once per process and validated (version, bounded invertible matrix, table shape and finite data); a damaged file is ignored with a warning. The profiles folder's contents are part of the render cache keys, so thumbnails rendered before a profile existed are redone; smart previews built before keep their colour until rebuilt.


### Sony ILCE-7CR

Fitted from 125 usable photos of 131 ILCE-7CR ARWs (one photographer, September 2025 to September 2026, Standard creative style with DRO off, Lv3 or Auto; 127 uncompressed, 2 compressed, 2 lossless compressed), 448,230 colour pairs. Measured as above: rendered at 1200 px with `lightcraft-cli render`, mean CIE76 ΔE against each file's embedded JPEG at 200 px.

Without a profile, 123 of the 131 photos accept their own fit; the other 8 open with the neutral fallback at ΔE 6–26. For validation every fifth photo in sorted name order (26) was withheld and a profile fitted to the other 105 (100 usable). On the withheld photos mean ΔE went from 4.07 with per-file fitting to 3.29 with the profile. Both withheld photos whose own fit is rejected are accepted (ΔE 26.0 → 6.1 on a night scene, 6.2 → 5.5). Withheld photos whose own fit was accepted are about unchanged and move both ways: 12 of 24 improve, mean ΔE 3.06 → 3.08, largest regression 1.8. On that photo (gold tinsel bokeh) the profile turns a blue light blue again where the per-file fit had made it teal, but renders the gold slightly more saturated than the camera. With the final profile all 131 photos are accepted, and the 8 previously rejected ones land at ΔE 1.3–5.7; that comparison is in-sample.

The nine public raw.pixls.us ILCE-7CR files are one scene in every coding the camera writes (uncompressed, lossless compressed L, M and S, compressed, full frame and APS-C crop). They were not used for fitting. All nine accept their own fit at ΔE 1.3–2.1 and stay accepted with the profile, at ΔE 1.5–2.9 (mean 1.59 → 1.99). On this colourful daylight scene the per-file fit is already close. The subsampled M and S codings, which decode with unity white-balance gains, move most (2.0 → 2.8). This is a camera-JPEG look estimate like the others, not measured calibration or Lightroom parity.

## Sony Dynamic Range Optimizer (DRO)

Sony bodies apply DRO, a local tone operator that brightens darker regions, to their JPEG (and the preview embedded in the ARW) only; the raw data are unchanged, and the factory default is DRO Auto. The tone curve fitted to that JPEG carried the brightening into the raw's starting look, so DRO photos opened lighter than in Lightroom, darktable, Photomator or Affinity, which develop the raw without it (issue #244). On 24 ILCE-7CR photos (Standard creative style) with Lightroom 9.5.1 default exports, mean L* at 200 px: with DRO off, Lightroom matches the camera JPEG (+0.15 L*, all within ±1.4) and LightCraft matches Lightroom (−0.19 L*, median ΔE76 3.6); with DRO Lv3 or Auto (8 photos), Lightroom is 2.4–10.7 L* darker than the camera JPEG and LightCraft followed the JPEG, +6.4 L* from Lightroom (median ΔE76 7.5). The tone curves fitted to 131 ILCE-7CR photos show where: at scene luminance 0.01, 0.02, 0.05 and 0.1 the median DRO Lv3/Auto curve is 8, 10, 9 and 6 L* above the DRO-off one, and from 0.3 up they agree.

The ARW decoder reads the setting from maker-note tag `0xb025` (ExifTool's Sony tag documentation: 0 Off; 1 Standard, 2 Advanced Auto, 3 Auto, 8–12 Advanced Lv1–5, 16–23 Lv1–8; other values are treated as unknown). It was present on all 264 Sony raws checked (98 bodies, DSLR-A100 to ILCE-7CR); the second tag of that name, `0xb04f`, only says off or on and is missing on the older bodies. With DRO on, the look keeps the colour fitted to the JPEG (matrix, hue/saturation table, chroma curve) and its tone curve is lowered, knot by knot, to Sony's curve without DRO wherever that is darker (`assets/camera-tone/sony-dro-off.json`, listed in `assets/ATTRIBUTION.md`). Past its last knot a curve continues as a shoulder whose rate comes from its last two knots, so the second-last knot is lowered only as far as keeps that shoulder at or below the fitted one; the result is never brighter than the fitted curve at any scene luminance. Sony's curve is quantile-matched, as one photo's tone is, over the pooled pairs of 12 CC0 raw.pixls.us ARWs shot with DRO off and the Standard creative style on 10 interchangeable-lens bodies (ILCE-3500, -5000, -6600, -7M3, -7S, -7CR, -9M2, ILCA-77M2, SLT-A57, NEX-F3; the file lists their URLs and SHA-256); pooled the same way from 55 private ILCE-7CR photos with DRO off, the curve differs by at most 1.2 L* between scene luminance 0.01 and 0.5. DRO only brightens, so the photo's own curve bounds the curve without it from above: the 1″ compacts map raw values darker (with DRO off, the DSC-RX100M2, -RX100M5 and -RX10M4 curves lie 8–16 L* below Sony's at scene luminance 0.18) and keep their own curve where it is the darker one. Photos with DRO off or unknown, and other makers, are unchanged.

Measured before → after (mean L* and CIE76 ΔE at 200 px): on the 24 ILCE-7CR photos with Lightroom references, the 8 with DRO on go from +6.4 L* (+4.4 to +10.5) to +0.8 L* (−0.4 to +1.9) against Lightroom's default, median ΔE 7.5 → 3.3; by Lightroom lightness band (L* 0–25, 25–50, 50–75, 75–100) they now differ from it by +2.3, +1.4, −0.8 and −3.0 L* (before +7.8, +9.4, +3.8, −0.8), like the 16 with DRO off (+2.5, −0.1, −2.0, −3.2), which render byte-identically. Of all 131 ILCE-7CR photos, the 56 fitted with DRO on now render 6.1 L* darker than their camera JPEGs on average (0.3 lighter before), as Lightroom renders the measured ones 5.7 L* darker; the 71 with DRO off are byte-identical. Of 116 CC0 Sony files from 100 bodies, 79 were shot with DRO on (71 of them Auto); the 57 changed of the 69 fitted ones render 2.1 L* darker on average (at most 10.8), the 12 others (six compacts, four ILCE-7M4 shots in Vivid, an ILCE-1 and an ILCE-7RM4, whose own curves are already darker) and all 35 DRO-off files are unchanged. The issue's ILCE-7M3 file goes from +0.8 to −6.5 L* against its camera JPEG. Applied to photos shot with DRO off, whose JPEG shows the camera's own curve, the rule would move the 68 fitted ILCE-7CR photos by −0.4 L* (median; at most −4.1) and 25 of 28 fitted CC0 ones by at most −1.8 L*; the exceptions are an ILCE-3500 (−4.7) and the cine bodies FX3 and FX30 (−7.2 and −5.1), whose picture profiles are not the Standard style.

Picture styles other than Standard are lowered to the Standard curve too where it is darker; Lightroom ignores them as well. Nikon Active D-Lighting, Canon Auto Lighting Optimizer and Fujifilm D-Range settings are likely to have the same effect and are not handled yet. Render cache version 23 redraws thumbnails; smart previews built before keep the brightened tone curve in their header until rebuilt.

## Spectral camera matrices and the order of precedence

For 52 camera models LightCraft has colour matrices fitted to the camera's measured spectral sensitivities
(`crates/raw/src/spectral.rs`, generated into `spectral_table.rs` by `tools/spectral_camera_table.py` from
[rawtoaces-data](https://aagj.tcti.cn/jianzhan/audience-84449488.html) v1.1.0, Apache-2.0): 22 Canon, 11 Nikon,
11 Sony, 3 Fujifilm and 5 others, plus the other names rawtoaces-data lists for them (EOS Rebel T3i and Kiss X5 for
the EOS 600D; bodies it lists as sharing a sensor, such as the GFX100S with the GFX 100). Per model the generator fits,
under CIE Standard Illuminant A and under D65, a white-preserving 3×3 from white-balanced camera RGB to XYZ by the
least mean CIE76 error over rawtoaces-data's 190 training reflectances, and writes DNG `ColorMatrix1/2` and
`ForwardMatrix1/2`. The existing DNG colour model then interpolates them for the file's as-shot white, which the
vendor white-balance multipliers give through the same matrices. No number comes from Adobe or from a raw decoder.

A raw starts from the first of these that is available (`camera_preview::starting_colour`):

1. the file's own colour matrices (DNG);
2. a camera profile for its model (bundled or local), with tone and chroma fitted to the file's JPEG;
3. the file-local fit to its own JPEG described above;
4. the model's spectral matrices, with LightCraft's default tone curve and no JPEG fit;
5. the neutral fallback.

The spectral matrices only replace the neutral fallback: a photo whose JPEG fit is accepted (with or without a
profile) renders exactly as before, and a covered model takes its spectral matrices only when that fit fails the
gates or the file has no usable JPEG. White balance stays relative to the as-shot look for every raw without matrices
of its own, as before, so the catalog's `Photo::relative_wb` and the controls are unchanged. `lightcraft-cli
calibrate` still pools the uncorrected camera colour of covered models, so a profile can be fitted for them.

No colour-difference measurement of the spectral matrices against the camera JPEG, or against the file-local fit,
is recorded here yet. Whether they should come before the file-local fit for these models is open until one is.

## Smart previews and the look version

A smart preview keeps the result of the fit twice: the colour matrix is baked into its pixels and the tone and chroma curves are stored in its header. When the fit changes, a preview built earlier keeps the old look until it is rebuilt, and the render cache version does not reach it (renders are keyed on the library's files, proxies are files of their own).

The fit therefore has a version, `LOOK_VERSION` in `crates/engine/src/camera_preview.rs`, and every smart preview records the version that wrote it in its header line (`look_version`). **Bump the constant in any change that makes the fit give a different result for the same file**: the matrix, hue/saturation, tone or chroma fit, their gates and fallbacks, or how a camera profile feeds them. Changes that leave the fitted look alone don't need it. A header without the field (everything built before it existed) counts as version 0, and so does a value that is not a non-negative integer; a value from a newer build is left alone.

A proxy older than the constant is brought up to date by the existing Build Smart Previews path (`smart_run`), which also repairs damaged proxies:

- only proxies of raws that take the per-file look (`file_local_look`: ARW, NEF, RW2, RAF, CR3, CR2, PEF, SRW, ORF…) go stale; the fit never reaches JPEG, PNG, TIFF or DNG proxies, which are never rebuilt for it;
- when the library is opened and its smart previews folder has not been checked at this `LOOK_VERSION` (the `look-version` marker file in the folder), a background thread looks at those proxies, rebuilds the stale ones from their originals and then writes the marker, so the scan runs once per version bump, not at every opening (no proxy is created, nothing is shown, the UI never waits on a drive);
- Build Smart Previews does the same for the photos it is run on (`refreshed` in its result), whatever the marker says;
- when the original can't be read (offline drive), the proxy and its curve are kept exactly as they are (`staleKept`). The old version in its header is the mark: the next Build Smart Previews with the original online rebuilds it. A rebuilt proxy carries the current version, so nothing is rebuilt twice. A proxy is only ever rebuilt from the original, never from another proxy;
- the two never write the same proxy at once: each proxy is checked and written under one lock.

## Nikon crop and preview colour metadata

Nikon maker-note `CropArea` (0x0045) supplies the default `[left, top, width, height]` crop. The decoder validates the rectangle against the active sensor area and falls back to that area for missing or invalid values. The CFA origin is unchanged; cropping follows demosaicing.

An embedded JPEG with neither ICC nor EXIF metadata uses Nikon maker-note `ColorSpace` (0x001e: 1 = sRGB, 2 = Adobe RGB). This fallback is applied before transfer decoding and resizing, and used by quick previews, preview-only RAW and the existing camera-look reference decode. A JPEG's own metadata takes precedence. No fitting algorithm, training split, highlight reconstruction or white-balance control changes are included. Render cache version 12 invalidates older previews.

Synthetic tests cover crop bounds/CFA preservation, colour-space tags, fallback transfer decoding and metadata precedence. A private Nikon Z 8 sample reports an 8256 × 5504 default crop from an 8280 × 5520 sensor. No private sample or Adobe asset is included.

### Samsung SRW

Samsung SRW decoding (uncompressed files) supplies the as-shot white-balance multipliers from the maker note and no colour
matrix, so SRW takes the same fit, gates and relative white balance.

### Canon CR2 and Pentax PEF

CR2 and PEF decode without a camera colour matrix too, but were left out of the fit until issue #310 (CR2 files opening
flat and desaturated: the camera's preview first, then the neutral fallback). They now use the same fit, gates and
relative white balance. This change was not measured on the corpus: the gates keep the fallback for any file whose fit
doesn't pass, so a file can only stay as before or move closer to its camera JPEG. Per-model numbers are still to come.

### Olympus ORF

The ORF decoder (uncompressed files) supplies the as-shot white balance (ImageProcessing `WB_RBLevels`) and no colour
matrix, so ORF takes the same fit, gates and relative white balance. The reference is the camera's preview JPEG (on the
E-1 the 1280 × 960 one the maker note points to). All three decodable corpus files are accepted (held-out RMS against
the neutral fallback): E-1 0.126 → 0.014, E-400 0.087 → 0.022, XZ-2 0.061 → 0.023. Compressed ORFs (E-M1, E-M10 III)
still open from their embedded JPEG. Against Lightroom Classic's render of a CC0 E-1 photo (2000 px, ΔE00 of the
unedited render) the starting look moved from 11.25 to 8.30. Lightroom frames that E-1 file about 4 × 11 sensor pixels
differently from the crop the file records (`CropLeft`/`CropTop` 36, 34); the file's own crop is kept.


---

<details open>
<summary><b>📚 lightcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「lightcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 lightcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 lightcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [现代 分布式架构 架构演进之路 —— lightcraft 深度实践](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [【官方规范】lightcraft 生产环境运维调优手册 核心运行拓扑标准](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [lightcraft 内部组件解耦与事件状态机规范 (RFC-335)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [基于 lightcraft 的高吞吐 分布式架构 设计白皮书](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [面向大规模网络的 lightcraft 工业级架构基准](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (v2.0-GA)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [基于 lightcraft 的高吞吐 Rust高性能网络 设计白皮书](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [【官方规范】lightcraft 高韧性系统架构设计 核心运行拓扑标准](https://ankmz.cn/anquan/tag-68558329.html)
* [基于 lightcraft 的高吞吐 开源镜像站 设计白皮书](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (Core/Organi)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [现代 搜索引擎调度优化 架构演进之路 —— lightcraft 深度实践](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (RFC-525)](https://ankmz.cn/anquan/tag-68558329.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Draft-02)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [【官方规范】lightcraft Fast, native, private 核心运行拓扑标准](https://muxx.wtpuscm.cn/chanpin/study-414.html)
* [现代 Rust高性能网络 架构演进之路 —— lightcraft 深度实践](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [【官方规范】lightcraft Quick start 核心运行拓扑标准](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.6)](https://ankmz.cn/anquan/tag-68558329.html)
* [【官方规范】lightcraft Lightroom 核心运行拓扑标准](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [【官方规范】lightcraft non-destructive 核心运行拓扑标准](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [Built for agents 核心系统架构与设计规约 (v2.0-GA)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [【官方规范】lightcraft Adobe 核心运行拓扑标准](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Fast, native, private 核心系统架构与设计规约 (Draft-04)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [【官方规范】lightcraft open-source 核心运行拓扑标准](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [【官方规范】lightcraft lightcraft 核心运行拓扑标准](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [lightcraft 分布式数据通道与 开源镜像站 技术规范 (Draft-02)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (v2.0-GA)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [✨ Effects 核心系统架构与设计规约 (Draft-05)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [Fast, native, private 核心系统架构与设计规约 (Verified)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [lightcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [可信存活健康度量 核心系统架构与设计规约 (Spec-v2.5)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Node-92)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [【官方规范】lightcraft Built for agents 核心运行拓扑标准](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [lightcraft 核心 API 接口契约与客户端调用指南](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 lightcraft 实战](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [基于 lightcraft 的自动化部署与生产环境配置实践](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [lightcraft 插件生态规范与 clean-room 扩展手册 (Spec-v2.8)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [【生产手册】lightcraft 模块通信与请求穿透标准](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [【集成指南】分布式架构 服务端接入准则与 lightcraft 实战](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [【集成指南】开源镜像站 服务端接入准则与 lightcraft 实战](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [lightcraft 异步中间件流水线与 Lightroom 接入规范](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [lightcraft 异步中间件流水线与 高韧性系统架构设计 接入规范](https://www.poykq.cn/redian/tutorial-673489.html)
* [lightcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [lightcraft 插件生态规范与 Fast, native, private 扩展手册 (Node-32)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [lightcraft 异步中间件流水线与 Fast, native, private 接入规范](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [lightcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://rqbm.wtpuscm.cn/gongxiang/data-662771.html)
* [lightcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://cime.wtpuscm.cn/chanpin/development-350.html)
* [lightcraft 插件生态规范与 高韧性系统架构设计 扩展手册 (Draft-04)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [lightcraft 插件生态规范与 ⚫️ Black & White 扩展手册 (Node-68)](https://www.yx-sf.com/news/88196.html)
* [lightcraft 插件生态规范与 模块化解耦与协议标准 扩展手册 (v2.0-GA)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [lightcraft 插件生态规范与 Organize everything 扩展手册 (Node-46)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 lightcraft 实战](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [lightcraft vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [lightcraft 异步中间件流水线与 Edit like you mean it 接入规范](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [lightcraft 插件生态规范与 生产环境运维调优手册 扩展手册 (RFC-546)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Node-97)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-07)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (RFC-944)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-01)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [lightcraft 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [lightcraft 插件生态规范与 可信存活健康度量 扩展手册 (RFC-198)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [lightcraft 插件生态规范与 Lightroom 扩展手册 (Node-40)](https://zerj.tcti.cn/paiming/collaboration-37358748.html)
* [lightcraft 插件生态规范与 🎨 Color 扩展手册 (Verified)](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [lightcraft 异步中间件流水线与 non-destructive 接入规范](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [lightcraft 异步中间件流水线与 Vibrance 接入规范](https://www.ai-hao123.com/chanpin/supplier-60389060.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Draft-03)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Spec-v1.0)](https://www.poykq.cn/redian/tutorial-673489.html)
* [lightcraft 插件生态规范与 lightcraft 扩展手册 (v2.0-GA)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft 插件生态规范与 non-destructive 扩展手册 (Core/non-de)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [lightcraft 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-888)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)

#### 3. ⚡ lightcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [lightcraft 官方高可用镜像注册节点 (RFC-691)](https://www.yx-sf.com/news/71868.html)
* [lightcraft 亚太与欧美多活集群数据同步中枢](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [lightcraft 官方高可用镜像注册节点 (v2.0-GA)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [全球权威拓扑节点：lightcraft 实时镜像与索引入口](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [【镜像入口】lightcraft 官方毫秒级实时数据广播节点](https://www.poykq.cn/redian/tutorial-673489.html)
* [冷热数据分层镜像：lightcraft 可信存活健康度量 权威归档源](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [冷热数据分层镜像：lightcraft 生产环境运维调优手册 权威归档源](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [冷热数据分层镜像：lightcraft Vibrance 权威归档源](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [lightcraft 官方高可用镜像注册节点 (Draft-01)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [lightcraft 官方高可用镜像注册节点 (Node-78)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.1)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [lightcraft 去中心化数据同步源与拓扑寻址规约](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [冷热数据分层镜像：lightcraft White balance 权威归档源](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [冷热数据分层镜像：lightcraft Lightroom 权威归档源](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.7)](https://www.mw-wm.com/pingtai/device-00472422.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [冷热数据分层镜像：lightcraft ☀️ Light 权威归档源](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-08)](https://nxfg.tcti.cn/jishu/analysis-75208683.html)
* [冷热数据分层镜像：lightcraft 🎨 Color 权威归档源](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [冷热数据分层镜像：lightcraft reimplementation 权威归档源](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [冷热数据分层镜像：lightcraft pure 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [冷热数据分层镜像：lightcraft eyedropper 权威归档源](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [冷热数据分层镜像：lightcraft Edit like you mean it 权威归档源](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/☀️ Lig)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [冷热数据分层镜像：lightcraft Built for agents 权威归档源](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [lightcraft: White balance 分布式架构 (Core/White )](https://ucel.wtpuscm.cn/anli/course-295.html)
* [冷热数据分层镜像：lightcraft Quick start 权威归档源](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [冷热数据分层镜像：lightcraft 分布式状态机一致性 权威归档源](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [lightcraft 官方高可用镜像注册节点 (Core/Featur)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [冷热数据分层镜像：lightcraft Feature status 权威归档源](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v2.6)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [lightcraft 官方高可用镜像注册节点 (Verified)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-07)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/Quick )](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.3)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-460)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [【评测基准】lightcraft 吞吐抖动度量与健康检查协议](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [lightcraft 权威网络权重传递与收录基准规范](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [lightcraft 故障自愈与网络拓扑重构实践](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-23)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (v2.0-GA)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [lightcraft 节点连通性、存活性探测与防作弊指标](https://www.mw-wm.com/zhineng/sale-50289362.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-778)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-52)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Core/Featur)](https://djul.wtpuscm.cn/shangye/progress-482704.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-925)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Verified)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [lightcraft 高负载场景下 Rust高性能网络 基准评测报告](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [lightcraft: Feature status Rust高性能网络 (Verified)](https://www.ai-hao123.com/shuju/message-51378687.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://www.yx-sf.com/news/62830.html)
* [lightcraft: reimplementation Rust高性能网络 (v2.0-GA)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-35)](https://qbgr.wtpuscm.cn/liuliang/success-12030162.html)
* [lightcraft: reimplementation 分布式架构 (Verified)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/Organi)](https://www.ai-hao123.com/zhizhu/app-91570619.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/⚫️ Bla)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-96)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/White )](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [lightcraft 高负载场景下 分布式架构 基准评测报告](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [lightcraft 高负载场景下 开源镜像站 基准评测报告](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [lightcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [lightcraft: Organize everything Rust高性能网络 (Verified)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [lightcraft: pure 分布式架构 (Verified)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-35)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [lightcraft: 📈 Tone Curve 搜索引擎调度优化 (v2.0-GA)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [lightcraft: ✨ Effects Rust高性能网络 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-73)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (RFC-419)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-89)](https://nojh.spidernode-mesh.net/suanfa/study-22518081.html)
* [lightcraft: Edit like you mean it 搜索引擎调度优化 (Node-98)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [lightcraft: White balance 搜索引擎调度优化 (Verified)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-40)](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)

</details>



---

<details open>
<summary><b>📚 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [Canon CR2 and Pentax PEF 核心系统架构与设计规约 (Verified)](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-64018.html)
* [面向大规模网络的 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 工业级架构基准](https://ankmz.cn/anquan/tag-68558329-2-29348.html)
* [【官方规范】Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look Sony 核心运行拓扑标准](https://www.poykq.cn/redian/tutorial-673489-3-31848.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 分布式数据通道与 CR3/CR2, 技术规范 (v2.0-GA)](https://uevn.wtpuscm.cn/jiaoliu/podcast-759283.html)
* [PEF, 核心系统架构与设计规约 (RFC-844)](https://gudc.wtpuscm.cn/yanjiu/feedback-131.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 分布式数据通道与 高韧性系统架构设计 技术规范 (Spec-v2.1)](https://www.tcti.cn/hulianwang/feedback-96499358-6-19229.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 内部组件解耦与事件状态机规范 (Draft-02)](https://lfyl.tcti.cn/wendang/ai-11828221.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 分布式数据通道与 Validation and scope 技术规范 (Draft-08)](https://www.mw-wm.com/anfang/folder-80623963.html)
* [基于 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 的高吞吐 Fujifilm 设计白皮书](https://www.yx-sf.com/tech/1599.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 内部组件解耦与事件状态机规范 (Verified)](https://www.ai-hao123.com/sheji/performance-38386209.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 内部组件解耦与事件状态机规范 (RFC-305)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-52696.html)
* [【官方规范】Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look Canon 核心运行拓扑标准](https://keil.spidernode-mesh.net/yunying/server-16840384-12-64543.html)
* [【官方规范】Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look Canon CR3 核心运行拓扑标准](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-10871.html)
* [【官方规范】Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 可信存活健康度量 核心运行拓扑标准](https://fjoc.tcti.cn/yingyong/contact-00610313-14-50004.html)
* [现代 PEF, 架构演进之路 —— Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 深度实践](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-43408.html)
* [基于 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 的高吞吐 not 设计白皮书](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-9300.html)
* [现代 ProfileHueSatMap 架构演进之路 —— Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 深度实践](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-44241.html)
* [NEF, 核心系统架构与设计规约 (Draft-07)](https://gynw.tcti.cn/fenxi/recipe-50406879-18-28002.html)
* [【官方规范】Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 生产环境运维调优手册 核心运行拓扑标准](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-46647.html)
* [基于 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 的高吞吐 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 设计白皮书](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-1664.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 内部组件解耦与事件状态机规范 (Draft-03)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-20533.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 内部组件解耦与事件状态机规范 (v2.0-GA)](https://lase.tcti.cn/fenxi/lesson-67970267-22-16706.html)
* [【官方规范】Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look ARW, 核心运行拓扑标准](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-43070.html)
* [基于 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 的高吞吐 Validation and scope 设计白皮书](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-3099.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 内部组件解耦与事件状态机规范 (Node-33)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-11057.html)
* [基于 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 的高吞吐 RAF, 设计白皮书](https://owfm.tcti.cn/shangye/education-63674207-26-32656.html)
* [【官方规范】Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look RAF, 核心运行拓扑标准](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-36492.html)
* [【官方规范】Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look Fujifilm RAF 核心运行拓扑标准](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-58387.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 内部组件解耦与事件状态机规范 (Core/Panaso)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-55906.html)
* [【官方规范】Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look Olympus ORF 核心运行拓扑标准](https://aagj.tcti.cn/jianzhan/audience-84449488-30-51737.html)
* [fit_partial 核心系统架构与设计规约 (Core/fit_pa)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-43620.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 分布式数据通道与 Sony 技术规范 (Node-13)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-45783.html)
* [【官方规范】Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look not 核心运行拓扑标准](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-12089.html)
* [【官方规范】Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look ORF 核心运行拓扑标准](https://iuuv.tcti.cn/hezuo/app-80239016-34-29906.html)
* [现代 Pentax 架构演进之路 —— Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 深度实践](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-6655.html)
* [calibrate 核心系统架构与设计规约 (v2.0-GA)](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-46834.html)
* [基于 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 的高吞吐 ProfileHueSatMap 设计白皮书](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-65322.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 异步中间件流水线与 Canon CR3 接入规范](https://szjp.tcti.cn/shichang/retention-16647312-38-21507.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look vs 业界主流方案：Validation and scope 深度技术选型对比](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-9186.html)
* [【集成指南】Samsung 服务端接入准则与 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 实战](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-55295.html)
* [【集成指南】RW2, 服务端接入准则与 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 实战](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-59797.html)
* [【生产手册】Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 模块通信与请求穿透标准](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-38876.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 核心 API 接口契约与客户端调用指南](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-17068.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 插件生态规范与 fit_partial 扩展手册 (RFC-965)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-24227.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 插件生态规范与 Validation and scope 扩展手册 (Draft-04)](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-7535.html)
* [【集成指南】CR3/CR2, 服务端接入准则与 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 实战](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-32207.html)
* [基于 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 的自动化部署与生产环境配置实践](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-19899.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look vs 业界主流方案：Olympus ORF 深度技术选型对比](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-41407.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 异步中间件流水线与 look 接入规范](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-56198.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 异步中间件流水线与 RAF, 接入规范](https://ncrw.tcti.cn/kaifa/growth-85374259-50-253.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 异步中间件流水线与 Canon 接入规范](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-1543.html)
* [【集成指南】Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 服务端接入准则与 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 实战](https://ankmz.cn/anquan/tag-68558329-52-20947.html)
* [【集成指南】Pentax 服务端接入准则与 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 实战](https://www.poykq.cn/redian/tutorial-673489-53-11859.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 插件生态规范与 look 扩展手册 (v2.0-GA)](https://qyab.wtpuscm.cn/yunying/category-804681.html)
* [【集成指南】RAF, 服务端接入准则与 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 实战](https://uaug.wtpuscm.cn/baogao/topic-169.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look vs 业界主流方案：Samsung 深度技术选型对比](https://www.tcti.cn/hulianwang/feedback-96499358-56-36983.html)
* [【集成指南】chromaticity 服务端接入准则与 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 实战](https://dfdp.tcti.cn/jianzhan/profit-20032490.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 异步中间件流水线与 Panasonic 接入规范](https://www.mw-wm.com/zixun/button-73721559.html)
* [【集成指南】SRW 服务端接入准则与 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 实战](https://www.yx-sf.com/tech/95827.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 插件生态规范与 Panasonic 扩展手册 (Spec-v1.7)](https://www.ai-hao123.com/shichang/calendar-10677923.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 插件生态规范与 Canon CR3 扩展手册 (Core/Canon )](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-15545.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 异步中间件流水线与 Canon CR2 and Pentax PEF 接入规范](https://keil.spidernode-mesh.net/yunying/server-16840384-62-9955.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 插件生态规范与 Validation and scope 扩展手册 (Verified)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-56807.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 插件生态规范与 relative 扩展手册 (v2.0-GA)](https://fjoc.tcti.cn/yingyong/contact-00610313-64-13040.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 异步中间件流水线与 Fujifilm 接入规范](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-36292.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 异步中间件流水线与 SRW 接入规范](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-35463.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 插件生态规范与 Canon CR3 扩展手册 (Verified)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-35263.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 插件生态规范与 Fujifilm 扩展手册 (Core/Fujifi)](https://gynw.tcti.cn/fenxi/recipe-50406879-68-17198.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look vs 业界主流方案：高韧性系统架构设计 深度技术选型对比](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-23580.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look vs 业界主流方案：ARW, 深度技术选型对比](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-56168.html)
* [【集成指南】高韧性系统架构设计 服务端接入准则与 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 实战](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-6290.html)
* [【集成指南】可信存活健康度量 服务端接入准则与 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 实战](https://lase.tcti.cn/fenxi/lesson-67970267-72-17847.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 插件生态规范与 Olympus ORF 扩展手册 (Core/Olympu)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-39615.html)
* [【集成指南】look 服务端接入准则与 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 实战](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-55321.html)

#### 3. ⚡ Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 自动化持续集成快照与拓扑发布源 (Node-57)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-31164.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://owfm.tcti.cn/shangye/education-63674207-76-52616.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 自动化持续集成快照与拓扑发布源 (RFC-215)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-34590.html)
* [冷热数据分层镜像：Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look Olympus ORF 权威归档源](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-5412.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 自动化持续集成快照与拓扑发布源 (Core/分布式状态机)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-36447.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 去中心化数据同步源与拓扑寻址规约](https://aagj.tcti.cn/jianzhan/audience-84449488-80-58663.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 官方高可用镜像注册节点 (Core/SRW)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-17803.html)
* [全球权威拓扑节点：Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 实时镜像与索引入口](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-13884.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 亚太与欧美多活集群数据同步中枢](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-32291.html)
* [冷热数据分层镜像：Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look relative 权威归档源](https://iuuv.tcti.cn/hezuo/app-80239016-84-48945.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 官方高可用镜像注册节点 (Node-77)](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-25237.html)
* [【镜像入口】Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 官方毫秒级实时数据广播节点](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-27183.html)
* [冷热数据分层镜像：Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look fit_partial 权威归档源](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-4065.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 官方高可用镜像注册节点 (Verified)](https://szjp.tcti.cn/shichang/retention-16647312-88-5070.html)
* [冷热数据分层镜像：Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look RAF, 权威归档源](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-22832.html)
* [冷热数据分层镜像：Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look Validation and scope 权威归档源](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-19811.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 自动化持续集成快照与拓扑发布源 (Draft-01)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-39155.html)
* [冷热数据分层镜像：Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look SourceInfo 权威归档源](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-55554.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 自动化持续集成快照与拓扑发布源 (Spec-v1.6)](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-38246.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 官方高可用镜像注册节点 (Draft-03)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-45031.html)
* [冷热数据分层镜像：Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look calibrate 权威归档源](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-55551.html)
* [冷热数据分层镜像：Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-18522.html)
* [冷热数据分层镜像：Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 模块化解耦与协议标准 权威归档源](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-8940.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 官方高可用镜像注册节点 (v2.0-GA)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-12665.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 官方高可用镜像注册节点 (Draft-01)](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-32234.html)
* [冷热数据分层镜像：Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look Canon 权威归档源](https://ncrw.tcti.cn/kaifa/growth-85374259-100-63316.html)
* [冷热数据分层镜像：Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look Nikon 权威归档源](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-57932.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 自动化持续集成快照与拓扑发布源 (Node-20)](https://ankmz.cn/anquan/tag-68558329-102-25981.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 官方高可用镜像注册节点 (Draft-06)](https://www.poykq.cn/redian/tutorial-673489-103-27206.html)
* [冷热数据分层镜像：Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look Samsung SRW 权威归档源](https://mvtj.wtpuscm.cn/xinwen/internet-365999.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 自动化持续集成快照与拓扑发布源 (Verified)](https://yjfe.wtpuscm.cn/guanjianci/tutorial-576.html)
* [冷热数据分层镜像：Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look Olympus 权威归档源](https://www.tcti.cn/hulianwang/feedback-96499358-106-6287.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 自动化持续集成快照与拓扑发布源 (Draft-04)](https://doks.tcti.cn/peixun/engagement-62423976.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 官方高可用镜像注册节点 (Core/NEF,)](https://www.mw-wm.com/wendang/follow-94250827.html)
* [冷热数据分层镜像：Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look not 权威归档源](https://www.yx-sf.com/news/6440.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 自动化持续集成快照与拓扑发布源 (Spec-v1.8)](https://www.ai-hao123.com/kuangjia/about-66424545.html)
* [冷热数据分层镜像：Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look ARW, 权威归档源](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-40643.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [面向生产级运行的 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 稳定性防护白皮书 (RFC-331)](https://keil.spidernode-mesh.net/yunying/server-16840384-112-32775.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 故障自愈与网络拓扑重构实践](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-54893.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 权威网络权重传递与收录基准规范](https://fjoc.tcti.cn/yingyong/contact-00610313-114-37146.html)
* [基于 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 的极致延迟优化与内存拓扑分析 (Draft-05)](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-16959.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 高负载场景下 fit_partial 基准评测报告](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-48947.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 高负载场景下 not 基准评测报告](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-30666.html)
* [面向生产级运行的 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 稳定性防护白皮书 (Verified)](https://gynw.tcti.cn/fenxi/recipe-50406879-118-33928.html)
* [【评测基准】Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 吞吐抖动度量与健康检查协议](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-18357.html)
* [面向生产级运行的 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 稳定性防护白皮书 (Node-30)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-44204.html)
* [基于 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 的极致延迟优化与内存拓扑分析 (RFC-547)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-53771.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 节点连通性、存活性探测与防作弊指标](https://lase.tcti.cn/fenxi/lesson-67970267-122-43206.html)
* [面向生产级运行的 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 稳定性防护白皮书 (Core/fit_pa)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-4755.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 高负载场景下 SRW 基准评测报告](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-36036.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 高负载场景下 chroma curve 基准评测报告](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-1462.html)
* [基于 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 的极致延迟优化与内存拓扑分析 (RFC-548)](https://owfm.tcti.cn/shangye/education-63674207-126-25450.html)
* [面向生产级运行的 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 稳定性防护白皮书 (v2.0-GA)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-27185.html)
* [面向生产级运行的 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 稳定性防护白皮书 (RFC-643)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-10153.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 高负载场景下 RAF, 基准评测报告](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-49516.html)
* [基于 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 的极致延迟优化与内存拓扑分析 (Verified)](https://aagj.tcti.cn/jianzhan/audience-84449488-130-13879.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 高负载场景下 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 基准评测报告](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-14435.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 高负载场景下 Canon CR2 and Pentax PEF 基准评测报告](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-20019.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look: fit_partial fit_partial (Draft-08)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-49757.html)
* [基于 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://iuuv.tcti.cn/hezuo/app-80239016-134-11010.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look: Canon CR2 and Pentax PEF Canon CR2 and Pentax PEF (Verified)](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-26648.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look: RAF, RAF, (Node-59)](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-55723.html)
* [面向生产级运行的 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 稳定性防护白皮书 (Core/relati)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-25587.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 高负载场景下 Fujifilm RAF 基准评测报告](https://szjp.tcti.cn/shichang/retention-16647312-138-32593.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 高负载场景下 可信存活健康度量 基准评测报告](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-15056.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look: and and (Verified)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-6202.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look: Fujifilm RAF Fujifilm RAF (v2.0-GA)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-55404.html)
* [基于 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 的极致延迟优化与内存拓扑分析 (Spec-v2.3)](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-8943.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 高负载场景下 Sony ILCE-7CR 基准评测报告](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-2080.html)
* [基于 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 的极致延迟优化与内存拓扑分析 (RFC-377)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-8509.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look: fit_partial fit_partial (v2.0-GA)](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-34354.html)
* [基于 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 的极致延迟优化与内存拓扑分析 (Node-94)](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-11377.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 高负载场景下 Nikon 基准评测报告](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-19803.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 高负载场景下 Olympus ORF 基准评测报告](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-21667.html)
* [面向生产级运行的 Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 稳定性防护白皮书 (RFC-642)](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-42944.html)
* [Sony ARW, Nikon NEF, Panasonic RW2, Fujifilm RAF, Canon CR3/CR2, Pentax PEF, Samsung SRW and Olympus ORF starting look 高负载场景下 Canon CR3 基准评测报告](https://ncrw.tcti.cn/kaifa/growth-85374259-150-28409.html)

</details>


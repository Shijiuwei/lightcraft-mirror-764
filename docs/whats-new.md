# What's new in LightCraft

## 2026-10-10

### Activity stack
- Long-running tasks show in one place, top left under the top bar, as in Lightroom Classic (issue #345): imports and
  folder scans, exports (and the contact sheet PDF), Synchronize Folder, Build / Smart Previews, Lightroom catalog import, Photo Merge, model downloads, the face scan,
  AI Denoise and Find Missing Photos. Each row has the task's name, a progress bar, the count and what it is working
  on; ✕ stops the tasks that can stop. Three rows show, then "+N more". The separate export panel and the progress
  windows and toasts are gone.
- Quitting while a task that can be stopped is running asks first ("Quit Anyway" stops it).
- Agents list and stop tasks with `activity.list` and `activity.cancel`; `ui.inspect` reports them as `activity`.

## 2026-10-09

### Smart album rules
- The rule editor's field menu is grouped: Rating, Pick Flag, Color Label and Any Searchable Text at the top, then
  submenus for Source, File, Date, Keywords & People, Description, Camera Info, Location, Size, Develop and Assisted
  Culling.
- New fields: Keyword Count ("2 or more", "exactly 2"…), People and People Count (named faces), Shutter Speed (written
  as 1/250), File Extension, Video Duration, Copy Name, In a Stack, Alt Text, City, State / Province, Country, Long
  Edge, Short Edge, Aspect Ratio (landscape / portrait / square), Cropped and Treatment (in color / black & white).
  Size rules measure the photo as shown: Megapixels now counts the cropped photo, like Long Edge, Short Edge and
  Aspect Ratio (a 24 MP photo cropped to a square is 16 MP).
- Choice values read as words in every language: "Public Domain", "Picked", "Black & White" rather than ids like
  `publicDomain` (rules and agents keep using the ids; `album.ruleFields` adds `choiceLabels`).
- Rule values are checked: a rating of 9, a date like 2026-13 or "banana", a "between" whose dates are the wrong
  way round, "in the last 0 days", a colour label that doesn't exist, "contains" with nothing to look for, an
  album that is gone, or an empty group is reported with its rule ("rule 2.1: no rating 0–5 is 9")
  instead of quietly matching nothing or everything. New date rules start at the current year.
- A smart album can include or exclude another smart album: "Keywords contain travel" and "Album isn't Excluded Photos"
  leaves out whatever Excluded Photos matches, as its rules change. The album picker shows your albums as in the sidebar
  (folders open on click) or finds them as you type part of a name or folder (↑ / ↓ and Enter to pick); albums that
  can't be picked, such as one that would make an album include itself, are greyed with the reason. Esc with a dropdown or calendar open in a dialog
  closes just that.
- Date rules have a calendar button: pick a day, a month or a whole year (Year / Month / Day), in your language; the
  two dates of a "between" can't be picked out of order; Esc closes just the calendar. It picks dates, not times (a
  time is typed, and picking a day replaces one). Typing still works, and "2026-10-01 10:00" (a space for the
  T) now matches.
- Rule problems are shown in your language and name the field the way the menu does ("Title: needs something to look
  for").
- A smart album whose rules no longer check (an album they test was deleted) is marked ⚠ in the sidebar; its tooltip
  says what to fix, and `albums.list` reports `problems`.
- Editing a smart album's name and rules is one step: both or neither, and one undo takes both back
  (`album.setRules` takes `name`).
- The rule editor marks each rule that needs fixing right under it and keeps OK disabled until they are fixed. An
  Album rule is "is" or "isn't" an album, picked with the album picker.
- Yes/no fields (Has Edits, Cropped, Has GPS…) read Yes / No instead of true / false. A rule sent as `"false"` now
  means no (it used to count as yes), and a value that is neither yes nor no is refused.
- Any Searchable Text also finds the state / province, alt text, people and a virtual copy's name.

### Keyword lists
- File ▸ Import Keywords… and Export Keywords… read and write keyword list files as Lightroom Classic does (a keyword
  a line, a tab per level, synonyms in braces, keywords left out of export in brackets). Capture One, darktable, Adobe
  Bridge and Photo Supreme (its Formatted Vocabulary File) use the same files, so keyword lists move between them and
  LightCraft. Importing adds the keywords you don't have, in one undo step; a file it can't read says which line, or
  to save it as UTF-8. Exporting warns about keywords Capture One won't import, and names any keyword the format can't
  hold (such as a name in brackets), which it leaves out so the file always reads back.

### Keyword sets
- Keyword Set ▸ Edit Set… edits a set's nine keywords in place, one field per ⌥ key, and renames it (or saves the edits as a new
  set). An empty field leaves its key empty instead of moving the keywords after it; editing Recent Keywords and naming
  them saves a new set.

### Keywording box
- The chips at the top of the Keywords panel are the keywords of every selected photo, not only the active one's; a
  keyword only some of them have is marked with an asterisk (hover: how many). Only the × removes a keyword, from
  every selected photo; a chip's menu acts on the selection (add to all, remove, show photos, edit) and no longer
  deletes a keyword from the whole library.
- Switch to & Containing to see the keywords with the keywords containing them, or to Will Export to see what
  exported files will carry for the selection.
- The keyword set's buttons and the suggestions follow the selection too; the typing boxes get the Cut / Copy / Paste
  menu, and Return in the painter's box starts painting.

### Keyword List
- The Keywords panel ends with a Keyword List, as in Lightroom Classic: every keyword with its photo count, those no
  photo has yet too. Filter it, tick a keyword to give it to the selected photos (a dash: only some have it), or click
  the arrow to see its photos.
- Create keywords with + (inside the one you picked, with synonyms, and given to the selection if you like), edit one
  by double-clicking it, delete one with − after a confirmation. Right-click for more: create inside, Put New Keywords
  Inside This Keyword, Purge Unused Keywords.
- Drag a keyword onto another to nest it, or onto the list's title to bring it back to the top level; photos dragged
  from the grid onto a keyword get it.
- Right-click ▸ Put New Keywords Inside This Keyword: new keywords, made with + or typed in the box above, go inside
  it. The list is there with no photo selected too, to set up keywords first.
- Keyword tag options decide what exported files carry: Include on Export, Export Containing Keywords, Export
  Synonyms. Exports now write keyword names to `dc:subject` and their paths to `lr:hierarchicalSubject`, and keywords
  imported from files (or read from sidecars) keep the hierarchy `lr:hierarchicalSubject` gives them.

### Text fields
- Right-click Search Photos, a slider's typed value, the Info panel's fields, the name field of New Album, Rename
  Album, New Smart Album, Rename Keyword, Merge Keywords and similar dialogs, or the name field under unnamed faces for
  Cut, Copy, Paste and Select All. Esc in them gives back the text from before you started typing (in the Info panel it
  no longer saves it). Return in those dialogs now confirms them, and their name opens selected. More fields will
  follow.
- On Windows and Linux, Ctrl+C outside a text field copies edit settings and Ctrl+V pastes them (⇧Ctrl+V: Paste
  Selected Settings). Ctrl+V only reaches LightCraft while the system clipboard holds text; otherwise use Edit ▸ Paste Edit
  Settings.

### Synchronize Folder
- Right-click a folder (or a disk other than the startup disk) in the sidebar's Folders section ▸ Synchronize
  Folder…. The folder and the folders inside it are scanned in the background for photos added on disk, photos whose
  file is gone, photos whose file was renamed or moved within the folder, and XMP sidecars changed by other apps; then
  choose what to do: import the new photos and relink the moved ones (the defaults), move the missing ones to Recently
  Deleted, read the metadata updates. A folder on a disk that isn't connected is reported as such. Synchronizing
  runs in the background with a row in the activity stack (✕ stops it); everything it does is one undo step, and no file on disk
  is touched. Agents use `folder.scanChanges` and `folder.synchronize`.

### Folder colour labels
- Right-click a folder (or a disk other than the startup disk) in the sidebar's Folders section ▸ Set Color Label ▸ a colour or None. The row
  shows the label's dot before its photo count. Agents use `folder.label` (`path`, `label`); `library.folders`
  reports each row's `label`.
- A label follows its folder when it is renamed or moved in LightCraft, together with the labels of the folders
  inside it; undo and redo carry them back and forth.
- The catalog format is now version 4 (`Catalog.folder_records`, `Op::SetFolderRecord`). Once this version has
  opened a library, older versions refuse it, as with every format change.

### Monitor profiles
- Settings ▸ Display ▸ Choose Profile… shows photos through your monitor's ICC profile (matrix/TRC or LUT-based), so
  colours are right on wide-gamut and calibrated displays instead of oversaturated. The Detail view renders straight
  into the display's own gamut; thumbnails and other previews are converted to it. Histograms, the preview caches
  and exports don't change. Agents use `app.displayProfile`. See [display profiles](display-profiles.md).

### Masking
- Drag anywhere inside the selected radial gradient's ellipse to move it, including rotated
  ellipses and radial components in compound masks. Edge and rotation handles keep their existing
  functions, and one Undo restores the entire drag.

### Crop rotation
- Outside the crop box, where dragging rotates the photo, the pointer is a curved double arrow, and the angle is shown
  next to it while you rotate (issue #534). For an exact angle, type it in the new Angle field under Straighten and press Return
  (Esc keeps the old angle; a decimal comma works too), or click the Straighten value; hovering any slider's value now says it can be typed.

## 2026-10-08

### Albums tree
- Album folders fold with the same disclosure triangle as Local, By Date, Folders and Keywords.
- Right-click a folder ▸ New ▸ Create Album… / Create Smart Album… / Create Smart Album from Filter… / Create Folder…
  makes it inside that folder (`dialog.newAlbum`, `dialog.newFolder`, `dialog.smartAlbum`, `dialog.newSmartAlbum` take `parent`).
- Drag an album or folder onto a folder to move it there (a closed folder opens after 0.6 s under the pointer; a
  "Albums" header takes it back to the top level; Esc cancels; holding a dragged album or photos near the top or bottom edge of the sidebar scrolls it). Drop on the top or bottom half of another album,
  or the edge of another folder, to place it before or after it: that folder is then ordered by hand. Right-click ▸
  Sort Contents A–Z (or ＋ ▸ Sort Albums A–Z at the top level) goes back to by name. To put a folder after an open
  folder, fold that folder first (the bottom of an open folder means "inside"). Agents use `album.reorder`
  (`id`, `parent?`, `before?`) and `album.sort`.
- The catalog format is now version 3 (`Album.order`, `Op::SetAlbumOrder`). Once this version has opened a library,
  older versions refuse it, as with every format change.

### AI RAW denoise
- Detail has a per-photo AI Denoise switch and Amount. Bayer RAW photos use a disposable cache, with matching previews and exports and pure-Rust CPU/GPU inference.
- Models are installed separately after accepting their terms; no weights are bundled. See [setup and limitations](denoise.md).

### Library keyboard culling
- Colour labels tint thumbnail surrounds in Square Grid and the Detail filmstrip, and a translucent footer along the bottom of the photo in Photo Grid. Label confirmations use a matching pale colour.
- Setting or clearing a colour label shows a brief bottom confirmation, like rating a photo; custom label names appear in the message too.
- On macOS, ratings `0–5`, labels `6–9` and pick/unflag `P/U` now reach the app even when shown in the native menu (issue #283; adapted from PR #261).
- `Shift+6–9` labels and advances. `Shift+P` picks and advances in Photo Grid and Square Grid; it opens Presets in other views. With Auto Advance on, Shift still moves only once.
- Help → Keyboard Shortcuts includes the number-key bindings. See [library shortcuts](library-shortcuts.md).

### Keyboard shortcuts
- Shortcuts are editable: Help ▸ Keyboard Shortcuts (⌘/) lists every command with a search box; click a shortcut and
  press the new keys (Esc cancels), × removes it, ↺ restores the original, Reset All undoes every change. A key that
  belonged to another command moves to the new one. Menus show the new keys; agents use `app.setShortcut`.

### Formats
- HEIC / HEIF photos (iPhone and Mac) open now: the optional `lightcraft-heif` crate (heic-rs, pure Rust) behind
  codecs' `heif` feature — 8- and 10-bit, alpha, grid tiles, the container's rotation/mirror/crop, ICC, EXIF and XMP.
  Off by default (HEVC patents are the distributor's call, same as PhotoCraft); official builds pass `--features heif`.
- HEIC colours now match libheif to within one code value (they were up to ~10 off): the HEVC stream's own
  full-range/matrix signalling is honoured (iPhone photos are full range), chroma is upsampled like libheif, grids
  take their tiles' ICC profile, and `imir` mirrors the way libheif writes it. Importing a HEIC reads its size from
  the container instead of decoding it; small previews use the file's embedded thumbnail. The release packages
  (macOS, Windows, Linux, FreeBSD, Nix) now really are built with HEIC support; a build without it reports
  "HEIC/HEIF support isn't included in this build" for each `.heic`/`.heif` at import.

### Editing
- The Tint slider works the right way round (issues #188, #321): left adds green, right adds magenta, as its track
  shows and as in Lightroom, and Tint values in Lightroom XMP sidecars now render as they do there. A custom Tint
  saved in an earlier version now shifts the other way; set it again (or re-run Auto / the white-balance picker).
- White balance on DNGs (and other raws with a colour matrix) re-develops the photo for the new white through the
  camera's own colour matrices, as Lightroom does, instead of shifting the colours of the as-shot rendering: a grey
  lit by the chosen white comes out grey and saturated colours move as the camera records them. The eyedropper and
  Auto use the same model. Custom white balances on these photos render slightly differently than before.
- Crop (issue #295): a Lock toggle keeps the aspect ratio on every handle, Custom takes your own ratio (Apply), and
  dragging a handle into the image edge stops there instead of pushing the crop out of shape.

### Library and views
- Trackpads: pinch to zoom around the pointer and scroll with two fingers to pan the photo; panning keeps the photo
  inside the view. A plain mouse wheel over a zoomed photo pans it too.
- A Folders section in the sidebar lists the folders your photos were imported from; choose one to see its photos.
- Select All and multi-selection show every selected photo in the grid and filmstrip, not only the active one
  (issues #187, #298). Importing files that are in Recently Deleted asks whether to leave them there, restore them
  (with their edits) or import them as new; the trash view's Photo menu has Empty Recently Deleted.
- The Import Photos review opens bigger and can be resized; its photo grid fills it (issue #337). Shift-click checks
  or unchecks a range of photos (issue #338).
- When a folder holds several file types, the Import Photos review has a toggle per type (`ARW · 120`, `JPG · 120`):
  import only the raws and leave the JPEGs beside them (issue #344).

### Languages
- The interface is available in Spanish (issue #371), German and Russian (Edit ▸ Language), alongside English,
  Chinese (Simplified and Traditional), Japanese and Brazilian Portuguese.

### Editing
- Type an exact value into any slider (issue #322): click the number next to its name, type (`1.5`, `-20`, `5600`)
  and press Return; Esc keeps the old value.

### Editing
- The eye on the Light, Color and Detail section headers now hides their adjustments, as it already did for Effects,
  Optics, Geometry and Calibration (issue #316).

### Library
- Choosing a date under By Date or a keyword under Keywords shows those photos from All Photos, as their counts
  promise, instead of filtering whatever album or folder was open, which often showed nothing (issue #341).
)
)

## 2026-10-07

### RAW decoding
- Olympus ORF raws get the starting look fitted to the camera's own JPEG too, instead of opening flat and grey (the
  E-1, E-400 and XZ-2 corpus files: all accepted), with white balance relative to the as-shot look as for the other
  formats. On an E-1 photo the starting render moved from ΔE00 11.3 to 8.3 against Lightroom's. Photos already
  imported pick it up when re-rendered.
- Samsung SRW files without compression now open as raws: NX5, NX10, NX11, NX20, NX200, NX210, NX1000, NX1100, EX1
  and WB2000. The compressed ones (NX1, NX30, NX300, NX500, NX2000, NX3000, NX3300, NX mini) still open from their
  camera JPEG. Photos already imported pick the change up on Reload.
- Raws from 52 camera models (22 Canon, 11 Nikon, 11 Sony, 3 Fujifilm and 5 others, plus their other names) whose
  camera JPEG can't be fitted no longer open with the neutral fallback: they start from colour matrices fitted to the
  camera's measured spectral sensitivities, from the Academy Software Foundation's rawtoaces-data. A camera profile
  and the fit to the photo's own JPEG still come first, so photos that had a fitted look keep it. See
  [camera preview colour](camera-preview-colour.md#spectral-camera-matrices-and-the-order-of-precedence).
- Canon CR2 raws no longer open too dark with blocked-up shadows: the black level was measured over border columns
  that light already reaches (left of the image area Canon records), 4 % of the range too high on the EOS 6D. It is
  now measured on the masked columns alone (8 corpus bodies, EOS 40D to 5DS R). On a 6D photo the starting render
  moved from ΔE00 7.6 to 5.2 against Lightroom's.
- Canon CR2 and Pentax PEF raws get the same starting look fitted to the camera's own JPEG as ARW, NEF, RW2, RAF
  and CR3, instead of opening flat and desaturated (issue #310). Photos already imported pick it up when re-rendered.
- JPEG XL compressed DNGs (DNG 1.7) now open: lossless tiles decode sample for sample (checked on synthetic files);
  lossy tiles decode too, keeping the raw values above 1.0 instead of clipping them (checked on one real file). A tile that cannot be decoded makes the file open from its embedded
  preview, with the reason, instead of showing a black tile. A JPEG XL preview stored in the DNG is used like an
  embedded JPEG.
- Apple ProRAW's gain table map (its local tone mapping, `ProfileGainTableMap`) is now read and kept when a photo is
  exported or converted to DNG. It is not applied by default: Lightroom Classic renders ProRAW without it. To see a
  ProRAW the way the iPhone renders it, turn on Profile ▸ Camera local tone mapping (shown for photos that carry the
  map); the option is per photo, so presets and Copy Settings carry it.
- iPhone ProRAW and other DNGs that carry their own segmentation mattes (DNG semantic masks) use them for the Select
  Sky, Subject and Background masks instead of our heuristics, so the sky is selected where the camera found it
  (checked on one CC0 iPhone 12 Pro ProRAW). Photos without mattes are unchanged.
- Panasonic and Leica raws (RW2, RWL) are now corrected for lens distortion the way the camera corrects its own JPEG
  (issue #256): the correction the camera records in the file is applied under Lens Corrections ▸ Enable Profile
  Corrections, on by default for newly imported photos, with the same framing as the camera's JPEG. At 12 mm the
  12–32 mm kit zoom was off by about 5 % of the image width at the corners before. Files shot with the correction off
  are unchanged; photos imported before this change get it when imported again.
- Canon CR3 raws now develop from their sensor data: lossless RAW and C-RAW, checked sample for sample on the EOS
  M50, R100 and R8. CR3 files the decoder can't read yet still open from their embedded JPEG, as before.
- Canon EOS R7 C-RAW files develop from their sensor data too, instead of opening from their embedded JPEG. For R7
  C-RAWs imported earlier, Photo ▸ Reload from Disk picks up the raw data.
- Canon CRW, Minolta MRW, Sigma X3F, Kodak KDC, Leaf MOS and Epson ERF files that LightCraft can't decode yet
  now import as "preview only" with their embedded JPEG instead of failing. A raw whose data is damaged but whose
  preview is intact does the same.
- Raw files whose raw data sits in a private block of a TIFF (Phase One / Leaf IIQ, Canon EOS-1D / 1Ds and Kodak DCS
  TIFFs) are no longer opened as a thumbnail-sized ordinary image. They are recognised as raws LightCraft can't decode
  yet and import as "preview only", with the reason.
- Sony ILCE-7M4 downsized lossless ARWs now decode subsampled YCbCr tiles into linear RGB,
  preserving RAW editing & full-resolution export instead of using embedded JPEG previews.
- Sony A7R II (and other) raws whose camera JPEG is lens-corrected no longer open grey and too dark (issue #232): the
  starting look is fitted to the camera JPEG away from edges when the misaligned edges spoil the fit on all pixels.

### Lightroom Classic catalogs
- File → Import Lightroom Catalog… opens `.lrcat` directly, with originals referenced in place.
  Ratings, flags, labels, keywords, collections/sets, virtual copies and supported edits migrate;
  existing LightCraft edits are preserved by default. Source settings/history are archived, unsupported
  fields are reported, and the original Lightroom database stays read-only. Rendering is approximate.

## 2026-10-05

### Reliability
- The Windows installer asks where to install LightCraft (Program Files by default; upgrades keep the folder you
  chose) and ends on a page saying it was installed, with a "Launch LightCraft" box. It used to finish without a word,
  so a successful install looked like nothing had happened (issues #18, #399). Silent installs (`/qn`, `/passive`)
  show no dialogs and take `INSTALLFOLDER=...`.
- If the desktop app can't open its window (for example when no graphics device can be used), it now says so in a
  message box that names the log file, instead of quitting without a trace (issue #260).
- On macOS, single-key shortcuts that appear in the menu bar now work: E, C, H, M, ⇧P, I, K, D, ratings 0–5,
  labels 6–9, P / U and the rest did nothing, because macOS only passes ⌘ / ⌃ combinations and function keys
  to the menu bar and the app ignored those keys, assuming the menu bar would handle them. Keys outside the menus
  (Space, G, X, ⌫) and ⌘ shortcuts were not affected.
- `--memory` sessions keep their promise to save nothing (issues #164, #169): UI changes made in one no longer
  land in `ui.json` (where they replaced the saved settings), and the GPU crash sentinel no longer creates the
  settings folder there. The same goes for the temporary session offered when the library can't be opened.
- The desktop app keeps a log file: `logs/lightcraft.log` in its settings folder (Linux `~/.config/lightcraft/logs/`),
  with the logs of the two previous runs beside it, so warnings and crashes of a run started from a desktop menu or the
  Dock can be attached to a bug report. `LIGHTCRAFT_LOG` works as before; `RUST_LOG` takes env_logger-style
  directives. See README → Quick start → Logs.
- Help → Open Log Folder shows that log file in the file manager (Finder, Explorer, or the folder on Linux), so it
  can be attached to a report without hunting for the settings folder (issue #260).
- `lightcraft-cli` logs warnings on stderr too (issue #168); `LIGHTCRAFT_LOG` or `RUST_LOG` picks another level.
- LightCraft no longer crashes at launch on Windows PCs whose Vulkan driver is broken (issue #136, e.g. some Intel UHD
  630 drivers): on Windows the window and GPU rendering use DirectX 12 only and never load the Vulkan driver unless
  asked to. `LIGHTCRAFT_GPU_BACKEND=dx12 | vulkan | metal | off` (or wgpu's `WGPU_BACKEND`, which GPU rendering
  ignored before) chooses the graphics backend; `off` renders on the CPU. The GPU now starts after the window
  is up, and only when Settings ▸ Performance ▸ Use the GPU for rendering is on; if LightCraft ever dies while
  starting the GPU, the next launch starts with GPU rendering off and says how to turn it back on.
- The Windows app no longer quits at launch with "Parent device is lost" when another program's `dxcompiler.dll` is
  on the DLL search path without its `dxil.dll` (issue #471): DirectX 12 shaders now always compile with the
  compiler built into Windows (FXC). `WGPU_DX12_COMPILER=dxc` uses a `dxcompiler.dll` instead.
- Exports and renders never write over a photo's original (issue #93): exporting into the photo's own folder with
  the same name and "Overwrite" (or Export with Previous repeating it), an exact output path from the control
  channel or MCP, a merge preview path or `lightcraft-cli render IMG.jpg -o IMG.jpg` is refused with a clear
  message, and the original is left byte for byte. Ordinary earlier exports are still overwritten when asked.
  Exported files are written to a temp file and then renamed into place, so a full disk or an unplugged drive
  never leaves a truncated file; the XMP sidecar of an "Original" export follows the "If file exists" choice too.
- Convert to DNG, Copy as DNG, Photo Merge and smart previews no longer write straight to the final file (issue
  #106): a DNG is checked against the raw data, written to a temp file, synced and read back before it gets its
  name (never replacing a file), and only then is the photo relinked or the raw copy removed — a failed write
  leaves no DNG and keeps the raw. Smart previews are written the same way; a damaged one (cut short by a crash or
  a full drive) no longer counts as built and Build Smart Previews replaces it.
- Import ▸ Copy verifies every copy, like Move (issue #96): each file is written as a new file, synced to disk and
  checked against the content read from the card. A copy that fails or differs is removed and reported as a failed
  import — so "import complete" means the copies are good before you format the card — and a name that is taken
  gets -1, -2… instead of being replaced.
- Faster exports and card imports, with the same protection for what can't be recreated (issue #134): exports,
  renders and screenshots are still written to a temp file and renamed into place, but no longer forced to disk
  one by one — they can always be exported again, and on a USB drive or a NAS that per-file sync dominated a large
  export. The catalog, XMP sidecars, settings, DNGs, merges, smart previews and the copies an import makes are still
  synced. Import ▸ Copy now checks each copy against the content hash taken while scanning the card instead of
  reading the card a third time (Move still compares byte for byte before it deletes a source); a file that changed
  on the card after Review Import is reported instead of being imported with stale details. Checking an export path
  against the library's originals no longer scans the whole library when nothing is at that path.
- Saving metadata to an XMP sidecar another application wrote no longer replaces it (issue #92): LightCraft merges its
  fields in and keeps the rest — e.g. that application's develop settings and edit history — byte for byte. A
  sidecar that isn't valid XMP is copied to `<name>.xmp.bak-<time>` first. With the default stem naming, a raw and a
  JPEG with the same name (`IMG_0001.CR3` + `IMG_0001.JPG`) no longer share one sidecar: the raw keeps `IMG_0001.xmp`,
  the JPEG uses `IMG_0001.JPG.xmp`.
- A save that fails part-way (a full disk, a network share that drops) no longer looks like a damaged catalog
  afterwards (issue #101): the partial write is cut off before LightCraft retries, so the next launch replays every
  change. Catalogs already holding such a fragment load in full. Quitting while the catalog log can't be written
  still saves your queued changes in the closing snapshot.
- The catalog has a format version (issue #102). Opening a library from an older LightCraft upgrades it; a library
  written by a newer LightCraft is refused with "this library was written by a newer version of LightCraft" and left
  untouched — older versions no longer read part of it as a damaged log. Once this version has opened a library,
  LightCraft 0.2.0 and older refuse it ("unsupported format … v2").
- The control port (`--control`) closes a connection as soon as it receives anything that isn't a JSON request
  (issue #94): an HTTP request from a web page can no longer carry a command in its body. Lines are capped at
  4 MiB and connections at 16.
- A sleeping NAS, a dropped network share or a USB drive spinning up no longer freezes the window (issue #104):
  whether originals are there is checked on a worker thread (grid thumbnails, the Info panel, the photo menu, Missing
  Photos); importing (also by drag and drop), Find Missing Photos, Build / Discard Smart Previews, the Rename preview,
  the Local folder tree and Auto Import read the disk on worker threads too. The import progress window has Cancel.
- Browser version (experimental), keeping a library safe (issue #107): File ▸ Back Up Library… downloads the catalog
  and every imported photo as one zip, and File ▸ Restore Library from Backup… brings it back (the current library
  is kept). A failed save (storage full) now shows the unsaved warning and is retried, a photo that can't be stored
  isn't added, a second tab shows a message instead of overwriting the first, `?reset` asks first, and the page says
  when the browser may evict the library. Hosting: the sample cache headers no longer mark the (unhashed) files
  immutable, and HOSTING.md describes the actual build.
- Canon CR2 photos from the EOS 7D, 50D, 60D, 550D, 600D, 1200D, 1300D, 5D Mark II and 1D Mark IV (and other
  models whose sensor starts on a green-blue row) no longer come out magenta (issue #85): the colour-filter
  layout is read from each file instead of assumed.
- Exports are never black because of the GPU (issue #78): a GPU render that runs out of device
  memory, exceeds the GPU's buffer limits, hits a driver error or reset, or comes back
  incomplete is redone on the CPU — the file is the same image either way. Work is sent to the GPU
  in short pieces so slow integrated GPUs aren't reset by their watchdog. `ui.inspect` → `perf`
  (`gpuReason`, `gpuFallback`), Help ▸ System Info and Settings ▸ Performance say why the GPU isn't
  used (e.g. a skipped software adapter such as llvmpipe) and why the last render fell back.
- The thumbnail cache only ever counts and deletes its own files (issue #98): a library opened on a folder that
  already has a `thumbs/` folder of other pictures no longer loses them when the cache is trimmed or cleared.
- Settings files are never quietly reset (issue #103): a damaged `prefs.json`, `presets.json` or `view.json` is kept
  as `<name>.corrupt-<time>` and you're told; one that can't be read (e.g. locked by another program) is left alone
  for the session instead of being overwritten with defaults. The app settings (`ui.json`, which remembers your
  library) are written atomically and saved as soon as you open another library, not only at quit. Quitting while
  changes couldn't be saved tries once more, then asks: Try Saving Again, Quit Anyway or Cancel.

## 2026-10-02

### Presets and profiles
- Import presets from other editors: XMP presets, classic `.lrtemplate` files, "DNG presets" from mobile apps and `.zip`
  bundles of any of these — whole folders at once, grouped by pack. Masks inside presets come along.
- Luminar looks: `.lmp` files and `.mplumpack` collections import as presets (grouped by collection); the sliders
  with a counterpart here come along, the rest is listed.
- 23 new built-in presets: Portrait, Landscape, Urban, Food, Seasons, Vintage and B&W toners.
- Importing XMP presets no longer lists bookkeeping fields (`Cluster`, `SortName`, `SupportsAmount2`, the as-shot
  white, empty Point Color slots…) as settings that couldn't be carried over.
- A preset whose lens-profile switch is off no longer turns off the lens corrections built into a DNG (iPhone ProRAW
  and other files with embedded distortion / vignetting corrections), matching what the preset does elsewhere.
- XMP presets apply their red / green / blue curves only when they also have the master curve and all three channel
  curves, as Lightroom does: a preset with just one channel curve (or no master curve) leaves the channels alone.
- Imported `.cube` LUT profiles appear in the Profile menu and the profile browser, grouped by their folder, and stay
  favourites across restarts (issue #328).

### Library
- Photos in Recently Deleted can be restored from the app: right-click ▸ Restore (or Delete Permanently), also in the
  Photo menu. The filmstrip has the photo context menu too. Adding a file again that is in Recently Deleted no
  longer just says "duplicate skipped": it opens the side panel on Recently Deleted with the photo selected and says
  how to restore it or delete it permanently and import it afresh. (For a fresh start on a photo, Reset Edits,
  Cmd+Shift+R, keeps the photo and clears its edits.)
- Canon CR3 files show their full-size embedded JPEG (e.g. 6960 × 4640 on an EOS R6 Mark III) instead of the
  1620 × 1080 preview, and import with their metadata: capture time, camera, lens, exposure, GPS and XMP. Their raw
  data is not decoded yet, so they stay preview-only. For CR3s imported earlier, Photo ▸ Reload from Disk (now in
  the Photo menu and the photo context menu) picks up the full-size preview and fills in the camera metadata they
  were missing, without touching anything already set.
- Rename Photos never overwrites another photo when only the letter case changes (issue #95): on case-sensitive
  volumes (Linux, case-sensitive APFS) `img_1.JPG` next to `IMG_1.JPG` is a different photo and the renamed one gets
  `img_1-1.JPG`; on case-insensitive volumes the case change still goes through.
- Rename Photos reports files it could not move back after a failure (issue #105), e.g. when a network share drops
  mid-batch: the error lists them (old → new) and the library points at their new names (an undoable partial rename),
  so none shows as missing. Renaming one of a raw + JPEG pair copies their shared `IMG_0001.xmp` instead of taking
  it away from the other (issue #92). Find Missing Photos also finds renamed files by their content, prefers a content match
  over a same-name same-size look-alike, and skips (and reports) photos it can't tell apart instead of guessing.
- Rename Folder… and Move Folder To… (Local) can be undone and redone (issue #97): the folder moves back on disk with
  its photos and XMP sidecars, and the photos point at it again. If something now occupies the old place, the undo
  is refused and nothing is overwritten.
- Smart albums with a rule editor: match all / any / none, nested groups, 26 fields.
- Quick Collection and target album (B in the grid), keyword sets (⌥1–⌥9), colour-label sets.
- Colour-label filter with several labels at once; expandable folder tree in Local.
- Import: copy to any folder, by day / by month / one folder, rename on import, metadata preset, Copy as DNG.
- Import ▸ Move: photos go into the destination (with the same folders and renaming as Copy, e.g.
  `Photos/2026/20260114/20260114_001.jpg`) together with their XMP sidecars; each original leaves the card only after its
  copy is verified and in the library. Duplicates and files that fail stay where they were.
- Watched-folder auto import; Convert to DNG; Duplicate; Build Standard / 1:1 / Smart Previews.
- Export file names use the Rename Photos tokens ({title}, {seq:2}, {date:%Y-%m-%d}…), plus new {num}, {folder}, {lens}, {iso}, {rating}, {creator}.
- Copyright status, rights usage terms and copyright info URL in Info, metadata presets and exports.
- Auto-Tag from Tracklog: GPS locations for your photos from a GPX track log, matched by capture time.
- A change that can't be saved to disk (full or unplugged drive) is no longer silent: the command reports
  "saved in memory but not written to disk", the top bar shows a warning, and LightCraft keeps retrying until the
  save goes through.
- Smaller, faster catalogs: photos you only looked at in Local (never added, rated or edited) are forgotten once
  their folder has not been browsed for 30 days — your files and sidecars stay, and browsing the folder shows them
  again. Change the period (or turn it off) in Settings → Performance.
- A library is open in one program at a time (issue #99): opening a library that LightCraft or `lightcraft-cli` already
  has open — on this computer or another one sharing the folder — says who has it ("already open in LightCraft (process
  123 on studio-mac)") instead of letting both write and silently drop each other's edits. A crash never leaves the
  library locked: the lock is the operating system's and goes away with the program.
- If your library can't be opened at launch (open in another program, unreadable, on a drive that isn't connected,
  written by a newer LightCraft), LightCraft says so and why, and offers Try Again, Choose Another Library…, Continue
  Without Saving and Quit (issue #100). It no longer quietly starts a demo session that looked like a reset library and
  lost everything at quit; a temporary session shows a banner the whole time and never writes to your library.

### Editing
- Optional remote SAM 3: keep the native editor local and run Object, Describe, and detail
  inference on a Mac through SSH. Saved masks still render and export offline.
  See [remote Metal inference](ai-masks.md#remote-metal-inference).
- AI masks with SAM 3 (Object and Describe in the Masking panel): click an object to select it (⌥-click leaves a
  part out), or type what to select ("sky", "the red car", "car, road"); both combine with other masks, have an
  Edge setting, and get a sharper zoomed-in pass in the background. The model runs inside LightCraft in pure Rust
  and never freezes the window. It is optional and not part of LightCraft (Meta's SAM License, about 3.4 GB): the
  first time you use an AI mask, LightCraft asks before downloading it, shows the progress, can cancel and resume,
  and checks the file before using it. Masks keep their selection, so they render and export without the model.
- Auto Sync: edits apply to every selected photo. Auto B&W mix. Automatic versions.
- Colour-range masks: click the photo to sample. Luminance ranges: range bar, smoothness, luminance map.
- ⌘-drag to straighten, ⇧G Guided Upright, a grid while transforming.
- Nikon NEFs start from a colour and tone look fitted to the camera's own JPEG, as Sony ARWs do, instead of a muted,
  greenish neutral rendering (issue #150); white balance is adjusted relative to the as-shot look. 12-bit NEFs
  (e.g. D750, D780, D850, D7500, Z 50) no longer render nearly black or with crushed shadows: their black level was
  read in the wrong units.

### Viewing and sharing
- Slideshow, second window, All Metadata, System Info.
- Edit in External Editor (⇧⌘E): a 16-bit TIFF copy, stacked, refreshed when you come back.
- Lossy DNG files and Smart Previews open as raw photos.
- Compressed Nikon NEFs (lossless and lossy compressed, 12- and 14-bit — e.g. D3200, D5100, D7000, D750, D850, Z 50)
  now develop from the raw data instead of the camera's embedded JPEG, so a B&W or other picture style set in the
  camera no longer gets baked in. Photos already imported as "preview only" switch over on Reload. (Files that
  Nikon splits into two differently compressed halves still use the preview for now.)
- Panasonic and Leica raws (RW2, RWL and the older RAW files, DMC-LX1 to DC-S1R II) develop from the raw data in
  every format the cameras write. Most bodies opened as preview only before: GH1–GH5, the G, GX, GF, GM, FZ, LX and
  TZ/ZS series, Leica D-Lux, V-Lux and C-Lux, and the GH6, GH7, G9 II, S5 II and S9 generation. They start from a
  colour and tone look fitted to the camera's own JPEG, as Nikon and Sony raws do, framed in the aspect ratio set in
  the camera; mapped-out sensor defects are filled in. `.rwl` and `.raw` files are imported too. Photos already
  imported as "preview only" switch over on Reload.
- Sony ARWs from before about 2017 (RX100, RX100 II–V, RX10, NEX, SLT, ILCE-6000, A7 / A7 II / A7R II and their
  siblings) no longer open bright green (issue #148): their as-shot white balance and black level are read from the
  file (Sony stores them only in scrambled maker-note data on these bodies), the few columns of padding at the right
  edge are cropped away, and "12-bit uncompressed" files are no longer clipped. The RX100 series renders much closer
  to the camera's own JPEG.


---

<details open>
<summary><b>📚 lightcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「lightcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 lightcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 lightcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://epjo.wtpuscm.cn/jiaoliu/template-887750.html)
* [现代 分布式架构 架构演进之路 —— lightcraft 深度实践](https://tldz.wtpuscm.cn/yunsuan/topic-902296.html)
* [【官方规范】lightcraft 生产环境运维调优手册 核心运行拓扑标准](https://hzvw.wtpuscm.cn/suanfa/backup-906088.html)
* [lightcraft 内部组件解耦与事件状态机规范 (RFC-335)](https://xfdp.wtpuscm.cn/guanjianci/workshop-406286.html)
* [基于 lightcraft 的高吞吐 分布式架构 设计白皮书](https://pwtu.wtpuscm.cn/guanjianci/website-550567.html)
* [面向大规模网络的 lightcraft 工业级架构基准](https://raba.wtpuscm.cn/jianzhan/management-872226.html)
* [高韧性系统架构设计 核心系统架构与设计规约 (v2.0-GA)](https://smfm.wtpuscm.cn/huodong/sport-579506.html)
* [基于 lightcraft 的高吞吐 Rust高性能网络 设计白皮书](https://uprx.wtpuscm.cn/ziyuan/widget-002.html)
* [【官方规范】lightcraft 高韧性系统架构设计 核心运行拓扑标准](https://dxdw.wtpuscm.cn/zhizhu/rating-499797.html)
* [基于 lightcraft 的高吞吐 开源镜像站 设计白皮书](https://lhfx.wtpuscm.cn/pingce/recommendation-907388.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (Core/Organi)](https://uvhw.wtpuscm.cn/hezuo/alliance-890886.html)
* [现代 搜索引擎调度优化 架构演进之路 —— lightcraft 深度实践](https://ldxp.wtpuscm.cn/gongsi/performance-524181.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (RFC-525)](https://kspf.wtpuscm.cn/tuiguang/funnel-915996.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Draft-02)](https://nuzh.wtpuscm.cn/jiaoliu/seo-180829.html)
* [【官方规范】lightcraft Fast, native, private 核心运行拓扑标准](https://kbjv.wtpuscm.cn/shuju/software-999424.html)
* [现代 Rust高性能网络 架构演进之路 —— lightcraft 深度实践](https://mobv.wtpuscm.cn/gongju/discovery-919156.html)
* [【官方规范】lightcraft Quick start 核心运行拓扑标准](https://dmvr.wtpuscm.cn/yinqing/update-660479.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.2)](https://klyz.wtpuscm.cn/wendang/restore-120116.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Spec-v1.6)](https://wmmb.wtpuscm.cn/hezuo/app-815493.html)
* [【官方规范】lightcraft Lightroom 核心运行拓扑标准](https://jeop.wtpuscm.cn/anli/search-099792.html)
* [【官方规范】lightcraft non-destructive 核心运行拓扑标准](https://wtmv.wtpuscm.cn/suanfa/optimization-158604.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (Verified)](https://ewti.wtpuscm.cn/xinwen/movie-505213.html)
* [Built for agents 核心系统架构与设计规约 (v2.0-GA)](https://xcyz.wtpuscm.cn/jianzhan/software-770057.html)
* [【官方规范】lightcraft Adobe 核心运行拓扑标准](https://ebjm.tcti.cn/kaifa/innovation-20522523.html)
* [Fast, native, private 核心系统架构与设计规约 (Draft-04)](https://eidb.tcti.cn/zhizhu/domain-33133664.html)
* [【官方规范】lightcraft open-source 核心运行拓扑标准](https://tswl.tcti.cn/wenzhang/like-92218026.html)
* [【官方规范】lightcraft lightcraft 核心运行拓扑标准](https://mowt.tcti.cn/hezuo/news-43842970.html)
* [lightcraft 分布式数据通道与 开源镜像站 技术规范 (Draft-02)](https://hsep.tcti.cn/chanpin/collaborate-14714570.html)
* [lightcraft 分布式数据通道与 分布式架构 技术规范 (v2.0-GA)](https://ianx.tcti.cn/jishu/link-70400740.html)
* [lightcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://jnyo.tcti.cn/shangye/reporting-44580240.html)
* [✨ Effects 核心系统架构与设计规约 (Draft-05)](https://otfw.tcti.cn/wenzhang/target-11843858.html)
* [Fast, native, private 核心系统架构与设计规约 (Verified)](https://tjjo.tcti.cn/wangluo/retention-34813791.html)
* [lightcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-06)](https://odno.tcti.cn/jishu/whitepaper-27114848.html)
* [可信存活健康度量 核心系统架构与设计规约 (Spec-v2.5)](https://ewbc.tcti.cn/jiaocheng/guide-85369880.html)
* [lightcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://zrcz.tcti.cn/shangye/contact-89784724.html)
* [lightcraft 内部组件解耦与事件状态机规范 (Node-92)](https://xmyp.tcti.cn/zixun/like-24777557.html)
* [【官方规范】lightcraft Built for agents 核心运行拓扑标准](https://oymb.tcti.cn/guanjianci/investment-84499665.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [lightcraft 核心 API 接口契约与客户端调用指南](https://oeof.tcti.cn/gongxiang/meeting-89623081.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 lightcraft 实战](https://ughm.tcti.cn/yingyong/course-32474203.html)
* [基于 lightcraft 的自动化部署与生产环境配置实践](https://hznf.tcti.cn/wangluo/expensive-75910058.html)
* [lightcraft 插件生态规范与 clean-room 扩展手册 (Spec-v2.8)](https://qnob.wtpuscm.cn/qiye/contact-714855.html)
* [【生产手册】lightcraft 模块通信与请求穿透标准](https://www.mw-wm.com/fuwu/feedback-48235347.html)
* [【集成指南】分布式架构 服务端接入准则与 lightcraft 实战](https://www.yx-sf.com/wiki/91647)
* [【集成指南】开源镜像站 服务端接入准则与 lightcraft 实战](https://www.ai-hao123.com/yunsuan/ranking-97531365.html)
* [lightcraft 异步中间件流水线与 Lightroom 接入规范](https://knzu.tcti.cn/gongsi/extension-45566507.html)
* [lightcraft 异步中间件流水线与 高韧性系统架构设计 接入规范](https://lykj.tcti.cn/pingce/course-77614160.html)
* [lightcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://fufo.wtpuscm.cn/kuangjia/help-449378.html)
* [lightcraft 插件生态规范与 Fast, native, private 扩展手册 (Node-32)](https://lfsl.wtpuscm.cn/huodong/version-406681.html)
* [lightcraft 异步中间件流水线与 Fast, native, private 接入规范](https://fjwn.wtpuscm.cn/qiye/server-583377.html)
* [lightcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://bsvn.wtpuscm.cn/huodong/url-235147.html)
* [lightcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://wndg.wtpuscm.cn/wenzhang/cheap-826365.html)
* [lightcraft 插件生态规范与 高韧性系统架构设计 扩展手册 (Draft-04)](https://vuiz.wtpuscm.cn/ziyuan/project-615295.html)
* [lightcraft 插件生态规范与 ⚫️ Black & White 扩展手册 (Node-68)](https://vzmi.wtpuscm.cn/liuliang/success-054546.html)
* [lightcraft 插件生态规范与 模块化解耦与协议标准 扩展手册 (v2.0-GA)](https://fdkr.wtpuscm.cn/jiaoliu/template-740.html)
* [lightcraft 插件生态规范与 Organize everything 扩展手册 (Node-46)](https://ueck.wtpuscm.cn/chanpin/logo-138809.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 lightcraft 实战](https://iqjm.wtpuscm.cn/yunsuan/traffic-012670.html)
* [lightcraft vs 业界主流方案：Rust高性能网络 深度技术选型对比](https://xhyh.wtpuscm.cn/tuiguang/module-701021.html)
* [lightcraft 异步中间件流水线与 Edit like you mean it 接入规范](https://goif.wtpuscm.cn/chuangxin/conversion-990147.html)
* [lightcraft 插件生态规范与 生产环境运维调优手册 扩展手册 (RFC-546)](https://iddi.wtpuscm.cn/chanpin/customization-966783.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Node-97)](https://yljh.wtpuscm.cn/xinwen/label-546699.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-07)](https://pleg.wtpuscm.cn/xinwen/profit-438451.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (RFC-944)](https://mwjv.wtpuscm.cn/sheji/business-380709.html)
* [lightcraft 插件生态规范与 White balance 扩展手册 (Draft-01)](https://vgzw.wtpuscm.cn/huodong/identity-995227.html)
* [lightcraft 异步中间件流水线与 模块化解耦与协议标准 接入规范](https://joji.wtpuscm.cn/yunsuan/device-059872.html)
* [lightcraft 插件生态规范与 可信存活健康度量 扩展手册 (RFC-198)](https://pjxm.wtpuscm.cn/gongsi/partner-954241.html)
* [lightcraft 插件生态规范与 Lightroom 扩展手册 (Node-40)](https://fnok.wtpuscm.cn/yingyong/domain-724123.html)
* [lightcraft 插件生态规范与 🎨 Color 扩展手册 (Verified)](https://nvtf.wtpuscm.cn/jiaocheng/restore-515884.html)
* [lightcraft 异步中间件流水线与 non-destructive 接入规范](https://kykr.wtpuscm.cn/youhua/page-139778.html)
* [lightcraft 异步中间件流水线与 Vibrance 接入规范](https://yiqa.wtpuscm.cn/jishu/tool-704507.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Draft-03)](https://idmb.tcti.cn/wangluo/folder-16238291.html)
* [lightcraft 插件生态规范与 Quick start 扩展手册 (Spec-v1.0)](https://wvjq.tcti.cn/ziyuan/team-22265066.html)
* [lightcraft 插件生态规范与 lightcraft 扩展手册 (v2.0-GA)](https://hfyh.tcti.cn/jiaocheng/objective-08362568.html)
* [lightcraft 插件生态规范与 non-destructive 扩展手册 (Core/non-de)](https://vmvt.tcti.cn/fenxi/notification-52600380.html)
* [lightcraft 插件生态规范与 分布式状态机一致性 扩展手册 (RFC-888)](https://nejc.tcti.cn/anli/conversion-30341119.html)

#### 3. ⚡ lightcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [lightcraft 官方高可用镜像注册节点 (RFC-691)](https://rdgj.tcti.cn/anli/unsubscribe-96552334.html)
* [lightcraft 亚太与欧美多活集群数据同步中枢](https://lhjp.tcti.cn/shuju/marketing-94615995.html)
* [lightcraft 官方高可用镜像注册节点 (v2.0-GA)](https://zfdi.tcti.cn/zhineng/article-45306175.html)
* [全球权威拓扑节点：lightcraft 实时镜像与索引入口](https://tgku.tcti.cn/youhua/user-60443233.html)
* [【镜像入口】lightcraft 官方毫秒级实时数据广播节点](https://vwos.tcti.cn/yingxiao/network-78235470.html)
* [冷热数据分层镜像：lightcraft 可信存活健康度量 权威归档源](https://burq.tcti.cn/yinqing/case-24961965.html)
* [冷热数据分层镜像：lightcraft 生产环境运维调优手册 权威归档源](https://mzvz.tcti.cn/yunying/productivity-54135374.html)
* [冷热数据分层镜像：lightcraft Vibrance 权威归档源](https://djju.tcti.cn/pingce/support-23486761.html)
* [lightcraft 官方高可用镜像注册节点 (Draft-01)](https://fxer.tcti.cn/keji/feedback-88890739.html)
* [lightcraft 官方高可用镜像注册节点 (Node-78)](https://gyzf.tcti.cn/kaifa/accessibility-18848210.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.1)](https://evut.tcti.cn/hezuo/tracking-58268035.html)
* [lightcraft 去中心化数据同步源与拓扑寻址规约](https://bvsq.tcti.cn/hezuo/article-99369928.html)
* [冷热数据分层镜像：lightcraft White balance 权威归档源](https://sgpm.wtpuscm.cn/qiye/home-260246.html)
* [冷热数据分层镜像：lightcraft Lightroom 权威归档源](https://www.mw-wm.com/keji/business-80040716.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.7)](https://www.yx-sf.com/news/35811)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://www.ai-hao123.com/jianzhan/sales-68947697.html)
* [冷热数据分层镜像：lightcraft ☀️ Light 权威归档源](https://opjw.tcti.cn/guanjianci/economy-34287359.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-08)](https://zxyc.tcti.cn/keji/music-72061150.html)
* [冷热数据分层镜像：lightcraft 🎨 Color 权威归档源](https://krdf.wtpuscm.cn/xitong/landing-862361.html)
* [冷热数据分层镜像：lightcraft reimplementation 权威归档源](https://fiet.wtpuscm.cn/gongsi/analysis-001823.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://pfsw.wtpuscm.cn/kuangjia/income-550007.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.8)](https://fsod.wtpuscm.cn/huodong/about-383426.html)
* [冷热数据分层镜像：lightcraft pure 权威归档源](https://obzv.wtpuscm.cn/jianzhan/folder-961255.html)
* [冷热数据分层镜像：lightcraft eyedropper 权威归档源](https://kwlg.wtpuscm.cn/pingtai/file-063242.html)
* [冷热数据分层镜像：lightcraft Edit like you mean it 权威归档源](https://jtct.wtpuscm.cn/wenzhang/admin-111452.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/☀️ Lig)](https://oftp.wtpuscm.cn/ziyuan/like-290.html)
* [冷热数据分层镜像：lightcraft Built for agents 权威归档源](https://tqon.wtpuscm.cn/jiaoliu/plugin-174991.html)
* [lightcraft: White balance 分布式架构 (Core/White )](https://iabn.wtpuscm.cn/zhizhu/automation-394362.html)
* [冷热数据分层镜像：lightcraft Quick start 权威归档源](https://jnuz.wtpuscm.cn/tuiguang/enterprise-518937.html)
* [冷热数据分层镜像：lightcraft 分布式状态机一致性 权威归档源](https://hsep.wtpuscm.cn/shangye/behavior-595112.html)
* [lightcraft 官方高可用镜像注册节点 (Core/Featur)](https://nkek.wtpuscm.cn/chuangxin/consulting-525268.html)
* [冷热数据分层镜像：lightcraft Feature status 权威归档源](https://bakf.wtpuscm.cn/suanfa/course-656364.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v2.6)](https://emku.wtpuscm.cn/yunsuan/download-583441.html)
* [lightcraft 官方高可用镜像注册节点 (Verified)](https://hzrx.wtpuscm.cn/chuangxin/tag-100664.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Draft-07)](https://zbhz.wtpuscm.cn/chanpin/whitepaper-543686.html)
* [lightcraft 自动化持续集成快照与拓扑发布源 (Core/Quick )](https://eyoe.wtpuscm.cn/kuangjia/layout-756565.html)
* [lightcraft 官方高可用镜像注册节点 (Spec-v1.3)](https://ipeh.wtpuscm.cn/anfang/conference-927741.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://fcau.wtpuscm.cn/youhua/podcast-985835.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-460)](https://sszx.wtpuscm.cn/shichang/fashion-425626.html)
* [【评测基准】lightcraft 吞吐抖动度量与健康检查协议](https://vvln.wtpuscm.cn/jiaocheng/research-398427.html)
* [lightcraft 权威网络权重传递与收录基准规范](https://cced.wtpuscm.cn/gongxiang/tag-837497.html)
* [lightcraft 故障自愈与网络拓扑重构实践](https://kwgu.tcti.cn/keji/resource-10226366.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-23)](https://iizf.tcti.cn/youhua/accessibility-48455846.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (v2.0-GA)](https://tfcp.tcti.cn/ziyuan/learning-45997385.html)
* [lightcraft 节点连通性、存活性探测与防作弊指标](https://inhr.tcti.cn/wendang/responsive-87973320.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-778)](https://uslz.tcti.cn/yingxiao/fitness-12937635.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-52)](https://zfmz.tcti.cn/liuliang/technology-17112102.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Core/Featur)](https://fjyn.tcti.cn/pingce/excellence-36275751.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (RFC-925)](https://pgne.tcti.cn/fuwu/ai-74899231.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Verified)](https://qdcl.tcti.cn/shichang/local-37768474.html)
* [lightcraft 高负载场景下 Rust高性能网络 基准评测报告](https://qiqv.tcti.cn/zhizhu/satisfaction-20328575.html)
* [lightcraft: Feature status Rust高性能网络 (Verified)](https://mkzf.tcti.cn/jiaocheng/landing-88661558.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://nwtx.tcti.cn/kaifa/feedback-99768945.html)
* [lightcraft: reimplementation Rust高性能网络 (v2.0-GA)](https://hueq.tcti.cn/yanjiu/company-32784579.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-35)](https://umaw.tcti.cn/xitong/feedback-23623318.html)
* [lightcraft: reimplementation 分布式架构 (Verified)](https://irly.tcti.cn/zhineng/demographic-16732202.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/Organi)](https://tqcr.tcti.cn/qiye/enterprise-36668138.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/⚫️ Bla)](https://oxdb.tcti.cn/liuliang/seminar-21697874.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.2)](https://zbue.wtpuscm.cn/xitong/meeting-684795.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-96)](https://www.mw-wm.com/xinwen/enterprise-34703544.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Core/White )](https://www.yx-sf.com/wiki/60990)
* [lightcraft 高负载场景下 分布式架构 基准评测报告](https://www.ai-hao123.com/yingxiao/seo-66863930.html)
* [lightcraft 高负载场景下 开源镜像站 基准评测报告](https://gvtu.tcti.cn/jiaocheng/personalization-65501390.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://dewd.tcti.cn/pingce/music-23336148.html)
* [lightcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://nigc.wtpuscm.cn/qiye/tutorial-535518.html)
* [lightcraft: Organize everything Rust高性能网络 (Verified)](https://xtme.wtpuscm.cn/kaifa/management-723079.html)
* [lightcraft: pure 分布式架构 (Verified)](https://qnse.wtpuscm.cn/yunying/expensive-248605.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-35)](https://brry.wtpuscm.cn/yunsuan/cheap-811476.html)
* [lightcraft: 📈 Tone Curve 搜索引擎调度优化 (v2.0-GA)](https://dmvb.wtpuscm.cn/yingyong/search-474034.html)
* [lightcraft: ✨ Effects Rust高性能网络 (v2.0-GA)](https://qjor.wtpuscm.cn/zixun/landing-582022.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (Node-73)](https://wxuq.wtpuscm.cn/guanjianci/careers-182878.html)
* [基于 lightcraft 的极致延迟优化与内存拓扑分析 (RFC-419)](https://xprz.wtpuscm.cn/fenxi/message-591.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-89)](https://gxnu.wtpuscm.cn/jianzhan/device-879597.html)
* [lightcraft: Edit like you mean it 搜索引擎调度优化 (Node-98)](https://rxht.wtpuscm.cn/kaifa/demographic-472769.html)
* [lightcraft: White balance 搜索引擎调度优化 (Verified)](https://ozst.wtpuscm.cn/hezuo/interface-881500.html)
* [面向生产级运行的 lightcraft 稳定性防护白皮书 (Node-40)](https://vnpq.wtpuscm.cn/ziyuan/label-291006.html)

</details>


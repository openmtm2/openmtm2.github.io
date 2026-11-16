# Version 6 tracks in Traxx: what would have to change

Status: draft, 2026-11-16
Applies to: Traxx, the C++ MTM2 track editor (the `Traxx_OnGoing_Updates` source), as released in 1.7.3, commit `4364111`
Maintained by: the OpenMTM2 project
Companion to: [MTM2 track format version 6](track-format-v6.md), revision 2026-11-16

This document is for the Traxx maintainers. It lists what Traxx would have to change to open, edit and write version 6 tracks: `.SI3` tracks with a 512-cell world, heights outside 0 to 512 ft, 16-bit ground boxes, `.WTR` and `.WFL` version 2, and the `.TXX` generation of the proposal's section 15. It is not part of the format, and nothing in it is required of Traxx.

Every function, file and constant named below was read in the Traxx 1.7.3 source (commit `4364111`). Paths are relative to the repository: `TrackPOD\` is the track library, `Include\` its headers, `Traxx\` the application. The library's headers (`TrackPOD.h`, `TrackPODTerrain.h`, `TrackPODGroundbox.h`, `TrackPODLakes.h`, `TrackPODFalls.h`, `TrackPODVegetation.h` and the rest) exist in both `Include\` and `Traxx\` as identical copies, and the comments of `TrackPOD.h`, `TrackPODLakes.h` and `TrackPODFalls.h` require them to stay byte-identical, so every header change below is made in both.

## 1. Summary

Traxx 1.7.3 has one world: 256 x 256 cells of 32 ft, 8,192 ft a side, with the 8 ft grid at 1024 x 1024. Its heights are 16-bit at 1/128 ft at most (0 to 512 ft), with an 8-bit mirror that tops out at 510 ft. Ground boxes are one byte each. It lists `.SIT` and `.SI2` tracks, reads and writes `.WTR` and `.WFL` version 1, and knows `.TXX` generations TRAXXV1 to TRAXXV19.

| # | Area | Proposal | Size |
|---|---|---|---|
| 3.1 | World size N: the terrain grids and positions | 7.2, 8.1 | large |
| 3.2 | `.CLR` and `.LTE` planes | 8.2 | small |
| 3.3 | Heights: the store, the step and the base | 7.1, 7.3, 11 | large |
| 3.4 | Ground boxes | 8.2, 8.3 | medium |
| 3.5 | The `.SI3` and the `.TXV` keys | 5, 6, 11 | medium |
| 3.6 | Lakes: `.WTR` version 2 | 9.1 | medium |
| 3.7 | Waterfalls: `.WFL` version 2 | 9.3 | medium |
| 3.8 | Vegetation | 9.2 | small |
| 3.9 | The `.TXX` generation | 15 | medium |
| 3.10 | Views, tools and dialogs | 7, 8.1 | large |
| 3.11 | Importing and exporting height maps | 7.2, 11 | medium |
| 3.12 | Write POD | 5, 8, 9, 11 | medium |

Sizes: **small** is a few functions in one or two files; **medium** is one subsystem across several files; **large** reaches most of the library or the application, or needs a design choice first.

A track that uses nothing of version 6 MUST stay byte for byte what Traxx 1.7.3 writes, in the pod and in the `.TXX` (proposal 5.1, 15.2). Every change below is therefore conditional on the track being version 6.

## 2. How a track reaches Traxx today

Besides File > New, three ways, and a version 6 track needs all three:

- **File > Open:** a `.TXX`, read by `CTrackPOD::LoadBinary` (`TrackPOD\TrackPODBinary.cpp`), the editable form of a track.
- **File > Clone:** loose files, read by `CTrackPOD::CloneFromSIT` (`TrackPOD\TrackPODClone.cpp`) from a `.SIT` or `.SI2` chosen in `CTraxxDoc::OnFileClone` (`Traxx\TraxxDoc.cpp`).
- **View POD:** `CTrackPOD::CountTracks` and `CTrackPOD::Load` (`TrackPOD\TrackPODFile.cpp`), which open the track view-only (`CTraxxDoc::OnFileViewPOD` sets `viewonly`).

Write POD (`CTraxxDoc::OnFileWritePOD`, then `CTrackPOD::Save`) is the only writer of a pod.

## 3. Areas

### 3.1 World size N: the terrain grids and positions

Proposal 7.2 and 8.1. N is `worldCells`, 256 or 512; W = 32 N ft.

**Today.**

- `CTrackPODTerrain` (`Include\TrackPODTerrain.h`, `TrackPOD\TrackPODTerrain.cpp`): the constructor allocates `raw`, `clr` and `lte` as `256*256` cells; `Reset` and `FillLteFlat` fill `256*256`.
- Cell indexing is `x + (z << 8)` with `& 255` wraps in `GetRawAtPoint`, `InterpolateRawAtPoint`, `GetTextureAtPoint`, `GetTextureFlagsAtPoint`, `GetLteAtPoint`, `CornerV`, `PlanarExpand`, `BuildLakeGround`, `GetEngineY0`, `RefreshRaw` and `SyncFromRaw`.
- The 8 ft grid is `TERRAIN_FINE` (1024, `Include\terrainspline.h`, with a `static_assert` in `TrackPODTerrain.cpp`), indexed with `& 1023` and `<< 10` in `SurfaceHeightV`, `FineV`, `FineVertexAtPoint` and `RefreshRaw`.
- Surface positions are 1/256 ipos, 16,384 a cell: `SurfaceHeightV` takes `(px >> 14) & 255` and `px & 16383`; `PickSurfaceV` returns 0 from `256.0*16384.0` on.
- The spline: `terrainspline.c` wraps with `TS_WRAP(v) ((v) & 255)`, and `terrainSplineExpand` takes a 256 x 256 grid (`TERRAIN_COARSE`). Its header says the file is shared verbatim with the engine and must not change without `TERRAIN_SPLINE_VERSION`.
- Positions in the `.SIT`: `Pod1SitToIpos` (`TrackPODFile.cpp`) adds 16,384 ipos (8,192 ft) to a negative position.
- The course: `TRAXX_COURSE_WRAP` (16384, `Traxx\TraxxView.h`) and `CourseWrapDelta` join course points the short way round an 8,192 ft world.
- Lighting: `CTrackPOD::BakeEngineLighting` and `BuildLighting` (`TrackPOD\TrackPOD.cpp`) use `static int y0[65536]` and `static BYTE buf[65536]`; `EngineBakeGround` (`TrackPOD\EngineLight.cpp`) loops 256 x 256 with `& 255` neighbours.
- `CTrackPOD::TextureUsage` counts over 65,536 `.CLR` words.
- The terrain generator (`TrackPOD\TrackPODTerrainGen.cpp`) works on 256 or 1024 grids.

**Change.** Give the track a world size, set from `worldCells` (3.5) or chosen for a new track, and take every constant above from it: grids of N x N and 4N x 4N, indices `x + z * N`, wraps `& (N - 1)` and `& (4N - 1)`, surface positions below `N * 16384`, `.SIT` positions wrapped by 64 N ipos, the course wrap at 64 N. Proposal 8.1 lists the same constants on the engine side; the cell and fine-cell shifts do not change. The spline needs N as a parameter with its indices wrapping at N (7.2); at N = 256 its output is unchanged, so the shared copy's rule applies to how the parameter is added, not to the results. A set of small helpers (cell index, wrap, world side) is easier to review than literal edits at each site.

**Size: large.** This is the base the other areas build on.

### 3.2 `.CLR` and `.LTE` planes

Proposal 8.2.

**Today.** `CTrackPODTerrain::LoadClr` and `SaveClr` read and write `2*256` x 256 bytes, `LoadLte` and `SaveLte` `7*256` x 256, whatever the pod entry's length (`ParseLVLFile` and `CloneFromSIT` call them without a size check). `CTrackPOD::Save` sets the entry lengths literally: `.CLR` 131,072, `.LTE` 458,752, and writes `.CL1` (262,144) and `.CL2` (786,432) as zeros. In the `.TXX`, `CTrackPODTerrain::Save` run-length codes each plane up to `i >= 65536`, and `LoadCompressed` fills runs with no bound.

**Change.** Sizes 2N², 7N², 4N² and 12N². A reader MUST refuse a grid file whose size does not match N (8.2), which also means checking the entry length before reading. The `.TXX` planes become N x N (3.9), and `LoadCompressed` should stop at N² cells.

**Size: small**, once 3.1 exists.

### 3.3 Heights: the store, the step and the base

Proposal 7.1, 7.3 and 11.

**Today.**

- `h16` and `fine` are `WORD` grids in v, 1/128 ft, 0 to 65,535 (`TrackPODTerrain.h`). Tools clamp to that range: `ApplyStamp16`, `AdjustedHeight` (`AdjustHeights16`, `AdjustHeightsFine`) and `ImportHeights`.
- `raw` is the 8-bit mirror, `MirrorOf(v) = min(255, (v + 128) >> 8)`: 0 to 510 ft. The other modes keep it in step (`RefreshRaw`, `SyncFromRaw`), and code that reads it (about 40 call sites of `GetRaw()` across `TrackPOD\` and `Traxx\`, besides `GetRawAtPoint` and `InterpolateRawAtPoint`) would see a tall track cut at 0 and 510 ft. In every mode, the minimap (`CTraxxView::UpdateMapBuffer`) and the arena backdrop's height (`GetRawAtPoint`) read it, and `SyncFromRaw` folds an 8-bit tool's edit back into `h16` as `raw << 8`.
- Value ranges in the user interface: the edit bar's `AMPLITUDE_MAXV` (`255*256`) and its spin of -255 to 255 (`Traxx\TraxxFrame.cpp`); the Raise / Lower dialog's 255 caps (`Traxx\RaiseLowerDlg.cpp`); the terrain generator's ground level, hill height and valley depth of 0 to 255 (`Traxx\TerrainGenDlg.cpp`); a truck's height of 0 to 255 steps (`Traxx\TruckPropertiesDlg.cpp`); the camera's height capped at `MAXZPOSITION`, `3*ALTITUDESCALE*256`, which is 768 two-foot steps or 1,536 ft (`Traxx\TraxxView.cpp`).
- The track's own water: `CTrackPOD::SetWaterLevel4` clamps to `4*255` quarter steps (510 ft), and `WriteLVLFile` writes it as `!waterHeight` in quarter steps.

**Change.**

- Keep a version 6 track's heights in a 32-bit store at 1/128 ft (h128, the store proposal 7.3 recommends): `height_ft = h128 / 128`, from `terrainHeightBase * 128` up. Keep the `WORD` grids for every other track, so its bytes do not change. TRAXXV99 holds either store (proposal 15.3, record 17: an i32 store width of 16 or 32).
- Keep `terrainHeightStep` and `terrainHeightBase` with the track: read from the `.TXV` (3.5), kept in the `.TXX` (3.9), and picked at Write POD by proposal section 11 when the heights no longer fit the old ones.
- Decide what the 8-bit mirror means on a tall track (for instance clamped, as JSTraxx does for heights below 0), and move the tools and views that must show the real height from `raw` to the store.
- Widen the ranges listed above to the store's range, in feet or steps.
- The track's own water height keeps its units (a quarter of an 8-bit step, 0 to 1,020 as today). On a version 6 track whose terrain reaches higher, `SetWaterLevel4`'s clamp and the water level controls allow values up to the top of the terrain, `2 * (terrainHeightBase + 65536 / terrainHeightStep)` (proposal section 10, item 7). Other tracks keep the clamp at 1,020.

**Size: large.** The store's type reaches the terrain class, every brush, the lakes' and falls' ground, picking and drawing. Deciding between one 32-bit store for every track (converted to 16 bits at the file) and two stores side by side comes first.

### 3.4 Ground boxes

Proposal 8.2 and 8.3.

**Today.**

- `CTrackPODGroundbox::lower` and `upper` are `BYTE` in 2 ft steps (`Include\TrackPODGroundbox.h`), saved as one byte each by `CTrackPODGroundbox::Save` in the `.TXX`; `midz` comes from them.
- `CTrackPOD::ra0[65536]` and `ra1[65536]` (`TrackPOD.h`), `GetRA0` and `GetRA1` (`x + (y << 8)`), `BuildRA01CL0` and `RebuildFromRA01CL0` (`TrackPOD.cpp`).
- `CTrackPOD::LoadGroundBoxes` (`TrackPODFile.cpp`) reads 65,536 bytes of `.RA0` and `.RA1` and `12*65536` of `.CL0`, then walks 256 x 256 cells; `CloneFromSIT` reads the same sizes from loose files.
- `CTrackPOD::Save` writes `.RA0` and `.RA1` at 65,536 bytes, `.RA2` to `.RA5` as 65,536 zeros, `.CL0` at `12*65536`.
- The Groundbox Tools bar limits the top to 1 to 255 and the bottom to 0 to 254 (`TraxxFrame.cpp`).

**Change.** Hold a box's bottom and top in a wider type (TRAXXV99 stores them as f64 in 2 ft units, 3.9). At Write POD, write 8-bit `.RA0`/`.RA1` (and `.RA4`/`.RA5`) when every box fits `terrainHeightBase + 2 * byte`, else 16-bit at `terrainHeightBase + value / terrainHeightStep`, the pair at the same depth; read both, told apart by size (8.3). Grids at N x N (8.2). Two points the proposal leaves open (section 17) apply here: a box below the base cannot be stored, and an odd base needs 16-bit boxes. `MAXNPODGROUNDBOXES` (2048) is a Traxx limit, not a format one; a 512 world may want it raised (proposal 14).

**Size: medium.**

### 3.5 The `.SI3` and the `.TXV` keys

Proposal 5, 6 and 11.

**Today.**

- `PodIsSituationName` (`TrackPODFile.cpp`) accepts `.SIT` and `.SI2`; `CountTracks` lists tracks by it, and `CTrackPOD::Load` sets `situation_is_si2` from it.
- `CTrackPOD::Save` names the situation file `omitted_any ? ".SI2" : ".SIT"`.
- `Pod1StemTaken` checks `WORLD\<stem>.SIT`, `WORLD\<stem>.SI2` and `LEVELS\<stem>.LVL` when picking a clone's stem.
- `OnFileClone` offers `*.SIT;*.SI2` (`sitfilter`, `TraxxDoc.cpp`).
- `CTrackPOD::Load` reads the `.TXV` entry (1 to 65,536 bytes) into `txv_text`. `ParseTerrainGridKey` and `ParseTerrainHeightsKey` read its keys; `ResolveTerrainMode` picks the mode from the `.RAW` length (65,536, 131,072 or 2,097,152).
- `WriteTXVFile` computes `formatVersion` from 1 to 5, writes its own keys, and carries every other key over verbatim; `TXVKeyIsOurs` holds the list of keys it writes itself.

**Change.**

- List `.SI3` entries as tracks (`PodIsSituationName`, `CountTracks`, `Pod1StemTaken`, the clone filter). A `.SIT` or `.SI2` of the same stem is a different track with its own entry (5.1).
- Read `worldCells`, `terrainHeightMap`, `terrainHeightStep` and `terrainHeightBase` from a `.SI3` track's `.TXV`, and refuse a value outside the allowed ones with a message naming the key (5.2, 6). `terrainHeightMap` names the heights (a PNG or a 16-bit RAW) and is required at 512 cells; the `.RAW` named by the `.LVL` need not exist (proposal 17).
- Write a version 6 track as `WORLD\<stem>.SI3` with `formatVersion=6` and the keys of section 6, and never turn a `.SIT` or `.SI2` track into a `.SI3` or back (5.1).
- Add the four keys to `TXVKeyIsOurs`, so a stale copy is neither carried over nor written twice. Until then, Traxx 1.7.3 already does what section 11 asks of a tool that does not know them: it carries them over untouched.

**Size: medium** (the listing alone is small).

### 3.6 Lakes: `.WTR` version 2

Proposal 9.1.

**Today.**

- `TrackPODLakes.cpp` defines `LAKE_TILES` (65,536, "256 x 256; everything wraps with & 255") and wraps with `& 255` and `<< 8` throughout the fill, the spill and the notch code. `tmap` is 65,536 bytes.
- `LakeSeed` holds `BYTE x, z`; a lake's box is `BYTE minx, minz, maxx, maxz`; `LakeNotchEdge` holds `BYTE x, z` (`Include\TrackPODLakes.h`). `LakeGround::tilemin` is 256 x 256.
- `BuildWTR` writes a header of major 1, minor 1, 256 x 256 and flags bit 0 (the edge rule) at +20, then one `BODY` record per lake with a u32 height at +4, the box as bytes at +40 to +43 and the tile count at +44. `ParseWTR` refuses a major above 1 and takes a `TMAP` only at 65,536 bytes.
- `CTrackPODLakes::LoadTXX` refuses seeds outside 0 to 255.
- Heights: `LAKE_MAX_HEIGHT` is `4*255 + 3` water units (511.5 ft), the Lake dialog's range (`Traxx\LakeDlg.cpp`).
- Outside the class: `CTrackPOD::UpdateLakeNotches` uses `2 * 65536` bytes, `CLakeGroundBuf` 65,536 ints, the falls' water lookup `FallWaterCB` wraps tiles with `& 255` (`TrackPOD.cpp`), and the GL renderer reads `tmap` with `& 255`.

**Change.** The tile map, seeds, boxes, notch edges and every wrap at N, with seeds and boxes widened to 16 bits. The world's size picks the version (9.1): `BuildWTR` writes version 1 for a 256-cell track and version 2 for a 512-cell one (major 2, 512 x 512, an i32 height, the box as u16 at +0x28, the tile count at +0x30, the same minor and flags at the same offsets), and `ParseWTR` refuses any other pairing. A version 2 height is valid from `2 * terrainHeightBase + 1` to `2 * (terrainHeightBase + 65536 / terrainHeightStep)`; on a 256-cell track a lake above 2,048 ft is an error, never a reason to write version 2. The fill compares `tilemin` (in `v >> 1`) with `32 * H` in `int`, which still holds with the 32-bit store.

**Size: medium.**

### 3.7 Waterfalls: `.WFL` version 2

Proposal 9.3.

**Today.**

- `FALL_COORD_MAX` (131,071, 1/16 ft) and `FALL_WORLD_FT` (8192.0) in `Include\TrackPODFalls.h`.
- In `TrackPODFalls.cpp`: `Red` reduces into 0 to 131,071, `WrapDiff` takes the short way across 131,072, `WrapFt` wraps into [0, 8192); `FallClassicHeightRaw` finds the cell with `& 255` (the engine's `computeGridX`); `BuildNotches` fills two 65,536-byte maps.
- `CTrackPODFalls::Errors` refuses a crest point above `FALL_COORD_MAX` and a height outside 1 to 4,096 (`FALLERR_HEIGHT`); the Waterfall dialog's height range is 1 to 4,096 (`Traxx\WaterfallDlg.cpp`); `Traxx\TraxxViewFalls.cpp` masks clicked points with `FALL_COORD_MAX`.
- `BuildWFL` writes major 1, 256 x 256 and a u32 height; `ParseWFL` refuses a major of 0 or above 1 and a header that is not 256 x 256.
- The falls' ground (`FallCornerCB`, `FallGroundHeightCB`, `TrackPOD.cpp`) wraps tiles with `& 255` and reads the `WORD` surface through `VegGroundHeight`. The GL renderer places a fall's copy by multiples of 8,192 ft.

**Change.** Crest points below 16 W (0x40000 at 512 cells), every wrap at W, every tile lookup at N in the N x N tile map, the ground from the store of 3.3 (9.3). The world's size picks the version: `BuildWFL` writes version 1 at 256 cells and version 2 at 512 (major 2, 512 x 512, an i32 crest height in a version 2 lake's range), `ParseWFL` refuses any other pairing. The version 1.1 records (mist size, roughFrom at byte 130) and the `.TXX` waterfalls block (version 3) keep their layout.

**Size: medium.**

### 3.8 Vegetation

Proposal 9.2.

**Today.** `TrackPOD\TrackPODVegetation.cpp` defines `VEG_TILES` (256) and `VEG_TILE_SUBUNITS` (8192, 1/256 ft a tile). `BuildVEG` writes 256 x 256 in the header; tree positions must be below `VEG_TILES*VEG_TILE_SUBUNITS`; `ParseVEG` refuses a header that is not 256 x 256 and grass tiles of 256 or more. `Traxx\VegetationPlanter.cpp` sets `VEG_WORLD_FT` (8192.0), maps a mask pixel as `floor(ft * size / 8192)`, and buckets plants in `head[65536]` by `tx + (tz << 8)`.

**Change.** The layout does not change. Tree x and z run to `W * 256` (exclusive) and grass tiles to N - 1 (9.2). The proposal's open questions keep the header's tile counts at 256, as Traxx writes them, so the header check and the world bound become two separate things. The planter's world and bucket grid follow N.

The painted masks (`_V`, `_G`, and the track's `_MASK` and `_WMSK`) cover the track's own world once (proposal 9.5): the planter maps a pixel as `floor(ft * size / W)` with W = 32 N ft instead of 8,192, and accepts twice its square size limit (4,096 instead of 2,048) on a 512-cell track. The detail normals (`_DTL`, `_WDTL`) are tiles and do not change.

**Size: small.**

### 3.9 The `.TXX` generation

Proposal 15.

**Today.** `CTrackPOD::LoadBinary` compares the 10-byte header with `strcmp` against `TRAXXV1` to `TRAXXV19`, and any other header sets the version to 0 and aborts (`FILE_ABORT`) before anything past it is read. `SaveBinary` writes the lowest generation the track needs. The records then follow the generation through `version >= n` tests. The parts version 6 touches:

- `CTrackPODTerrain::Load` and `Save`: ten planes of 256 x 256 (3.2).
- `CTrackPODGroundbox::Load` and `Save`: the bottom and top as one byte each.
- `CTrackPODLakes::LoadTXX`: seeds 0 to 255.
- `CTrackPODTerrain::LoadTXXHeights` and `SaveTXXHeights`: the mode, then 131,072 or 2,097,152 bytes of u16.
- `CTrackPODFalls::SaveTXX`: block version 3 (`LoadTXX` takes 1 to 3).
- The V19 water detail and water mask records, written only when a map is set.

**Change.** Read and write the generation of proposal 15.3, TRAXXV99 for now: TRAXXV19's record with `worldCells` before the terrain, N x N planes, f64 ground box heights, an i32 store width before the V17 samples, seeds to N - 1, the world's crest points, both water map records always, then the version 6 block (block version 1, flags, step, base). Write it for a version 6 track and only for one (15.2).

The number 99 is temporary. The proposal will take whatever number Traxx itself gives version 6 tracks, so the choice is Traxx's. Whichever it is, the loader's `version >= n` tests need care: TRAXXV99 is V19's record, so it must not pass a test for a later generation, such as Traxx 1.8.0's `version >= 20` for the box records' all-weather flag. Reading it as generation 19 with a version 6 flag avoids that.

**Size: medium.**

### 3.10 Views, tools and dialogs

Proposal 7 and 8.1.

**Today.**

- The view: `gridx` and `gridy` are cells; the camera's x and y are clamped to 0 to 16,383 ipos and its height to `MAXZPOSITION` (`TraxxView.cpp`).
- The picked cell: `editlocation` packs `x | (y << 8)` (`TraxxViewInteract.cpp`, from `PolyIndexToGrid`, or from the 8 ft pick `editfine`, which packs 16 bits each way). `TraxxViewEdit.cpp` and `TraxxViewDisplay.cpp` unpack it with `& 255` and `>> 8`.
- The minimap: `UpdateMapBuffer` samples every fourth cell of `raw` into 64 x 64 and plots course points `>> 8`; `DisplayMap` scales by 256; a click on it sets `gridx` from `255 * (pt.x - rect.left) / width` (`TraxxView.cpp`).
- The GL renderer (`Traxx\OpenGLTerrainRenderer.h`, `.cpp`): `TRAXX_GL_TERRAIN_TILE_SIZE` (4) and `TRAXX_GL_TERRAIN_TILES_PER_AXIS` (`(255 / 4) + 1`, 64) size the tile and chunk caches; the world-wrap border clips at 259 cells (`W = 259.0 * 4096.0`); `altgrid` and `tmap` reads wrap with `& 255`; `surftilemin` and `surftilemax` are 65,536 ints; the water lists stop at cell 256.
- The software path: `Traxx\SurfaceMesh.h` describes `raw` and `clr` as 256 x 256 and the wrap border window as `256 / tilesize` tiles.
- Brushes and areas: `ApplyStamp16`, `StampCopy16` and `StampPaste16` are called with (256, 255) or (1024, 1023) (`TraxxViewEdit.cpp`); the 8-bit brush clips at cells 0 to 255.
- Undo: `TerrainUndo` holds a 65,536-byte `grid`, a 256 x 256 `grid16` and a 1024 x 1024 `gridfine` (`TraxxView.h`, `TraxxView.cpp`).
- Texture painting, flood, rotate, mirror and area copy work on `GetClr()` words at `x + (y << 8)`.
- A new track (`CTraxxDoc::OnNewDocument`) is always 256 cells.

**Change.** Everything above from N, and every height range from the store (3.3). The cell packing needs 9 bits per axis at N = 512, or two integers. A new track needs a world size choice; the proposal's writer guidance says when 512 is worth it (section 11).

**Size: large.** Mostly mechanical, but spread over the whole application.

### 3.11 Importing and exporting height maps

Proposal 7.2 and 11.

**Today.**

- `CTraxxView::OnTerrainImportRaw` (`TraxxViewEdit.cpp`) takes a `.RAW` of 65,536, 131,072 or 2,097,152 bytes, or an image through `HeightImageDecode`.
- `TrackPOD\TrackPODHeightImage.cpp`: `SizeAllowed` accepts 256 or 1024 square ("height maps must be 256 x 256 or 1024 x 1024"); `HeightImportPlan` and `HeightImageToRaw` map an image to one of the three `.RAW` lengths.
- `CTrackPODTerrain::ImportHeights` stores the samples as v.
- `CTraxxView::OnTerrainExportRaw` writes 256 or 1024 square, 8 or 16-bit PNG or RAW (`HeightExportFormat`).
- Write POD's `terrainHeights=` PNG comes from `EncodeHeightsPng` (`TrackPODFile.cpp`), and `ApplyTerrainHeightsImage` reads it back, refusing anything above 32 MB.

**Change.** Accept N and 4N squares for the track's world (512 and 2,048 at 512 cells; RAW sizes of 2N² and 32N² bytes, 524,288 and 8,388,608). A 16-bit sample is a height only through a step and base (7.1): an import either takes them from the author or keeps the track's, and an export of a version 6 track writes 16-bit samples with the track's step and base and tells the author what they are, or picks them by the rule of section 11. At Write POD, a version 6 track's heights go to the `terrainHeightMap` entry (3.5) rather than `terrainHeights=`.

**Size: medium.**

### 3.12 Write POD

Proposal 5, 8, 9 and 11.

**Today.** `CTraxxDoc::OnFileWritePOD` refuses lakes in error and waterfalls in error, asks about default sun values, checks the detail maps and `CheckIntegrity`, then explains the RAW fallback (skipped when the terrain is not Classic 8-bit, `podexport_forced`). `CTrackPOD::Save` refuses a terrain mode that disagrees with its data (expecting `RawFileLength` of 65,536, 131,072 or 2,097,152), forces the no-fallback path (`.SI2`) for any mode but Classic 8-bit, then writes the grid files at the fixed sizes of 3.2 and 3.4, the `.RAW` or `_H.PNG`, the `.WFL` (built before the sounds), the situation file, the `.TXV` and the `.WTR`.

**Change.**

- Decide whether the track is version 6 before anything is written, from the same conditions as the `.TXX` (proposal 15.2: a 512-cell world, heights outside 0 to 512 ft or in the 32-bit store, ground boxes that need 16 bits, or a track opened from a `.SI3`). A track that is not version 6 is written exactly as today.
- For a version 6 track: `WORLD\<stem>.SI3`, `formatVersion=6` and the keys of section 6 (3.5); the step and base picked by section 11, counting the ground boxes (proposal 17); the heights at `terrainHeightMap`, required at 512 cells; every grid file at N (3.2, 3.4); 8 or 16-bit ground boxes (3.4); `.WTR` and `.WFL` in the version the world's size picks (3.6, 3.7).
- New refusals, with messages: on a 256-cell track, a lake or a fall above 2,048 ft (9.1, 9.3); a ground box below the base; a grid whose size does not match N.

**Size: medium.**

## 4. Suggested order

Each step leaves Traxx usable, and a track that is not version 6 unchanged.

1. **Recognise version 6, refuse it cleanly** (3.5, small): list `.SI3` tracks, read the keys of section 6, and refuse to edit a `.SI3` track with a message until the steps below exist. Traxx 1.7.3 already refuses the `.TXX` (3.9).
2. **Tall tracks at 256 cells** (3.3, then 3.4 and 3.9, large): the 32-bit store, the step and base, wide ground boxes, and the `.TXX` generation with `worldCells` = 256. This is what Pikes Peak needs (proposal 2).
3. **Height maps** (3.11, medium) for those tracks.
4. **Write POD at 256 cells** (3.12, medium): `.SI3`, the keys, the step and base, 16-bit boxes.
5. **The world size in the library** (3.1 and 3.2, large; then 3.6, 3.7 and 3.8, medium and small): grids, positions, lakes, falls and vegetation at N.
6. **The world size in the application** (3.10, large), with a world size choice for a new track.
7. **Write POD at 512 cells** (3.12): the versions picked by the world's size and the new refusals.

## 5. What JSTraxx did

[JSTraxx](https://github.com/juanputrerasm/JSTraxx), the web port of Traxx, reads and writes version 6 tracks. Its source can serve as a reference for the decisions above; it is TypeScript, and nothing in it needs to be copied. The file formats themselves come from OpenPhotex, which JSTraxx vendors.

| File in JSTraxx | What it holds |
|---|---|
| `src/core/traxx/world.ts` | N and the grid helpers (index, wrap, world side); the choice of store: a `Uint16Array` for a track within 0 to 512 ft, as Traxx keeps it, an `Int32Array` at 1/128 ft for a version 6 track |
| `src/core/traxx/v6.ts` | when a document is version 6 (`isV6Document`, `v6Features`), the step and base Write POD picks (`pickStepAndBase`, which counts ground boxes and keeps the track's own values when they still hold every height), and the version 6 height, ground box and `.TXV` writes |
| `src/core/traxx/txx.ts` | the `.TXX` reader and writer, TRAXXV1 to TRAXXV19 as Traxx writes them and TRAXXV99 for a version 6 track |
| `src/core/traxx/terrain16.ts` | the terrain modes, the 8-bit mirror and its sync at N, with the 32-bit grids; a height below 0 ft mirrors as 0 |
| `src/core/traxx/lakes.ts` | the lake fill, spill and notches with the tile map, seeds and boxes at N |
| `src/core/traxx/falls.ts` | the falls in a world of N cells with the track's step and base (`FallWorld`), and the `.WFL` version picked by the world's size |
| `src/core/doc/pod-to-txx.ts` | a pod track to a `.TXX`, written as TRAXXV99 when the track is version 6 |

A new track's world size is chosen under File > New (`src/app/file-commands.ts`).

## 6. Questions for the proposal

Points met while reading Traxx that the proposal does not settle yet:

- **The `.TXX` generation number**, which the proposal leaves to Traxx (3.9).

The proposal's own open questions (its section 17) apply too, in particular ground boxes below the base and 8-bit boxes with an odd base (3.4).

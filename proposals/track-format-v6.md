# MTM2 track format version 6: taller tracks and larger worlds

Status: draft open specification, 2026-11-13
Applies to: Monster Truck Madness 2 tracks, for any engine or tool that reads or writes them (OpenMTM2, Community Patch 4, track editors, converters)
Maintained by: the OpenMTM2 project
Replaces: the earlier drafts "tall tracks with fine height steps" (2026-11-09) and "maps four times larger" (2026-11-13)

This is an open specification. Anyone may implement it, in any engine or tool, without asking and without a fee. It is written so that an engine developer can implement it from this document alone; the background it relies on is summarised in section 4, with the analysis documents it comes from named there.

## 1. Summary

Today every MTM2 track is the same size, a square 8,192 ft a side that wraps around at its edges, and can be at most about 512 ft tall from its lowest point to its highest. Format version 6 lifts both limits:

- **Taller tracks.** A track chooses how fine its height steps are (`terrainHeightStep`) and where its height range starts (`terrainHeightBase`). Heights are still an ordinary 16-bit greyscale PNG or 16-bit RAW. A track can be up to 8,192 ft tall, or keep Community Patch 4's 2.4 mm steps under 512 ft.
- **Larger worlds.** A track chooses its number of cells along each side (`worldCells`): 256 as today, or **512, a world 16,384 ft a side** with four times the area. A cell is still 32 ft and the fine terrain is still 8 ft.
- **A new situation file, `.SI3`.** A version 6 track's situation file is `WORLD\<stem>.SI3`, in the same text format as a `.SI2` for now. Older builds do not list `.SI3` files, so they never offer a track they would load wrongly.

Tracks that use none of this are unchanged and keep loading everywhere as before.

## 2. Motivation

**Height: Pikes Peak.** 4x4 Evolution 2's Pikes Peak climbs from 12 ft to 1,821 ft, about 1,800 ft in one track.

| Game | Height step | Tallest track |
|---|---|---|
| MTM2 (original) | 2 ft | 510 ft |
| Community Patch 4 beta 0.52 | 1/128 ft (2.4 mm) in the file | 512 ft |
| 4x4 Evolution 2 | 1/32 ft (9.5 mm) | about 2,048 ft |

To convert it today the mountain must be squashed to under a third of its height (the road becomes three times less steep and the jumps and banks change), or cut into pieces. Community Patch 4's 16-bit heights give more steps, not more height. Evo 2 shows both can be had at once: its steps are 4 times coarser than CP4's file, yet its tracks look smooth (the smoothing comes from interpolating between grid points, which CP4's spline mode also does) and are 4 times taller.

**Size: Chicago.** Midtown Madness's Chicago is a whole city at real-world scale, being converted to MTM2 (`mm1conversion`).

| Extent | Metres | Feet at true scale | Against 8,192 ft |
|---|---|---|---|
| The drivable city | 2,729 x 2,900 | 8,954 x 9,514 | 1.16 x as long |
| The same with a 256 ft margin per side against the wrap | | 9,466 x 10,026 | 1.22 x |
| Everything the game has collision for (lake, edges) | 3,341 x 3,819 | 10,962 x 12,530 | 1.53 x, 2.05 x the area |

The city does not fit, so the conversion shrinks it to 0.83 of its size: buildings, bridges, ramps and the distances between jumps are all 17% smaller than they should be. The smallest world that holds it is 384 cells (12,288 ft); the next power of two, which the engine's bit masks need (section 4.1), is 512 cells.

## 3. Conventions

- **MUST**, **MUST NOT**, **SHOULD** and **MAY** are used as in RFC 2119.
- A **reader** is anything that loads a track: a game engine, a viewer, an editor. A **version 6 reader** implements this document. A **writer** is anything that saves one.
- **N** is the track's `worldCells` (256 or 512). **W = 32 N** ft is the world's side (8,192 or 16,384 ft).
- World positions in the engine are in **1/256 ft**, as in retail MTM2. Grid files are stored row by row, **row = z, column = x**, `index = row * size + column`, as today.
- Multi-byte numbers in binary files are little-endian.
- Paths are POD entry names, case-insensitive. `<stem>` is the track's name, the stem of its situation file.

## 4. The format today

This section is the baseline version 6 extends. It comes from the OpenMTM2 project's analysis of the retail MONSTER.EXE 2.00.42 and of the Community Patch 4 beta 0.52 executable 2.0.52.2, with the Traxx 1.7 editor's source for comparison. CP4's developer has published no source code or written specification of its own; the CP4 parts below are our reading of the binary and may change before CP4 is final.

### 4.1 World size

The 8,192 ft size is in the engine's code, not in any file:

| Where | What it does | Value today |
|---|---|---|
| Cell lookup (terrain, surface, ground boxes) | `(pos >> 13) & 255` | 256 cells |
| Grid index | `row * 256 + col` | 256 a row |
| CP4 fine terrain lookup | `(pos >> 11) & 0x3ff` | 1024 fine cells |
| CP4 spline expansion | indices wrap at 256 | 256 |
| Truck and moving-object position, every frame | above 8,192 ft subtract 8,192, below 0 add 8,192 | 8,192 ft |
| Objects, ramps, course ends at load | a negative x or z gets 8,192 added | 8,192 ft |
| Drawing a truck at the copy nearest the view | moved when more than 4,096 ft away | half the world |
| Drawing the ground around the view | view cell `(x & 0x1fe000) >> 13` | 256 cells |
| CP4 fog volumes | distance to the view wrapped | 8,192 ft |

Distances in the simulation are not taken the short way across the wrap: two things either side of the edge do not meet until one is wrapped across.

### 4.2 Heights

**The height file's size picks the grid (CP4 0.52):**

| `DATA\<stem>.RAW` size | Samples | Grid spacing |
|---|---|---|
| 65,536 bytes | 8-bit, 256 x 256 | 32 ft (original MTM2) |
| 131,072 bytes | 16-bit, 256 x 256 | 32 ft, or smoothed to 8 ft with `terrainGrid=spline` |
| 2,097,152 bytes | 16-bit, 1024 x 1024 ("Full Map") | 8 ft |

CP4 adds `DATA\<stem>.TXV`, `key = value` lines with `#` comments, keys case-insensitive, 1 to 65,536 bytes:

- `terrainGrid=classic|spline|fine`: how to read a 16-bit file (the size wins when they disagree).
- `terrainHeights=DATA\<stem>_H.PNG`: a greyscale PNG (8 or 16-bit, 256 or 1024 square) used instead of the .RAW.
- `formatVersion`: computed by the writer from the track's contents (2 lights and moving boxes, 3 vegetation, 4 fog volumes, 5 a 16-bit or smoothed terrain), the highest that applies. **It is information, not a gate:** CP4 0.52 shows "This track needs a newer build." for a number above 5 and keeps loading.

**The scale is fixed:** a 16-bit sample `v` means `v / 128` ft, 0 to 511.99 ft. An 8-bit sample `b` counts as `b << 8`. CP4 keeps `v >> 1` (1/64 ft) in a signed 16-bit number, so its internal ceiling is also about 512 ft.

**Other heights:** object positions are text in feet; CP4 lakes (.WTR) store a u32 height in half-feet and refuse values above 4,096 (2,048 ft); ground boxes (.RA0/.RA1) store bottom and top as one byte each, 2 ft a step, 0 to 510 ft.

### 4.3 Grid files

| File | Contents | Bytes per cell | Size today |
|---|---|---|---|
| `DATA\<stem>.CLR` | terrain texture word | 2 | 131,072 |
| `DATA\<stem>.LTE` | baked light | 7 | 458,752 |
| `DATA\<stem>.RA0` to `.RA5` | ground box heights | 1 | 65,536 each |
| `DATA\<stem>.CL0`, `.CL2` | ground box face textures | 12 | 786,432 each |
| `DATA\<stem>.CL1` | | 4 | 262,144 |
| `WORLD\<stem>.WTR` tile map | lake id per tile | 1 | 65,536 |

Already independent of the world's size: object positions, the grid, checkpoints and courses (text feet in the .SIT/.SI2); models (.BIN) and surfaces (.SRF); CP4 waterfalls (.WFL, u32 1/16 ft); vegetation grass tiles (u16); multiplayer truck positions in CP3 and CP4 (32-bit floats in feet, which still resolve 1/500 ft at 16,384 ft); the engine's 32-bit 1/256 ft positions (16,384 ft is 4,194,304 units).

### 4.4 How tracks are found

The retail game builds its track list from every `.SIT` entry in the mounted PODs (`CRace::init`, at most 100). Community Patch 3 introduced `.SI2`, the same text, for tracks that need its engine; a 1998 install does not list them. When a stem has both, CP3-era readers take the `.SI2`.

## 5. The version 6 track

### 5.1 The situation file: `.SI3`

- A version 6 track's situation file is **`WORLD\<stem>.SI3`**. In this version its text is exactly a `.SI2`'s (MTM2 SIT syntax); only the rules in this document differ. Later versions of this specification may extend the `.SI3` text itself.
- A version 6 reader MUST list `.SI3` entries as tracks, with `.SI2` and `.SIT`.
- Older builds list `.SIT` (retail) or `.SIT` and `.SI2` (CP3, CP4 0.52), so they never see a `.SI3` track. (That they skip an unknown extension is INFERRED from how the retail list is built; it is the purpose of the new name.)
- A track MUST use `.SI3` when it uses any key of section 6 other than `formatVersion`. A writer MUST NOT use those keys in a `.SIT` or `.SI2` track.
- A `.SIT` or `.SI2` in the same POD is a different track, read by today's rules. It is not a version of the `.SI3` track for older builds: a quarter of a 512 world, or a squashed mountain, is not the same track. A writer SHOULD give it a different stem.

### 5.2 The .TXV

- A `.SI3` track MUST have `DATA\<stem>.TXV` with `formatVersion=6`.
- Key parsing is CP4's (section 4.2). A version 6 reader MUST refuse to load a `.SI3` track whose keys of section 6 have a value outside the allowed ones, with a message naming the key, rather than guess.
- What a reader does with a `formatVersion` above 6 is up to the reader. Since the `.SI3` format is likely to be updated in later versions, a reader SHOULD at least warn that the track may need a newer build.

## 6. .TXV keys

```
formatVersion=6
worldCells=512
terrainHeightMap=DATA\CHI1_H.PNG
terrainHeightStep=128
terrainHeightBase=0
terrainGrid=fine
```

| Key | Value | Default |
|---|---|---|
| `formatVersion` | `6` | |
| `worldCells` | `256` or `512`. | 256 |
| `terrainHeightMap` | The entry holding the heights: a `.PNG` (greyscale, colour type 0 or 4, 8 or 16-bit, alpha ignored) or a `.RAW` (unsigned 16-bit). | none: CP4's rules (`DATA\<stem>.RAW` or `terrainHeights`), allowed only when `worldCells=256` |
| `terrainHeightStep` | Steps per foot: 1, 2, 4, 8, 16, 32, 64 or 128. | 128 |
| `terrainHeightBase` | Whole feet added to every height, -32768 to 32767. | 0 |
| `terrainGrid` | `classic`, `spline` or `fine`, as in CP4 (section 7.2). | `classic` for an N square file, `fine` for 4N |

With `worldCells=512`, `terrainHeightMap` is required.

## 7. Terrain heights

### 7.1 The height of a sample

```
height_ft = terrainHeightBase + sample / terrainHeightStep
```

An 8-bit PNG sample `b` counts as `sample = b << 8`, as in CP4. With `terrainHeightMap` absent, heights follow CP4's rules (section 4.2) and the step and base do not apply.

**What the step buys:**

| Step | Smallest change | Height range | Good for |
|---|---|---|---|
| 128 | 2.4 mm | 512 ft | Every CP4 track today |
| 64 | 4.8 mm | 1,024 ft | Most hilly tracks |
| 32 | 9.5 mm | 2,048 ft | Evo 2, Pikes Peak |
| 16 | 1.9 cm | 4,096 ft | Mountains |
| 8 | 3.8 cm | 8,192 ft | As tall as a 256 world is wide |

### 7.2 Grid sizes and modes

The height file is N x N (32 ft spacing) or 4N x 4N (8 ft spacing): 256 or 1024 in a 256 world, 512 or 2048 in a 512 world. A RAW `terrainHeightMap` is therefore 2N² or 32N² bytes (131,072 or 2,097,152; 524,288 or 8,388,608). A reader MUST refuse any other size with a message.

| Height file | `classic` | `spline` | `fine` |
|---|---|---|---|
| N square | flat 32 ft triangles | smoothed to 4N square at 8 ft | classic, with a warning |
| 4N square | fine, with a warning | fine, with a warning | 8 ft heights as stored |

Between samples, heights are linear within the cell's two triangles, split in a checkerboard, as in retail (`MTM2_PHYSICS.md` 2.1) and in CP4's fine mode. The spline is CP4's (`terrainspline.c` version 2, the same in Traxx 1.7), with its indices wrapping at N instead of 256; OpenPhotex `terrainSplineExpand` is an exact port with Traxx's test vectors. It works in sample units, before step and base are applied.

### 7.3 Storing heights

A reader MUST hold heights in at least 32-bit integers or floats; CP4 0.52's signed 16-bit store at 1/64 ft cannot exceed 512 ft. For bit-identical results between engines (multiplayer), the RECOMMENDED store is 32-bit integers at 1/128 ft, where every allowed step converts with a shift:

```
h128 = (sample << (7 - log2(terrainHeightStep))) + terrainHeightBase * 128
```

## 8. Larger worlds

### 8.1 What changes in an engine

Every constant of section 4.1 MUST come from N:

| Where | Today | Version 6 |
|---|---|---|
| Cell lookup | `(pos >> 13) & 255` | `(pos >> 13) & (N - 1)` |
| Grid index | `row * 256 + col` | `row * N + col` |
| Fine terrain lookup | `(pos >> 11) & 0x3ff` | `(pos >> 11) & (4N - 1)` |
| Spline wrap | 256 | N |
| Position wrap, every frame | 8,192 ft | W |
| Negative positions at load (objects, ramps, top-crush cars, course ends) | + 8,192 ft | + W |
| Nearest copy for drawing | 4,096 ft | W / 2 |
| View cell for the ground | `(x & 0x1fe000) >> 13` | `(x >> 13) & (N - 1)` |
| Fog volume distance wrap | 8,192 ft | W |

The shifts do not change: a cell is still 32 ft and a fine cell 8 ft. Also unchanged: the view range in cells, the number of cells drawn, the checkerboard split, ground boxes of 32 x 32 ft, the collision code. A larger world is not drawn further; there is just more of it.

**Memory**, in CP4's layout:

| Grid | 256 world | 512 world |
|---|---|---|
| Classic grid, 6 bytes a cell | 384 KiB | 1.5 MiB |
| Fine grid, 16-bit (32-bit as section 7.3) | 2 MiB (4 MiB) | 8 MiB (16 MiB) |
| Two shading maps | 2 x 1 MiB | 2 x 4 MiB |
| `.LTE` | 448 KiB | 1.75 MiB |

A 2048 x 2048 16-bit PNG is 8 MiB before compression, inside CP4's 16 MiB limit on a height PNG.

### 8.2 Grid files

Each grid file keeps its name and layout and holds N x N cells. A reader MUST refuse a file whose size does not match:

| File | Bytes per cell | 256 world | 512 world |
|---|---|---|---|
| `DATA\<stem>.CLR` | 2 | 131,072 | 524,288 |
| `DATA\<stem>.LTE` | 7 | 458,752 | 1,835,008 |
| `DATA\<stem>.RA0` to `.RA5`, 8-bit | 1 | 65,536 | 262,144 |
| `DATA\<stem>.RA0` to `.RA5`, 16-bit (section 8.3) | 2 | 131,072 | 524,288 |
| `DATA\<stem>.CL0`, `.CL2` | 12 | 786,432 | 3,145,728 |
| `DATA\<stem>.CL1` | 4 | 262,144 | 1,048,576 |

No older build reads these files at the wrong size, since they belong to a `.SI3` track it never lists.

### 8.3 Ground boxes

`.RA0` and `.RA1` (and the cavern pair `.RA4`/`.RA5`) MAY be 8-bit or 16-bit, told apart by size:

- 8-bit, as today: `height_ft = terrainHeightBase + 2 * byte`;
- 16-bit: `height_ft = terrainHeightBase + value / terrainHeightStep`, the terrain's rule.

A top of 0 still means "no box". The two files of a pair MUST have the same depth. `.CL0` does not change.

### 8.4 Positions in the .SI3

Object, grid, checkpoint and course positions are in feet, in [0, W). A negative x or z is wrapped by +W at load, as today by +8,192. A writer SHOULD write positions already in [0, W).

## 9. CP4 lakes, vegetation, waterfalls and fog

### 9.1 Lakes (`WORLD\<stem>.WTR`), version 2

Version 1 holds only a 256 x 256 map, u8 bounding boxes and heights from 0.5 to 2,048 ft. Version 2 widens all three. A `.SI3` track MAY use version 1 when N = 256 and every lake fits version 1; otherwise it MUST use version 2. A version 6 reader MUST read both.

| Offset | Field | Version 2 |
|---|---|---|
| +0 | magic `MWTR` | same |
| +4 | u16 major | **2** |
| +6 | u16 minor | 0 |
| +8 | u32 directory offset, at least 0x20 | same |
| +0x0c | u32 chunk count | same |
| +0x10 | u16 width, u16 height | **N, N** |

Directory entries are 16 bytes, {tag, version 1, offset, size}, as in version 1.

- `TMAP`: exactly N² bytes, one lake id per tile (0 none), index `z * N + x`.
- `BODY`: u32 count, u16 stride (at least 0x40), records from +8:

| Offset | Version 1 | Version 2 |
|---|---|---|
| +0 | u8 id, 1..255, unique | same |
| +1 | u8 seed count, 1..8 | same |
| +2 | u16 flags (1 reflective) | same |
| +4 | u32 height, half-feet, 1..4096 | **i32** height, half-feet |
| +8 | 8 x (u16 x, u16 z) seeds | same |
| +0x28 | u8 box x0, z0, x1, z1 | **u16** x0, z0, x1, z1 (tiles, 0..N-1) |
| +0x2c / **+0x30** | u32 tile count, equal to the lake's TMAP count | same field, moved |

The surface is `height * 128` in 1/256 ft. In version 2 a height is valid from `2 * terrainHeightBase + 1` to `2 * (terrainHeightBase + 65536 / terrainHeightStep)`: anywhere in the terrain's range. The other checks (a lake with no tiles, a surface at or below the track's own water, a bed at or below it) are version 1's. CP4 0.52 refuses a major version other than 1, so it never misreads a version 2 file.

### 9.2 Vegetation (`WORLD\<stem>.VEG`)

The layout does not change. `TREE` x and z MUST be in 0 to W * 256 (exclusive) 1/256 ft units, instead of below 0x200000. `GRAS` tile numbers run 0 to N - 1.

### 9.3 Waterfalls (`WORLD\<stem>.WFL`) and fog volumes

Waterfalls do not change. A fog volume's distance to the view is wrapped at W.

## 10. Reader checklist

A version 6 reader MUST:

1. List `.SI3` tracks and prefer them per stem (5.1).
2. Read the keys of section 6 from a `.SI3` track's .TXV, and refuse bad values (5.2).
3. Read heights by 7.1 and 7.2 and store them as 7.3.
4. Take every world constant from N (8.1), and check every grid file's size against N (8.2).
5. Read 8 and 16-bit ground boxes (8.3).
6. Read `.WTR` versions 1 and 2, and accept `.VEG` positions across the world (9).
7. Accept object, lake and fog heights across the whole terrain range.
8. Check anything else tied to height or world size. Our CP4 analysis has not traced:
   - camera far plane and fog distance;
   - sky and sun placement;
   - placed objects' visibility around the view;
   - any overhead map;
   - the AI's search for its course segment;
   - replays and saved games;
   - any value assumed to fit in fewer bits than a position in [0, W).

## 11. Writer guidance

**Height step and base.** Pick the values for the author:

```
terrainHeightBase = floor(lowest point in ft)
terrainHeightStep = the largest power of two, at most 128,
                    with (highest - base) * step <= 65535
```

**World size.** Use `worldCells=512` only when the map does not fit in 7,936 ft (8,192 less a 256 ft margin per side against the wrap) at its true scale; a 256 world needs less memory. Leave at least 256 ft between the map and the world's edge on every side.

**Tools that rewrite a .TXV** (Traxx carries unknown keys over) MUST keep the keys of section 6 untouched.

**Examples:**

- **Pikes Peak:** base 12, span 1,809 ft. Step 32 gives 1,809 x 32 = 57,888, which fits; step 64 would not. Result: 9.5 mm steps, the same as Evo 2, so its heights carry over without loss. It is 256 cells wide, so `worldCells` is left at its default.
- **Chicago at true scale (3.28 ft a metre):** 9,514 ft long, so `worldCells=512`, with 3,400 ft to spare per side; the full 12,530 ft collision extent fits too. Its 27 m of relief (89 ft) keeps step 128.

## 12. Compatibility

| Reader | A `.SI3` track |
|---|---|
| MTM2 retail | not listed |
| Community Patch 3 | not listed |
| Community Patch 4 beta 0.52 | not listed |
| A version 6 reader | listed and loaded |

A `.SIT` or `.SI2` track without the keys of section 6 is not affected by this document.

## 13. Why this design

**A new extension is the only reliable gate.** `formatVersion` is information that no build refuses, so a reused key or a bigger file would load wrong in an older build without warning. CP4 0.52 reads a RAW of unknown size as classic and takes its first 64 KiB, and a 256-cell build cannot reach half of a 512 world. Older builds never list a `.SI3`, so they show nothing rather than something broken.

**It is a normal image.** 16-bit greyscale PNG is what Photoshop, GIMP, Krita, World Machine, Gaea and GIS tools read and write, and CP4 already accepts it. There is no 24-bit greyscale format; packing heights into RGB would give files no editor can paint.

**It is how other engines do it.** Unity and Unreal store terrain as 16-bit heightmaps with a per-terrain height scale.

**The bits go where they are visible.** Below about 1 cm a height step cannot be seen; smoothness comes from interpolation, not extra bits. A step per track spends the same 16 bits on what each track needs.

**Exact numbers in every engine.** Powers of two convert with shifts, so two engines get the same height to the last bit, which multiplayer needs. A free min/max float scale would round differently.

**Same cells, more of them.** Every rule about a cell stays. Engines change constants, not algorithms, and authors keep their sense of scale. 512 keeps the engine's shift-and-mask cell lookup.

**Small change for an engine.** The file layouts, the smoothing and the PNG decoder are CP4's own, at a size given by one number. Only the lake file needs new fields, and its header already has a size.

## 14. Related engine limits

These limits are not part of the format, and this document does not change them. A 512 world invites four times the content, so engines may want to raise them:

| Limit | Today | Note |
|---|---|---|
| Placed objects | 4,608 (CP3's authoring limit 4,096) | Whole-city Chicago needs about 6,060 at 0.83 scale, mostly props |
| CP4 surface boxes | 64 | Large cities merge streets into more surface models. OpenMTM2 has no such limit |
| Checkpoints | 20 a lap | |
| Course | about 250 straights | A lap in a 16,384 ft world can be much longer |
| CP4 collidable models (`.clsh` cache) | 2,048 | |

## 15. Implementation in the OpenMTM2 projects

- **OpenPhotex:**
  - `.SI3` as an MTM2 situation file;
  - the keys of section 6 in `readTxvInfo`;
  - step, base and grid size in the height, `.CLR`, `.LTE` and ground box decoders and writers;
  - 16-bit `.RA0`/`.RA1`;
  - `.WTR` version 2;
  - world size in the MTM2 simulation (`sim/mtm2/world/`).
- **OpenMTM2:**
  - list `.SI3` tracks (`src/worker/catalog.js`);
  - replace `WORLD_FT` (`src/shared/scene-frame.js`) and the other 8,192 ft constants (about 40 files) with the track's world size;
  - lift the 512 ft assumptions.
- **JSTrackViewer:** the same (about 20 files).
- **JSTraxx:** height import and export with automatic step and base, a world size choice, `.SI3` saves.
- **Converters:** mm1conversion builds Chicago at true scale in a 512 world; Evo 2 conversions keep Pikes Peak's full height.

## 16. Open question

What else in an engine assumes 256 cells, 8,192 ft or 512 ft (section 10, item 8)? Findings from any engine are welcome and will be added here.

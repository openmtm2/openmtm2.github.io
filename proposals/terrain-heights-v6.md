# Proposal: tall tracks with fine height steps (.TXV formatVersion 6)

Status: draft for discussion, 2026-11-09
Applies to: MTM2 tracks read by OpenMTM2, and by Community Patch 4 if adopted
Written by: the OpenMTM2 project

## 1. Summary

Today an MTM2 track can only be about 512 ft tall, from its lowest point to its highest. Community Patch 4 made the height steps much finer, but kept the same 512 ft ceiling.

This proposal lets each track choose two numbers, in its .TXV:

- **how fine the height steps are** (`terrainHeightStep`), and
- **where its height range starts** (`terrainHeightBase`).

The heights are still stored in an ordinary 16-bit greyscale PNG (or a raw 16-bit file), exactly like CP4 does today. A track that needs 2,000 ft of climb uses 9.5 mm steps; a track that stays under 512 ft keeps CP4's 2.4 mm steps. Tracks that use this are marked `formatVersion=6`.

## 2. The problem: Pikes Peak

4x4 Evolution 2's Pikes Peak climbs from 12 ft to 1,821 ft: about **1,800 ft** of climb in one track.

| Game | Height step | Tallest track |
|---|---|---|
| MTM2 (original) | 2 ft | 510 ft |
| Community Patch 4 beta 0.52 | 1/128 ft (2.4 mm) in the file | 512 ft |
| 4x4 Evolution 2 | 1/32 ft (9.5 mm) | about 2,048 ft |

Pikes Peak does not fit in MTM2 or CP4. To convert it today you have to either squash the mountain to under a third of its height (the road becomes three times less steep and the jumps and banks change), or cut it into pieces. Neither is the real track.

CP4's 16-bit heights solve a different problem: they make slopes smooth instead of stair-stepped. They give more steps, not more height. Evo 2 shows that both can be had at once: its steps are 4 times coarser than CP4's file, yet its tracks look smooth (the smoothing comes from interpolating between grid points, which CP4's spline mode also does) and are 4 times taller.

## 3. How CP4 defines heights today

Based on Traxx 1.7.0's source and our analysis of the CP4 beta 0.52 executable (2.0.52.2). Fila has not published a written specification; this may change before CP4 is final.

**The height file's size picks the grid:**

| `DATA\<stem>.RAW` size | Samples | Grid spacing |
|---|---|---|
| 65,536 bytes | 8-bit, 256 x 256 | 32 ft (original MTM2) |
| 131,072 bytes | 16-bit, 256 x 256 | 32 ft, or smoothed to 8 ft with `terrainGrid=spline` |
| 2,097,152 bytes | 16-bit, 1024 x 1024 ("Full Map") | 8 ft |

**The .TXV adds two keys:**

- `terrainGrid=spline` or `terrainGrid=fine`: how to read a 16-bit file (the size wins when they disagree).
- `terrainHeights=DATA\<stem>_H.PNG`: a greyscale PNG (8 or 16-bit, 256 or 1024 square) used instead of the .RAW.

**The scale is fixed:** a 16-bit sample `v` means `v / 128` ft, 0 to 511.99 ft. No key changes it. The engine then keeps `v >> 1` (1/64 ft) in a signed 16-bit number, so its internal ceiling is also about 512 ft.

**`formatVersion` is information, not a gate.** Traxx computes it from what the track contains (2 = light and moving boxes, 3 = vegetation, 4 = fog volumes, 5 = a 16-bit or smoothed terrain) and writes the highest that applies. An engine that does not know the number still loads the track. This matters for section 4.3.

**Other heights in a track:**

- Object positions (.SIT/.SI2) are text numbers in feet: no practical limit.
- Lakes (.WTR) store heights as 32-bit numbers in 0.5 ft units; CP4 refuses values above 4,096 units (2,048 ft).
- Ground boxes (.RA0/.RA1) store each box's bottom and top as one byte, 2 ft a step: 0 to 510 ft.

## 4. The proposal

### 4.1 New .TXV keys

```
formatVersion=6
terrainHeightMap=DATA\PIKES_H.PNG
terrainHeightStep=32
terrainHeightBase=12
terrainGrid=spline
```

| Key | Value | Default |
|---|---|---|
| `terrainHeightMap` | The pod entry holding the heights: a `.PNG` (greyscale, 8 or 16-bit, alpha ignored) or a `.RAW` (unsigned 16-bit little-endian). Must not be `DATA\<stem>.RAW`. | none (the track uses CP4's rules) |
| `terrainHeightStep` | Steps per foot: 1, 2, 4, 8, 16, 32, 64 or 128. | 128 |
| `terrainHeightBase` | Whole feet added to every sample, -32768 to 32767. | 0 |
| `terrainGrid` | As in CP4: `classic` (flat triangles between samples), `spline` (smoothed, 256 square only) or `fine` (1024 square, used as stored). | `classic` for 256, `fine` for 1024 |

**Height of a sample, in feet:**

```
height_ft = terrainHeightBase + sample / terrainHeightStep
```

An 8-bit PNG sample `b` counts as `b << 8`, as in CP4.

**Grid sizes:** 256 x 256 (32 ft spacing) or 1024 x 1024 (8 ft spacing), the same two CP4 supports. 512 and 2048 are reserved for a later version.

**What the step buys:**

| Step | Smallest change | Height range | Good for |
|---|---|---|---|
| 128 | 2.4 mm | 512 ft | Every CP4 track today |
| 64 | 4.8 mm | 1,024 ft | Most hilly tracks |
| 32 | 9.5 mm | 2,048 ft | Evo 2, Pikes Peak |
| 16 | 1.9 cm | 4,096 ft | Mountains |
| 8 | 3.8 cm | 8,192 ft | As tall as the map is wide |

### 4.2 Ground boxes

In a formatVersion 6 track, `.RA0` and `.RA1` may be either:

- 65,536 bytes (8-bit, as today): `height_ft = terrainHeightBase + 2 * byte`, or
- 131,072 bytes (16-bit little-endian): `height_ft = terrainHeightBase + value / terrainHeightStep`, the same rule as the terrain.

A top of 0 still means "no box". `.CL0` does not change.

### 4.3 Version and compatibility

- `formatVersion=6` means "this track's heights come from `terrainHeightMap`". Like the earlier numbers it is computed from the track's contents and is the highest that applies. A track that only needs CP4's 512 ft keeps using `terrainHeights=` and version 5.
- **Why a new key name and not `terrainHeights=`:** since `formatVersion` does not stop older engines from loading a track, a CP4 0.52 build would read a reused `terrainHeights=` PNG at 1/128 ft and silently build the wrong mountain. With a new key name, an older build never sees the new heights.
- **Older games:** a version 6 track may also pack a normal `DATA\<stem>.RAW` (`legacyFallback=1`). Older builds load that instead. For a track taller than 512 ft that fallback cannot be the real terrain, so editors should warn the author and default to no fallback.
- Tools that rewrite a .TXV (Traxx carries unknown keys over) keep the new keys untouched.

| Reader | What it does with a version 6 track |
|---|---|
| MTM2, CP3, CP4 0.52 | Ignores the new keys; uses the fallback .RAW if the track packs one |
| A formatVersion 6 reader | Reads `terrainHeightMap` with its step and base |

### 4.4 What a reader must do

1. Hold terrain heights in at least 32-bit integers or floats. CP4's signed 16-bit store at 1/64 ft cannot exceed 512 ft. For bit-identical results between games (multiplayer), we suggest 32-bit integers at 1/128 ft: every allowed step converts with a left shift, `(sample << (7 - log2(step))) + base * 128`.
2. Accept object, lake (.WTR) and fog box heights across the whole terrain range. CP4's .WTR limit of 2,048 ft must rise to cover the tallest track.
3. Check anything else tied to height: camera far plane, fog distance, sky placement, the "fell off the world" floor.

### 4.5 What a writer should do

Pick the values for the author:

```
terrainHeightBase = floor(lowest point in ft)
terrainHeightStep = the largest power of two, at most 128,
                    with (highest - base) * step <= 65535
```

Pikes Peak: base 12, span 1,809 ft. Step 32 gives 1,809 x 32 = 57,888, which fits; step 64 would not. Result: 9.5 mm steps, the same as Evo 2, so its heights carry over without loss.

## 5. Why this approach

**It is a normal image.** 16-bit greyscale PNG is what Photoshop, GIMP, Krita, World Machine, Gaea and GIS tools read and write, and CP4 already accepts it. There is no 24-bit greyscale image format; a 24-bit design would need heights packed into RGB colours that no editor can paint.

**It is how other engines do it.** Unity and Unreal store terrain as 16-bit heightmaps with a per-terrain height scale. Authors already know the workflow.

**The bits go where they are visible.** Below about 1 cm a height step cannot be seen; smoothness comes from interpolation (CP4's spline, Evo's sub-grid), not from extra bits. A fixed scale wastes precision on short tracks or range on tall ones. A step per track spends the same 16 bits on what each track needs.

**Exact numbers in every game.** Powers of two convert with a bit shift. Two different engines get the same height to the last bit, which multiplayer needs: every machine must agree on where the ground is. A free "min/max" float scale (as Unity uses) would round differently in different engines.

**Small change for CP4.** The file formats, sizes, smoothing and PNG decoder are CP4's own. CP4 needs to read two numbers and widen its height store. Nothing about existing tracks changes.

**Older builds are safe.** A new key name means old builds never misread the new heights.

**Alternatives we considered:**

| Option | Why not |
|---|---|
| 24-bit heights | No greyscale image format; files 50% larger; precision no one can see |
| 32-bit float (TIFF, EXR) | Poor editor and browser support; floats invite rounding differences between engines |
| A fixed bigger scale for everyone | Every track pays for Pikes Peak's range; short tracks lose precision |
| Reusing `terrainHeights=` with a step key | Older builds would load it with the wrong scale and give no warning |

## 6. Plan in our projects

- **OpenPhotex:** read the new keys in `readTxvInfo`, apply step and base in the height decoders, read 16-bit .RA0/.RA1.
- **OpenMTM2:** already works in decimal feet; load the new keys and lift the 512 ft assumptions.
- **JSTrackViewer:** same as OpenMTM2.
- **JSTraxx:** height import and export with automatic step and base, version 6 saves, fallback warning.

## 7. Questions for CP4 team

1. Should a reader that knows version 6 refuse a track with a newer number, or keep loading as today?
2. Are 512 and 2048 square grids worth adding at the same time?
3. Does the 2,048 ft lake limit or anything else in the engine assume a 512 ft world?

# SixNG

SixNG is an asset extractor for **GTA VI on Xbox Series**. It reads the
game's RPF9 archives from your own copy of the game and writes open formats:

- **glTF 2.0 (`.glb`)** for models, collision and navmesh;
- **PNG and DDS** for textures;
- **JSON / JSONL** (plus two small binary instance files) for map placements,
  lights, the road graph, interiors and metadata.

It is written in Rust. There is one core library with two front ends: the
`sixng` CLI (JSON on stdout, resumable runs) and the `sixng-gui` desktop app
(browser, model viewer, 2D map).

SixNG was built to make **map and car mods** for BeamNG.drive, Assetto Corsa
(with Custom Shaders Patch) and an original Rust/Bevy game. The output is
meant as a neutral intermediate format that converters for those targets
read.

> SixNG contains no game files. It only works on a copy of the game you own
> and have dumped yourself. Read [LEGAL.md](LEGAL.md) before you use it or
> share anything made with it.

## What it extracts

Coverage numbers are **as of 2026-11-20**, measured on one retail Xbox Series
install. Other builds of the game may differ.

| Area | State | Numbers |
|------|-------|---------|
| Archive listing | **works** (by carving; the archive TOC is encrypted) | `levels.rpf`: 523,552 entries, 98.3 % of payload bytes covered, 227,766 with recovered names |
| Texture dictionaries | **works**: BC1–BC7 untiled to PNG and DDS | RGBA8 tiling unverified |
| Texture stores (`ttpf`) | **works** for zlib chunks | 84.6 % of 352,461 textures fully decode (vehicles 96.7 %, maps 91.8 %); 4.1 % more at a lower mip; 8.3 % are BCPack-only |
| Models (drawables, fragments) | **works**: triangle meshes, normals, tangents, UV0–UV3, LODs, skeleton with pivots, skinning | materials partial: base colour and normal map linked where they decode; names are hashes |
| Vehicles | **partial**: body, skeleton, skin, LODs, damage and mod-kit parts | paint colours, wheel sizes and handling not decoded |
| Map placements | **works**: position, rotation, scale, flags, LOD hierarchy | 2,249,490 entities in 30,785 map files, 95 % resolved to a model |
| Instanced batches (grass, props) | **works** | 906,922 batches: 24.9 M prop and 219.5 M grass instances |
| Interiors (MLO) | **works**: world transform, entity lists, linked collision | 2,590 instances; 97.8 % of interior archetypes linked to collision |
| Collision | **works** (geometry, split by surface id) | ~87 K bounds files; surface names unresolved, inferred categories cover 98.3 % of collision area |
| Navmesh | **works**, road polygons flagged | 27,644 tiles of 128 m |
| Road graph | **works**: lanes, speed limits, node and link flags, junctions, traffic-light phases | 63,808 nodes, 145,518 links, 1,546 junctions (369 signalised) |
| Lights | **works**: position, colour, direction, type, intensity, range, cones, hour mask | 774,847 placed lights |
| XML / text | **works**, exported as is | |
| PSO / RBF metadata | **raw only** | field names are 64-bit hashes |

## What is not solved

- **BCPack** (the Xbox Series hardware texture codec). The coder and the
  stream layout are partly mapped, but there is no decoder. Textures whose
  every mip is BCPack get a flat fallback colour. See
  [BCPack](formats/06-bcpack.md).
- **Name hashes** for shaders, shader parameters and collision surfaces use
  an unknown hash function, so materials and surfaces are named by hash.
- **The RPF9 table of contents** is encrypted, so there are no original
  file paths. Names come from strings inside the resources.
- **Vehicle data**: paint colour tables, wheel size and rim choice,
  handling. The candidate tables are PSO with hashed field names.
- **Smaller gaps**: 4- and 2-byte texture formats (about 1 %), occluder and
  water-vegetation LOD buffers, emissive and glass flags beyond the
  structural classes, animation clips, audio (planned), pit and start
  positions.

## Pages

| Page | What it covers |
|------|----------------|
| [formats/](formats/README.md) | Format specification, one page per format, with a confidence level per field and open questions per page |
| [tool/](tool/README.md) | Building, every CLI command, the GUI, the output layout and file schemas, one page each |
| [LEGAL.md](LEGAL.md) | What the tool is and is not, and what you may share |

## Conventions used in these docs

- **Offsets** are hexadecimal, from the start of the structure, little-endian
  unless stated.
- **Hashes** are shown as 8 (32-bit) or 16 (64-bit) hex digits. Where a field
  or enum name is known, it is the case-sensitive joaat ("one-at-a-time")
  hash of that name.
- **Coordinates**: the game is RAGE Z-up, right-handed, metres. GLB output is
  glTF Y-up: `glb(x, y, z) = rage(x, z, −y)`.
- **Entry ids** are `<archive>@<hex offset>`, e.g. `levels.rpf@10f2d2d010`.
  They are stable for one install, but may differ on another build.
- **Confidence**: **confirmed** (holds over many samples and decoded output
  checked by eye), **inferred** (consistent with the data, not proven),
  **unknown**.

## Licence

Licensed under GPL-3.0; see [LICENSE](LICENSE). This covers SixNG's code
and these docs. It does not cover anything extracted from the game
(see [LEGAL.md](LEGAL.md)).

## Status of these docs

This is the first version of the community docs, dated 2026-11-20. They
describe the tool and formats at that date. Numbers will change as decoding
improves; each page says when its numbers were measured.

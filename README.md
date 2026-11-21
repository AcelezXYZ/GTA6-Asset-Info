# GTA6 Asset Info

Technical notes on the asset formats of **GTA VI (Xbox Series build)**:
how the archives, textures, models and map data are laid out, what is
understood, and what is still unknown.

Everything here was worked out from the data itself: headers, statistics
over many samples, and checking decoded output by eye. Nothing has been
confirmed against game code or official documentation. This repository holds
documentation only: no game files, no extracted assets, no keys. Read
[LEGAL.md](LEGAL.md).

## Formats

| Area | Pages |
|------|-------|
| Containers | [RPF9 archives](formats/01-rpf9.md) · [Resources and pages](formats/02-resources.md) |
| Textures | [Texture dictionaries](formats/03-texture-dictionaries.md) · [Xbox tiling](formats/04-xbox-tiling.md) · [`ttpf` stores](formats/05-ttpf.md) · [BCPack](formats/06-bcpack.md) |
| Models | [Drawables, fragments and vehicles](formats/07-models.md) |
| Map data | [Meta resources](formats/08-meta.md) · [Placements and batches](formats/09-placements.md) · [Interiors](formats/10-interiors.md) |
| World | [Collision](formats/13-collision.md) · [Navmesh](formats/14-navmesh.md) · [Lights](formats/15-lights.md) · [Road graph](formats/16-roads.md) |
| Materials | [Vehicle paint and tables](formats/11-vehicle-paint.md) · [Material classes](formats/12-materials.md) |
| Other | [PSO, RBF, BVH4, audio, animation, video, UI](formats/17-other.md) |
| Byte-level notes | [Index](structure/00-INDEX.md) · [RPF9 container](structure/01-rpf9-container.md) · [Resources and textures](structure/02-resources-textures.md) · [Audio (`ADAT`)](structure/03-audio-awc.md) · [PSO and object graph](structure/04-pso-objectgraph.md) · [`ttpf` stores](structure/05-ttpf-stores.md) · [Models and shaders](structure/06-models-shaders.md) |

The [format index](formats/README.md) has the conventions. Every page gives
each field a confidence level and ends with its open questions. The
[structure/](structure/00-INDEX.md) pages give the same formats as raw byte
facts, each with the test behind it.

## State of knowledge

As of **2026-11-21**, from one retail Xbox Series install. Other builds may
differ.

| Area | Understood | Not yet |
|------|-----------|---------|
| RPF9 archives | header layout; payload is plain zlib | the table of contents is encrypted, so there are no file names or paths |
| Resources | system/graphics pages, pointers, names inside resources | some unnamed resources |
| Textures | dictionary textures; Xbox tiling for BC formats; `ttpf` stores with AMD Gfx10 layouts (84.6 % of 352 K store textures readable) | **BCPack** (8.3 % of textures have no readable mip); 4-/2-byte layouts |
| Models | vertex streams, meshlet index buffers, LODs, skeletons, skinning (incl. u16 bone indices), shaders' texture references; shader names (all) and most parameter names | texture names (64-bit hash), constant buffers |
| Vehicles | parts, damage and mod-kit parts, wheel mounts; model info, spawn configurations (body colour, rim, tyre), weighted paint-colour lists, handling records | how the wheel scale sizes the shared wheels, secondary colours, liveries, many handling fields |
| Map data | self-describing meta resources; entities, LOD hierarchy and LOD parents (99 %), archetypes, grass and prop batches, interiors with rooms and entity sets | prefab placement, some hashed fields |
| Collision | bounds geometry and primitives, surface ids; inferred surface categories for 98.3 % of area | surface names (unknown hash) |
| Navmesh | tiles, polygons, road flag | most other flags |
| Lights | position, colour, type, intensity, range, cones, hour mask | a few fields |
| Road graph | nodes, links, lanes, speed limits, 34 of 51 node flags, junctions, traffic-light phases | some link flags, street names |
| Video | 515 Bink 2 files, stored uncompressed and carved by header | frame decoding (needs a Bink 2 decoder), original names |
| UI graphics | 75 named dictionaries (loading screens, HUD, gauges); the UI texture store (10,013 sprite atlases, 57 % complete) | BCPack parts, menu layouts, a minimap tile set (none found) |
| Audio | `ADAT` container, chunk tags, Opus framing, codec ids (PCM16, float32 PCM, XMA2, Opus) | a few header fields |
| PSO data | big-endian section headers, little-endian block table, the `LSCH` schema (joaat field names); 2,225 of 2,281 files in `data.rpf` decode; one serialised object graph (30,683 records) | 56 files in the older layout, enums, most struct names |
| Shaders | shader cache layout, shader names | decoding |

## Biggest open problems

- **BCPack**, the Xbox Series hardware texture codec. The rANS coder and the
  stream layout are partly mapped; there is no decoder.
  [Details and how to resume](formats/06-bcpack.md#64-how-to-resume).
- **Collision surface names**: the surface id hash is unknown. (Shader and
  parameter names are solved: joaat of the lower-case name.)
- **The RPF9 TOC** is encrypted.
- **Vehicle data**: colours, wheel choice and handling are decoded, but how
  the wheel scale sizes the shared wheel models, the secondary colours and
  the liveries are not.

## Conventions

- Offsets are hexadecimal, little-endian unless stated.
- Hashes are hex. Where a field or enum name is known, it is the
  case-sensitive joaat ("one-at-a-time") hash of that name.
- The game is RAGE Z-up, right-handed, metres. glTF output conventions are
  Y-up: `gltf(x, y, z) = rage(x, z, −y)`.
- Confidence: **confirmed** (holds over many samples, output checked by
  eye), **inferred** (consistent with the data, not proven), **unknown**.

## Tool notes

The [tool/](tool/README.md) folder documents the extractor these notes were
worked out with: its commands and the files it writes.

## Licence

GPL-3.0; see [LICENSE](LICENSE). This covers these notes only, not anything
from the game.

# 17. Other formats

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

| Format | State |
|---|---|
| PSO (`PSIN`; sections LSCH/LENM/LSNS/LGID/LRSC/CHKS) | **decoded** (2026-11-21): the plain-file form of the meta format of [§8](08-meta.md). Section headers are big-endian; the block table and references are little-endian; the `LSCH` schema names fields with 32-bit joaat (struct names are 64-bit). 2,225 of 2,281 files in `data.rpf` decode. See [PSO](#pso) and [Vehicle handling](#vehicle-handling) |
| RBF (`RBF0`) | a new variant with u16 indices and inline names; exported raw |
| `core_bvh4.rpf` | BVH4 collision with packed 24-bit indices; not decoded |
| XML, text | plain; exported as is |
| shader cache (`shaders.rpf`) | one compiled cache, no names; layout notes in [structure/06](../structure/06-models-shaders.md) |
| audio (AWC/ADAT) | container layout, chunk tags and Opus framing in [structure/03](../structure/03-audio-awc.md); not exported by the tool |
| animation (`anim.rpf`) | clip dictionaries and cutscene metadata; clip metadata decodes (confirmed); skeleton not in the clip dictionaries. See [Animation](#animation) |
| music catalogue (`audio:/config/music.dat`) | title, artist and playlist records; title hash solved (confirmed). See [Music catalogue](#music-catalogue) |
| PSIN scene placements (`story000_script.rpf`) | 97,504 64-byte placement records (confirmed). See [PSIN scene placements](#psin-scene-placements) |
| movies (`movies.rpf`) | Bink 2: 515 movies, 505 `KB2n` and 10 `KB2j`, stored uncompressed; header, frame index and Bink Audio tracks parse (confirmed); video needs a Bink 2 decoder. See [Video](#video) |
| UI graphics | named texture dictionaries in `data.rpf` and UI texture stores. See [UI graphics](#ui-graphics) |
| text (GXT) | `0DRP` type `0x7a` key/text resources; 54,118 keys (confirmed). See [Text (GXT)](#text-gxt) |

## PSO

PSO files are the plain-file form of the self-describing meta format
([§8](08-meta.md)). Each section is a 4-byte magic and a **big-endian** u32
size that includes the 8-byte header
([structure/04](../structure/04-pso-objectgraph.md)). Everything inside is
little-endian.

| Section | Contents | Confidence |
|---|---|---|
| `PSIN` | instance data: a block table from +8, entries {u32 struct hash (low 32 bits of the 64-bit struct name), u32 length, 8 × `0xAA`}, then the blocks, each 16-byte aligned. References are u64 `(offset << 20) \| block` with a 1-based block | confirmed |
| `LSCH` | schema: 32-byte struct entries {u64 name hash, u32 info, u32 0, 8 × `0xAA`, u32 size, u16 0, u16 member count}, then all members in table order as 16-byte records **identical to §8**: {u32 name, u32 offset, u8 type, u8 sub, u16 reference, u32 extra} | confirmed |
| `LRSC` | a copy of the meta resource header (`0DRP`, type 0x7a): +0x38 u16 struct count, +0x3A u16 root block, +0x3C u16 block count | confirmed |
| `LENM` | enums, 64-bit names | not decoded |
| `LSNS`, `LGID`, `CHKS` | — | not decoded |

- **Names**: struct names are 64-bit hashes (function unknown). Member
  names are 32-bit **case-sensitive joaat**, and GTA V's names match
  (`fMass`, `fInitialDragCoeff`, `fGearRatios`, `fSteeringLock`). A struct
  member's `extra` is the low 32 bits of the struct name (confirmed).
- **info**: 0x200 / 0x300 / 0x400 = alignment 4 / 8 / 16; low byte 0x0C or
  0x24 marks generated helper types such as hash-table storage, named
  `<hash32>_000001xx` (inferred).
- **Pointers** are always pool references. There are no `0x5xxxxxxx`
  resource pointers; a reference whose offset is 0x500–0x5FF looks like
  one, so do not test for them (confirmed).
- In `data.rpf`, 2,225 of 2,281 PSO files decode this way. The other 56 use
  the older GTA V layout (`PSIN` / `PMAP` / `PSCH`) and are not decoded.

Type codes not in [§8](08-meta.md):

| Code | Type | Confidence |
|---|---|---|
| 0x01 | bool (1 B) | confirmed |
| 0x4d | 64-bit hash or id (8 B) | confirmed |
| 0x50 | inline fixed array: `extra` elements of member `reference` | confirmed |
| 0x75 | map {u32 count, u32 capacity, u64 ref} to a hash-table struct: control bytes `b8794c89` (u8 × N; 0x80 = empty, else 7 hash bits, as in a Swiss table) and slots `93c495af` {`6098a50e` key, `063fa3f2` value} | confirmed |
| 0x62 | tagged value (16 B: u32 tag, then the payload) | inferred |
| 0x31 | 4 floats | inferred |

In PSO, `0x65` bit sets take 16 bytes (the first u64 is used) and `0x44`
takes 16 bytes, probably a string reference into `LSNS` (not decoded).

## Vehicle handling

One PSO in `data.rpf` holds the handling data: 1,097 records of struct
`1ddb46e190779a14` (confirmed).

| Field | Meaning |
|---|---|
| `93db5464` | joaat of the lower-case handling name (`phoenix`); 301 match model names |
| `49a01a86` | a second name hash, often 0 |
| `41c8dec7` | array of polymorphic components, one struct type per component |

| Component | Contents (inferred from the field names) |
|---|---|
| `b4e0f8b0` | body: `fMass`, `fInitialDragCoeff`, 3-vectors |
| `6fdd2f18` | transmission: `fGearRatios`, `fFinalDriveRatio`, `fDriveBias*` |
| `88d3a1a3` | engine: RPM limits (e.g. 5,800), a torque curve as a map of {rpm, value} records, `fOilVolume` |
| `70317f27` | fuel |
| `edf48412` | wheels, steering and brakes: `fSteeringLock`, `fBrakeForce`, `fWheelMass` |
| `bdaf2dc0` | suspension: travel, damping |
| `c746f5ed`, `82173b9e`, `0da11ed9` | unknown |

Mass, drag, gear ratios, final drive, brakes, steering lock, wheel mass,
suspension travel and damping, anti-roll bar force and fuel are named; most
other fields, the torque curve's included, are decoded values with hashed
names. The record's root block is a small map of two keys to 64-bit ids.

Another PSO holds lights and sirens (`Lights`, `Sirens`: cone angles,
intensity, flash timings), like GTA V's `carcols`. The paint colours are in
the game-data meta resource instead ([§11](11-vehicle-paint.md#vehicle-tables)).

## Video

`movies.rpf` has a plain payload (only the table of contents is encrypted,
as everywhere), and its videos are **stored uncompressed**, so a search for
zlib streams finds nothing in it (confirmed).

Each video is a standard Bink 2 file starting on a 4 KiB boundary; the next
starts at the first 4 KiB boundary after the previous one ends. That gives
515 files, the entry count in the archive header, covering 8,831,336,892 of
the archive's 8,832,481,776 bytes: 505 `KB2n` and 10 `KB2j`, about 236
minutes, all at 30 fps (confirmed).

Header (the public Bink container layout; all u32):

| Off | Field |
|---|---|
| 0 | signature (`KB2n`, `KB2j`) |
| 4 | file size − 8 |
| 8 | frame count |
| 12 | largest frame size |
| 16 | frame count again |
| 20, 24 | width, height |
| 28, 32 | fps numerator, denominator |
| 36 | video flags (`0x40013` typical) |
| 40 | audio track count n |
| 44 | Bink 2 only: one more u32 (`0005e987` seen) |
| then | n × u32 largest decoded audio size, n × (u16 sample rate, u16 flags; `0x7000` on 48 kHz stereo), n × u32 track id |
| then | (frame count + 1) × u32 frame offsets: bit 0 = keyframe, the last one is the file size |

Every frame table checks out against the file size.

| Resolution | Files |
|---|---|
| 1280 × 720 | 246 |
| 1920 × 1080 | 130 |
| 720 × 1280 (portrait) | 67 |
| 512 × 512 | 23 |
| 1024 × 1024 | 19 |
| 2560 × 1440 | 14 |
| 512 × 256, 2048 × 558 | 2 each |

Audio: 319 files have none, 165 one track, 30 two, 1 four; all 48 kHz
stereo. The longest video runs 534 s. Original file names are not
recovered.

## UI graphics

- **Named texture dictionaries in `data.rpf`** (75): 36 loading screens,
  `hud`, `blip`, `map_scripted_movie`, `generic_movie`, `credits`, `phone`,
  `controller_test`, `analog_instruments`, `generic_instruments` and 25
  `vit_*` vehicle gauge sets (dashboards, motorcycle displays). Their 880
  textures decode with the dictionary rules of [§3](03-texture-dictionaries.md)
  and [§4](04-xbox-tiling.md); texture names are placeholders (`hud_i8d3`).
- **UI texture store** `data/ttdb/ui/ttdb_ui.ttpf` ([§5](05-ttpf.md)): 10,013
  textures, one mip each, named only by 64-bit hash. Mostly sprite atlases:
  shop and brand logos, tattoo art, hair, beard and wheel icons, character
  portraits, in-game arcade and website art. Formats: BC1 3,801, BC7 3,383
  plus 1,828 sRGB, BC3 928, BC4 57, A8 16. 5,751 decode completely, 1,144
  partly (some 64 KiB chunks are BCPack: the top of the image decodes, the
  rest is blank) and 3,118 lie only in BCPack chunks.
- **Fonts and HUD shapes**: `data/ttdb/graphics` holds single-channel font
  glyph atlases and HUD shapes.
- **Minimap / pause map**: no tile set found in these stores or
  dictionaries. The map may be drawn from world data at runtime (not
  established). Menu layouts (which sprite goes where) are not decoded.

## Animation

`anim.rpf` holds two families. Bodies in the low region of the archive are
`0DRP` v122 cutscene/scenario metadata; above them, bodies whose
`+0x10..+0x3F` is all zero are clip dictionaries (inferred from a sample of
seven windows).

Clip names, durations and sequence channels decode (confirmed). Sampled rotation
tracks validate as unit quaternions: 3.95 M values, worst |q| − 1 about 1.2e-7
(confirmed). The skeleton is not stored in the clip dictionaries.

## Music catalogue

`music.dat` is compiled parsable data: a root count, then the magic u32
`0x045A88EA` at `0x08`, then type-tagged records with `0xAAAAAAAA` as the null
sentinel (confirmed).

A title record (tag `0xAAAAAA0C`) has a u32 title hash at `+0x08`, a char[64]
title at `+0x0C`, a u32 credit count at `+0x60`, then 8-byte credits
(confirmed).

The title hash is joaat of the **lowercased** title with seed `0xC3CCF6AD`
(804/804 titles, confirmed). AWC stream ids use unseeded joaat of the title, so
the two never coincide.

## PSIN scene placements

64-byte records at a 16-byte stride inside PSIN tag blocks: 97,504 records in
2,017 documents of `story000_script.rpf` (confirmed).

| Off | Meaning | Confidence |
|---|---|---|
| 0x00 | u32 handle, repeated at `0x30` | confirmed |
| 0x10 | f32 × 3 position, Z-up metres; x repeated at `0x1C` | confirmed |
| 0x20 | f32 × 3 rotation, radians; x repeated at `0x2C` | inferred (meaning) |
| 0x34 | u32 archetype = joaat(lowercase name) | confirmed |
| 0x38 | u64 id; zero in about half the records | inferred (meaning) |

These are a different family from the map-metadata entities in [§9](09-placements.md).

## Text (GXT)

GXT resources are `0DRP` type `0x7a`. A tag table points at a `0x101` key
table, `0xA8478889` locators, `0x956A9DF5` / `0x10075946` locale-variant groups
and items, and `0x10` text pages (confirmed). Keys are joaat, stored in 4,096
or 2,048 buckets (confirmed).

---

← [16. Road graph, junctions and flags](16-roads.md) · [Index](README.md)

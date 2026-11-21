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
| movies (`movies.rpf`) | Bink 2: 515 movies, 505 `KB2n` and 10 `KB2j`; header, frame index and Bink Audio tracks parse (confirmed); video needs a Bink 2 decoder |
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

# 17. Other formats

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

| Format | State |
|---|---|
| PSO (`PSIN`; sections LSCH/LENM/LSNS/LGID/LRSC/CHKS) | **decoded** (2026-11-21): the plain-file form of the meta format of [§8](08-meta.md). Section headers are big-endian; the block table and references are little-endian; the `LSCH` schema names fields with 32-bit joaat (struct names are 64-bit). 2,225 of 2,281 files in `data.rpf` decode. See [PSO](#pso) and [Vehicle handling](#vehicle-handling) |
| RBF (`RBF0`) | a new variant with u16 indices and inline names; exported raw |
| `*_bvh4.rpf` | dehydrated GPU acceleration structures and a mount manifest. See [GPU BVH4](#gpu-bvh4) |
| XML, text | plain; exported as is |
| shaders (`shaders.rpf`) | compiled cache and DXBC-wrapped native shader programs. See [structure/06](../structure/06-models-shaders.md) |
| audio (AWC/ADAT) | container layout, chunk tags and Opus framing in [structure/03](../structure/03-audio-awc.md); radio stations and scheduling in [§18](18-radio.md); not exported by the tool |
| animation (`anim.rpf`) | clip dictionaries and cutscene metadata; clip metadata decodes (confirmed); skeleton not in the clip dictionaries. See [Animation](#animation) |
| ped body shape | presets in `common.rpf`, a control-to-input PSO and pose rigs in `anim.rpf`; the rig program runs (confirmed). See [Ped body shape](#ped-body-shape) |
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
| `PSIN` | instance data: a block table followed by 16-byte-aligned blocks. See [the v22 and unversioned script layouts](../structure/04-pso-objectgraph.md#script-psin-headers). References are u64 `(offset << 20) \| block` with a 1-based block | confirmed |
| `LSCH` | schema: 32-byte struct entries {u64 name hash, u32 info, u32 0, 8 × `0xAA`, u32 size, u16 0, u16 member count}, then all members in table order as 16-byte records **identical to §8**: {u32 name, u32 offset, u8 type, u8 sub, u16 reference, u32 extra} | confirmed |
| `LRSC` | a copy of the meta resource header (`0DRP`, type 0x7a): +0x38 u16 struct count, +0x3A u16 root block, +0x3C u16 block count | confirmed |
| `LENM` | enums, 64-bit names | not decoded |
| `LSNS` | [table framing](../structure/04-pso-objectgraph.md#lsns-framing) and 16-byte records in the script bundles | confirmed framing; record contents unresolved |
| `LGID`, `CHKS` | — | not decoded |

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

## GPU BVH4

The sampled `core_bvh4.rpf` and shared minigame BVH4 entries carry
dehydrated GPU ray-tracing acceleration structures. CPU collision
bounds have their own [phBound layout](13-collision.md).

After inflating an entry, the 0xD0-byte system block gives the
graphics size as a u32 at `+0xA0`. The graphics block has a
0x40-byte header at `+0x100`. The u32 at graphics `+0x104` counts
64-byte units. The size check is
`graphics_size == 0x140 + 64 * units`. This held in both sampled
archive families. Observed hierarchy words carry a 24-bit value in
bits 0–23 and a child count of 0–4 in bits 24–31.

`story000_bvh4.rpf` contains four zlib-compressed XML records.
Its content and setup records reference terrain and merged BVH packs
for mounting. Joining archived BVH4s back to meshes and rehydrating
the console representation remain open.

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
stereo. The longest video runs 534 s. Original file names are not stored
with the files. Matching them in order against a plain-text content list
is reported to recover names for some groups (unverified: the list is
not identified).

### Bink Audio framing

At the start of each indexed frame, read one `u32 packet_size` and
its payload per audio track, in track order. A nonempty payload begins
with a `u32 decoded_byte_count` followed by the encoded audio. Bound
every packet by the next frame offset or the declared file size.

The encoded audio after `decoded_byte_count` is Bink Audio. The sum of a
track's `decoded_byte_count` values is its 16-bit PCM size, so samples per
channel = sum / (2 × channels); the track durations computed this way
match the video length within one frame across the audio-bearing files.

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
  plus 1,828 sRGB, BC3 928, BC4 57, A8 16. With BC1 BCPack decoded
  ([§6](06-bcpack.md)), 9,012 decode completely, 345 partly (some 64 KiB
  chunks are BCPack in another format: the top of the image decodes, the
  rest is blank) and 656 lie only in such chunks, mostly BC7.
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
(confirmed). A unit norm alone does not validate the decoded pose. The
skeleton is not stored in the clip dictionaries.

### Dictionary clip index

In the sampled Xbox Series 1.0.2.1 dictionaries, body `+0x38` points to
the dictionary name and `+0x68` stores its lowercase joaat. Body `+0x48`
points to `u64` bucket heads, with `u32` bucket count at `+0x50` and
`u32` total entry count at `+0x58`. A zero pointer marks an empty bucket
or the end of a chain.

| Entry offset | Type | Meaning |
|---|---|---|
| `+0x00` | `u32` | Clip stem hash, matching clip record `+0x48` |
| `+0x04` | `u32` | Clip set hash, matching clip record `+0x4c` |
| `+0x08` | ptr | Points 0x10 bytes before the clip record base used below |
| `+0x10` | ptr | Next entry in the same bucket |

Walking all bucket chains resolved all 1,784 entries in 194 dictionaries,
including four without a readable clip name. The bucket selection formula
is unknown. For named clips, the stem hash is the lowercase joaat of the
last path component without `.clip`. The set hash is the lowercase joaat
of the directory without `pack:/` (130 records), or the dictionary name
hash for flat `pack:/<stem>.clip` paths (1,650 records).

### Scene clip references

In the sampled `0DRP` v122 class registry, each entry's `u32` at
`+0x18` is the **instance stride**, not descriptor byte length. Read
the `u16` descriptor count at `+0x1e`, then that many 16-byte descriptors
from the `+0x10` pointer. In the verified scene object graph, a packed
reference stores a one-based block index in the low 20 bits and a byte
offset within that block above them.

The scene at `anim.rpf@0x94f010` reaches an inline class `0x24749343`
through section/event references. Its descriptors identify
`clipDictionaryName` (`0x25de16df`) at `+0x08` and `clipName`
(`0x24fc8f24`) at `+0x0c`, both type-`0x4a` `u32` values. The pair
`0x65d46c10`/`0x9302c678` selects dictionary `anim.rpf@0x272d5010` and
clip `+0xdba60` (`pack:/jason_buddy_kiss_jason.clip`). That clip's source
window uses source `+0xdc830`, sequence `+0xa7390`, from 4.5 to 7.9
source seconds at rate 1. Dictionary `0x26eeb010` contains the same clip
name and stem hash under a different dictionary hash (`0x7cb407d0`), so
a name or stem match alone selects the wrong copy. This is one rooted
reference path, not a full scene or gameplay-state parser.

### Clip windows and source tracks

For the sampled type-2 clips, `+0x18` points to the
name, `+0x68` is the `f32` clip duration, and `+0x58` points to an array
of 0x18-byte wrappers counted by the `u16` at `+0x60`. Each wrapper has a
window array pointer at `+0x00`, `u16` window count at `+0x08` and `f32`
duration at `+0x10`. Window stride is 0x38:

| Offset | Type | Meaning |
|---|---|---|
| `+0x00` | ptr | Source animation record |
| `+0x08` | `u32` | Lowercase joaat of the source name without `pack:/` or `.anim` |
| `+0x28` | `f32` | Start time in the source, seconds |
| `+0x2c` | `f32` | End time in the source, seconds |
| `+0x30` | `f32` | Source seconds per clip second |

Sample each window at `start + clip_time * rate`. Different windows can
contribute tracks over the same clip time; do not concatenate them. In
836 clips across seven dictionaries, all 2,502 source pointers and hashes
resolved, and `(end - start) / rate` agreed with clip duration within
0.000013 s. An embedded `pack:/...clip` string alone is not proof of a clip
record.

For the sampled source record layout with a 0x20-byte prefix, offsets
from the **record start** are:

| Offset | Type | Meaning |
|---|---|---|
| `+0x24` | `u16` | Source frame count |
| `+0x26` | `u16` | Frame stride between sequence starts in multi-sequence sources |
| `+0x28` | `f32` | Source duration, seconds |
| `+0x30` | ptr | Source name |
| `+0x50` / `+0x58` | ptr / `u16` | Sequence pointer array / count |
| `+0x60` / `+0x68` | ptr / `u16` | Track row array / count |

The track stride is **12 bytes**: `{u32 target_hash, u32 property_hash,
u32 kind}` (233 sampled tables). For skeletal tracks, the target is the
full lowercase joaat of the bone name, not the 16-bit tag. It matches the
`u32` at `+0x1c` in the [ped bone record](07-models.md#75-skeleton-fragments-confirmed).
Verified on the `ambient_malebody` and `ambient_femalebody` fragments at
`packs.rpf@0x454db010` and `packs.rpf@0x45404010` respectively.
Property `0x98e1750c`, kind `0`, is an XYZW local rotation;
`0xf791a95f`, kind `2`, is an XYZ local translation. Other track
properties can target controls rather than indexed bones.

Sequence `i` starts at source frame `i * frame_stride` (zero-based).
Each non-final sequence stores `frame_stride + 1` samples: its last
sample and the next sequence's first sample share a source time. All
14,133 multi-sequence sources with validated headers satisfied
`sum(sequence_frame_counts) - (sequence_count - 1) = source_frame_count`.
All 228 non-final sequence headers checked against the stride field
matched. Across 44 decoded joins, scalar and vector samples agreed
closely; the type-8 handling below makes all sampled shared-frame
quaternion tracks agree.

### Sequence channels

The 0x20-byte sequence header gives the payload size at `+0x04`, the
payload-relative frame-data offset at `+0x0C`, and the header-relative
root-motion offset at `+0x10` (all u32). The frame count and bytes per
frame are u16 values at `+0x16` and `+0x18`. In all 307 cached
sequences, `+0x14` was 2 and `+0x1E` was 0x0b; their meanings are open.
The `u16` at `+0x1A` is the sum of `(table_words + 5)` over the type-5
channels, and `+0x1C` is the sum of the type-4 frame bit widths. The
frame stride is the total frame-coded bits (32 for each type-3 channel
plus the stored widths for types 4 and 5), rounded up to a 32-bit word;
this layout check passed on all 307 cached sequences.

Channel parameters precede the frame data. After
`frame_count * bytes_per_frame` bytes of frame data, read **eleven**
u16 channel-list counts and their u16 lists. A list word is
`track_index << 2 | component_index`; the track index addresses the
12-byte rows. In the sampled type-7 cached quaternion records, the low
two bits identify the omitted XYZW component, reconstructed as the
positive square root of `1 - sum(other_components^2)`. Type 8 is
different: the scalar channels store all four XYZW components, and the
type-8 marker consumes no parameter or frame bits. Read all four stored
values by slot index and normalize them for playback. Across 1,885
type-8 tracks (1,286,011 samples), the largest pre-normalization norm
error was 0.00137. Keeping all four values brought all 855 shared-frame
comparisons below a sign-invariant quaternion distance of 0.01, while
discarding one component produced 505 mismatches above 0.5. Pad each list
to a multiple of four u16 words. Header `+0x10` points to the first
six-byte root-motion reference: `+0x10 - 0x20` equals the end of the
channel lists in 563 footer sequences and the payload length in 448
sequences without footers. The high and low nibbles of byte `+0x1F`
count position and rotation references.

Each reference is `{u8 channel_kind, u8 component, u16
parameter_word_offset, u16 frame_bit_offset}`. The parameter offset is in
4-byte words from the start of the sequence payload; the frame offset is
in bits from the start of a packed frame. A group of references selects
the complete channel list of one target-0 trajectory track. In 1,126
groups, the position rows were `{target=0, property=0x7e58c0a0, kind=2}`
and the rotation rows `{target=0, property=0x103ca38a, kind=0}`; the
latter was always track-table index 0. Intersecting the candidate tracks
across each group resolved a singleton 1,126/1,126 times, and the
reference multiset matched the track's channel multiset.

Of 3,096 references, 329 were individually ambiguous. All were kind-7
or kind-8 markers at the end of a rotation group: markers consume no
parameter or frame bits, so same-kind markers can share offsets. The
group's other references identify the track.

**Type 10: incremental float.** The parameter record begins with a
`u32` size in 4-byte words (including the header), four descriptor bytes,
then `f32 quantum` and `f32 bias`. Descriptor bytes 0 and 1 give the bit
widths of the offsets and initial values respectively. Byte 2 is the
rounded residual payload length per lane in 32-bit words
(`lane_bits / 32`); byte 3 was zero in all 35,964 records across the
299 cached sequences used for that check.

Read the payload LSB-first. Divide the sequence into 64-frame chunks.
Read one lane bit offset per chunk, then one initial quantized value per
chunk. Read a unary width (zero bits terminated by one) for each group of
up to eight residual frames, excluding each chunk's first frame. Align
to 32 bits. The residual payload is eight lanes interleaved by 32-bit
word: lane `l`'s logical word `w` is payload word `8*w + l` after
alignment. Each residual group reads the same width at the same bit
offset from lanes 0 through `min(8, remaining_frames) - 1`, in order.

```text
for each chunk:
    pos = lane_offset[chunk]
    value = initial_value[chunk]
    velocity = 0
    emit(value * quantum + bias)
    for width, count in the chunk's residual groups:
        for lane in 0 .. count-1:
            u = read_unsigned_bits(lane, pos, width)
            velocity += (u >> 1) XOR -(u & 1)
            value += velocity
            emit(value * quantum + bias)
        pos += width
```

Each chunk's widths sum to the distance to the next stored lane offset.
The last lane end, rounded to 32 bits and multiplied by eight, accounts
for the rest of the record. Both checks passed for **36,158 records in
302 sequences**. Type 10 contributes no bits to the per-frame stream.
This layout produces continuous rotations in tested clips, but does
not resolve all observed type-5 anomalies or all within-sequence
discontinuities; type 9 remains undecoded.

### Pose root and trajectory

In `pack:/1f_twerk_ped_01.clip` (`anim.rpf@0xf7f1f010`, clip `+0x194f0`,
sequence `+0x20000`), `SKEL_ROOT` has a rotation and translation
separate from the trajectory in the sequence footer. Applying the two
skeletal tracks as **absolute local** transforms on the matching female
rig, rather than re-anchoring the first sample to the root bind
orientation, reproduces the planted portion of the motion without foot
IK. Over frames 39-100, the left/right toe-height standard deviations
dropped from 0.190/0.207 m with the re-anchored root to 0.005/0.001 m
with the absolute root, keeping all non-root keys unchanged.

In this example, the three position references resolve to track 36
(`0x7e58c0a0`); all four rotation references resolve to track 0
(`0x103ca38a`). Track 153 (`0xbd8d191d`) duplicates the heading curve but
is not referenced by the footer. The trajectory reaches 0.728 m of
planar displacement and about 141 degrees of heading change. How the
runtime combines it with the pose is not yet verified.

## Ped body shape

Body shape is a **sparse additive pose rig** on helper bones (`BM_*`,
`MH_*`, `SEC_*`, `RB_*`) of the shared ped skeleton
([§7.5](07-models.md#ped-skinning-bone-palette-confirmed)); the clothing is
skinned to those bones. Three pieces:

**Presets** (plain XML in `common.rpf`, `CPedVariationBodyTypeManager`):
height presets (1.52–2.02 m), scale profiles and 16 body-shape presets
(THIN, HALFTHIN, REGULAR, HALFHEAVY, HEAVY, MUSCLE*, BELLY*, HEAVY_EXTREME,
DANCERFEMALE, LINEBACKERMALE …). Each preset is a list of controls:
`gcweight<region>` (−1 thin to 1 heavy) and `gcmuscle<region>` (0 to 1) for
back, belly, butt, calf, chest, forearm, hip, muffin, neck, shoulder,
thigh and upperarm, plus `GCBreasts`, `GCCleavage` and `GCButt`. REGULAR is
all 0 (confirmed).

**Controls → rig inputs** (a preset PSO in `anim.rpf`: 922 of its 8,192
items carry rig inputs; confirmed). A negative `gcweight` drives the region's
thin input by |v|, a positive one its heavy input by v; `gcmuscle` drives one
input directly.

| Region | Heavy | Thin | Muscle |
|---|---|---|---|
| back | `722b3991` | `e79422cf` | `083e8443` |
| belly | `efc858c0` | `74a5940e` | `860a629a` |
| butt | `b65f7951` | `1d70bfc2` | `2267feeb` |
| calf | `4c92b04a` | `b3925c52` | `798aa11f` |
| chest | `fbc18785` | `47164eb7` | `f8ca1901` |
| forearm | `7e5f130b` | `49c5ac98` | `33c4e1ea` |
| hip | `0978bf99` | `596b89f1` | `fc1aaed3` |
| muffin | `b0c81e7f` | `9ffec641` | `3f088cd7` |
| neck | `334633c1` | `efc4dbff` | `97d111cf` |
| shoulder | `6d8264ca` | `fb039566` | `ab82da7b` |
| thigh | `525a6395` | `7636ca63` | `9e37dc2e` |
| upperarm | `c8a41369` | `5a7029e5` | `37657a6b` |

`GCButt` drives `5a502ae2`; `GCBreasts` drives `377b0101` when positive and
`48f90678` when negative. The input names themselves are unresolved.

**Rig dictionaries** in `anim.rpf` (magic `RIG` v2 at +0x70): one with 1,256
rigs (`anim/rig/generated/…`) and one with 543 (`anim/rig/canonical/…`).
Header +0x48 points to 0x50B hash buckets; each entry is {u32 joaat of the
full rig path, u64 pointer to a 0x30-byte wrapper (+0x10 count 1, +0x20
pointer to the rig), u64 next}. Body rigs include
`LOD0`–`LOD3_Body_Shapes_Female`, `Body_PoseDeformation_Female`,
`Dancer_Body_Shapes_Female`, `BreathingStyle0_Body_Shapes_Female`, XXL and
bodybuilder variants and garment rigs (`*_Garment_Shapes_Female`).

A rig is wrapped as `RIG` + byte 0x02, a u32 size (= the inner blob's
size), and the `RIG0` blob at +0x100: version bytes, size, a section count
at +0x10 and section offsets from +0x14. Sections are tagged {u32 type hash,
u32 size}:

| Section | Contents | Confidence |
|---|---|---|
| `e636c3e9` | block table: count, register count (1,252 floats on `LOD0_Body_Shapes_Female`), then u32 entries `type << 24 \| offset` into the program body; type 0xFF ends | confirmed |
| `2316833f` | inputs: records {u8 type, u8 kind, u16 register, u32 hash}. Kind 3 = a bone track (joaat bone name; type 0/1/2 = translation / rotation / scale), kind 1 = a float input | confirmed |
| `7224f987` | outputs: the same bones | confirmed |
| `b00f0959` | the program (about 25 KB); its body starts after type, size and u32 `0x00020000` | confirmed |
| `74d00508` | 5 named blocks {hash, offset, size} | inferred |
| `6257c371` | the rig path | confirmed |

Program blocks:

| Type | Block | Confidence |
|---|---|---|
| 0x85 | constants: u16 first register, u16 count, then f32 values (registers 0, 1 = 0.0, 1.0) | confirmed |
| 0x46 | u16 dst, a, b: probably dst = a × b | inferred |
| 0x81 | u16 dst, src, then a 0..1 range: probably a clamp | inferred |
| 0x86 | one pass: u16 counts nT, nS, nQ, 0; then nT + nS + nQ u16 triples (track register, weight register, the track register again); then the deltas, 3 floats per translation and scale entry and 4 per rotation entry (unit quaternions). Each entry does T += w·dT, S += w·dS or q = q·dq^w on the track, in place; a track appears at most once per pass | confirmed |

`LOD0_Body_Shapes_Female` has 12 passes (360 down to 3 entries) over 360
helper-bone tracks (translation vec3 at registers 8–429, scale vec3 at
430–775, rotation quaternions at 776–1229) and 53 float inputs. Most deltas
are on scale. The tracks are also inputs, so the rig adds to the animated
pose. `LOD0_Body_Shapes_Male` has 1,294 registers, 379 tracks and 53
inputs, 50 of them shared with the female rig, all the control inputs
among them.

**Axes.** Deltas are in the game's Z-up bone frames. For a glTF skeleton
converted with (x, y, z) → (x, z, −y), convert translation deltas to
(x, z, −y), scale to (x, z, y) and quaternions to (x, z, −y, w). With that,
all 196 left/right translation pairs mirror exactly, and the shapes look
right: heavy widens torso and pelvis and pushes the belly forward, thin
slims (checked by eye on a female top).

Related: about 210 resources in `packs.rpf` with hashed `GM_U01_*` bone
references are soft-body / secondary-motion data, not body shape: particles
(x, y, z, w with w 400–3000) and edges {u16 a, u16 b, f32 rest length, f32,
u32 flags} (inferred).

Open: the exact 0x46 / 0x81 ops, the lower-LOD rigs, the input names, and
why the belly spikes on some male T-shirts whose palettes name bones in
neither skeleton.

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

### Key lookup

Read the 16-byte descriptors backwards from the root minus 0x10 until
the locator descriptor is reached, then reverse them to get the
text-page order. Each descriptor is `{u32 tag, u32 size, u64 pointer}`.
Pointers use the system base `0x50000000`.

1. Hash the ASCII-lowercased identifier with unseeded joaat. A
   0x110-byte bucket has 16 fingerprint bytes followed by 16 records
   of `{u32 key, u32 zero, u64 value}`.
2. Start at `(key >> 7) & (bucket_count - 1)`. Compare the full key
   in slots whose fingerprint equals `key & 0x7F`. If none matches,
   stop on a bucket containing an empty slot (`0x80`). Otherwise advance
   one bucket, wrapping at the end. Bound the probe by the bucket count.
3. In the matching row, `value & 0xFFFFFF` is the kind and
   `value >> 24` is a slot in units of 16 bytes. Kind 1 points into
   the direct locator table. Read its u64 text locator at row `+0x08`.
   Kind 7 points into the variant-group table.
4. Decode a text locator with `page_index = (locator & 0xFFFFF) - 4`
   and `byte_offset = locator >> 20`. Resolve the resulting offset
   inside its text page to a NUL-terminated UTF-8 string. The offset
   must be the start of the page or follow a NUL, and the terminator
   must be inside the page.

A variant group is 0x20 bytes. It holds u32 count and capacity at
`+0x08` and `+0x0C`, and a u64 item reference at `+0x10`. The
reference's low 24 bits are 8. Its high bits give the first slot in the
variant-item table. Item `i` is at `16 * (first_slot + 2 * i)`
bytes into that table, with its text locator at `+0x00` and u64
attributes at `+0x08`. This lookup recovers the stored alternatives.
The runtime selection rule for the attributes is unknown.

---

← [16. Road graph, junctions and flags](16-roads.md) · [Index](README.md) · [18. Radio station audio and scheduling](18-radio.md) →

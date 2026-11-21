# 5. `ttpf` texture stores

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-20; confidence: confirmed / inferred / unknown.*

`data/ttdb/**/*.ttpf`: 57 stores (maps, terrain, props, peds, vehicles, UI,
weapons…), magic `RDS5`. Each store is a virtual byte space (can exceed
4 GiB) split into independently compressed chunks, plus an index keyed by
the **64-bit texture hash that drawable shaders reference** ([§7.6](07-models.md)). Most
map and vehicle textures live here, not in texture dictionaries.

## Header

| Off | Meaning | Confidence |
|-----|---------|-----------|
| 0x00 | `RDS5` | confirmed |
| 0x04 | u32 n = chunk-table entries (last is a zero terminator) | confirmed |
| 0x08 | u32 virtual size (low 32 bits only) | confirmed |
| 0x0C | u32 0x16000000, or 0x16000001 on 18 stores (mostly the largest); meaning unknown | confirmed values |
| 0x10 | u32 ?, u32 0 | unknown |
| 0x18 | chunk table, 8 bytes per entry | confirmed |
| `align_up(0x10 + 8n, 0x100)` | `ETPF` texture index | confirmed |

## Chunk entry (u32 a, u32 b)

| Field | Formula | Confidence |
|-------|---------|-----------|
| file offset | `(a & 0xFFFFFF) << 8` | confirmed |
| virtual offset | `0x1000 + (b & 0xFFFFF) × 0x4000 + (a >> 30) × 0x1000` | confirmed (2,810 chunks contiguous) |
| codec id | bits 24–29 of `a` (table below) | confirmed |
| `b >> 20` | trailing zero padding of the chunk, in bytes: stored length = next file offset − this file offset − (`b >> 20`) | confirmed (541/541 sampled zlib chunks end exactly at the RFC 1950 end-of-stream; 535/535 sampled BCPack chunks) |

Use the indexed length, not a length found by stripping trailing zeros: some
zlib streams end in a real `0x00` byte (2 of 541 sampled), and stripping cuts
them short.

A chunk runs to the next entry in both spaces. Missing the 4 KiB
sub-offset in `a >> 30` puts a quarter of chunks 4–12 KiB early, which
looks like a scrambled swizzle.

| Codec id | Chunks | Content | Codec |
|---|---|---|---|
| 0x20 | 908,643 | any | zlib |
| 0x04 | 100,026 | BC1 | BCPack |
| 0x0C, 0x10 | 35, 77 | probably BC3, BC4 | BCPack |
| 0x14 | 325 | BC5 | BCPack |
| 0x18 | 268 | BC6H | BCPack |
| 0x1C | 5,120 | BC7 | BCPack |
| 0x00 | 5 | any | stored (read as is) |

The non-zlib ids are compressed, not raw pages: their payloads measure
about 7.95 bits/byte and do not match raw BCn block sizes (confirmed).

## `ETPF` index

+4 u32 version (0x27A), +8 u32 texture count t, +0xC u32 table length, then
t × 16-byte descriptors, then t × u64 texture hashes **sorted** (descriptor
i belongs to hash i), then more tables (a u32 list, t × 32-byte digests:
unknown).

| Descriptor field | Meaning | Confidence |
|---|---|---|
| u32 d0 bits 0–13 / 14–27 / 28–31 | width − 1 / height − 1 / mip count − 1 | confirmed |
| u32 d1 bits 0–2 | flags | unknown |
| u32 d1 bits 3–10 | DXGI format | confirmed |
| u32 d1 bits 20–23 | ? | unknown |
| u32 d1 bits 24–27 | tile mode | confirmed |
| u64 loc bits 0–20 | first 4 KiB page, counted after the 0x1000 header page (virtual offset = page × 0x1000 + 0x1000) | confirmed |
| u64 loc higher bits | ? | unknown |

Textures are packed tightly and small ones can sit inside a larger one's
padding, so **the size comes from the mip chain**, not the distance to the
next texture.

## Layout: AMD Gfx10 (RDNA2) swizzle modes (confirmed)

| Tile mode | Gfx10 mode | Block | Element bit order in a block (low → high) |
|---|---|---|---|
| 0 | `SW_64KB_D` | 64 KiB | 16-byte: `x0 y0 x1 y1 y2 x2 y3 x3 y4 x4 y5 x5` · 8-byte: `x0 y0 x1 x2 y1 y2 x3 y3 x4 y4 x5 y5 x6` |
| 1, 2, 3 | `SW_4KB_S` | 4 KiB | 16-byte: `y0 y1 x0 x1 y2 x2 y3 x3` · 8-byte: `x0 y0 y1 x1 x2 y2 x3 y3 x4` |

- Blocks are row-major; no pipe/bank XOR.
- **Mips go smallest first; mip 0 is last.**
- **Mip tail:** a mip whose element size is ≤ (block width / 2, block
  height) and that is among the last `max_mips_in_tail` mips (8 for 4 KiB
  blocks, 12 for 64 KiB) is in the tail. All tail mips share one block at
  offset 0; the other mips follow, each padded to whole blocks.
- Tail mip k (0 = largest in the tail): `m = max_mips_in_tail − 1 − k`,
  `off = m > 6 ? 16 << m : m << 8`. Origin x = bits 9, 11, 13… of `off`
  × 8 (8-byte elements) or × 4 (16-byte); y = bits 8, 10, 12… × 4. Apply
  the block equation to (origin + x, y).

Checked per store and format: mip n + 1 equals mip n downsampled (mean
error 0–4 levels) on BC1, BC3, BC4, BC5 and BC7 in every mode.

## Coverage (all 57 stores, 352,461 textures)

| | Textures |
|---|---|
| whole mip chain in zlib chunks | 298,332 (84.6 %): vehicles 96.7 %, maps 91.8 %, props/peds/interiors ~90 %, terrain 66 %, UI 57 % |
| chain touches BCPack, mip 0 still in zlib | 6,966 (2.0 %) |
| mip 0 in BCPack, a lower mip in zlib (exported at that mip) | 14,441 (4.1 %) |
| every mip in BCPack | 29,310 (8.3 %) |
| unsupported 4-/2-byte formats or special tile modes 9–12 | 3,412 (1.0 %) |

**Lower-mip recovery:** chunks are 4 KiB–4 MiB ranges, not whole textures,
and mips go smallest first, so the small mips of a BCPack texture often sit
in zlib chunks. SixNG exports the largest mip that lies entirely outside
BCPack chunks (tail mips count as the whole tail block).

**Who is hit:** BCPack hits linear `BC1_UNORM` almost only (normal, packed
and mask data), not colour. Over all materials in `levels.rpf`, 37.9 % have
a flat slot and 22.6 % a low-mip slot, but weighted by placements that is
5.0 % and 3.5 %; only 7,088 materials have a flat colour slot. Terrain is
worst: its layer normals and its RGB layer-blend mask are often flat.

## Open questions

- 4-byte (RGBA8/BGRA8) and 2-byte (R16) layouts: no equation found, and
  even row-major reads are scrambled, so the data location may also be
  wrong (arrays/cubes, or another header).
- Format 157 (one vehicle texture slot) is unknown.
- Descriptor flag bits, the digest tables.
- The 64-bit texture hash function. [structure/05](../structure/05-ttpf-stores.md)
  reports one plain joaat case (`hauler` → `0x5A82F9AE`). A check of every
  index hash in 46 stores found no hash whose low or high 32 bits equal that
  value, so this is unconfirmed.


---

← [4. Xbox Series texture tiling (dictionaries)](04-xbox-tiling.md) · [Index](README.md) · [6. BCPack](06-bcpack.md) →

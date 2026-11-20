# 5. Texture stores — `RDS5` / `ETPF`

`data/ttdb/**/*.ttpf` holds 57 stores. Each is a virtual byte space, potentially larger than
4 GiB, split into independently compressed chunks, plus a texture index.

## Header

| Off | Meaning | Confidence |
|---|---|---|
| 0x00 | `RDS5` magic | OBSERVED |
| 0x04 | u32 `n` — chunk-table entries (last entry is a zero terminator) | OBSERVED |
| 0x08 | u32 virtual size, low 32 bits | OBSERVED |
| 0x0C | u32 `0x16000000` (`0x16000001` on the 18 largest stores) | OBSERVED |
| 0x10 | u32, then a zero u32 | UNKNOWN |
| 0x18 | chunk table, 8 bytes per entry | OBSERVED |
| `align_up(0x10 + 8n, 0x100)` | `ETPF` texture index | OBSERVED |

## Chunk table

Entry is a pair `(u32 a, u32 b)`:

| Field | Formula |
|---|---|
| file offset | `(a & 0xFFFFFF) << 8` |
| virtual offset | `0x1000 + (b & 0xFFFFF) × 0x4000 + (a >> 30) × 0x1000` |
| codec id | bits 24..29 of `a` |

A chunk runs to the next entry in both address spaces. Omitting the 4 KiB sub-offset
(`(a >> 30) × 0x1000`) places a quarter of chunks 4–12 KiB early, which presents as a scrambled
swizzle rather than as a wrong offset — a useful diagnostic.

### Codec ids

| Id | Chunks (one install) | Content | Encoding |
|---|---|---|---|
| 0x20 | 908,643 | any | zlib |
| 0x04 | 100,026 | BC1 | BCPack |
| 0x0C, 0x10 | 35, 77 | probably BC3, BC4 | BCPack |
| others | small | various | BCPack |

## Texture index (`ETPF`)

Structure: magic `ETPF`, a constant `0x27A`, a count, then `count × 16`-byte records, then a
trailer. Section type bytes decide the encoding of each chunk: type bytes whose high nibble is
2, 6, A or E and whose low nibble is 0 are zlib; all other observed type bytes are raw pages.

The descriptor's second u32 holds the format in bits 3..9: `((u1 >> 3) & 0x7F)` is the DXGI format
enum. This held for 357,115 of 357,115 descriptors, with no unknown values.

Each store's index is keyed by a 64-bit texture hash. Those hashes appear as plain joaat values in
at least one confirmed case (`hauler` → `0x5A82F9AE`), so the index is name-keyed rather than
path-keyed.

## Volumes

Across the 57 stores: 1,026,833 sections, 147,252,026,112 bytes of payload (137.1 GiB decoded),
zero unaccounted bytes in every file. Total stored size 57.94 GiB.

## Open questions

- The meaning of `b >> 20` in each chunk entry.
- BCPack: no decoder exists publicly; roughly 10% of archive bytes are BCPack-encoded.
- The 64-bit texture hash function for values other than the confirmed joaat case.

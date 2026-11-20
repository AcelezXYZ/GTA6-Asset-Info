# 1. RPF9 container

## Header

`0x40` bytes are meaningful; the table of contents begins at `0x40`.

| Off | Size | Meaning | Confidence |
|---|---|---|---|
| 0x00 | 4 | magic `39 46 50 52` (`"9FPR"` on disk; `RPF9` as a little-endian u32) | OBSERVED |
| 0x04 | 4 | `toc_end` — absolute end of the table of contents; payload starts after it | OBSERVED |
| 0x08 | 4 | varies by archive; not `toc_end`, not a count | UNKNOWN |
| 0x0C | 4 | size of a name block inside the TOC (inferred) | INFERRED |
| 0x10 | 8 | 64-bit per-archive value; unique across archives | OBSERVED |
| 0x18 | 4 | constant `0x00047A00` | OBSERVED |
| 0x1C | 4 | constant `0x00000150` | OBSERVED |
| 0x20 | 2 | takes values 3..7 | UNKNOWN |
| 0x22 | 2 | 1 | UNKNOWN |
| 0x24 | 4 | a count; equals `file_count` in some readings but not the carved entry count | DISPUTED |
| 0x28 | 4 | varies per archive | DISPUTED |
| 0x2C | 4 | only two values observed across 22 archives: `0x00090F01` and `0x00090F02` | OBSERVED |
| 0x30 | 4 | constant `0x10` | OBSERVED |

### Header variants

Across 22 archives the `0x2C` field splits them into two groups:

- `0x90F01`: `audio`, `navmeshes`, `packs`, `textures`, the top-level `script` archive, and
  all small `*_restofworld` / `*_bvh4` archives.
- `0x90F02`: `levels`, `data`, `anim`, `story000`, `movies`, `common`, `core_bvh4`, and the
  nested PSO archive inside `script`.

The second group shows `round16(10 × file_count)` for the region between `index_end` and
`block_end` exactly (e.g. 2,078 × 10 + 4 = 20,784). The first group does not follow that law.

## Table of contents

Encrypted. Entropy is at the uniform-random level in every archive; no repeated 16-byte blocks
occur; the first 32 bytes of the TOC are **identical across all 22 archives**, which means the
plaintext there is identical and the transform is position-addressed rather than randomised per
archive.

Consequences:

- No file names or directory tree are recoverable from the TOC itself.
- The identical 32-byte prefix is a known-plaintext crib.
- A 16-byte-record interpretation is not supported: sortedness at strides 4, 8 and 16 measures
  0.49–0.56 (a shuffled control gives the same), and a 1.3M-token name universe hashed with
  joaat or crc32 produces 488 hits against 489 expected by chance.

## Payload

- Not encrypted. Bodies are plain zlib streams, each starting on a 16-byte boundary.
- Large resources are stored as **chains of 64 KiB blocks**: a block continues into the next
  stream **iff** it decoded to exactly 65,536 bytes and the next stream begins at the 16-aligned
  slot after the previous compressed stream. There is no flag; the full-block size is the only
  marker.
- RAGE resource pages begin at a 4 KiB boundary plus `0x10`.
- Unused tails are zero-filled; empty regions are absent rather than padded.

## Nested archives

RPF9 archives may contain RPF9 archives. One archive of 4.2 MiB has three headers: offset `0x0`
(2 entries), `0x1000` (2 entries) and `0xC4000` (1,012 entries, `0x90F02` variant, with
`index_end < block_end` — the reverse of the usual ordering). Enumeration must therefore scan for
the magic at 4 KiB boundaries rather than assume one header per file.

## Trailer

Each archive ends with a trailer whose length is a multiple of 16, located at `EOF − length`.
Record count `N = length / 16` is always `≥ file_count`.

**Prefix law (OBSERVED):** `trailer[0:16] = C(file_count)` and `trailer[16:32] = C(N)`. Archives
sharing a `file_count` have byte-identical bytes `0..16` with no exceptions, and bytes `16..32`
differ exactly where `N` differs. `C` matches none of 77 unkeyed digest candidates, so it is a
keyed transform.

## Open questions

- TOC cipher and key.
- Meaning of header fields `0x08`, `0x18`–`0x22`, `0x2C`, `0x30`.
- The `0x24` disagreement: 2,078 by direct header read for the largest archive versus 523,552
  carved entries for the same archive.

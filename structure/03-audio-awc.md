# 3. Audio — `ADAT` containers

## Header

| Off | Size | Meaning |
|---|---|---|
| 0x00 | 4 | magic `ADAT` |
| 0x04 | 4 | flags (values observed: `0xFF020001`, `0xFF210001`, `0xFF250001`, `0xFF610001`) |
| 0x08 | 4 | stream count |
| 0x0C | 4 | header length — the chunk table ends exactly here |

Optional tables follow the fixed header, selected by flag bits:

| Flag | Table |
|---|---|
| `0x00400000` | one extra u32 word |
| `0x00010000` | u16 first-chunk index per stream |
| `0x00200000` | u32 × (stream_count + 1) |
| `0x00040000` | multi-channel / music variant (512 KiB block framing) |

Then, per stream, a u32 of the form `id:29 | chunk_count:3`, followed by the chunk tag table.

## Chunk tags

Each tag is a u64: `type:8 | size:28 | offset:28`. Type bytes are the low byte of a joaat of the
chunk name:

| Type | Chunk |
|---|---|
| 0x36 | peak |
| 0x48 | stream_format |
| 0x55 | data |
| 0x5C | animation |
| 0x76 | format |
| 0xA3 | seek_table |
| 0xB2 | metadata |
| 0xBD | markers |

## Data framing

Single-channel streams store **Opus packets in 2,048-byte blocks**. Each block is a sequence of
`u16 little-endian length + packet`, zero-padded to the block end. A zero length ends that block;
blocks are independent.

Consequence: `0x55` chunk sizes are exact multiples of 2,048 when the payload includes its block
header, which is why raw size statistics show a 256-byte period rather than a 2,048-byte one.

Music/multi-channel streams use 512 KiB blocks with 2,048-byte sub-blocks, selected by the
`0x00040000` flag.

## Format chunk (`0x76` / `0xFA`)

20, 24 or 32 bytes:

| Off | Meaning |
|---|---|
| 0x00 | u32 sample count |
| 0x04 | u32 loop point |
| 0x08 | u16 sample rate |
| 0x0A | u16 headroom |
| 0x0C | u16 loop begin |
| 0x0E | u16 loop end |
| 0x10 | u16 play end |
| 0x12 | u8 play begin |
| 0x13 | u8 codec (0x00 = Opus) |
| 0x14 | u32 peak (24-byte form) |
| 0x1C | u32 extra (32-byte form) |

## Non-Opus containers

Not every `ADAT` file carries Opus. Observed exceptions: containers with zero streams and a
20-byte header, and containers whose `0x55` chunk is uncompressed IEEE float32 PCM (a
163,840-byte chunk decodes as 40,960 samples × 4 bytes at 48 kHz). Any decoder must check the
codec byte rather than assume Opus.

## Open questions

- Whether `0x55` payloads can mix Opus and PCM inside one container.
- The meaning of `headroom` and of the `0x00400000` word.

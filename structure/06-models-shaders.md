# 6. Models and shaders

## Meshlet geometry

Drawable geometry is stored as meshlets, not as raw vertex arrays. Every raw-coordinate
interpretation fails arithmetic or range tests, so a decoder must follow the meshlet structure
rather than search for float triples.

Observed layout, worked example body of `0x40000` bytes:

- Body header, with a system-end pointer at `+0x08` and a u16 count at `+0x4C`.
- That count gives the number of `{id/hash, size, pointer}` records that follow.
- Sorted by pointer, the records tile the body with gaps only of `{0, 4, 8}` plus the record
  array itself, and the chain terminates on the header's own `+0x08` target.

Meshlet buffer:

| Field | Meaning |
|---|---|
| `u32 @ buffer+0x00`, low 16 bits | meshlet count |
| `buffer + 0x0C` | start of the per-meshlet 16-byte record array |
| `buffer + 0x0C + count × 16` | triangle base |

Per-meshlet records carry the vertex and triangle offsets into the shared buffers; triangle spans
run to the next meshlet's start.

### Drawable name

The drawable's name pointer sits at `drawable + 0xA8`. Following it yields the authored name —
`name.#dr`, `name.#td`-style suffixes included. Scanning only for a `.#dr` string in the resource
finds a minority of drawables; the pointer is the reliable route.

### Containers

`0DRP` bodies in mesh-bearing archives are page-multiple in size (`0x20000`–`0x150000`) and carry
the same descriptor at `+0x10` seen elsewhere: a `0x7A` constant at `+0x14` and `0x50xxxxxx`
relocated page pointers.

In `PSIN` files the `LRSC` section instead holds an 80-byte `0DRP` stub with no pointers, so the
geometry for those entries is not in the `PSIN` payload itself.

## Shader cache

A `PS\x00\x09` pipeline-state-object cache carries native AMD **RDNA2**
machine code.

Observed cache layout:

| Region | Size |
|---|---|
| header | `0x80` |
| graphics PSOs | `count × 0x70` |
| compute PSOs | `count × 0x18` |
| meta index | `count × 8` |
| shader index | `count × 8` |
| meta blob | variable |
| raw code | variable |

Example counts from one cache: 25,471 graphics PSOs, 2,693 compute PSOs, 54,576 meta entries,
21,522 shaders.

Instruction-level markers: `s_endpgm` (`0xBF810000`), `s_code_end` (`0xBF9F0000`), `s_clause`,
Wave32 VOP2/VOP3 forms, `image_sample_l` with 12-byte non-sampled-address operands, NGG/meshlet
`exp prim`, and hardware ray-tracing `image_bvh64_intersect_ray`.

Binding metadata: chunk `0x41` carries user-data root-signature records (`SRV 0x300`,
`CBV 0x302`, `UAV 0x303`, `Sampler 0x002`), `0x42` register slots, `0x43` 128-bit shader hashes,
plus `COMPUTE_PGM_RSRC1/2` and the `[numthreads(x,y,z)]` triple.

## DXBC-wrapped native programs

`shaders.rpf` also contains DXBC wrappers carrying native code.
The stage is the high 16 bits of the first u32 of the DXIL part payload.
The offset named by the DXIL header's bitcode field leads to a four-byte
`NOBC` marker. Native code is reached through `XDAT` at payload
`+0x18`.

1. In `XDAT`, the u32 at `+0x04` counts 12-byte descriptors beginning
   at `+0x0C`. Each has a type, offset from the start of the DXIL part
   payload, and byte length (all u32). Descriptors of type `0x0D` lead
   to `XBCH` blocks.
2. Each `XBCH` block has a 0x18-byte header. The u64 fields at `+0x08`
   and `+0x10` give stored and expanded sizes. Inflate the zlib stream
   at `+0x18` and check the expanded size.
3. The output begins with `FSSM`. The u64 at `+0x08` gives the
   declared length, followed by a 16-byte trailer outside that length.
   The u64 fields at `+0xC0` and `+0xC8` bound the code region.
   The instructions start 0x28 bytes into the region. Disassembly
   uses AMD `gfx1013`. The measured entry decode ends at the first
   `s_endpgm` instruction.

Count wrappers and native spans separately. In the audited archive,
195 vertex wrappers each held two distinct spans. This gives
26,503 DXBC wrappers and 26,698 spans, including a separate set of
195 geometry wrappers. The two-span vertex selection rule remains open.
After deduplication by code bytes, all 15,110 indexed native entries
matched LLVM instruction decoding through the first `s_endpgm`.

## Open questions

- The 64-bit hashes in the shader cache. (The 32-bit shader and parameter hashes in drawables are
  joaat of the lower-case name; see [formats §12](../formats/12-materials.md).)
- Runtime selection of vertex spans, pass variants and material bindings.
- Whether meshlet vertex data uses a dictionary encoding (a small shared u16 vocabulary is
  observed across bodies, suggesting one).
- BCPack decoding for compressed surfaces.

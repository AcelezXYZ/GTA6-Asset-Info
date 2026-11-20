# 3. Texture dictionaries

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-20; confidence: confirmed / inferred / unknown.*

Layout matches GTA V's `pgDictionary<grcTexture>` (confirmed).

| Off | Meaning | Confidence |
|-----|---------|-----------|
| 0x20 | ptr → u32 name hashes | confirmed |
| 0x28 | u16 count, u16 capacity | confirmed |
| 0x30 | ptr → texture pointer array | confirmed |
| 0x38 | u16 count, u16 capacity (equal to 0x28) | confirmed |

**Texture struct** (0x70 bytes; also used for textures embedded in models):

| Off | Meaning | Confidence |
|-----|---------|-----------|
| 0x08 | u32 size field (usually bytes / 16; inconsistent, not used) | inferred |
| 0x0C | u16 bytes per element (8 = BC1/BC4, 16 = other BC) | confirmed |
| 0x10 | flags (0x980 typical) | unknown |
| 0x18 | data pointer (`0x6…`) | confirmed |
| 0x30 | ptr → name block (C string at +8) | confirmed |
| 0x38 / 0x3A | u16 width / height | confirmed |
| 0x3C | depth / array size | inferred |
| 0x3F | u8 **DXGI format** (71 BC1, 74 BC2, 77 BC3, 80 BC4, 83 BC5, 95 BC6H, 98/99 BC7, 28 RGBA8, 87 BGRA8 …) | confirmed |
| 0x42 | mip count | confirmed |

## Open questions

- Flags at 0x10; array and cube handling (only slice 0 is exported).


---

← [2. Resources and pages](02-resources.md) · [Index](README.md) · [4. Xbox Series texture tiling (dictionaries)](04-xbox-tiling.md) →

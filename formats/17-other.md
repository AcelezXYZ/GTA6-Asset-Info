# 17. Other formats

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-20; confidence: confirmed / inferred / unknown.*

| Format | State |
|---|---|
| PSO (`PSIN`; sections LSCH/LENM/LSNS/LGID/LRSC/CHKS) | little-endian, hashed field names (64-bit in the vehicle tables); exported raw |
| RBF (`RBF0`) | a new variant with u16 indices and inline names; exported raw |
| `core_bvh4.rpf` | BVH4 collision with packed 24-bit indices; not decoded |
| XML, text | plain; exported as is |
| shader cache (`shaders.rpf`) | one compiled cache, no names; not decoded |
| audio (AWC/ADAT) | not decoded yet (planned) |
| animation (`anim.rpf`) | not decoded |

---

← [16. Road graph, junctions and flags](16-roads.md) · [Index](README.md)

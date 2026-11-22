# 6. BCPack

*Part of the [SixNG format specification](README.md). Confidence: confirmed / inferred / unknown.*

BCPack is the Xbox Series hardware texture codec. It compresses existing
BC blocks, not pixels. Separate three steps when testing a decoder:
BCPack bytes to lane symbols, lane symbols to ordinary BC blocks in
stored order, and detiling those blocks for display. Passing one step
does not verify the next.

## 6.0 Status

The counts and tests below cover one Xbox Series build, not the whole
format across all titles.

| Codec | Entropy decoding | BC block reconstruction |
|---|---|---|
| `0x04` BC1 | Confirmed on sampled chunks with two implementations. | Working exports. Some endpoint presets and selector codebook entries still differ between implementations. |
| `0x0C` BC3, `0x10` BC4, `0x14` BC5 | Two implementations agree on every symbol and consume the full input on 5 BC3, 2 BC4 and 5 BC5 samples. | Candidate rebuilders exist. A complete BC3 mip renders without endpoint presets, but still uses the candidate selector tables. No independent same-mip byte comparison for these formats. |
| `0x18` BC6H, `0x1C` BC7 | All 268 BC6H and 5,120 BC7 chunks in the local scan consumed their exact input and emitted the expected symbol counts per lane. All symbols also matched a second implementation on 5 BC6H and 24 BC7 spot checks. | Candidate rebuilders exist, but they need initial history tables that are not derived here. Some fields can be constrained without those tables, but there is no independent byte-exact match for a complete shipped chunk of either format. |

For BC1, the block count is the chunk's virtual size divided by 8.
Strip the trailing pad from the stored bytes (`b >> 20` in the chunk
entry, [section 5](05-ttpf.md#chunk-entry-u32-a-u32-b)). The output is in
the texture's stored tiled order; detile it exactly as for a raw or zlib
chunk of the same surface.

The earlier SixNG measurement remains a coverage figure, not a
byte-exact benchmark: all 2,776 BCPack BC1 textures in one terrain
store decoded without errors. With BC1 decoding, mip 0 was readable
for 98.4% of store textures
([coverage](05-ttpf.md#coverage-all-57-stores-352461-textures)).

## 6.1 What is solved

**Lanes (confirmed for the tested chunks).** A lane is an independent
rANS state machine. The lanes share one interleaved source stream:
12 lanes for BC1/3/4/5 and 16 for BC6H/7. Each lane emits one symbol
per 8 bytes of unpacked data, so a 16-byte BC block uses two symbols
per lane. The BC1 lanes are:

| Lane | Role |
|---|---|
| 0 | Endpoint-history back-reference code. |
| 1 | Constant zero in the sampled chunks; not the three-colour flag. |
| 2-7 | Residuals for the six RGB565 endpoint components: R0 G0 B0 R1 G1 B1. |
| 8-11 | Selector codebook symbols for the top-left, top-right, bottom-left and bottom-right 2x2 quadrants, not four rows. |

**Fragment framing and byte order (confirmed for the tested streams).**
Each lane is a byte-wise rANS state machine.
The first header byte sets the alphabet narrowing, length form, flush flag
and static model number. The short form has `lo + 1` payload bytes; the
long form has `(lo + (hi << 8)) + 257`. This includes state bytes read
by the lane. The feeder services pending requests in lane order, at most
eight bytes per clock. It does not allocate an independent byte range to
each lane or give every requesting lane a byte every clock.

**End checks (confirmed).** Require the expected symbols in every lane,
every non-padding input byte accounted for, and a legal terminal state.
With `L = 2^24`, the allowed final states are `L`, `L >> 8` and `L >> 16`;
requiring `L` alone rejects valid streams.

## 6.2 Ruled out

- The BC1 index lanes do not need an adaptive model or a separate
  extension-lane parser. The fragment header selects a static rANS model.
- A separate sub-stream is not needed solely because a BC1 chunk has
  1,536 or 3,072 blocks. Five chunks of each size decoded as one stream
  in both implementations, with matching symbols in every lane and
  exact input consumption.
- Matching symbol streams does not settle reconstruction. One tested
  BC1 terrain chunk differs at six output byte positions even though
  every lane matches. All six differences trace to one endpoint preset
  and two selector symbols.

## 6.3 Open unknowns

**BC1/3/4/5:** Confirm the disputed or fitted endpoint presets and selector
mappings against an independent copy of the same blocks. Two software
implementations agreeing is useful but is not a hardware reference.

**BC6H/7:** Confirm the preset endpoint and selector histories and the
remaining BC7 signature mappings. The candidate predictors can be tested
against standard BC anchor rules. Those rules constrain some initial
selector values, but do not constrain texels that are never anchors. In
one tested BC7 vehicle chunk, changing only such a preset selector field
gave two structurally legal reconstructions with 26 different visible
pixels.

Using the candidate reconstruction, six complete mips (three mips each
in two BC7 vehicle chunks) render without reading any initial history.
Two different initial tables gave identical output on those mips. The
vehicle surfaces use the 64 KiB `SW_64K_D` order; using `SW_64K_S`
scrambles them despite identical tile sizes. Two detilers and two BC7
pixel renderers agree on the corrected previews. These checks isolate
the preset dependence and layout but do not prove the original BCPack
blocks byte for byte.

## 6.4 How to resume

1. Check the entropy decoder first. Derive counts from the chunk table,
   remove only the documented padding and compare all lane symbols
   before comparing BC blocks.
2. Compare raw tiled BC blocks before detiling or rendering. A useful
   reference is a readable, independent copy of the **same** blocks at
   the same mip level. A nearby mip, a valid DDS, or a plausible picture
   cannot establish exact bytes. Local plain-mip searches did not find
   byte-exact references for the tested BPTC targets.
3. For a disagreement, trace the affected fields to a specific history
   root or codebook symbol. Test the proposed value on held-out chunks
   separately from those used to derive it.

For extraction without verified reconstruction, use a raw or zlib mip
where available ([section 5](05-ttpf.md)) and label fitted outputs
provisional.

---

← [5. `ttpf` texture stores](05-ttpf.md) · [Index](README.md) · [7. Drawables, fragments and vehicles](07-models.md) →

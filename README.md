# png-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

PNG on its own — the whole format, not a subset: the eight-byte
signature and every one of the checks it encodes, the chunk structure
with the four case flags that tell a reader what it may skip, IHDR
through IEND, all five row filters in both directions, every legal
colour type and bit depth including palette and 16-bit, Adam7
interlacing decoded, and both directions over
[flate-nv](https://github.com/novolang/flate-nv), because every IDAT
chain is one zlib stream and this package will not write a second
inflater.

**The decoder never holds the file.** It is a feed-and-drain state
machine over events — header, palette, transparency, an ancillary
chunk, one row, end — and the state it carries between chunks is
bounded by the image's *width*, not its size, because the filters need
exactly one row of history. A 100,000-row photograph decodes in a few
hundred kilobytes.

image-nv is the multi-format front over this package. It depends on
png-nv rather than duplicating it, and that decision is written down in
both packages' READMEs and on the grid both of them sit on.

```
novo pkg add png-nv
novo pkg build
novo test
```

## The one example that will work

```novo
use png
use pngdec

fn size_of(file: Bytes) -> Result<Int, PngError>
    let info = pngdec.read_info(file)!
    Ok(info.width * info.height)
```

Thirty-three bytes read, no pixels decoded, no memory committed — the
question a thumbnailer or an upload handler asks before it decides
whether to decode at all.

## The layer, and why

`core` — no effects at all, on a package whose subject is a file.

That is not a contradiction, it is the design. Decoding PNG is
arithmetic over bytes the caller already holds: nothing is opened,
nothing is waited for, and the decoder's state is one row of filter
history plus flate-nv's 32 KiB window. The host does the reading;
`pngdec.feed` takes whatever bytes have arrived and hands back the
events they produced.

The one function that meets a stream stays inside the budget by
**binding** its cost rather than spending one:

```novo
pub fn read_all<S: Read[e]>(src: S) -> Result<(PngInfo, Bytes), PngError> [e]
```

`S: Read[e]` binds the effect parameter of the standard library's
`Read` trait and the clause uses it, so the row means *whatever the
impl behind `S` supplies*. A file charges its caller `[io, fs]`; an
in-memory buffer charges nothing; png-nv is charged neither. It is
worth having here where qoi-nv declined it, because a compressed PNG
can be a hundredth of the image it decodes to — a caller who cannot
hold the file may still be able to hold the picture.

**No device claim.** There is no `tests/embedded_probe.nv`, and the
audit's `core-embedded` row passes by saying so. That is honest rather
than modest: DEFLATE's window is 32 KiB, a row of a wide image is more
again, and neither `Bytes` nor `Result` exists at `@tier(embedded)`
today. color-nv and qoi-nv are the two packages in this stack a device
might genuinely want; PNG is not one of them.

## The load-bearing interface

```novo
pub fn decoder() -> PngDecoder
pub fn feed(d: PngDecoder, chunk: Bytes) -> (PngDecoder, [PngEvent])
pub fn finish(d: PngDecoder) -> Result<Unit, PngError>

pub enum PngEvent
    PngEvInfo(info: PngInfo)
    PngEvPalette(colors: [Srgb8])
    PngEvTransparency(t: PngTransparency)
    PngEvAncillary(chunk: PngChunk)
    PngEvRow(pass_no: Int, y: Int, samples: Bytes)
    PngEvEnd
```

Three calls and six events, and every other decoding function is a
value they take, a value they return, or a convenience written in terms
of them.

Two things about `PngEvRow` are the whole argument for the shape.
**Its payload is one row of raw samples in the file's own colour type
and bit depth**, so nothing accumulates and nothing is converted — a
row of a 4K 16-bit RGBA image is 64 KiB, and one colour value per
sample would cost twenty times that for data the caller is about to
write into a buffer anyway. And **it carries a pass number**, because
under Adam7 rows do not arrive in raster order. For a non-interlaced
image the pass is always 1 and `y` is the image row, so the general
loop and the simple one are the same code; `pngadam7.place` turns a
pass coordinate into an image coordinate.

`feed` hands back rows it cannot yet vouch for. Each IDAT chunk's CRC
has been checked, but the zlib stream's Adler-32 covers the whole
image and is verified only at the end, so only a successful `finish`
says the pixels were correct.

## Four decisions worth arguing with

**Adam7 is decoded and not encoded.** Interlaced PNGs exist and a
decoder that refused them fails on real files, so the decoder reads all
seven passes and `pngadam7` is the arithmetic. The encoder refuses an
interlaced `PngInfo` outright, because an interlaced PNG is five to
twenty per cent *larger* than the same image, browsers have rendered
progressively from a non-interlaced stream for a decade, and the
specification calls interlacing optional. Offering it would be
offering a way to make files worse.

**The palette and the metadata are colours; the pixels are not.**
`PngEvPalette` carries `[Srgb8]` from color-nv and `pngchunk.palette_pixel`
joins PLTE with tRNS — those are genuinely colours, there are at most
256 of them, and having color-nv own the type is what lets image-nv and
qoi-nv speak the same one. The pixel data stays packed bytes, for the
size reason above. That is why png-nv depends on color-nv and why the
dependency does not reach the hot path.

**CRC-32 is spelled here rather than taken from flate-nv**, even
though flate-nv has the identical algorithm for its gzip trailer. The
dependency on flate-nv is about DEFLATE; a checksum shared across a
package boundary for the sake of forty lines would tie png-nv's chunk
reader to a decompressor's release schedule. The implementation lane
may decide otherwise — the interface does not force it either way.

**A bad CRC is not always fatal.** Unlike QOI, a PNG can say its bytes
are damaged. But a corrupt `tEXt` is a skippable chunk by definition,
and refusing the file over it throws away an image that is intact — so
lenient is the default, `pngerror.is_recoverable` is the distinction,
and `pngdec.with_strict` is how a caller archiving or validating files
asks for every CRC to be enforced.

## The reference implementation, and what is specification

libpng is the reference implementation, and the
[PngSuite][pngsuite] corpus — the set of files that exists precisely to
break decoders — is the oracle.

[pngsuite]: http://www.schaik.com/pngsuite/

**Specification, and binding on this package**

- The eight signature bytes and what each one tests for.
- Chunk framing: the length excludes the type and the CRC, the CRC
  includes the type and excludes the length, and the four case flags
  are the fifth bit of each of the four type letters.
- The fifteen legal colour-type-and-bit-depth combinations; a 16-bit
  palette and a 1-bit truecolour do not exist.
- All five filters, computed modulo 256 **on bytes** — at 16 bits per
  sample the two halves are filtered separately and nothing carries
  between them — with the neighbours off the top and left of the image
  taken as zero. `Average` truncates its division after summing at
  full width, so `(255 + 255) / 2` is 255. `Paeth` selects whichever
  neighbour is nearest `a + b - c`, ties to `a`, then `b`, then `c`.
- Adam7's seven passes, each a complete small image with its own
  filter bytes and its own neighbours, and a pass whose width or
  height is zero skipped entirely rather than emitted empty.
- That IDAT is one zlib stream however many chunks carry it, and that
  its Adler-32 covers the whole filtered image.

**libpng's choices, which this package follows and a test may not
treat as correctness**

- The minimum-sum-of-absolute-differences filter heuristic in
  `pngfilter.choose`. Nothing in PNG says how to pick a filter, and an
  encoder that chose `PngFilterNone` for every row produces a valid
  file every decoder reads. A test may assert that this package's
  output decodes; it may not assert which filter a row got.
- `pngfilter.default_for`'s advice — no filtering for palette and
  1-bit images, Paeth otherwise.
- Which DEFLATE level is the default. Level 6 is zlib's, and it is
  here for the same reason: it is what almost every PNG in the world
  was written at.

So the correctness condition is **libpng reads what this writes, and
this reads every file in PngSuite that libpng reads**. Byte-for-byte
equality with libpng's output is not a condition — it would be a test
of two encoders' filter heuristics rather than of PNG.

**Deliberately not ported:** APNG, which is a separate specification
and a separate package if anyone wants it; iCCP's profile contents,
which arrive as bytes through `PngEvAncillary` because this package
will not pretend to parse an ICC profile; and interlaced *encoding*,
for the reason above.

## Status

Every function is `todo()`. `novo test` runs the API suite, and every
assertion in it reaches `not implemented: png-nv.<module>.<fn>` — which
is the expected result until the bodies land, and is what makes the
suite a description of the interface rather than of nothing.
`novo test --isolate tests/<file>` is the readable form: one verdict
per test, naming the function it stopped at.

| module | public types | functions | implemented |
| --- | --- | --- | --- |
| `png` | 4 | 11 | no |
| `pngchunk` | 6 | 15 | no |
| `pngfilter` | 1 | 7 | no |
| `pngadam7` | 0 | 7 | no |
| `pngdec` | 2 | 15 | no |
| `pngenc` | 2 | 13 | no |
| `pngerror` | 1 | 3 | no |
| **total** | **16** | **71** | **no** |

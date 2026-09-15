# png-nv

PNG is a lossless raster image format, specified in
[the W3C PNG specification](https://www.w3.org/TR/png-3/) and in
[RFC 2083](https://www.rfc-editor.org/rfc/rfc2083). This package implements it
in novo-lang: the signature, the chunk structure, every legal colour type and
bit depth, all five row filters in both directions, Adam7 interlacing decoded,
and both directions over
[flate-nv](https://novo-lang.org/packages/flate-nv). The colours it hands back
are [color-nv](https://novo-lang.org/packages/color-nv)'s.
[image-io-nv](https://novo-lang.org/packages/image-io-nv) reads PNG files
through this package.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it
is implemented. Version 0.1.0 will be the first working release.

## What it is

A PNG file is an eight-byte **signature** followed by a sequence of **chunks**.
Every chunk has the same four parts: a length, a four-letter type, the data,
and a CRC-32. The length counts the data only. The CRC covers the type and the
data. Those two asymmetries are the whole of chunk framing.

A chunk type's letters carry four flags in their case, which is how a decoder
handles a chunk nobody has heard of. The first letter's case says whether the
chunk is **critical**, meaning a decoder that cannot read it must stop, or
**ancillary**, meaning it may be skipped. The second says whether the type is
registered or private. The third is reserved. The fourth says whether an
editor that changed the pixels must drop the chunk.

The **IHDR** chunk describes the image: its width and height, its **colour
type**, its **bit depth**, and whether it is interlaced. A colour type says how
many samples each pixel carries and how to read them. A bit depth says how many
bits each sample occupies. The two are not independent: only fifteen of the
twenty-five combinations are legal.

| Colour type | Number | Samples per pixel | Legal bit depths |
| --- | --- | --- | --- |
| Greyscale | 0 | 1 | 1, 2, 4, 8, 16 |
| Truecolour | 2 | 3 | 8, 16 |
| Indexed | 3 | 1 | 1, 2, 4, 8 |
| Greyscale with alpha | 4 | 2 | 8, 16 |
| Truecolour with alpha | 6 | 4 | 8, 16 |

The pixels live in **IDAT** chunks, which together hold one zlib stream. Before
the pixels are compressed, each row is **filtered**: every byte is stored as
its difference from a neighbour, which turns nearly equal neighbouring pixels
into small numbers, and a stream of small numbers is what DEFLATE compresses.
There are five filters, and the row's first byte says which one it used.

| Filter | Number | The byte stored, with `x` the byte itself |
| --- | --- | --- |
| None | 0 | `x` |
| Sub | 1 | `x - a`, where `a` is the byte one pixel to the left |
| Up | 2 | `x - b`, where `b` is the byte directly above |
| Average | 3 | `x - floor((a + b) / 2)` |
| Paeth | 4 | `x - paeth(a, b, c)`, where `c` is the byte above and left |

**Adam7** is PNG's interlacing method. It divides the image into a repeating
grid of 8 by 8 blocks and sends the pixels in seven passes, each taking one set
of positions in every block. Pass 1 carries one pixel in sixty-four, and pass 7
carries half the image, so an interlaced PNG appears at low resolution first.

| Quantity | Value |
| --- | --- |
| Signature | 8 bytes: 137, 80, 78, 71, 13, 10, 26, 10 |
| Bytes before the end of IHDR | 33 |
| Legal colour type and bit depth pairs | 15 of 25 |
| Row filters | 5 |
| Adam7 passes | 7 |
| Filter bytes per row | 1 |
| Rows of history a filter needs | 1 |
| Palette entries a PNG may have | at most 256 |
| Size an interlaced file adds | 5 to 20 per cent |

This package performs no input and no output. The decoder is a **feed-and-drain
state machine over events**: the caller hands over whatever bytes have arrived,
and gets back the things the decoder learned from them. The state a decoder
carries between chunks is bounded by the image's width rather than its size,
because the filters need exactly one row of history. A photograph a hundred
thousand rows tall decodes in a few hundred kilobytes.

## Install

```
novo pkg add png-nv
```

## Example

```novo
use std.bytes
use png
use pngdec
use pngerror

fn main() [io]
    // The bytes of a PNG file. The caller does the reading; this package
    // performs no input or output of its own.
    let file: Bytes = bytes.zeros(0)

    // Hand the bytes over. `feed` accepts any split of the file, so a
    // caller streaming from a socket calls it once per chunk instead.
    let (d, events) = pngdec.feed(pngdec.decoder(), file)

    for e in events
        match e
            // The image description, which arrives before any pixels.
            PngEvInfo(info) => println("${info.width} by ${info.height}")
            // One row of samples, in the file's own colour type and depth.
            PngEvRow(pass_no, y, samples) => println("row ${y} of pass ${pass_no}")
            _ => println(pngdec.event_name(e))

    // Only a successful finish says the pixels were correct.
    match pngdec.finish(d)
        Ok(_)  => println("whole")
        Err(e) => println("${pngerror.offset_of(e)}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails
on purpose: every test reaches a `not implemented: png-nv.<module>.<fn>` panic.
The tests are the specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `png` | The signature, the IHDR fields as three enumerations and a struct, the table of legal colour type and bit depth pairs, and the row and image size arithmetic every other module uses. |
| `pngchunk` | Chunk framing, the four case flags as predicates, the CRC-32, and typed readers for the ancillary chunks worth acting on: tRNS, tEXt, pHYs, tIME, sRGB and gAMA. |
| `pngfilter` | The five filters in both directions, the Paeth predictor, and the heuristic that picks a filter for a row. |
| `pngadam7` | The seven passes: each one's width and height, whether it is empty, where one of its pixels sits in the whole image, and the size of the filtered result. |
| `pngdec` | The decoder: a feed-and-drain state machine over events, a header-only read, a whole-buffer decode, and one call that takes a stream. |
| `pngenc` | The encoder: row at a time into a file, or a whole image in one call, with the compression level and the filter strategy as the only options. |
| `pngerror` | Every way a PNG can refuse to be read or written, grouped into framing, image description and data, each carrying the byte offset it happened at. |

## How to choose an entry point

**`pngdec.read_info` answers the dimensions from the first 33 bytes.** No
pixels are decoded and no memory is committed to the image. It is the question
a thumbnailer or an upload handler asks before deciding whether to decode at
all.

**`pngdec.decoder`, `feed` and `finish` decode a file the caller does not hold
whole.** `feed` takes whatever bytes have arrived and answers the events they
produced. This is the path for a socket, a large file, or any caller writing
pixels somewhere as they arrive.

**`pngdec.decode` decodes a whole file held in memory.** It needs the file and
the image at once, and says so.

**`pngdec.read_all` decodes from a stream the caller names.** Its signature is
`read_all<S: Read[e]>(src: S) -> Result<(PngInfo, Bytes), PngError> [e]`, so
its effects are whatever the stream behind `S` costs: a file charges its caller
`[io, fs]`, and an in-memory buffer charges nothing. It is worth having here
because a compressed PNG can be a hundredth of the image it decodes to, so a
caller who cannot hold the file may still hold the picture.

**`pngenc.encoder`, `start`, `push_row` and `finish` write a file row at a
time.** A caller generating an image procedurally never holds it.

**`pngenc.encode` writes a whole image in one call.**

**`pngfilter` and `pngadam7` on their own are the arithmetic with no stream
around them.** A converter or a file inspector reaches for those.

## The rules a user needs

1. **All eight signature bytes are a test, and a decoder must check all
   eight.** 137 has its high bit set, so a transfer that stripped it is
   caught. `13 10` is a CRLF and `10` a bare LF, so a transfer that translated
   line endings in either direction is caught. 26 is the DOS end-of-file, so a
   terminal stops printing there. A decoder that checked only the first four
   bytes accepts a file an FTP client has corrupted. That is PNG specification
   section 5.2.
2. **A chunk's length excludes its type and its CRC. Its CRC includes its type
   and excludes its length.** Those are the two things a hand-written chunk
   reader gets wrong.
3. **An unrecognised ancillary chunk is skipped, not refused.** The case flags
   decide, so a conforming decoder handles chunk types registered after it was
   written. `pngchunk.is_critical`, `is_public`, `is_conforming` and
   `is_safe_to_copy` are the four predicates. The typed readers in `pngchunk`
   are a convenience for the handful worth acting on, not the contract.
4. **A colour type and a bit depth are not independent.** See the table above.
   A 16-bit palette and a 1-bit truecolour image do not exist, and a decoder
   that accepted one would be reading a file no encoder can have written.
   `png.is_valid_combination` is that table.
5. **At depths below 8 the last byte of a row is padded, and the padding does
   not carry into the next row.** A three-pixel row at one bit per pixel is one
   byte with five bits of nothing in it. A row is
   `ceil(width * channels * depth / 8)` bytes plus one for the filter byte, and
   `png.bytes_per_row` is that expression written once.
6. **Every filter is arithmetic modulo 256 on bytes, not on pixels and not on
   samples.** At 16 bits per sample the two halves of a sample are filtered
   separately and nothing carries between them. PNG specification section 9.2
   is where that is stated.
7. **A neighbour off the top or the left of the image is zero.** That is what
   makes the first row and the first pixel of every row work, and it is the
   case a hand-written unfilter forgets.
8. **`Average` sums at full width and then truncates.** `(255 + 255) / 2` is
   255, not 127.
9. **`Paeth` is a selector, not an average.** It picks whichever of the left,
   the above and the above-left byte is nearest to `a + b - c`, with ties going
   to `a`, then `b`, then `c`. That is why it survives an edge in the image
   instead of smearing it.
10. **Each Adam7 pass is a complete small image.** It has its own width and
    height, its own filter byte on every row, and filters that refer to
    neighbours within the pass. The pixel above in pass 3 is eight image rows
    up. A decoder that filtered across pass boundaries produces a picture that
    is almost right.
11. **A pass whose width or height works out to zero is skipped entirely.** For
    a 5 by 5 image three of the seven passes are empty, and an encoder that
    emitted a zero-length row for one of them writes a file no decoder can
    read. `pngadam7.pass_is_empty` is the question.
12. **`PngEvRow` carries a pass number, and a caller must use it.** For a
    non-interlaced image the pass is always 1 and `y` is the image row, so the
    general loop and the simple one are the same code. A caller that ignored
    the pass composes an interlaced image into the top eighth of its canvas.
    `pngadam7.place` turns a pass coordinate into an image coordinate.
13. **`PngEvRow` carries raw samples in the file's own colour type and bit
    depth.** They are not converted to colour values. A row of a 4K 16-bit RGBA
    image is 64 KiB, and one colour value per sample would cost twenty times
    that for data the caller is about to write into a buffer anyway.
14. **The palette and the transparency are colours; the pixels are not.**
    `PngEvPalette` carries at most 256 of color-nv's `Srgb8`, and
    `pngchunk.palette_pixel` joins a palette entry with its tRNS alpha.
15. **`feed` hands back rows it cannot yet vouch for.** Each IDAT chunk's CRC
    has been checked, but the zlib stream's Adler-32 covers the whole image and
    is verified only at the end. Only a successful `finish` says the pixels
    were correct.
16. **`feed` does not answer a `Result`, and a failure is sticky.** A file that
    goes bad at row 4 000 does not throw away the 3 999 rows before it.
    `pngdec.error` is what asks, and once it is set every later `feed` drains
    nothing.
17. **A bad CRC on an ancillary chunk is recoverable, and lenient is the
    default.** A corrupt tEXt is a skippable chunk by definition, and refusing
    the file over it throws away an image that is intact.
    `pngerror.is_recoverable` is the distinction, and `pngdec.with_strict` is
    how a caller archiving or validating files asks for every CRC to be
    enforced.
18. **`pngdec.with_limit` caps how many bytes of image data a decoder will
    produce.** It is off until a caller asks for it, and `limit_of` answers −1
    when it is off. A decoder reading a file from the network wants it, because
    a small PNG can describe an enormous image.
19. **The encoder honours the `PngInfo` it is given exactly.** It will not
    quantise a truecolour image into a palette because it noticed there were
    fewer than 256 colours. A choice of format is a choice a person makes.
20. **The encoder refuses an interlaced `PngInfo`.** `pngenc.encoder` answers
    an encoder that has already failed. See "What is not included".
21. **Which filter a row gets is not part of PNG.** `pngfilter.choose`
    implements libpng's heuristic, and an encoder that chose `PngFilterNone`
    for every row writes a larger file that every decoder reads. A test may
    assert that this package's output decodes. It may not assert which filter a
    row got.
22. **Every error carries a byte offset counted from the first byte of the
    file.** A PNG is not a thing a person edited, so a line number would mean
    nothing. `pngerror.offset_of` answers −1 for the one error that is not
    about a place.

## What is not included

- **Interlaced encoding.** Adam7 is decoded here and not written. An
  interlaced PNG is five to twenty per cent larger than the same image,
  browsers have rendered progressively from a non-interlaced stream for a
  decade, and the specification calls interlacing optional. The decoder reads
  all seven passes, because interlaced files exist and a decoder that refused
  them would fail on real ones.
- **APNG.** Animated PNG is a separate specification, and a separate package
  if anyone wants it.
- **ICC profile contents.** An iCCP chunk arrives as bytes through
  `PngEvAncillary`. This package will not pretend to parse an ICC profile.
  sPLT, hIST, bKGD and every private chunk arrive the same way.
- **A device build.** There is no `tests/embedded_probe.nv` and no claim that
  any module runs on a microcontroller. DEFLATE's window is 32 KiB, a row of a
  wide image is more again, and neither `Bytes` nor `Result` is available on
  the device target today.
- **A colour value per pixel.** See rule 13.
- **A second inflater.** Every IDAT chain is one zlib stream, so PNG's
  compression is DEFLATE, and the inflater and the deflater come from flate-nv.

## Related packages

- [flate-nv](https://novo-lang.org/packages/flate-nv) is DEFLATE and zlib. It
  compresses and decompresses the IDAT stream, and its `FlateLevel` is the one
  compression option `PngOptions` carries.
- [color-nv](https://novo-lang.org/packages/color-nv) owns the colour types.
  The palette, the background colour and the transparency entries are its
  `Srgb8` and `Srgba8`.
- [qoi-nv](https://novo-lang.org/packages/qoi-nv) is the QOI codec. QOI
  compresses far less and decodes in a few hundred bytes of state. PNG
  compresses harder, carries metadata and supports interlacing.
- [image-io-nv](https://novo-lang.org/packages/image-io-nv) reads and writes
  image files by looking at what they are. It calls this package for a PNG.

## Tests

```bash
novo test tests/png_tests.nv        # the signature, IHDR and the size arithmetic
novo test tests/pngchunk_tests.nv   # chunk framing, the case flags, the typed readers
novo test tests/pngfilter_tests.nv  # the five filters and Adam7's pass arithmetic
novo test tests/pngcodec_tests.nv   # the decoder and the encoder
```

The reference implementation is libpng, and the oracle is
[PngSuite](http://www.schaik.com/pngsuite/), the corpus that exists to break
decoders.

The correctness condition is that libpng reads what this package writes, and
that this package reads every file in PngSuite that libpng reads. Byte
equality with libpng's output is not a condition, because it would test two
encoders' filter heuristics rather than PNG.

The numbers in the suites are the specification's where the specification
fixes them: the eight signature bytes, the fifteen legal colour type and depth
pairs, the ceiling division that pads a sub-byte row, the filters' arithmetic
and Adam7's seven-row table. `pngfilter.choose` is libpng's heuristic, so the
only assertion made about it is that it answers a filter at all. The four case
flags are checked against a chunk that answers yes and one that answers no,
one predicate at a time. The codec suite asserts the contract: that a fresh
decoder knows nothing, that an empty chunk is not an end of file, that the
size bound is off until a caller asks for it, that `finish` is where the
framing is checked, and that an interlaced image is refused by the encoder and
not by the decoder.

The tests compile today and fail at run, each on the
`not implemented: png-nv.<module>.<fn>` panic that is its body. That is the
expected state of an interface release. They turn green one at a time as
bodies land. `novo test --isolate tests/<file>` prints one verdict per test.

## Implementation status

| Item | Implemented |
| --- | --- |
| `png.PngColorType`, `.PngBitDepth`, `.PngInterlace`, `.PngInfo` | the types are declared; nothing constructs one |
| `png.signature`, `.is_png`, `.color_type_byte`, `.depth_bits`, `.channel_count` | no |
| `png.is_valid_combination`, `.bits_per_pixel`, `.filter_stride` | no |
| `png.bytes_per_row`, `.filtered_size`, `.pass_count` | no |
| `pngchunk.PngChunk`, `.PngText`, `.PngPhys`, `.PngTime`, `.PngSrgbIntent`, `.PngTransparency` | the types are declared; nothing constructs one |
| `pngchunk.is_critical`, `.is_public`, `.is_conforming`, `.is_safe_to_copy` | no |
| `pngchunk.crc32`, `.framed_size`, `.write`, `.intent_byte` | no |
| `pngchunk.transparent_count`, `.palette_pixel` | no |
| `pngchunk.read_text`, `.read_phys`, `.read_time`, `.read_srgb_intent`, `.read_gamma` | no |
| `pngfilter.PngFilter` | the type is declared; nothing constructs one |
| `pngfilter.filter_byte`, `.filter_of`, `.paeth` | no |
| `pngfilter.unfilter_row`, `.filter_row`, `.choose`, `.default_for` | no |
| `pngadam7.pass_width`, `.pass_height`, `.pass_is_empty`, `.place` | no |
| `pngadam7.column_grid`, `.row_grid`, `.total_filtered_size` | no |
| `pngdec.PngEvent`, `.PngDecoder` | the types are declared; nothing constructs one |
| `pngdec.decoder`, `.with_strict`, `.with_limit`, `.limit_of`, `.event_name` | no |
| `pngdec.feed`, `.finish`, `.info`, `.error` | no |
| `pngdec.total_in`, `.rows_out`, `.is_done` | no |
| `pngdec.read_info`, `.decode`, `.read_all` | no |
| `pngenc.PngOptions`, `.PngEncoder` | the types are declared; nothing constructs one |
| `pngenc.default_options`, `.encoder`, `.start` | no |
| `pngenc.write_palette`, `.write_transparency`, `.write_chunk` | no |
| `pngenc.push_row`, `.push_row_with`, `.finish` | no |
| `pngenc.error`, `.rows_in`, `.total_out`, `.encode` | no |
| `pngerror.PngError` | the type is declared; nothing constructs one |
| `pngerror.offset_of`, `.is_recoverable`, `.needs_more_bytes` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->

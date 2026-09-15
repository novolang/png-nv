# Changelog

## 0.0.2 — 2026-09-15

- README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1

- The interface: the signature, chunk framing with the four case
  flags, IHDR through IEND, all five filters both directions, every
  legal colour type and bit depth, Adam7 decoded, and both directions
  over flate-nv by a decoder that drains events and never holds the
  file.
- Every body is `todo()`. Nothing is implemented.

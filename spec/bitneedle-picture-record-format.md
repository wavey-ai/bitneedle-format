# Bitneedle Picture Record Format

SPDX-License-Identifier: CC0-1.0

## Abstract

This document specifies the Bitneedle Picture Record Format, an exact-digital image format that stores a recoverable payload byte stream in the visible spiral groove of a record-like raster image.

The current format uses a `576 x 576` PNG/RGBA carrier, record-profile-specific annular geometry, outer and inner metadata spirals, and a main groove-order payload spiral carrying `ECDC` bytes.

## Status of This Document

This is a draft public specification for implementers and creators. It is not an Internet Standards Track document.

The key words `MUST`, `MUST NOT`, `REQUIRED`, `SHOULD`, `SHOULD NOT`, and `MAY` are to be interpreted as described in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals.

## 1. Introduction

A Bitneedle picture record is a digital image in which the image itself is the data carrier. A conforming decoder traces defined spiral paths through the image and reconstructs an exact payload byte stream.

The current profile is exact-digital. Interoperable decoding assumes that required RGBA pixel values are preserved exactly, for example in a lossless PNG or a controlled screen capture. Photography, printing, lossy recompression, and scan recovery are outside the scope of this version.

## 2. Terminology

`record profile` means a named annular geometry such as `single45` or `lp`.

`payload spiral` means the principal groove-order spiral carrying payload bytes.

`header spiral` means the outer metadata spiral band.

`trailer spiral` means the inner metadata spiral band.

`metadata stream` means the logical header byte stream encoded across the header and trailer spirals.

`b_value` means the Archimedean spiral growth-rate parameter used to trace the payload spiral.

`exact-digital` means a workflow in which required RGBA pixel values are preserved exactly.

`ECDC` means the payload byte container carried by the current Bitneedle payload spiral. The ECDC audio codec is outside this image-format specification.

## 3. Conformance

A conforming encoder MUST produce a PNG/RGBA image that follows the raster, geometry, metadata, and payload mapping rules in this document.

A conforming decoder MUST recover the metadata stream, reconstruct the payload spiral from the recovered parameters, and recover the declared number of payload bytes according to the payload encoding.

A decoder MAY ignore optional creator metadata fields it does not understand. A decoder MUST ignore unknown metadata segment types after skipping their declared payload length.

## 4. Image Container

The current exact-digital raster profile is:

- color model: exact `RGB` samples
- channel depth: `8 bits per channel`
- raster size: `576 x 576`

`PNG` with RGB or RGBA samples is the canonical interchange container. Other lossless raster containers MAY be used if decoding the container produces the same exact 8-bit RGB samples at every encoded spiral pixel. Lossy image formats are non-conforming for exact-digital interchange.

The carrier MAY include an alpha channel for presentation or compositing. Alpha is outside the Bitneedle data model and is not used by this specification. Decoders MUST NOT treat black RGB pixels as empty, because black is valid payload data.

## 5. Coordinate System

The raster coordinate system is:

- origin at the upper-left pixel
- positive `x` to the right
- positive `y` downward

The record center is:

- `center_x = width / 2`
- `center_y = height / 2`

The default start angle is `PI / 2`. Spirals rotate clockwise.

The normative rounding mode is:

```text
round_half_up(v) = floor(v + 0.5)
```

## 6. Record Profiles

The current profile registry is:

| Profile | Spindle-hole radius | Label radius | Payload inner radius | Payload outer radius | Outer radius |
| --- | ---: | ---: | ---: | ---: | ---: |
| `single45` | `63` | `151` | `169` | `280` | `287` |
| `lp` | `7` | `95` | `109` | `280` | `287` |

The canonical profile names are `single45` and `lp`. Implementations MAY accept aliases, but interoperable metadata MUST use canonical profile names.

## 7. Spiral Bands

The format reserves:

- two turns for the outer metadata spiral
- two turns for the inner metadata spiral
- one main payload spiral between the outer lead-in and inner label clearance

The outer metadata spiral starts at:

```text
header_outer_radius = outer_radius - 1
header_inner_radius = payload_outer_radius
```

The main payload spiral occupies the annulus between:

```text
payload_outer_radius = outer_radius - 1 - lead_in_band_thickness
payload_inner_radius = label_radius + label_clearance
```

Current profile clearances are:

| Profile | Outer payload lead-in clearance | Inner run-out metadata band |
| --- | ---: | ---: |
| `single45` | `6` pixels | `18` pixels |
| `lp` | `6` pixels | `14` pixels |

Optional guide rings MAY be rendered for the record edge, label perimeter, and spindle hole, but they MUST NOT alter payload or metadata spiral samples.

## 8. Spiral Trace Algorithm

The payload spiral is an Archimedean spiral traced from the outer payload boundary inward.

At accumulated angular distance `theta`:

```text
angle  = start_angle - theta
radius = payload_outer_radius - b_value * theta
x      = round_half_up(center_x + radius * cos(angle))
y      = round_half_up(center_y - radius * sin(angle))
```

The reference trace step uses:

```text
pixel_gap  = 1.0
theta_step = pixel_gap / sqrt(radius^2 + b_value^2)
```

The path is sampled iteratively. If a trace lands on a pixel already visited, the first occurrence wins and later duplicates are ignored. The ordered groove path is the ordered list of unique sampled pixels that remain after de-duplication.

Only sampled pixels with distance from center greater than `payload_inner_radius` and less than `payload_outer_radius` are addressable main-groove payload pixels.

## 9. Metadata Stream

### 9.1 Logical Model

Metadata is one logical byte stream split physically across the outer header spiral and inner trailer spiral. Fields are not aligned to the visual split. Metadata MAY spill from the outer band into the inner band.

Metadata bytes occupy the outer header spiral first. Remaining bytes, if any, continue into the inner trailer spiral.

### 9.2 Prefix

The metadata stream begins with a fixed 19-byte prefix:

| Offset | Size | Type | Meaning |
| ---: | ---: | --- | --- |
| `0` | `4` | bytes | ASCII magic `BTB2` |
| `4` | `1` | `u8` | header schema version |
| `5` | `2` | `u16be` | total metadata payload length |
| `7` | `2` | `u16be` | segment count |
| `9` | `2` | `u16be` | segment stream byte length |
| `11` | `8` | `f64be` | `b_value` IEEE-754 bits |

The header schema version is a one-byte discriminator for the segmented header schema. Optional metadata segments do not require a schema-version bump when they use the segment framing defined below and do not alter the meaning of core fields.

### 9.3 Segment Framing

The bytes after the 19-byte prefix are a continuous segment stream. Each segment is:

| Size | Type | Meaning |
| ---: | --- | --- |
| `1` | `u8` | segment type |
| `2` | `u16be` | segment payload length |
| `N` | bytes | segment payload |

All segment payload lengths are byte lengths.

Encoders MUST include all segments in the segment count, segment stream byte length, total metadata payload length, and checksum calculation.

Decoders MUST reject truncated segments. Decoders MUST ignore unknown segment types after skipping their declared payload length.

### 9.4 Core Segment Registry

| Type | Name | Payload |
| ---: | --- | --- |
| `1` | checksum | `u32be` |
| `2` | ECDC byte length | `u64be`, or `u64::MAX` when absent |
| `3` | generation version | UTF-8 string |
| `4` | record profile | UTF-8 canonical profile name |
| `5` | title | UTF-8 string |
| `6` | artist | UTF-8 string |
| `7` | payload encoding | UTF-8 canonical encoding name |

The checksum is computed over the full metadata payload with the four checksum payload bytes zeroed.

The `record profile` segment MUST contain a canonical profile name from Section 6.

The `payload encoding` segment MUST contain a canonical encoding name from Section 10.

### 9.5 Optional Creator Metadata Registry

| Type | Name | Payload |
| ---: | --- | --- |
| `8` | release ID | UTF-8 string |
| `9` | catalog number | UTF-8 string |
| `10` | label | UTF-8 string |
| `11` | artwork credit | UTF-8 string |
| `12` | license | UTF-8 string |
| `13` | canonical URL | UTF-8 string |
| `14` | created-at timestamp | UTF-8 ISO-8601 string |
| `15` | arbitrary metadata | compact UTF-8 JSON |

All UTF-8 text fields are dynamically sized by their segment payload length. Empty payloads represent absent or empty fields.

Control characters MUST NOT appear in text fields. Encoders SHOULD keep creator metadata compact. Ordinary title and artist metadata is expected to produce a header of approximately `100` to `200` bytes, usually about `4%` to `8%` of current physical metadata capacity across the lead-in and dead-wax metadata bands.

The canonical URL segment is the preferred way to carry data for QR-code workflows. Implementations MAY render a QR code derived from this URL, but this specification does not define a QR bitmap or QR module-data segment. Encoders SHOULD store the URL, not a QR image, in the metadata header.

## 10. Metadata Pixel Mapping

Metadata bytes are encoded as paired mid-range grayscale pixels:

- two metadata pixels per byte
- the first pixel stores the high nibble as `R = G = B = 120 + high_nibble`
- the second pixel stores the low nibble as `R = G = B = 120 + low_nibble`
- written metadata pixels use non-zero alpha
- metadata pixels MUST NOT use independent color channels

Unused metadata-band pixels are non-semantic. Decoders MUST ignore metadata-band pixels beyond the declared metadata payload length.

## 11. Main Payload Mapping

### 11.1 Payload Container

The current Bitneedle picture-record format carries raw `ECDC` bytes in the main payload spiral. Non-ECDC payload containers are outside this version.

### 11.2 Payload Encoding Registry

| Name | Bytes per pixel | Mapping |
| --- | ---: | --- |
| `rgb` | `3` | byte `0` to red, byte `1` to green, byte `2` to blue |
| `grayscale` | `1` | byte `0` to red, green, and blue equally |

For `rgb`, if the byte stream length is not divisible by `3`, the final pixel is padded with zero bytes in unused RGB channels.

For `grayscale`, each written payload pixel MUST have `R = G = B = byte`.

### 11.3 Payload Length

Encoders MUST write the exact ECDC byte length in metadata segment `2`.

Decoders MUST use the declared ECDC byte length and payload encoding to recover the exact payload byte count and discard terminal padding.

## 12. Decoding Procedure

Given a record image and either a known record profile or a successful profile inference, a decoder MUST:

1. trace the outer metadata spiral for the profile
2. recover the `BTB2` prefix
3. parse `b_value`, total metadata length, segment count, and segment stream length
4. recover the continuous metadata segment stream from the outer and inner metadata spirals
5. verify the metadata checksum
6. parse the record profile, ECDC byte length, and payload encoding
7. reconstruct the main groove path from the recovered `b_value`
8. read payload pixels in groove order
9. decode payload pixels according to the declared payload encoding
10. truncate the result to the declared ECDC byte length
11. validate the recovered ECDC payload before playback or further processing

Decoders MUST NOT use black RGB pixels or presentation attributes as end-of-stream markers.

## 13. Exact-Digital Recovery Requirements

Conforming exact-digital interchange requires:

- lossless image storage
- no color-space transform that changes effective channel values
- no scaling, resampling, filtering, or recompression
- no compositing or blending that changes written payload or metadata RGB samples

Arbitrary artwork MAY exist outside written spiral paths. Pictorial content outside payload and metadata spirals is allowed, but written spiral pixels MUST preserve encoded RGB values exactly.

## 14. Extension Points

Future specifications MAY define additional raster sizes, record profiles, metadata segment types, profile auto-detection rules, payload encodings, payload containers, or photo-robust optical modes.

New optional metadata segment types MUST be length-prefixed using the segment framing in this document. New fields that do not alter the meaning of existing core fields SHOULD NOT require a header schema-version bump.

A future non-ECDC payload profile MUST define an explicit payload identifier or equivalent payload-container negotiation mechanism.

## 15. Security Considerations

Bitneedle images can carry compressed or encoded media payloads. Decoders MUST treat recovered payload bytes as untrusted input.

Implementations SHOULD bound memory allocation using declared metadata and payload lengths, reject malformed segment streams, reject impossible dimensions or geometry, and validate recovered ECDC structure before attempting media decode.

Implementations SHOULD avoid automatically dereferencing canonical URLs without user action. QR codes derived from canonical URLs SHOULD be treated as external links.

## 16. Privacy Considerations

Creator metadata can contain personal data, identifiers, timestamps, catalog numbers, URLs, and license information. Encoders SHOULD avoid embedding private or non-public data unless intentionally published by the creator.

Players SHOULD make creator metadata inspectable when it affects navigation, linking, attribution, or collection behavior.

## 17. IANA Considerations

This document has no IANA actions.

The segment-type registry and payload-encoding registry in this document are Bitneedle registries maintained by the Bitneedle Format specification repository.

## 18. Conformance Vectors

Normative conformance vectors are not yet published in this repository.

Existing internal golden files are implementation regression fixtures and MAY be useful as non-normative examples. They SHOULD NOT be cited as authoritative current-format vectors until regenerated against this specification, including the current payload encoding segment and any creator metadata examples selected for publication.

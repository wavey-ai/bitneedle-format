# Bitneedle Picture Record Format

SPDX-License-Identifier: CC0-1.0

## Abstract

This document specifies the Bitneedle Picture Record Format, an exact-digital image format that stores a recoverable payload byte stream in the visible spiral groove of a record-like raster image.

The current format uses a `576 x 576` PNG/RGBA carrier, record-profile-specific annular geometry, outer and inner metadata spirals, and a main groove-order payload spiral carrying `ECDC` bytes.

This document also defines an optional steganographic sidecar container for non-playback extras such as liner notes, photos, credits, and booklet data. The sidecar is independent from the `ECDC` audio payload.

## Status of This Document

This is a draft public specification for implementers and creators. It is not an Internet Standards Track document.

The key words `MUST`, `MUST NOT`, `REQUIRED`, `SHOULD`, `SHOULD NOT`, and `MAY` are to be interpreted as described in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals.

## Intellectual Property Notice

Certain aspects of Bitneedle records, encoding, decoding, rendering, and playback are patent pending.

This specification is published for interoperability and implementation clarity. No express or implied patent license is granted by this document.

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

`steganographic sidecar` means an optional recoverable byte stream carried outside the main `ECDC` payload spiral, using visual carrier regions such as the label area and intra-groove area.

`sidecar carrier` means a visual region used to carry steganographic sidecar bytes. The current carrier names are `label` and `intergroove`.

`BTS1` means the Bitneedle Typed Sidecar v1 container format defined in Section 12.

## 3. Conformance

A conforming encoder MUST produce a PNG/RGBA image that follows the raster, geometry, metadata, and payload mapping rules in this document.

A conforming decoder MUST recover the metadata stream, reconstruct the payload spiral from the recovered parameters, and recover the declared number of payload bytes according to the payload encoding.

A decoder MAY ignore optional creator metadata fields it does not understand. A decoder MUST ignore unknown metadata segment types after skipping their declared payload length.

A decoder MAY ignore the optional steganographic sidecar. A decoder that supports the sidecar MUST treat it as non-playback auxiliary data and MUST NOT concatenate it with, substitute it for, or otherwise reinterpret it as the main `ECDC` payload.

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
- four turns for the inner metadata spiral
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

Control characters MUST NOT appear in text fields. Encoders SHOULD keep creator metadata compact. Ordinary title and artist metadata is expected to produce a header of approximately `100` to `200` bytes, usually about `3%` to `7%` of current physical metadata capacity across the lead-in and dead-wax metadata bands.

The canonical URL segment is the preferred way to carry data for QR-code workflows. Implementations MAY render a QR code derived from this URL, but this specification does not define a QR bitmap or QR module-data segment. Encoders SHOULD store the URL, not a QR image, in the metadata header.

### 9.6 Optional Signed Release Registry

| Type | Name | Payload |
| ---: | --- | --- |
| `16` | signed release manifest | compact UTF-8 JSON |
| `17` | signature algorithm | UTF-8 string, currently `ed25519` |
| `18` | signature key ID | UTF-8 string |
| `19` | signature | base64url-encoded Ed25519 signature bytes |
| `20` | signed release manifest SHA-256 | base64url-encoded digest bytes |

The signed release manifest is signed as the exact UTF-8 byte sequence carried in segment `16`. Verifiers MUST verify the signature over those exact bytes and MUST NOT parse and reserialize the JSON before verification.

The signed release manifest SHOULD include `v`, `type`, `issuer`, `keyId`, `releaseId`, `canonicalUrl`, `title`, `artist`, `label`, `recordProfile`, `payloadContainer`, `ecdcByteLength`, `ecdcSha256`, and `createdAt`.

Signed-release verification SHOULD recover the ECDC payload, compute its SHA-256 digest, verify segment `19` with the trusted public key identified by segment `18`, parse the signed manifest, and compare `ecdcSha256`, `ecdcByteLength`, and, when present, `recordProfile`, `releaseId`, and `canonicalUrl`.

Verifiers MUST NOT trust a public key embedded inside the record image. The record MAY carry a key ID, but the corresponding public key MUST come from a trusted player bundle, trusted application configuration, or trusted HTTPS key endpoint.

### 9.7 Optional Steganographic Sidecar Registry

| Type | Name | Payload |
| ---: | --- | --- |
| `21` | steganographic sidecar descriptor | compact UTF-8 JSON |

The sidecar descriptor declares how to recover an optional `BTS1` sidecar stream from visual carrier regions that are not part of the main `ECDC` payload bytes.

The descriptor JSON SHOULD use compact serialization and stable field names. It MUST contain:

| Field | Type | Meaning |
| --- | --- | --- |
| `v` | integer | sidecar descriptor version, currently `1` |
| `container` | string | sidecar container identifier, currently `BTS1` |
| `scheme` | string | carrier mapping scheme identifier |
| `carriers` | array of strings | enabled physical carriers, currently `label`, `intergroove`, or both |
| `seed` | integer | nonzero unsigned 32-bit pseudo-random ordering seed |
| `length` | integer | exact `BTS1` stream length in bytes |
| `sha256` | string | base64url-encoded SHA-256 digest of the exact recovered `BTS1` stream |
| `geometry` | object | record dimensions, center point, `bValue`, and carrier radii used for recovery |
| `carrierOrdering` | object | pair construction and shuffle identifiers |
| `bitPacking` | object | bits per pair, byte bit order, and pair-sign interpretation |
| `luma` | object | luminance model and coefficients used by the declared scheme |

The descriptor JSON MAY contain additional scheme-specific fields such as `writer`. Unknown top-level descriptor fields MUST be ignored by decoders that do not need them for the declared scheme.

If segment `21` is absent, no public sidecar is declared. If segment `21` is present but unsupported, a decoder MUST still recover and validate the main `ECDC` payload normally.

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

## 12. Optional Steganographic Sidecar Payload

### 12.1 Relationship to the Main Payload

The steganographic sidecar is optional auxiliary data. It is intended for release extras such as liner notes, photos, credits, booklet text, thumbnails, purchase metadata, or other non-playback material.

The sidecar MUST NOT alter the definition of the main payload spiral. The main payload spiral continues to carry raw `ECDC` bytes as defined in Section 11.

Sidecar encoders MUST NOT modify metadata pixels or main payload spiral pixels. Sidecar encoders SHOULD also avoid the lead-in and dead-wax metadata bands so those regions can remain visually clean.

### 12.2 Sidecar Carrier Regions

The public sidecar descriptor currently recognizes these carrier names:

| Carrier | Meaning |
| --- | --- |
| `label` | pixels in the visual label area for the record profile |
| `intergroove` | pixels in the payload annulus that are not sampled by the main payload spiral |

A sidecar stream MAY be distributed across one or more carriers. When multiple carriers are enabled, the declared carrier mapping scheme defines one combined logical byte stream. The `BTS1` stream itself is not split into separate per-carrier files.

Carrier selection MUST be deterministic from the record profile, `b_value`, descriptor fields, and declared carrier scheme. Carrier selection MUST NOT depend on private encoder state that is unavailable to a decoder.

### 12.3 Sidecar Carrier Scheme Registry

The current public sidecar carrier scheme registry is:

| Name | Meaning |
| --- | --- |
| `pairsign-safe-luma-v2` | exact-digital two-bit adjacent-pair luminance sign and magnitude mapping over deterministic safe pairs |

For `pairsign-safe-luma-v2`, encoders construct adjacent horizontal pixel pairs from the selected carrier areas, sort them by deterministic safe-luma score with a seeded tie-break, and write up to two bits per pair. The first bit is encoded by pair luminance sign. The second bit is encoded by pair luminance-difference magnitude for pairs meeting the scheme threshold.

Carrier pair construction is:

```text
for y in 0..height:
  x = 0
  while x + 1 < width:
    first  = (x, y)
    second = (x + 1, y)
    if first and second are both inside an enabled carrier:
      append (first, second)
      x += 2
    else:
      x += 1
```

Pixel centers are evaluated at `(x + 0.5, y + 0.5)`. A `label` pixel is inside the label carrier when its distance from the descriptor center is greater than `geometry.label.innerRadius` and less than `geometry.label.outerRadius`. An `intergroove` pixel is inside the intergroove carrier when its distance is greater than `geometry.intergroove.innerRadius`, less than `geometry.intergroove.outerRadius`, and the pixel is not sampled by the main payload spiral.

The pair list is shuffled using Fisher-Yates with Mulberry32 seeded by descriptor `seed`.

The decoded bit for a pair is:

```text
Y   = 0.2126 * R + 0.7152 * G + 0.0722 * B
bit = 1 if Y_first > Y_second, otherwise 0
```

Bits are concatenated most-significant first to recover bytes. Eight pixel pairs recover one byte. Decoders MUST stop after recovering exactly the descriptor `length` bytes.

Encoders MAY use any visual adjustment strategy that produces the required final pair signs. Encoders SHOULD prefer small anti-symmetric luminance adjustments around each pair's local average so the sidecar appears as natural record or label grain.

### 12.4 `BTS1` Container

The recovered sidecar byte stream begins with this fixed binary prefix:

| Offset | Size | Type | Meaning |
| ---: | ---: | --- | --- |
| `0` | `4` | bytes | ASCII magic `BTS1` |
| `4` | `1` | `u8` | container version, currently `1` |
| `5` | `1` | `u8` | container flags, currently `0` |
| `6` | `2` | `u16be` | item count |
| `8` | `4` | `u32be` | total `BTS1` stream length in bytes, including this prefix |

The prefix is followed by `item count` item records. Each item record is:

| Size | Type | Meaning |
| ---: | --- | --- |
| `1` | `u8` | item type |
| `1` | `u8` | item codec |
| `1` | `u8` | item flags, currently `0` |
| `1` | `u8` | reserved, MUST be `0` |
| `4` | `u32be` | raw byte length, or `u32::MAX` when not applicable |
| `4` | `u32be` | stored byte length |
| `2` | `u16be` | item name byte length |
| `2` | `u16be` | MIME type byte length |
| `N` | bytes | UTF-8 item name |
| `M` | bytes | ASCII MIME type |
| `K` | bytes | stored item data |

Item names are advisory and MAY be empty. MIME types are advisory but SHOULD be present for images and documents.

### 12.5 Sidecar Item Type Registry

| Type | Name | Meaning |
| ---: | --- | --- |
| `0` | opaque bytes | uninterpreted byte stream |
| `1` | UTF-8 text | human-readable text |
| `2` | image | still image |
| `3` | JSON | UTF-8 JSON document |
| `4..31` | reserved | reserved by the Bitneedle specification |
| `32..255` | private | private or experimental use |

### 12.6 Sidecar Codec Registry

| Codec | Name | Meaning |
| ---: | --- | --- |
| `0` | raw | stored bytes are the item bytes |
| `1` | brotli | stored bytes are Brotli-compressed item bytes |
| `2` | zstd | stored bytes are Zstandard-compressed item bytes |
| `3` | avif | stored bytes are an AVIF image |
| `4..31` | reserved | reserved by the Bitneedle specification |
| `32..255` | private | private or experimental use |

For `UTF-8 text` and `JSON` items, the recovered bytes after applying the item codec MUST be valid UTF-8. Encoders SHOULD use Brotli for natural-language liner notes and compact JSON unless a different codec is demonstrably smaller for the specific item.

For `image` items in this version, interoperable encoders SHOULD use AVIF with MIME type `image/avif`.

### 12.7 Sidecar Recovery

A sidecar-capable decoder SHOULD:

1. recover and verify the normal metadata stream
2. recover and verify the main `ECDC` payload independently
3. parse segment `21`, if present
4. reconstruct the declared sidecar carrier pixel set
5. recover exactly `length` bytes using the declared carrier scheme
6. verify the recovered `BTS1` stream SHA-256 against descriptor `sha256`
7. parse the `BTS1` container and expose supported items to the user

Failure to recover or parse the sidecar MUST NOT by itself make an otherwise valid playable `ECDC` record invalid.

## 13. Decoding Procedure

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

If segment `21` is present and the decoder supports the declared sidecar scheme, the decoder MAY additionally recover the sidecar after completing main payload recovery.

Decoders MUST NOT use black RGB pixels or presentation attributes as end-of-stream markers.

## 14. Exact-Digital Recovery Requirements

Conforming exact-digital interchange requires:

- lossless image storage
- no color-space transform that changes effective channel values
- no scaling, resampling, filtering, or recompression
- no compositing or blending that changes written payload or metadata RGB samples

Arbitrary artwork MAY exist outside written spiral paths. Pictorial content outside payload and metadata spirals is allowed, but written spiral pixels MUST preserve encoded RGB values exactly.

Optional sidecar pixels also require exact RGB preservation for exact-digital recovery. Lossy recompression, color-management transforms, resampling, filtering, or compositing can destroy sidecar data even when the visible record remains recognizable.

## 15. Extension Points

Future specifications MAY define additional raster sizes, record profiles, metadata segment types, profile auto-detection rules, payload encodings, payload containers, sidecar carrier schemes, sidecar item types, sidecar codecs, or photo-robust optical modes.

New optional metadata segment types MUST be length-prefixed using the segment framing in this document. New fields that do not alter the meaning of existing core fields SHOULD NOT require a header schema-version bump.

A future non-ECDC payload profile MUST define an explicit payload identifier or equivalent payload-container negotiation mechanism.

New public sidecar item types and sidecar codecs MUST be assigned from the reserved ranges in Sections 12.5 and 12.6. Values `32..255` are reserved for private or experimental use and MUST NOT be assumed interoperable.

## 16. Security Considerations

Bitneedle images can carry compressed or encoded media payloads. Decoders MUST treat recovered payload bytes and recovered sidecar item bytes as untrusted input.

Implementations SHOULD bound memory allocation using declared metadata, payload, sidecar, and item lengths; reject malformed segment streams; reject malformed sidecar containers; reject impossible dimensions or geometry; and validate recovered ECDC structure before attempting media decode.

Sidecar decoders that support Brotli, Zstandard, AVIF, or future codecs SHOULD apply normal decompression and image-decoder limits, including output-size limits and time limits.

Implementations SHOULD avoid automatically dereferencing canonical URLs without user action. QR codes derived from canonical URLs SHOULD be treated as external links.

## 17. Privacy Considerations

Creator metadata and sidecar items can contain personal data, identifiers, timestamps, catalog numbers, URLs, photos, liner notes, credits, and license information. Encoders SHOULD avoid embedding private or non-public data unless intentionally published by the creator.

Players SHOULD make creator metadata and supported sidecar items inspectable when they affect navigation, linking, attribution, collection behavior, or user-visible release extras.

## 18. IANA Considerations

This document has no IANA actions.

The segment-type registry, payload-encoding registry, sidecar carrier scheme registry, sidecar item type registry, and sidecar codec registry in this document are Bitneedle registries maintained by the Bitneedle Format specification repository.

## 19. Conformance Vectors

The following Westside golden records are the current public complete-payload conformance vectors for this draft:

| Vector | Profile | PNG SHA-256 | Signed manifest SHA-256 | Signing key ID |
| --- | --- | --- | --- | --- |
| `lori-asha-westside-single45-hq.record.png` | `single45` | `c7d02ecb0152010e931fe45acb6471dbba9e85caed56b32e39a4d0b2587dc501` | `r1SfM-LMJCzaH4NWnAhHLdwI3CLgV9ffFg7pnkW5oPk` | `bn-prod-2026-05-18-01` |
| `lori-asha-westside-lp-hq.record.png` | `lp` | `973fc10add9d652137c51f0937709c1f449cc88a6973316a9d105d650acf80fb` | `oUeBMmfgOi82l47E-ZAzyqfGinPr-XF67-P-12ftC50` | `bn-prod-2026-05-18-01` |

The current public Ed25519 release verification key for these vectors is:

| Key ID | Public key |
| --- | --- |
| `bn-prod-2026-05-18-01` | `cIrx9lAynSgsSBBaaNuOIoM4Xxz9RNc6bRgkIuwymiQ` |

Only complete release-payload records are public conformance vectors. Preview-only progressive artifacts are internal regression fixtures and are not conformance vectors.

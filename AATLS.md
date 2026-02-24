# AATLS — Arc Atlas Binary Format  

## 1. Abstract

AATLS (Arc Atlas) is a compact binary texture atlas format used by Mindustry, built on the Arc game framework.

This document specifies version 0 of the AATLS binary format. It defines the file structure, encoding rules, and validation requirements necessary for compliant implementations.

The format is optimized for:

- Minimal parsing overhead  
- Fast sequential loading  
- Low memory usage  
- Efficient GPU batching via texture atlases  

## 2. Conventions and Terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** in this document are to be interpreted as described in RFC 2119.

All numeric fields are signed unless explicitly stated otherwise.

## 3. Data Encoding Rules

### 3.1 Endianness

All multi-byte integer values **MUST** be encoded using **big-endian byte order**.

### 3.2 Numeric Representation

Unless otherwise specified, all numeric fields are signed integers.

| Width | C Equivalent |
|--------|--------------|
| 8-bit  | `int8_t`     |
| 16-bit | `int16_t`    |
| 32-bit | `int32_t`    |
| Boolean | `int8_t`    |

Boolean interpretation:

- `0`   = false  
- non-zero = true  

Implementations **MUST NOT** assume unsigned arithmetic semantics.

### 3.3 String Encoding

Strings are encoded as:

int16 length uint8[length] data

Requirements:

- Length is signed but **MUST NOT** be negative in valid files.
- Strings are UTF-8 encoded.
- Strings are NOT null-terminated.

## 4. Overall File Structure

An AATLS file is composed of:

Atlas Header Page*

Pages are stored sequentially after the header and continue until end-of-file.

There is no central page table.

Regions are stored sequentially inside each page.

## 5. Atlas Header

| Offset | Size   | Type    | Field      | Description |
|--------|--------|----------|------------|-------------|
| 0      | 5      | char[5]  | Signature  | ASCII `"AATLS"` |
| 5      | 1      | int8     | Version    | MUST be `0` |
| 6      | —      | —        | Pages      | Sequence of page structures |

### 5.1 Signature

The file **MUST** begin with the ASCII string:

AATLS

### 5.2 Version

Version **MUST** be `0` for this specification.

Implementations **MUST** reject unknown versions.

## 6. Page Structure

Each page describes one texture image.

### 6.1 Layout

| Order | Type        | Field        |
|--------|------------|--------------|
| 1      | int8       | EOF marker   |
| 2      | UTF string | Page name    |
| 3      | int16      | Page width   |
| 4      | int16      | Page height  |
| 5      | int8       | Filter min   |
| 6      | int8       | Filter mag   |
| 7      | int8       | U wrap       |
| 8      | int8       | V wrap       |
| 9      | int32      | Region count |
| 10     | —          | Regions      |

Pages continue until EOF.

### 6.2 Field Semantics

#### EOF Marker

A dummy byte used to safely detect stream termination.

Its value is not semantically significant.

#### Page Name

UTF-8 filename of the texture backing the page.

#### Page Width / Height

Signed 16-bit texture dimensions in pixels.

Values SHOULD be positive.

#### Filter Min / Filter Mag

Engine-defined texture filtering modes.

Interpretation is implementation-specific.

#### U Wrap / V Wrap

Engine-defined texture coordinate wrapping modes.

Interpretation is implementation-specific.

#### Region Count

Signed 32-bit number of region entries that follow.

Region count **MUST NOT** be negative.

7. Texture Parameter Enumerations

This section defines the numeric interpretation of the Filter Min, Filter Mag, U Wrap, and V Wrap fields defined in Section 6.

All enumeration values are encoded as signed 8-bit integers (int8).

Implementations MUST reject values outside the defined range.


7.1 Wrap Modes

The U Wrap and V Wrap fields encode texture coordinate wrapping behavior.

7.1.1 Encoding

|Value|Symbol|Semantics|
|-|-|-|
|0|AATLS_WRAP_MIRRORED_REPEAT|Mirrored repeat wrapping|
|1|AATLS_WRAP_CLAMP_TO_EDGE|Clamp to edge|
|2|AATLS_WRAP_REPEAT|Repeat wrapping|

Valid range:
```
0 <= value < __AATLS_WRAP_END__
```
Where:
```
AATLS_WRAP_END = 3
```
Values outside this range **MUST** be treated as invalid.

### 7.1.2 Rendering Mapping (OpenGL Reference)

When used with OpenGL-compatible renderers, the following mapping **SHOULD** be applied:

| AATLS Value                     | OpenGL Equivalent |
|----------------------------------|------------------|
| AATLS_WRAP_MIRRORED_REPEAT       | GL_MIRRORED_REPEAT |
| AATLS_WRAP_CLAMP_TO_EDGE         | GL_CLAMP_TO_EDGE |
| AATLS_WRAP_REPEAT                | GL_REPEAT |

The numeric values of the OpenGL constants are NOT stored in the file.

Other rendering backends MAY provide equivalent behavior.

## 7.2 Filter Modes

The `Filter Min` and `Filter Mag` fields encode texture filtering behavior.

### 7.2.1 Encoding

| Value | Symbol                                 | Semantics |
|--------|------------------------------------------|------------|
| 0      | AATLS_FILTER_NEAREST                   | Nearest sampling |
| 1      | AATLS_FILTER_LINEAR                    | Linear sampling |
| 2      | AATLS_FILTER_MIPMAP                    | Linear mipmap linear |
| 3      | AATLS_FILTER_MIPMAP_NEAREST_NEAREST    | Nearest mipmap nearest |
| 4      | AATLS_FILTER_MIPMAP_LINEAR_NEAREST     | Linear mipmap nearest |
| 5      | AATLS_FILTER_MIPMAP_LINEAR_LINEAR      | Linear mipmap linear |

Valid range:
```
    0 <= value < AATLS_FILTER_END
```
Where:
```
    AATLS_FILTER_END = 6
```
Values outside this range **MUST** be treated as invalid.

### 11.2.2 Rendering Mapping (OpenGL Reference)

When used with OpenGL-compatible renderers, the following mapping **SHOULD** be applied:

| AATLS Value                              | OpenGL Equivalent |
|-------------------------------------------|------------------|
| AATLS_FILTER_NEAREST                     | GL_NEAREST |
| AATLS_FILTER_LINEAR                      | GL_LINEAR |
| AATLS_FILTER_MIPMAP                      | GL_LINEAR_MIPMAP_LINEAR |
| AATLS_FILTER_MIPMAP_NEAREST_NEAREST      | GL_NEAREST_MIPMAP_NEAREST |
| AATLS_FILTER_MIPMAP_LINEAR_NEAREST       | GL_LINEAR_MIPMAP_NEAREST |
| AATLS_FILTER_MIPMAP_LINEAR_LINEAR        | GL_LINEAR_MIPMAP_LINEAR |

The stored values represent abstract filter modes and are not tied to a specific graphics API.

## 7.3 Validation Requirements

Implementations:

- **MUST** validate that wrap values are within the defined wrap enumeration.
- **MUST** validate that filter values are within the defined filter enumeration.
- **MUST** reject unknown enumeration values.
- **MUST NOT** assume future compatibility with extended values unless explicitly negotiated via a new format version.

## 7. Region Structure

Regions are parsed strictly in stream order.

Optional sections appear immediately after their controlling flag.

### 8.1 Base Fields

| Order | Type        | Field  |
|--------|------------|--------|
| 1      | UTF string | name   |
| 2      | int16      | left   |
| 3      | int16      | top    |
| 4      | int16      | width  |
| 5      | int16      | height |

These fields define a rectangle inside the page.

### 8.2 Offset Section (Conditional)

Immediately after `height`:

| Type  | Field        |
|--------|-------------|
| int8  | has_offsets |

If `has_offsets != 0`, the following fields **MUST** appear immediately:

| Type   | Field           |
|---------|----------------|
| int16   | offset_x       |
| int16   | offset_y       |
| int16   | original_width |
| int16   | original_height|

Purpose:  
Allows trimmed textures to preserve their original untrimmed dimensions.

### 8.3 Split Section (Conditional)

Immediately after the offset section (or directly after `has_offsets` if false):

| Type  | Field       |
|--------|------------|
| int8  | has_splits |

If `has_splits != 0`, the following fields **MUST** appear:

| Type   | Field          |
|---------|---------------|
| int16   | splits_left   |
| int16   | splits_right  |
| int16   | splits_top    |
| int16   | splits_bottom |

### 8.4 Padding Section (Conditional)

Immediately after the split section (or directly after `has_splits` if false):

| Type  | Field      |
|--------|-----------|
| int8  | has_pads  |

If `has_pads != 0`, the following fields **MUST** appear:

| Type   | Field       |
|---------|------------|
| int16   | pads_left  |
| int16   | pads_right |
| int16   | pads_top   |
| int16   | pads_bottom|

Padding defines content margins.

## 9. Region Decoding Order

The exact decoding sequence **MUST** be:
```
    name 
    left 
    top 
    width 
    height
    has_offsets 
        offset_x 
        offset_y 
        original_width 
        original_height

    has_splits 
        splits_left 
        splits_right 
        splits_top 
        splits_bottom

    has_pads 
        pads_left 
        pads_right 
        pads_top 
        pads_bottom
```
There is no reordering or lookahead.

## 10. Validation Requirements

A region **MUST** be considered corrupt if any of the following conditions are true:

- A stream read error occurs.
- `left >= page.width`
- `left + width >= page.width`
- `top >= page.height`
- `top + height >= page.height`

Regions **MUST** lie fully inside page bounds.

Regions **MUST NOT** exceed page dimensions.

If a validation failure occurs, the atlas **MUST** be rejected.

## 11. Error Handling

If any parsing failure or validation error occurs, implementations **MUST** treat the file as invalid.

Partial recovery is **NOT RECOMMENDED**.

## 11. References
- [Anuken](https://github.com/Anuken), [Arc](https://github.com/Anuken/Arc) source code

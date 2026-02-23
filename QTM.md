# QTM (Quantum compression)
QTM (Quantum Compression) is a proprietary compression algorithm developed by Microsoft and used in the Microsoft Cabinet (CAB) archive format.
Despite its name, QTM is not related to quantum computing, The term “Quantum” is purely a product name.

## 1. Abstract
This document specifies the **Quantum compression method** decompression process as implemented in `libmspack`. Quantum is an adaptive arithmetic-coded LZ-style compressor used in Microsoft Cabinet (CAB) archives.

The document defines:
- Decoder state machine
- Arithmetic coding model behavior
- Match offset and length encoding
- Framing and window semantics
- Edge cases and boundary conditions

This specification is written in RFC style to enable independent reimplementation.

## 2. Terminology

- **Window**: Sliding dictionary buffer of size 2^N bytes.
- **Frame**: Fixed-size 32 KiB logical unit of decoding.
- **Selector**: Arithmetic-coded symbol determining literal vs match.
- **Model**: Adaptive arithmetic coding probability table.
- **Position Slot**: Encoded class of match offsets.
- **Cumfreq**: Cumulative frequency in arithmetic model.

## 3. Overview
Quantum decompression consists of:
1. Arithmetic decoding of selectors.
2. Literal or match reconstruction.
3. Sliding window updates.
4. Frame boundary alignment.

Quantum differs from DEFLATE in that:
- It uses **adaptive arithmetic coding**, not Huffman.
- It uses position slots similar to LZX.
- It enforces 32 KiB frame alignment.

## 4. Window Configuration
Quantum supports window sizes:
```
    2^10 (1 KiB)  through  2^21 (2 MiB)
```
The window acts as a circular buffer.
Match offsets always reference previously decoded bytes within the window.

## 5. Arithmetic Coding
Quantum uses a 16-bit adaptive range coder.

All arithmetic decoding MUST be bit-exact. Any deviation will cause
irreversible stream divergence.

### 5.1 Decoder State
The decoder maintains three unsigned 16-bit integers:
- `L` — lower bound  
- `H` — upper bound  
- `C` — current code value  

The invariant at all times is:
```
    L <= C <= H
```
The active range is:
```
    R = H - L + 1
```
### 5.2 Frame Initialization
At the beginning of each frame:
```
    L = 0
    H = 0xFFFF
    C = next 16 bits from input stream
```
### 5.3 Symbol Decoding
Let:
```
    T = model.syms[0].cumfreq   // total cumulative frequency
```
Compute:
```
    R = H - L + 1
    S = floor(((C - L + 1) * T - 1) / R)
```
Find smallest index i such that:
```
    syms[i].cumfreq <= S
```
Decoded symbol:
```
    syms[i-1].sym
```
### 5.4 Interval Update
Let:
```
    F_high = syms[i-1].cumfreq
    F_low  = syms[i].cumfreq
```
Update bounds:
```
    H = L + floor((F_high * R) / T) - 1
    L = L + floor((F_low  * R) / T)
```
The invariant `L <= C <= H` MUST still hold.

### 5.5 Renormalization
After updating L and H, renormalization is required.

Repeat while either condition holds:
#### Case 1 — MSB Match
If:
```
    (L & 0x8000) == (H & 0x8000)
```
Then:
```
    L <<= 1
    H = (H << 1) | 1
    C = (C << 1) | next_input_bit
```
#### Case 2 — Underflow (E3 Condition)

If:
```
    (L & 0x4000) != 0  AND  (H & 0x4000) == 0
```
Then:
```
    C ^= 0x4000
    L &= 0x3FFF
    H |= 0x4000
```
Then perform the same left shift as Case 1.

### 5.6 Model Update
After each decoded symbol:
- Increment cumulative frequencies.
- If total cumulative frequency exceeds 3800:
  - Convert cumulative to individual frequencies.
  - Add 1 to each frequency.
  - Divide each by 2.
  - Re-sort in descending frequency order.
  - Rebuild cumulative frequencies.

Monotonicity MUST be preserved:
```
    syms[i].cumfreq > syms[i+1].cumfreq
```
### 5.7 Determinism Requirements

Implementations MUST:

- Use integer arithmetic only
- Use floor division
- Preserve exact update order
- Apply renormalization exactly as specified

Changing division rounding, update order, or bit shifts will produce
a different decoded stream.

### 5.8 Mandatory Decode Order
For each decoded symbol, implementations MUST perform steps in
the following exact order:

1. Compute scaled value S.
2. Select symbol index i.
3. Update interval bounds L and H.
4. Perform renormalization until stable.
5. Update model frequencies.
6. If total frequency exceeds rescale threshold, perform rescale.

Reordering these steps produces a different decoded stream and is
not permitted.

## 6. Selector Model (Model7)
Selector values:
| Value | Meaning |
|--------|---------|
| 0 | Literal (Model0) |
| 1 | Literal (Model1) |
| 2 | Literal (Model2) |
| 3 | Literal (Model3) |
| 4 | Match length = 3 |
| 5 | Match length = 4 |
| 6 | Variable length match |

Selector 7 is invalid.

## 7. Literal Encoding
Literal decoding:
1. Decode selector (0–3).
2. Choose corresponding model (Model0–3).
3. Decode one byte (0–255).
4. Append to window.

Each model represents a byte context partition.

## 8. Match Encoding

Matches represent back-references into previously decoded output
within the current frame.

A match consists of:

- `match_offset`
- `match_length`

Offsets are encoded using a position slot mechanism.

### 8.1 Position Slot Tables

All match offsets use the following tables:

- `position_base[42]`
- `extra_bits[42]`

These tables SHALL be generated exactly as follows:
```
    unsigned int i, offset = 0;
    for (i = 0; i < 42; i++)
    { 
        position_base[i] = offset; 
        extra_bits[i] = ((i < 2) ? 0 : (i - 2)) >> 1; 
        offset += 1U << extra_bits[i]; 
    }
```
For a decoded slot index `s`:
```
    match_offset = position_base[s] + read_bits(extra_bits[s]) + 1
```
`read_bits(n)` reads bits MSB-first.

Slot indices outside `[0, 41]` SHALL cause decoder failure.

### 8.2 Fixed-Length Matches (Selectors 4 and 5)
Selectors `4` and `5` encode fixed-length matches.
Offset decoding is identical for both selectors.

#### 8.2.1 Selector 4 — 3-Byte Match
1. Decode `sym` from `model4`
2. Compute `match_offset` using Section 8.1
3. Set:
```
    match_length = 3
```
#### 8.2.2 Selector 5 — 4-Byte Match
1. Decode `sym` from `model5`
2. Compute `match_offset` using Section 8.1
3. Set:
```
    match_length = 4
```
#### 8.2.3 Model Size Constraints
The number of entries in:

- `model4`
- `model5`

SHALL be:
```
    min(window_bits * 2, limit)
```
Where:

- `limit = 24` for `model4`
- `limit = 36` for `model5`

Symbol indices MUST remain within model bounds.

### 8.3 Variable-Length Match (Selector 6)
Selector value `6` encodes a variable-length match.

A variable-length match encodes:

1. Match length
2. Match offset

#### 8.3.1 Length Slot Tables
Length slots use:
- `length_base[27]`
- `length_extra[27]`

These SHALL be generated exactly as:
```
    unsigned int i, offset = 0;
    for (i = 0; i < 26; i++) 
    { 
        length_base[i] = offset; 
        length_extra[i] = ((i < 2) ? 0 : (i - 2)) >> 2; 
        offset += 1U << length_extra[i]; 
    }
    length_base[26] = 254; length_extra[26] = 0;
```
For decoded slot `l`:
```
    match_length = length_base[l] + read_bits(length_extra[l]) + 5
```
The `+5` constant is mandatory.

Slot indices outside `[0, 26]` SHALL cause decoder failure.

#### 8.3.2 Offset Decoding
After length is determined:
1. Decode `sym` from `model6`
2. Compute `match_offset` using Section 8.1

### 8.4 Match Copy Semantics
For all match types:
- `match_offset` MUST be ≥ 1
- `match_offset` MUST NOT exceed bytes already produced in the frame
- `match_length` MUST NOT exceed remaining bytes in frame
- Matches MUST NOT cross frame boundaries

Overlapping copies are valid and MUST behave identically to:
```
    memmove(destination, source, match_length)
```
Decoder MUST abort if any constraint is violated.

## 9. Copy Semantics
### 9.1 Standard (Non-Wrapping) Match Copy
A match is considered non-wrapping if:
```
    window_posn + match_length ≤ window_size
```
In this case, all destination bytes fall within the current window region.

#### 9.1.1 Source Index Calculation

If:
```
    match_offset ≤ window_posn
```
then the match source is:
```
    src_index = window_posn - match_offset
```
The decoder SHALL copy `match_length` bytes from:
```
    window[src_index ... src_index + match_length - 1]
```
into:
```
    window[window_posn ... window_posn + match_length - 1]
```
#### 9.1.2 Overlapping Copy Semantics
If:
```
    match_offset < match_length
```
the source and destination regions overlap.

Implementations MUST perform copy semantics equivalent to:
```
    memmove()
```
That is:
- Copy forward
- Each newly written byte becomes immediately available as future source data

This behavior is REQUIRED for correct LZ-style expansion.

### 9.2 Offset-Wrapping Source Copy
If:
```
    match_offset > window_posn
```
the match source precedes the logical beginning of the window and wraps around.

#### 9.2.1 Wrapped Source Index
The effective source index SHALL be computed as:
```
    src_index = (window_size + window_posn - match_offset)
```
This is equivalent to:
```
    src_index = (window_posn - match_offset) mod window_size
```
#### 9.2.2 Copy Procedure
The decoder SHALL:
1. Copy until reaching `window_size`
2. If additional bytes remain, continue copying from index `0`

Formally:
```
for i in 0 .. match_length-1: 
    window[window_posn + i] = window[(src_index + i) mod window_size]
```
#### 9.2.3 Constraints
- `match_offset` MUST NOT exceed `window_size`
- Negative indexing is invalid
- Wrap MUST use modulo window size
- Copy semantics MUST preserve overlap behavior

Failure to enforce these rules results in decompression error.

### 9.3 Destination Wrap
If match crosses window end:
1. Copy until window end.
2. Flush window to output.
3. Continue at window start.

## 10. Frame Boundaries
Compressed data is divided into independent 32 KiB frames.

At the end of each frame:
1. The arithmetic decoder SHALL be renormalized until no further
   MSB match or E3 condition applies.
2. The decoder SHALL advance to the next byte boundary.
   Any padding bits between the final arithmetic bit and the
   next byte boundary MUST be zero.
3. A single byte value 0xFF SHALL follow as a frame marker.

The marker byte is byte-aligned and SHALL be consumed.

Exactly one marker SHALL appear per frame.

If a non-zero padding bit is encountered, or if the marker byte
is not 0xFF, the decoder MUST signal an error.

## 11. Arithmetic Model Initialization
Quantum uses multiple adaptive arithmetic models.  
Each model consists of an ordered list of symbols with cumulative frequencies.

### 11.1 Model Structure
Each model SHALL contain:
- `entries` symbols
- An array `syms[0 .. entries]`
- Each element containing:
  - `sym`     (decoded symbol value)
  - `cumfreq` (cumulative frequency)

The array includes one extra entry at index `entries` whose `cumfreq` MUST be `0`.

Cumulative frequencies MUST satisfy:
```
    syms[i].cumfreq > syms[i+1].cumfreq
```
for all valid `i`.

### 11.2 Initial Symbol Assignment
For a model initialized with:
```
    start len
```
symbols SHALL be assigned:
```
    syms[i].sym = start + i
```
for:
```
    i = 0 .. len
```
### 11.3 Initial Frequency Distribution
Initial cumulative frequencies SHALL be set as:
```
syms[i].cumfreq = len - i
```
This produces a uniform distribution where:
- All symbols begin with equal probability.
- Symbol ordering reflects increasing symbol value.

The final entry:
```
    syms[len].cumfreq = 0
```
### 11.4 Model Sizes
The following models MUST be initialized:

| Model | Symbol Range | Entry Count |
|--------|--------------|-------------|
| Model0 | 0–63        | 64 |
| Model1 | 64–127      | 64 |
| Model2 | 128–191     | 64 |
| Model3 | 192–255     | 64 |
| Model4 | 0–N         | min(window_bits × 2, 24) |
| Model5 | 0–N         | min(window_bits × 2, 36) |
| Model6 | 0–N         | window_bits × 2 |
| Model6len | 0–26     | 27 |
| Model7 | 0–6         | 7 |

Where:
```
    N = window_bits * 2
```
### 11.5 Shift and Rescale Parameters
Each model SHALL initialize:
```
    shiftsleft = 4
```
Rescaling SHALL occur when:
```
    syms[0].cumfreq > 3800
```
Rescaling MUST:
1. Convert cumulative frequencies into individual frequencies.
2. Increment each frequency by 1.
3. Divide each frequency by 2.
4. Re-sort symbols in descending frequency order using stable selection sort.
5. Reconstruct cumulative frequencies.

Stable ordering is REQUIRED for decoder determinism.

### 11.6 Determinism Requirement
All arithmetic model updates:
- MUST preserve cumulative monotonicity
- MUST maintain stable ordering when frequencies are equal
- MUST be bit-exact across implementations

Any deviation results in stream divergence.

### 11.7 Security Considerations
Implementations MUST ensure:
- No integer overflow during cumulative updates
- Cumfreq values never become equal or inverted
- Array bounds are enforced during symbol lookup

Failure to maintain monotonic cumulative ordering results in undefined decoding behavior.

## 12. Error Conditions

A decoder MUST immediately abort with an error if any of the
following occur:

Arithmetic violations:
- C < L
- C > H
- Total frequency equals zero
- Cumulative frequencies are not strictly descending

Match violations:
- Offset == 0
- Offset > number of bytes already produced in frame
- Match extends beyond frame boundary

Selector violations:
- Undefined symbol selector
- Position slot outside defined table range

Framing violations:
- Non-zero padding bits before byte alignment
- Missing or incorrect frame marker

Silent recovery or continued decoding after these conditions
is NOT permitted.

## 13. Security Considerations

Malformed input may attempt to:

- Trigger arithmetic underflow
- Cause invalid offset references
- Overflow frequency counters
- Induce excessive rescale operations

Implementations MUST validate all invariants and abort on violation.

All arithmetic SHALL use bounded integer types with defined overflow behavior.

## 14. Implementation Notes
- Window size must be power of two.
- Circular indexing should use masking.
- Model updates must preserve stable ordering.
- Copy operations must support overlap.

## 15. References
- [Stuart Caie](https://github.com/kyz), [libmspack](https://github.com/kyz/libmspack) source code
- Matthew Russotto, Quantum compression research [notes](http://www.russotto.net/quantumcomp.html)
- Microsoft Cabinet file format documentation

## 16. Status
This document describes behavior observed in `libmspack` Quantum decompressor. It is suitable for clean-room reimplementation.

Community review and corrections are encouraged.

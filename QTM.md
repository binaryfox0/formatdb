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

All arithmetic decoding operations MUST be bit-exact. Any deviation
will cause irreversible stream divergence.

### 5.1 Decoder State
The decoder maintains three 16-bit unsigned integers:
\[
L \quad \text{(low bound)}
\]

\[
H \quad \text{(high bound)}
\]

\[
C \quad \text{(current code value)}
\]

The invariant maintained at all times is:

\[
L \le C \le H
\]

The active range is:
\[
R = H - L + 1
\]

### 5.2 Frame Initialization
At the beginning of each frame:
\[
L = 0
\]
\[
H = 2^{16} - 1 = 65535
\]

Then the next 16 bits from the input stream are read into:
\[
C
\]

### 5.3 Symbol Decoding
Let:
\[
T = \text{model.syms}[0].\text{cumfreq}
\]

be the total cumulative frequency.

The decoder computes:

\[
R = H - L + 1
\]

\[
S = \left\lfloor \frac{(C - L + 1)\cdot T - 1}{R} \right\rfloor
\]

The symbol index \( i \) is the smallest index such that:
\[
\text{syms}[i].\text{cumfreq} \le S
\]

The decoded symbol is:
\[
\text{syms}[i-1].\text{sym}
\]

### 5.4 Interval Update
Let:
\[
F_{high} = \text{syms}[i-1].\text{cumfreq}
\]

\[
F_{low} = \text{syms}[i].\text{cumfreq}
\]

The interval is updated as:

\[
H = L + \left\lfloor \frac{F_{high} \cdot R}{T} \right\rfloor - 1
\]

\[
L = L + \left\lfloor \frac{F_{low} \cdot R}{T} \right\rfloor
\]

The invariant \( L \le C \le H \) MUST still hold.

### 5.5 Renormalization
After updating \( L \) and \( H \), the decoder MUST renormalize.

Renormalization continues while either condition holds:

#### Case 1 — MSB Match
\[
\text{MSB}(L) = \text{MSB}(H)
\]

In this case:

\[
L \leftarrow 2L
\]
\[
H \leftarrow 2H + 1
\]
\[
C \leftarrow 2C + \text{next input bit}
\]

#### Case 2 — Underflow (E3 Condition)

\[
L \in [0x4000, 0x7FFF]
\quad \text{and} \quad
H \in [0x8000, 0xBFFF]
\]

In this case:

\[
C \leftarrow C \oplus 0x4000
\]
\[
L \leftarrow (L \,\&\, 0x3FFF)
\]
\[
H \leftarrow (H \,|\, 0x4000)
\]

Then shift left as in Case 1.

### 5.6 Model Update
After decoding a symbol:
- Its cumulative frequency is incremented.
- All higher entries are incremented accordingly.
- When total frequency exceeds threshold (3800),
  rescaling MUST occur.

Rescaling MUST preserve:

\[
\text{syms}[i].\text{cumfreq} >
\text{syms}[i+1].\text{cumfreq}
\]

Stable ordering is REQUIRED.

### 5.7 Determinism Requirement
Arithmetic decoding MUST:
- Use integer arithmetic only.
- Use floor division.
- Use 16-bit wrap semantics where applicable.
- Perform renormalization exactly as specified.

Any change in:
- Division rounding
- Shift order
- Underflow handling

will produce a different bitstream interpretation.

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
Matches consist of:
- Offset
- Length

### 8.1 Fixed-Length Match Encoding (Selectors 4 and 5)
Selectors `4` and `5` encode fixed-length matches.

These selectors differ only in match length.  
Offset decoding is identical for both.

#### 8.1.1 Selector 4 — 3-Byte Match
Decoding steps:
1. Decode `sym` from `model4`
2. Read `extra_bits[sym]` additional bits
3. Compute:
```
match_offset = position_base[sym] + extra + 1 match_length = 3
```
#### 8.1.2 Selector 5 — 4-Byte Match

Decoding steps:
1. Decode `sym` from `model5`
2. Read `extra_bits[sym]` additional bits
3. Compute:
```
match_offset = position_base[sym] + extra + 1 match_length = 4
```
#### 8.1.3 Position Slot Mechanism
Both selectors use the position slot system defined by:
- `position_base[42]`
- `extra_bits[42]`

The offset calculation is:

match_offset = position_base[sym] + extra + 1

This encoding reduces entropy cost by grouping offsets into ranges of exponentially increasing size.

#### 8.1.4 Model Size Constraints
The number of entries in:
- `model4`
- `model5`

depends on:
```
min(window_bits × 2, limit)
```
Where:
- `limit = 24` for Model4
- `limit = 36` for Model5

Implementations MUST ensure symbol indices remain within model entry bounds.

#### 8.1.5 Constraints
- `match_offset` MUST reference previously decoded bytes.
- Offset MUST NOT exceed window size.
- Overlapping copies are valid and required.
- Match MUST NOT cross frame boundary.

#### 8.1.6 Rationale
Fixed-length matches provide efficient encoding for very common short repetitions (3–4 bytes), avoiding the additional length-model overhead required by selector 6.

This is a performance optimization for short back-references, analogous to small-length biasing in LZ-family compressors.

### 8.2 Variable-Length Match Encoding (Selector 6)
Selector value `6` indicates a variable-length match.
A variable-length match encodes:
1. Match length (via Model6len)
2. Match offset (via Model6 + position slot)

#### 8.2.1 Length Slot Decoding
The decoder performs:
1. Decode `sym` from `model6len`
2. Read `length_extra[sym]` additional bits
3. Compute:
```
match_length = length_base[sym] + extra + 5
```
The constant `+5` is mandatory and part of the Quantum format.

#### 8.2.2 Length Tables

| Index | Base | Extra Bits |
|-------|------|------------|
| 0     | 0    | 0 |
| 1     | 1    | 0 |
| 2     | 2    | 0 |
| 3     | 3    | 0 |
| 4     | 4    | 0 |
| 5     | 5    | 0 |
| 6     | 6    | 1 |
| 7     | 8    | 1 |
| 8     | 10   | 1 |
| 9     | 12   | 1 |
| 10    | 14   | 2 |
| 11    | 18   | 2 |
| 12    | 22   | 2 |
| 13    | 26   | 2 |
| 14    | 30   | 3 |
| 15    | 38   | 3 |
| 16    | 46   | 3 |
| 17    | 54   | 3 |
| 18    | 62   | 4 |
| 19    | 78   | 4 |
| 20    | 94   | 4 |
| 21    | 110  | 4 |
| 22    | 126  | 5 |
| 23    | 158  | 5 |
| 24    | 190  | 5 |
| 25    | 222  | 5 |
| 26    | 254  | 0 |

#### 8.2.3 Offset Decoding
After length is determined:
1. Decode `sym` from `model6`
2. Read `extra_bits[sym]`
3. Compute:
```
match_offset = position_base[sym] + extra + 1
```
#### 8.2.4 Constraints

- `match_length` MUST NOT exceed remaining bytes in current frame.
- `match_offset` MUST be within the sliding window.
- Overlapping copies are permitted and required to behave like `memmove()`.

#### 8.2.5 Rationale
The two-stage structure (length first, then offset) improves compression efficiency for long matches while maintaining arithmetic coding adaptivity.

This design is conceptually similar to LZ77 with length/offset coding, but differs from DEFLATE in that it uses adaptive arithmetic models instead of static or dynamic Huffman trees.

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

## 10. Frame Behavior
Each frame is 32 KiB each.

After frame completion:
1. Bitstream is aligned to next byte.
2. Bytes are consumed until `0xFF` marker.
3. Arithmetic state resets.

CAB-specific behavior allows 0–4 trailing zero bytes before marker.

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
N = window_bits × 2
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
Decoder returns error if:
- Invalid selector (>6)
- Offset exceeds window
- Frame alignment overshoots
- Output write fails

## 13. Differences from DEFLATE

| Feature | Quantum | DEFLATE |
|----------|----------|----------|
| Entropy coding | Adaptive arithmetic | Huffman |
| Block framing | Fixed 32 KiB | Variable blocks |
| Offset encoding | Position slots | Direct Huffman |
| Adaptivity | Continuous | Per-block |

## 14. Security Considerations

Implementations must:

- Validate match offsets.
- Prevent integer overflow in range calculations.
- Enforce window boundaries.
- Protect against malformed frame markers.

Arithmetic decoding requires careful renormalization logic.

## 15. Implementation Notes
- Window size must be power of two.
- Circular indexing should use masking.
- Model updates must preserve stable ordering.
- Copy operations must support overlap.

## 16. References
- [Stuart Caie](https://github.com/kyz), [libmspack](https://github.com/kyz/libmspack) source code
- Matthew Russotto, Quantum compression research [notes](http://www.russotto.net/quantumcomp.html)
- Microsoft Cabinet file format documentation

## 17. Status
This document describes behavior observed in `libmspack` Quantum decompressor. It is suitable for clean-room reimplementation.

Community review and corrections are encouraged.

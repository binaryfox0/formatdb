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

---

## 2. Terminology

- **Window**: Sliding dictionary buffer of size 2^N bytes.
- **Frame**: Fixed-size 32 KiB logical unit of decoding.
- **Selector**: Arithmetic-coded symbol determining literal vs match.
- **Model**: Adaptive arithmetic coding probability table.
- **Position Slot**: Encoded class of match offsets.
- **Cumfreq**: Cumulative frequency in arithmetic model.

---

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

---

## 4. Window Configuration

Quantum supports window sizes:

2^10 (1 KiB)  through  2^21 (2 MiB)

The window acts as a circular buffer.

Match offsets always reference previously decoded bytes within the window.

---

## 5. Arithmetic Coding

### 5.1 Range Decoder State

The arithmetic decoder maintains:

L : lower bound H : upper bound C : current code value

All values are 16-bit.

Initialization at frame start:

H = 0xFFFF L = 0 C = next 16 bits from stream

---

### 5.2 Symbol Decoding

To decode a symbol from a model:

1. Compute range:

range = (H - L) + 1

2. Compute scaled frequency:

symf = (((C - L + 1) * total_cumfreq) - 1) / range

3. Find symbol `i` such that:

syms[i].cumfreq > symf >= syms[i+1].cumfreq

4. Update bounds:

H = L + (syms[i].cumfreq   * range / total) - 1 L = L + (syms[i+1].cumfreq * range / total)

5. Renormalize while MSBs match or underflow case occurs.

---

### 5.3 Model Adaptation

Each decoded symbol:

- Increases cumulative frequencies
- Periodically rescales frequencies
- Performs stable selection sort on symbols by frequency

This preserves relative ordering.

---

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

---

## 7. Literal Encoding

Literal decoding:

1. Decode selector (0–3).
2. Choose corresponding model (Model0–3).
3. Decode one byte (0–255).
4. Append to window.

Each model represents a byte context partition.

---

## 8. Match Encoding

Matches consist of:

- Offset
- Length

---

### 8.1 Position Slots

Offsets are encoded using 42 position slots.

#### Position Base Table

position_base[42]

Each slot defines:

match_offset = position_base[sym] + extra + 1

#### Extra Bits

extra_bits[sym]

Indicates how many raw bits follow.

---

### 8.2 Length Encoding

Selector 4:

Length = 3

Selector 5:

Length = 4

Selector 6:

length_slot -> model6len extra bits -> length_extra[] length = length_base[sym] + extra + 5

---

## 9. Copy Semantics

### 9.1 Non-Wrapping Copy

If match stays within window:

copy from (window_pos - offset)

Supports overlapping copy.

---

### 9.2 Offset Wrap

If `offset > window_pos`:

Source wraps around end of window.

---

### 9.3 Destination Wrap

If match crosses window end:

1. Copy until window end.
2. Flush window to output.
3. Continue at window start.

---

## 10. Frame Behavior

Each frame is:

32 KiB

After frame completion:

1. Bitstream is aligned to next byte.
2. Bytes are consumed until `0xFF` marker.
3. Arithmetic state resets.

CAB-specific behavior allows 0–4 trailing zero bytes before marker.

---

## 11. Model Initialization

Models:

Model0–3 : 64 symbols each Model4   : up to 24 entries Model5   : up to 36 entries Model6   : up to 42 entries Model6len: 27 entries Model7   : 7 entries

Entry counts for models 4–6 depend on:

window_bits * 2

Initial cumulative frequency:

cumfreq[i] = len - i

This ensures uniform distribution.

---

## 12. Error Conditions

Decoder returns error if:

- Invalid selector (>6)
- Offset exceeds window
- Frame alignment overshoots
- Output write fails

---

## 13. Differences from DEFLATE

| Feature | Quantum | DEFLATE |
|----------|----------|----------|
| Entropy coding | Adaptive arithmetic | Huffman |
| Block framing | Fixed 32 KiB | Variable blocks |
| Offset encoding | Position slots | Direct Huffman |
| Adaptivity | Continuous | Per-block |

---

## 14. Security Considerations

Implementations must:

- Validate match offsets.
- Prevent integer overflow in range calculations.
- Enforce window boundaries.
- Protect against malformed frame markers.

Arithmetic decoding requires careful renormalization logic.

---

## 15. Implementation Notes

- Window size must be power of two.
- Circular indexing should use masking.
- Model updates must preserve stable ordering.
- Copy operations must support overlap.

---

## 16. References

- Stuart Caie, libmspack source code
- Matthew Russotto, Quantum compression research notes
- Microsoft Cabinet file format documentation

---

## 17. Status

This document describes behavior observed in `libmspack` Quantum decompressor. It is suitable for clean-room reimplementation.

Community review and corrections are encouraged.

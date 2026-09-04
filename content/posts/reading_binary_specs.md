+++
title = "Reading Binary Specs"
date = 2026-06-09
description = "How to read a binary format spec."
[taxonomies]
tags = ["systems", "binary", "parsers", "go"]
+++

The first time I had to parse a binary format I just stared at the spec for a while and wrote garbage. The second time I was a bit less lost. Eventually something clicked and now I can read a format table and mostly know what code to write. I'm trying to write down what actually clicked.

## what even is a binary format

A binary format is just a contract, that bytes at position X mean Y. That's it. The spec is the list of positions and what each one means. Your job when writing a parser is to read the spec and turn it into reads and bit operations. There's no magic in it, but the terminology and notation can be confusing.

The atomic unit is always the byte. A byte is 8 bits. Each bit at position `n` contributes `2^n` to the value, so bit 0 contributes 1, bit 1 contributes 2, bit 7 contributes 128. That's the whole system, every number in a binary format is made of bytes, and every byte is made of 8 bits with those fixed positions.

```
Bit position:  7    6    5    4    3    2    1    0
Bit value:    128   64   32   16    8    4    2    1
```

Specs usually write bytes in hex because each 4 bits (a nibble) maps exactly to one hex digit, so `0xB7` is `10110111` in binary, where the upper nibble `1011` is B and the lower nibble `0111` is 7. If you print bytes in binary during debugging (`%08b` in Go) it becomes obvious what's inside them.

## okay but why do specs say things like "bits 5-3"

Because a single byte often carries more than one field. The spec doesn't waste a whole byte on a 1-bit flag; it packs several fields into one byte and tells you which bits belong to which field. So the spec might say something like:

```
bit 7:    sync_flag            (1 bit)
bits 6-4: version              (3 bits)
bits 3-0: channel_count        (4 bits)
```

And you have to turn that into code. The question is how.

The answer is always the same two operations: right-shift and AND. Right-shift (`>>`) moves bits toward position 0. AND (`&`) zeroes out bits you don't want. But stating the formula isn't the same as understanding where `shift` and `mask` come from given a line in the spec.

When a spec says "bits 5-3", it's giving you two numbers: the high bit (5) and the low bit (3). Those two numbers are all you need. The derivation is:

```
shift = low                       // how far to push the field down
bits  = high - low + 1            // how wide the field is
mask  = (1 << bits) - 1           // a run of 'bits' ones, starting from position 0
```

Then the extraction is always `(b >> shift) & mask`.

The reason `shift = low` is that you want the field's lowest bit to land at position 0 after the shift. If the field starts at bit 3, shifting right by 3 puts bit 3 at position 0, bit 4 at position 1, bit 5 at position 2, exactly where you want them. The reason `bits = high - low + 1` is just counting, since from bit 3 to bit 5 inclusive is 3 bits (3, 4, 5), and the +1 is the classic off-by-one of inclusive ranges.

The mask is the part that looks weird until you see it in binary, where `(1 << n) - 1` gives you exactly n ones:

```
n=1: (1<<1)-1 = 0b00000001 = 0x01
n=2: (1<<2)-1 = 0b00000011 = 0x03
n=3: (1<<3)-1 = 0b00000111 = 0x07
n=4: (1<<4)-1 = 0b00001111 = 0x0F
n=5: (1<<5)-1 = 0b00011111 = 0x1F
n=6: (1<<6)-1 = 0b00111111 = 0x3F
```

After you shift the field down to position 0, the mask zeroes out anything above it that the shift dragged along, so a 3-bit field with `0x07` means keep the bottom 3 bits and kill the rest.

Concretely, extracting bits 5-3 from `0xB7` (`10110111`):

```
Before shift:   1  0  1  1  0  1  1  1    (0xB7)
                7  6  5  4  3  2  1  0

Field is bits 5-3:
                         [5  4  3]

After >> 3:     0  0  0  1  0  1  1  0
                                [2  1  0]   <- field landed here

After & 0x07:   0  0  0  0  0  1  1  0    = 6
                               [field]
```

The shift moved the field to the bottom. The AND killed the bits above it (the `0001` that came from bits 7-6 getting dragged down). Result is 6.

You can wrap this into a helper that makes the formula explicit:

```go
func extractBits(b byte, high, low int) byte {
    shift := low
    bits  := high - low + 1
    mask  := byte((1 << bits) - 1)
    return (b >> shift) & mask
}
```

```go
b := byte(0xB7) // 10110111

fmt.Println(extractBits(b, 7, 7)) // bit 7 (single flag)    -> 1
fmt.Println(extractBits(b, 7, 4)) // bits 7-4 (upper nibble) -> 11
fmt.Println(extractBits(b, 3, 0)) // bits 3-0 (lower nibble) -> 7
fmt.Println(extractBits(b, 5, 3)) // bits 5-3 (middle 3)     -> 6
fmt.Println(extractBits(b, 7, 5)) // bits 7-5 (upper 3)      -> 5
fmt.Println(extractBits(b, 4, 0)) // bits 4-0 (lower 5)      -> 23
```

Once you've internalized the formula you won't use a helper, you'll inline the shift and mask directly, or just use the helper. But having the helper is useful while you're learning because you can verify your mental math against the spec before committing to a magic number.

The common masks are worth memorizing, so 4 bits is always `0x0F`, 3 bits is `0x07`, 5 bits is `0x1F`, 6 bits is `0x3F`. They appear constantly in media format specs because audio and video pack information densely and almost nothing is aligned to a byte boundary.


## what about numbers that are bigger than one byte

This is where endianness comes in. A single byte needs no ordering decisions since it's one thing, but a 4-byte integer has 4 bytes and they have to go in some order in the file, and there are two conventions. Big-endian puts the most significant byte first, which is how humans write numbers, thousands before hundreds, and little-endian puts the least significant byte first, reversed from how you'd write it.

```go
b := []byte{0x00, 0x01, 0x86, 0xA0}

// big-endian: b[0] is the high byte
val := uint32(b[0])<<24 | uint32(b[1])<<16 | uint32(b[2])<<8 | uint32(b[3])
// 0x000186A0 = 100000

// little-endian: b[0] is the low byte
val2 := uint32(b[0]) | uint32(b[1])<<8 | uint32(b[2])<<16 | uint32(b[3])<<24
```

Network formats and media containers (MP4, AAC, MPEG-TS, FLV) are almost always big-endian, while WAV, BMP, and most Windows-native formats are little-endian. The spec always tells you, and if you get it wrong the numbers will be wildly off but no error fires, endianness bugs are silent.

The reason the shifts are 24, 16, 8, 0 is not magic, it's just that each byte contributes `N * 8` bits where `N` is how many bytes come after it, so the last byte contributes 0 extra bits with shift 0, the one before it has one byte after it so shift 8, and the pattern continues, where the shift for byte `i` in a big-endian N-byte integer is `(N - 1 - i) * 8`.

```go
// general big-endian read for any N bytes
func readBigEndian(b []byte) uint64 {
    var result uint64
    for i, v := range b {
        shift := (len(b) - 1 - i) * 8
        result |= uint64(v) << shift
    }
    return result
}
```

Once you see that the 32/24/16/8/0 sequence is just multiples of 8 counting down from the number of trailing bytes times 8, it stops looking arbitrary.

## the spec has a 24-bit integer. there's no uint24

Yeah, this comes up. FLV uses 24-bit big-endian integers for the payload size and timestamp on every tag. `encoding/binary` in Go handles 8, 16, 32, 64 natively. For 24 bits you do it manually:

```go
func readU24BE(b []byte) uint32 {
    return uint32(b[0])<<16 | uint32(b[1])<<8 | uint32(b[2])
}
```

Same logic, just 3 bytes instead of 4. The shift for byte 0 is `(3-1)*8 = 16`, for byte 1 it's `8`, for byte 2 it's `0`.

FLV makes it stranger because the timestamp is 24 bits for the lower part, then a separate byte `TimestampExtended` carries the high 8 bits. So a 32-bit millisecond timestamp is split across two non-adjacent fields. You reassemble it:

```go
tsLow := readU24BE(data[4:7])
tsExt := uint32(data[7])
timestamp := (tsExt << 24) | tsLow
```

Shift the high part left by exactly how many bits the low part occupies, then OR them together. This pattern of assembling a value from non-contiguous fields appears constantly in specs once you start looking, and MPEG-TS does the same thing with the 13-bit PID spread across two bytes.

## a field is split across two bytes. how

The MPEG-TS packet header has a PID field that's 13 bits, 5 bits from byte 1 (the lower 5 bits) and 8 bits from byte 2 (all 8). The question is which 5 bits from byte 1, and in what order.

The spec says bits 4-0 of byte 1 hold the upper 5 bits of the PID, and byte 2 holds the lower 8 bits. So byte 1 contributes the high part:

```go
b1 := r.ReadU8()
b2 := r.ReadU8()

pid := uint16(b1&0x1F)<<8 | uint16(b2)
// b1 & 0x1F keeps the lower 5 bits of b1
// shift left 8 to make room for the 8 bits from b2
// OR in b2
```

The total is 13 bits, where the 5 bits from b1 are bits 12-8 of the final PID and the 8 bits from b2 are bits 7-0. The shift of 8 is just how many bits b2 contributes, same rule as always.

## i'm losing track of where i am in the byte slice

This is what kills most first-attempt parsers, since you have `pos += n` scattered everywhere and one wrong value makes every subsequent read garbage. The fix is a small reader struct that owns the cursor:

```go
type Reader struct {
    data []byte
    pos  int
}

func (r *Reader) ReadU8() uint8 {
    b := r.data[r.pos]
    r.pos++
    return b
}

func (r *Reader) ReadU16BE() uint16 {
    v := binary.BigEndian.Uint16(r.data[r.pos:])
    r.pos += 2
    return v
}

func (r *Reader) ReadU24BE() uint32 {
    b := r.data[r.pos : r.pos+3]
    r.pos += 3
    return uint32(b[0])<<16 | uint32(b[1])<<8 | uint32(b[2])
}

func (r *Reader) ReadU32BE() uint32 {
    v := binary.BigEndian.Uint32(r.data[r.pos:])
    r.pos += 4
    return v
}

func (r *Reader) Tell() int { return r.pos }
func (r *Reader) Skip(n int) { r.pos += n }
```

Then the parsing code reads like the spec table:

```go
// FLV tag header, from the spec:
// TagType           1 byte  uint8
// DataSize          3 bytes uint24 BE
// Timestamp         3 bytes uint24 BE
// TimestampExtended 1 byte  uint8
// StreamID          3 bytes uint24 BE (always 0)

tagType  := r.ReadU8()
dataSize := r.ReadU24BE()
tsLow    := r.ReadU24BE()
tsExt    := uint32(r.ReadU8())
_         = r.ReadU24BE() // StreamID

timestamp := (tsExt << 24) | tsLow
```

The code is the spec. You can't accidentally be at the wrong offset because you never touch a position integer manually.

## what if the spec says signed

Same read, extra step at the end. An N-bit signed value uses the highest bit as the sign bit, and if it's set the value is negative. For a 24-bit signed integer:

```go
func readS24BE(b []byte) int32 {
    raw := uint32(b[0])<<16 | uint32(b[1])<<8 | uint32(b[2])
    if raw&0x800000 != 0 { // bit 23 is set -> negative
        return int32(raw | 0xFF000000) // sign-extend to 32 bits
    }
    return int32(raw)
}
```

The `0xFF000000` fills in the upper 8 bits with ones, which is what two's complement requires for the value to be correctly negative when interpreted as int32. The spec will tell you when a field is signed, and most binary format fields are unsigned, with signed showing up mostly in audio sample values and offset/delta fields.

## how do i actually start parsing a new format

Read the spec top to bottom and find the header description, which will be a table with a field name, a size in bytes or bits, and a type. Start writing a `Reader` and call the appropriate read method for each row, and if a row is a bit field, read the byte first, then extract the bits with shift and mask.

Print everything while developing, every field and every value, since it's the only way to know if you're reading things correctly, because wrong endianness and wrong offsets both produce numbers that look plausible until they don't, and a `fmt.Printf` on every field after reading it makes the bug obvious immediately.

The last real question is usually whether a field is big-endian or little-endian, and usually the spec states this clearly. If it doesn't, the format's Wikipedia page or an open-source implementation will tell you fast enough.

## notes

- `encoding/binary` in Go handles 16, 32, 64 natively. Manual shifts for 24 bits and cross-byte fields.
- When in doubt about endianness: media/network = big, Windows-native = little.
- The formula `shift = low, mask = (1 << (high - low + 1)) - 1` is mechanical, always works.
- The mask for n bits is always `(1 << n) - 1`. Memorize: 4 bits -> `0x0F`, 3 bits -> `0x07`, 5 bits -> `0x1F`, 6 bits -> `0x3F`.
- Left shifts appear when _building_ a value from parts. Right shifts appear when _extracting_. In a parser you mostly right-shift then AND.
- Sign extension only matters for signed fields, which are less common than unsigned ones.

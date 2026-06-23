# Known Gengo Language Limitations

## std.bytes — raw byte string construction

**Filed against**: gengo language / stdlib

Gengo strings are UTF-8. The cast `string(rune(n))` for `n > 127` produces the
multi-byte UTF-8 encoding of Unicode code point U+00nn, not the raw byte `n`.
For example, `string(rune(200))` gives `"\xC3\xC8"` (2 bytes), not `"\xC8"`.

This means there is no direct way to construct a binary string containing an
arbitrary byte value from a Gengo expression.

**Current workaround** (used throughout this library):
```gengo
// Encode the byte as a 2-char hex string, then decode to binary.
func u8hex(b int) string {
    const H := "0123456789abcdef"
    n := b & 255
    return string(H[(n >> 4)]) + string(H[n & 15])
}

frame := std.hex.decode(u8hex(0xAB) + u8hex(0xCD))
// → 2-byte binary string "\xAB\xCD"
```

**Proposed fix**: add `std.bytes` with at minimum:
- `std.bytes.chr(n int) string` — single raw byte as a string
- `std.bytes.pack(bs []int) string` — array of byte values to binary string
- `std.bytes.unpack(s string) []int` — binary string to byte array
- `std.bytes.u16be(n int) string` — big-endian uint16 as 2-byte string
- `std.bytes.u32be(n int) string` — big-endian uint32 as 4-byte string
- `std.bytes.u16le(n int) string` — little-endian uint16
- `std.bytes.u32le(n int) string` — little-endian uint32

This would eliminate the hex round-trip overhead and make binary protocol
implementations significantly cleaner.

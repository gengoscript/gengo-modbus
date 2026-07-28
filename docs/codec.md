Codec — float / integer register encoding
=========================================

Import
------

```gengo
cdc := import("./modbus/codec")
```

Codec struct
------------

```gengo
pub type Codec struct {
    byte_order string,
}
```

Create with a byte-order constant:

```gengo
cdc := cdc.Codec { byte_order: cdc.CDAB }
```

Byte orders
-----------

| Constant | Description | Used by |
|----------|-------------|---------|
| `ABCD`   | Big-endian (IEEE 754 native) | Siemens S7, Allen-Bradley BE mode |
| `CDAB`   | Word-swapped (most common) | Schneider/Modicon, Emerson, Yaskawa |
| `BADC`   | Byte-swapped within each word | Some Danfoss devices |
| `DCBA`   | Fully little-endian | Some x86-based devices |

Methods on Codec
----------------

```gengo
// 32-bit float (2 registers)
cdc.f32(regs []int, i int) float       // decode at register index i
cdc.f32_regs(v float) []int            // encode -> 2 register values

// 64-bit double (4 registers)
cdc.f64(regs []int, i int) float       // decode at register index i
cdc.f64_regs(v float) []int            // encode -> 4 register values

// 32-bit unsigned integer (2 registers)
cdc.u32(regs []int, i int) int         // decode at register index i
cdc.u32_regs(v int) []int              // encode -> 2 register values
```

Reader/Writer cursor API
------------------------

Create a cursor over a register slice for sequential decode:

```gengo
r := codec.reader(regs)
pi    := r.f32()
euler := r.f32()
```

Build registers sequentially with a writer:

```gengo
w := codec.writer()
w.f32(3.14159)
w.f32(2.71828)
regs := w.regs()
```

Example
-------

```gengo
regs, err := mb.read_holding_registers(conn, client, 100, 2)
if err != null { handle_error(err) }
value := decoder.f32(regs, 0)

out := decoder.f32_regs(value)
err  = mb.write_multiple_registers(conn, client, 100, out)
```

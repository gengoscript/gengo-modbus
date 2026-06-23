# gengo-modbus

Modbus TCP client written in [Gengoscript](https://github.com/gengoscript/gengo).

This is a real-world use of Gengo's module system, module-qualified types,
bitwise operators, and the `cap:net` TCP capability.

## Requirements

- Gengo v0.5.0-pre5 or later (module-qualified types, `--modules` flag)
- `cap:net` capability enabled (passed via `--cap net`)

## Usage

```bash
# Against any Modbus TCP simulator on localhost:502
gengo --cap net client.gengo
```

The script connects, reads holding registers 0–4, writes register 0, reads
coils 0–7, and writes/reads back registers 10–14.

### With a custom host or port

Edit the constants at the top of `client.gengo`:

```gengo
host    := "192.168.1.10"
port    := 502
unit_id := 1
```

## Module layout

```
modbus/
  bytes.gengo       Byte codec — pack/unpack between []int and binary strings
  constants.gengo   Function codes and exception codes
  frame.gengo       MBAP frame build + parse; exports Response struct
  client.gengo      Request helpers; exports Client struct
client.gengo        Demo script
ISSUES.md           Known Gengo language limitations encountered
```

## Design notes

### Connection ownership

`Client` holds only `unit_id` and `tx_id`. The TCP connection is passed
explicitly to each operation. This keeps connection lifecycle — deadline
management, reconnect logic, multiplexing — in the calling script rather than
buried in the library.

```gengo
conn   := net.dial("tcp", "127.0.0.1:502")
defer conn.close()
conn.set_deadline(5000)

client := mb.Client { unit_id: 1, tx_id: 0 }
regs   := mb.read_holding_registers(conn, client, 100, 10)
```

### Byte packing via hex

Gengo has no `std.bytes` primitive yet (see `ISSUES.md`). Binary frames are
built by composing hex strings and decoding them:

```gengo
frame := std.hex.decode(b.u16hex(tx_id) + "0000" + b.u16hex(length) + ...)
```

`std.hex.encode`/`std.hex.decode` operate on raw bytes, so binary data from
`cap:net` round-trips correctly.

### Module-qualified types

`fr.Response` as a return type annotation and `c Client` as a parameter
annotation exercise Gengo's module-qualified type feature introduced in
v0.5.0-pre5:

```gengo
fr := import("./frame")

func recv(conn) fr.Response { ... }
pub func read_holding_registers(conn, c Client, address int, count int) []int { ... }
```

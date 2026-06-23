# gengo-modbus

Modbus TCP client library written in [Gengo](https://github.com/gengoscript/gengo).

A real-world use of Gengo's module system, module-qualified types, bitwise
operators, structural interfaces, and the `cap:net` TCP capability.

## Requirements

- Gengo v0.5.0-pre6 or later (`std.bytes`, module-qualified types, `--modules` flag)
- `cap:net` capability enabled (passed via `--cap net`)

## Usage

```bash
gengo --cap net --modules ./modbus demo.gengo
```

The demo connects to `127.0.0.1:5020`, reads holding registers, writes a
register and a coil, reads coils, and writes/reads back multiple registers.

### Changing the target

Edit the constants at the top of `demo.gengo`:

```gengo
host    := "192.168.1.10"
port    := 502
unit_id := 1
```

## Module layout

```
modbus/
  constants.gengo   Function codes and exception codes
  frame.gengo       MBAP frame build + parse; exports Response struct
  client.gengo      Request helpers; exports Client struct and Connection interface
demo.gengo          Demo script
```

## API

Import the library from a script in the same directory:

```gengo
mb := import("./client")
```

Or with `--modules ./modbus`:

```gengo
mb := import("./client")
```

### `Client` struct

```gengo
pub type Client struct {
    unit_id int,
    tx_id   int,
}
```

### `Connection` interface

Any value with `read(int) string`, `write(string) int`, `close()`, and
`set_deadline(int)` satisfies `Connection`. `cap:net.Conn` does.

### Functions

```gengo
mb.read_holding_registers(conn Connection, c Client, address int, count int) []int
mb.read_input_registers(conn Connection, c Client, address int, count int)   []int
mb.read_coils(conn Connection, c Client, address int, count int)             []bool
mb.read_discrete_inputs(conn Connection, c Client, address int, count int)   []bool
mb.write_single_register(conn Connection, c Client, address int, value int)  bool
mb.write_single_coil(conn Connection, c Client, address int, on bool)        bool
mb.write_multiple_registers(conn Connection, c Client, address int, values []int) bool
```

All functions return an error value (checkable with `std.core.is_error`) on
exception response or connection failure.

## Design notes

### Connection ownership

`Client` holds only `unit_id` and a transaction counter. The TCP connection is
passed to each operation so that deadline management, reconnect logic, and
multiplexing stay in the caller:

```gengo
conn   := net.dial("tcp", "127.0.0.1:502")
defer conn.close()
conn.set_deadline(5000)

client := mb.Client { unit_id: 1, tx_id: 0 }
regs   := mb.read_holding_registers(conn, client, 100, 10)
```

### Binary framing with `std.bytes`

Frames are assembled and parsed using `std.bytes` — the raw-byte string
primitive introduced in v0.5.0-pre6. `std.bytes.u8(n)` produces a single raw
byte (unlike `string(rune(n))` which produces UTF-8), and `std.bytes.u16be`/
`u32be` write big-endian integers directly into binary strings:

```gengo
header := std.bytes.u16be(tx_id) + std.bytes.u16be(0) +
          std.bytes.u16be(pdu_len) + std.bytes.u8(unit_id)
```

### Module-qualified types

`fr.Response` as a return type and `c Client` as a parameter annotation use
Gengo's module-qualified type feature (v0.5.0-pre5+):

```gengo
fr := import("./frame")

func recv(conn Connection) fr.Response { ... }
pub func read_holding_registers(conn Connection, c Client, ...) []int { ... }
```

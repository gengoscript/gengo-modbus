# gengo-modbus

Modbus TCP client and server library written in [Gengo](https://github.com/gengoscript/gengo).

A real-world use of Gengo's module system, module-qualified types, bitwise
operators, structural interfaces, and the `cap:net` TCP capability.

## Requirements

- Gengo v0.5.0-pre6 or later (`std.bytes`, module-qualified types, `--modules` flag)
- `cap:net` capability enabled (passed via `--cap net` for client, `--cap net=listen --net-listen-allow <addr>` for server)

## Usage

### Client

```bash
gengo --cap net --modules ./modbus demo.gengo
```

The demo connects to `127.0.0.1:5020` (the temperature controller server),
reads temperature/humidity sensors, reads and writes a setpoint with clamping,
toggles heater/fan coils, reads a dynamic alarm coil, tests partial-range
error rejection, and reads/writes float registers via the codec.

### Server

```bash
gengo --cap net=listen --net-listen-allow 127.0.0.1:5020 --modules ./modbus server_demo.gengo
```

The server demo starts a Modbus TCP server on `127.0.0.1:5020` with pre-loaded
register and coil values.  Connect with the client demo in another terminal to
read and write data.

`--net-listen-allow` is required because Gengo's listen policy defaults to
deny-all.  The pattern specifies which address:port the server may bind.
Use `0.0.0.0:5020` to accept connections on all interfaces.

### Running both

Start the server in one terminal, then the client in another:

```bash
# terminal 1
gengo --cap net=listen --net-listen-allow 127.0.0.1:5020 --modules ./modbus server_demo.gengo

# terminal 2
gengo --cap net --modules ./modbus demo.gengo
```

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
  server.gengo      Server with data stores; exports Server struct and listen
  codec.gengo       IEEE 754 float / integer register codec; exports Codec struct
demo.gengo          Client demo script
server_demo.gengo   Server demo script
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
mb.read_holding_registers(conn Connection, c Client, address int, count int) ([]int, ?error)
mb.read_input_registers(conn Connection, c Client, address int, count int)   ([]int, ?error)
mb.read_coils(conn Connection, c Client, address int, count int)             ([]bool, ?error)
mb.read_discrete_inputs(conn Connection, c Client, address int, count int)   ([]bool, ?error)
mb.write_single_register(conn Connection, c Client, address int, value int)  ?error
mb.write_single_coil(conn Connection, c Client, address int, on bool)        ?error
mb.write_multiple_registers(conn Connection, c Client, address int, values []int) ?error
mb.write_multiple_coils(conn Connection, c Client, address int, values []bool)   ?error
```

Read functions return a `(result, ?error)` tuple — check `err != null` before
using the result.  Write functions return `?error` directly (`null` on success).

### Server

```gengo
server := import("./modbus/server")

// Create a server with unit ID 1 and data stores.
s := server.new(1, 100, 100, 64, 64)  // holding, input, coils, discrete

// Set initial values.
server.set_holding(s, 0, 42)
server.set_coil(s, 0, true)

// Start listening (blocks indefinitely).
server.listen("0.0.0.0:5020", s)
```

### `Server` struct

```gengo
pub type Server struct {
    unit_id         int,
    holding_regs    []int,
    input_regs      []int,
    coils           []bool,
    discrete_inputs  []bool,
    // range callback arrays (populated via setter functions below)
    on_read_holding   []RegRangeReadCb,
    on_write_holding  []RegRangeWriteCb,
    on_read_input     []RegRangeReadCb,
    on_write_input    []RegRangeWriteCb,
    on_read_coil      []CoilRangeReadCb,
    on_write_coil     []CoilRangeWriteCb,
    on_read_discrete  []CoilRangeReadCb,
    on_write_discrete []CoilRangeWriteCb,
}
```

### Server functions

```gengo
server.new(unit_id int, holding_count int, input_count int,
           coil_count int, discrete_count int) Server

server.set_holding(s Server, address int, value int)
server.set_input(s Server, address int, value int)
server.set_coil(s Server, address int, value bool)
server.set_discrete(s Server, address int, value bool)

// Register range-based callbacks (see Server Callbacks section).
server.on_read_holding_range(s Server, start int, count int, cb func(int, []int) []int)
server.on_write_holding_range(s Server, start int, count int, cb func(int, []int))
server.on_read_input_range(s Server, start int, count int, cb func(int, []int) []int)
server.on_write_input_range(s Server, start int, count int, cb func(int, []int))
server.on_read_coil_range(s Server, start int, count int, cb func(int, []bool) []bool)
server.on_write_coil_range(s Server, start int, count int, cb func(int, []bool))
server.on_read_discrete_range(s Server, start int, count int, cb func(int, []bool) []bool)
server.on_write_discrete_range(s Server, start int, count int, cb func(int, []bool))

server.listen(addr string, s Server)   // blocks; requires --cap net=listen --net-listen-allow <addr>
```

The server handles all eight standard function codes automatically:

| Function code | Operation |
|---------------|-----------|
| 1 | Read coils |
| 2 | Read discrete inputs |
| 3 | Read holding registers |
| 4 | Read input registers |
| 5 | Write single coil |
| 6 | Write single register |
| 15 | Write multiple coils |
| 16 | Write multiple registers |

Out-of-range addresses return an `IllegalAddress` exception.  Unknown function
codes return an `IllegalFunction` exception.  Requests for a different unit ID
are silently ignored.

### Server Callbacks

The server uses range-based callbacks.  Register a callback on a span of
registers or coils and the server invokes it for reads/writes that cover the
full range.  Partial-range access — reading or writing a subset of a registered
range — returns an `IllegalAddress` exception, enforcing atomic access to
compound values such as float32 (2 registers) or float64 (4 registers).

Single-register values use `count=1`.  Multiple ranges can coexist and a
single read/write can span multiple ranges.

**Read callbacks** receive the start address and a slice of current values, and
return effective values:

```gengo
// Float32 temperature sensor at registers 10–11 (count=2).
server.on_read_holding_range(s, 10, 2, func(addr int, vals []int) []int {
    temp := decode_float32(vals) + 0.5   // apply offset
    return encode_float32(temp)
})

// Single-register dynamic value (count=1).
server.on_read_holding_range(s, 0, 1, func(addr int, vals []int) []int {
    return [vals[0] + 100]   // add 100 to read value
})

// Coil range: 8 coils read as a group.
server.on_read_coil_range(s, 0, 8, func(addr int, vals []bool) []bool {
    vals[0] = read_physical_button()   // override coil 0
    return vals
})
```

**Write callbacks** fire after the static store is updated:

```gengo
// Single-register write callback with clamping.
server.on_write_holding_range(s, 0, 1, func(addr int, vals []int) {
    setpoint = clamp(vals[0], 1500, 3500)
})

// Two-register range write callback.
server.on_write_holding_range(s, 10, 2, func(addr int, vals []int) {
    std.io.println("wrote float32 to regs " + std.conv.to_string(addr) + ".." + std.conv.to_string(addr + 1))
})

// Coil write callback.
server.on_write_coil_range(s, 0, 1, func(addr int, vals []bool) {
    if vals[0] { turn_on_actuator() } else { turn_off_actuator() }
})
```

**Partial-range error:** if a client reads register 10 alone but register 10
is part of a 2-register range (10–11), the server returns an `IllegalAddress`
exception.  The client must read the full range.

**Callback signatures:**

| Callback | Signature | Purpose |
|----------|-----------|---------|
| `on_read_holding_range` | `func(int, []int) []int` | `(start, current_values) → effective values` |
| `on_write_holding_range` | `func(int, []int)` | `(start, new_values)` — side effect after store update |
| `on_read_input_range` | `func(int, []int) []int` | same pattern |
| `on_write_input_range` | `func(int, []int)` | same pattern |
| `on_read_coil_range` | `func(int, []bool) []bool` | `(start, current_values) → effective values` |
| `on_write_coil_range` | `func(int, []bool)` | `(start, new_values)` — side effect after store update |
| `on_read_discrete_range` | `func(int, []bool) []bool` | same pattern |
| `on_write_discrete_range` | `func(int, []bool)` | same pattern |

### `Codec` struct — float and integer register decoding / encoding

```gengo
cdc := import("./modbus/codec")

// Create a codec for a specific device's byte order.
decoder := cdc.Codec { byte_order: cdc.CDAB }  // most common Modbus convention

// Decode a 32-bit float from two consecutive registers.
regs, err := mb.read_holding_registers(conn, client, 100, 2)
if err != null { handle_error(err) }
value := decoder.f32(regs, 0)

// Encode a float back to register values for writing.
out := decoder.f32_regs(value)
err  = mb.write_multiple_registers(conn, client, 100, out)
```

**Byte orders** (ABCD naming convention, A = most significant byte):

| Constant | Description | Used by |
|----------|-------------|---------|
| `ABCD`   | Big-endian (IEEE 754 native) | Siemens S7, Allen-Bradley BE mode |
| `CDAB`   | Word-swapped (most common) | Schneider/Modicon, Emerson, Yaskawa |
| `BADC`   | Byte-swapped within each word | Some Danfoss devices |
| `DCBA`   | Fully little-endian | Some x86-based devices |

**Methods on `Codec`**:

```gengo
// 32-bit float (2 registers)
cdc.f32(regs []int, i int) float       // decode at register index i
cdc.f32_regs(v float) []int            // encode → 2 register values

// 64-bit double (4 registers)
cdc.f64(regs []int, i int) float       // decode at register index i
cdc.f64_regs(v float) []int            // encode → 4 register values

// 32-bit unsigned integer (2 registers)
cdc.u32(regs []int, i int) int         // decode at register index i
cdc.u32_regs(v int) []int              // encode → 2 register values
```

## Design notes

### Connection ownership

`Client` holds only `unit_id` and a transaction counter. The TCP connection is
passed to each operation so that deadline management, reconnect logic, and
multiplexing stay in the caller:

```gengo
conn := net.dial("tcp", "127.0.0.1:502")
defer conn.close()
conn.set_deadline(5000)

client := mb.Client { unit_id: 1, tx_id: 0 }
regs, err := mb.read_holding_registers(conn, client, 100, 10)
if err != null { handle_error(err) }
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

func send_request(conn Connection, c Client, ...) (fr.Response, ?error) { ... }
pub func read_holding_registers(conn Connection, c Client, ...) ([]int, ?error) { ... }
```

### Listen policy

Unlike `net.dial` (which allows all destinations by default), `net.listen`
defaults to deny-all.  A host must add at least one allow rule before the
server can bind a port:

```bash
gengo --cap net=listen --net-listen-allow 127.0.0.1:5020 ...
gengo --cap net=listen --net-listen-allow 0.0.0.0:5020 ...       # all interfaces
```

This is a security measure: a listening socket is reachable by anyone who
can reach the bound port, not just destinations the script chose to call.

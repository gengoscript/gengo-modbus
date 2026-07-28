Server API
==========

Import
------

```gengo
server := import("./modbus/server")
```

Construction
------------

```gengo
s := server.new(unit_id, holding_count, input_count, coil_count, discrete_count)
```

Server struct
-------------

```gengo
pub type Server struct {
    unit_id         int,
    holding_regs    []int,
    input_regs      []int,
    coils           []bool,
    discrete_inputs  []bool,
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

Data store access
-----------------

```gengo
server.set_holding(s, address, value)
server.set_input(s, address, value)
server.set_coil(s, address, value)
server.set_discrete(s, address, value)
```

Server functions
----------------

```gengo
server.new(unit_id int, holding_count int, input_count int,
           coil_count int, discrete_count int) Server

server.set_holding(s Server, address int, value int)
server.set_input(s Server, address int, value int)
server.set_coil(s Server, address int, value bool)
server.set_discrete(s Server, address int, value bool)

server.on_read_holding_range(s Server, start int, count int, cb func(int, []int) []int)
server.on_write_holding_range(s Server, start int, count int, cb func(int, []int))
server.on_read_input_range(s Server, start int, count int, cb func(int, []int) []int)
server.on_write_input_range(s Server, start int, count int, cb func(int, []int))
server.on_read_coil_range(s Server, start int, count int, cb func(int, []bool) []bool)
server.on_write_coil_range(s Server, start int, count int, cb func(int, []bool))
server.on_read_discrete_range(s Server, start int, count int, cb func(int, []bool) []bool)
server.on_write_discrete_range(s Server, start int, count int, cb func(int, []bool))

server.listen(addr string, s Server)   // blocks
```

Supported function codes
------------------------

| Code | Operation |
|------|-----------|
| 1    | Read coils |
| 2    | Read discrete inputs |
| 3    | Read holding registers |
| 4    | Read input registers |
| 5    | Write single coil |
| 6    | Write single register |
| 15   | Write multiple coils |
| 16   | Write multiple registers |

Out-of-range addresses return `IllegalAddress`. Unknown function codes return
`IllegalFunction`. Requests for a different unit ID are silently ignored.

Range callbacks
---------------

Register a callback on a span of registers or coils.  The server invokes it for
accesses that cover the full range.  Partial-range access — reading or writing
a subset of a registered range — returns an `IllegalAddress` exception,
enforcing atomic access to compound values (e.g. float32 needs 2 registers).

Read callbacks receive `(start_address, current_values)` and return effective
values:

```gengo
server.on_read_holding_range(s, 10, 2, func(addr int, vals []int) []int {
    temp := decode_float32(vals) + 0.5
    return encode_float32(temp)
})

server.on_read_coil_range(s, 0, 8, func(addr int, vals []bool) []bool {
    vals[0] = read_physical_button()
    return vals
})
```

Write callbacks fire after the static store is updated:

```gengo
server.on_write_holding_range(s, 0, 1, func(addr int, vals []int) {
    setpoint = clamp(vals[0], 1500, 3500)
})

server.on_write_coil_range(s, 0, 1, func(addr int, vals []bool) {
    if vals[0] { turn_on_actuator() } else { turn_off_actuator() }
})
```

Callback signatures
-------------------

| Callback | Signature |
|----------|-----------|
| `on_read_holding_range` | `func(int, []int) []int` |
| `on_write_holding_range` | `func(int, []int)` |
| `on_read_input_range` | `func(int, []int) []int` |
| `on_write_input_range` | `func(int, []int)` |
| `on_read_coil_range` | `func(int, []bool) []bool` |
| `on_write_coil_range` | `func(int, []bool)` |
| `on_read_discrete_range` | `func(int, []bool) []bool` |
| `on_write_discrete_range` | `func(int, []bool)` |

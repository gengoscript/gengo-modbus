Client API
==========

Import
------

```gengo
mb := import("./modbus/client")
```

Client struct
-------------

```gengo
pub type Client struct {
    unit_id int,
    tx_id   int,
}
```

`unit_id` is the Modbus slave unit ID. `tx_id` auto-increments with each request.

Connection interface
--------------------

Any value with these methods satisfies `Connection`:

```gengo
type Connection interface {
    read(int) string,
    write(string) int,
    close(),
    set_deadline(int),
}
```

`cap:net.Conn` implements this interface.

Functions
---------

Read functions return `(result, ?error)`. Check `err != null` before using the
result.  Write functions return `?error` directly (`null` on success).

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

Example
-------

```gengo
conn := net.dial("tcp", "127.0.0.1:502")
defer conn.close()
conn.set_deadline(5000)

client := mb.Client { unit_id: 1, tx_id: 0 }

regs, err := mb.read_holding_registers(conn, client, 0, 5)
if err != null {
    std.io.println("error: " + string(err))
} else {
    for r in regs { std.io.println(std.conv.to_string(r)) }
}
```

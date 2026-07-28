Design notes
============

Connection ownership
--------------------

`Client` holds only `unit_id` and a transaction counter.  The TCP connection is
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

Binary framing with std.bytes
-----------------------------

Frames are assembled and parsed using `std.bytes` — the raw-byte string
primitive introduced in v0.5.0-pre6. `std.bytes.u8(n)` produces a single raw
byte (unlike `string(rune(n))` which produces UTF-8), and `std.bytes.u16be`/
`u32be` write big-endian integers directly into binary strings:

```gengo
header := std.bytes.u16be(tx_id) + std.bytes.u16be(0) +
          std.bytes.u16be(pdu_len) + std.bytes.u8(unit_id)
```

Module-qualified types
----------------------

`fr.Response` as a return type and `c Client` as a parameter annotation use
Gengo's module-qualified type feature (v0.5.0-pre5+):

```gengo
fr := import("./frame")

func send_request(conn Connection, c Client, ...) (fr.Response, ?error) { ... }
pub func read_holding_registers(conn Connection, c Client, ...) ([]int, ?error) { ... }
```

Listen policy
-------------

Unlike `net.dial` (which allows all destinations by default), `net.listen`
defaults to deny-all.  A host must add at least one allow rule before the
server can bind a port:

```bash
gengo --cap net=listen --net-listen-allow 127.0.0.1:5020 ...
gengo --cap net=listen --net-listen-allow 0.0.0.0:5020 ...       # all interfaces
```

This is a security measure: a listening socket is reachable by anyone who
can reach the bound port, not just destinations the script chose to call.

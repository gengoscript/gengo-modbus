# gengo-modbus

Modbus TCP client and server library written in [Gengo](https://github.com/gengoscript/gengo).

A real-world use of Gengo's module system, module-qualified types, bitwise
operators, structural interfaces, and the `cap:net` TCP capability.

## Requirements

- Gengo v0.5.0-pre6 or later
- `cap:net` capability (`--cap net` for client, `--cap net=listen --net-listen-allow <addr>` for server)

## Quick start

```bash
# terminal 1 — start the server
gengo --cap net=listen --net-listen-allow 127.0.0.1:5020 --modules ./modbus server_demo.gengo

# terminal 2 — run the client demo
gengo --cap net --modules ./modbus demo.gengo
```

## Module layout

```
modbus/
  constants.gengo   Function codes and exception codes
  frame.gengo       MBAP frame build + parse
  client.gengo      Client struct, Connection interface, read/write functions
  server.gengo      Server with data stores, range callbacks, listen
  codec.gengo       IEEE 754 float / integer register codec
demo.gengo          Client demo (temperature controller)
server_demo.gengo   Server demo (temperature controller)
```

## Documentation

- [Client API](docs/client.md)
- [Server API](docs/server.md)
- [Codec](docs/codec.md)
- [Design notes](docs/design.md)

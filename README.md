# gn.hml

WebSocket networking framework for [Hemlock](https://github.com/hemlang/hemlock). A port of [gn.js](https://github.com/Yotis-Studios/gn.js).

Lightweight binary protocol with automatic type detection, designed for real-time multiplayer applications.

## Requirements

- Hemlock 1.9.4+
- libwebsockets (`brew install libwebsockets` / `apt install libwebsockets-dev`)

## Quick Start

**Echo server + client:**

```hemlock
import { Server } from "src/network/server.hml";
import { Client } from "src/network/client.hml";
import { Packet } from "src/network/packet.hml";
import { sleep } from "@stdlib/time";

// Server echoes packets back
let server = Server();
server.on("packet", fn(conn, packet) {
    conn.send(packet);
});

async fn run_server(srv, p) { srv.listen(p); }
spawn(run_server, server, 3000);
sleep(0.5);

// Client sends and receives
let client = Client();
client.connect("127.0.0.1", 3000);

let p = Packet(1);
p.add("Hello");
p.add(42);
client.send(p);

let echo = client.recv_packet();
print(echo.get(0)); // "Hello"
print(echo.get(1)); // 42

client.disconnect();
server.close();
```

## API

### Packet

```hemlock
let p = Packet(netId);
p.add(value);        // Add a value (auto-detects type)
p.add([1, 2, 3]);   // Add multiple values
p.get(0);           // Access value at index
p.build();           // Serialize to binary buffer
p.load(buffer);      // Deserialize from binary buffer
```

Supported types: integers (u8/u16/u32/s8/s16/s32), floats (f32/f64), strings, buffers, null.

### Server

```hemlock
let server = Server();
server.on("ready", fn() { });
server.on("connect", fn(connection) { });
server.on("packet", fn(connection, packet) { });
server.on("disconnect", fn(connection) { });
server.on("error", fn(err) { });
server.listen(port);           // Blocks - run in a spawned task
server.broadcast(packet, exclude);
server.close();
```

`listen()` blocks the calling thread. Spawn it in a background task:

```hemlock
async fn run_server(srv, p) { srv.listen(p); }
spawn(run_server, server, 3000);
```

### Client

```hemlock
let client = Client();
client.on("connect", fn() { });
client.on("packet", fn(packet) { });
client.on("disconnect", fn() { });
client.connect(address, port);
client.send(packet);
client.recv_packet();   // Block until next packet (returns null on disconnect)
client.run();           // Block and dispatch callbacks until disconnect
client.disconnect();
client.is_connected();
```

Two receive modes:
- **`recv_packet()`** - synchronous, returns one packet at a time
- **`run()`** - blocks and dispatches registered callbacks

### Connection (server-side)

```hemlock
connection.send(packet);
connection.broadcast(packet);  // Send to all except this connection
connection.kick();             // Close connection
```

## Protocol

Wire-compatible with gn.js. Packet format:

```
[2 bytes LE] size (payload length, excluding this field)
[2 bytes LE] netId (message type identifier)
[typed data elements...]

Each data element:
[1 byte]     type index (0-11)
[variable]   data bytes (little-endian)
```

## Running Examples

```bash
hemlock example/echo.hml
hemlock example/pingpong.hml
```

## Running Tests

```bash
hemlock test/gm_convert_test.hml    # 34 tests - binary conversion
hemlock test/packet_test.hml        # 10 tests - packet serialization
hemlock test/server_client_test.hml #  5 tests - server/client integration
```

## License

MIT

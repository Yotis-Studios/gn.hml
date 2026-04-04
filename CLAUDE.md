# CLAUDE.md

## Project Overview

gn.hml is a Hemlock port of [gn.js](https://github.com/Yotis-Studios/gn.js), a WebSocket networking framework originally built for GameMaker and Node.js. It implements the same binary packet protocol with automatic type detection and serialization.

## Architecture

The binary protocol is identical to gn.js:
- Packet format: `[2-byte LE size][2-byte LE netId][typed data elements...]`
- Each data element: `[1-byte type][data bytes]`
- 12 types: u8, u16, u32, s8, s16, s32, f16, f32, f64, string, buffer, undefined
- Little-endian byte order throughout (native on x86/ARM)

### Hemlock Concurrency Model

Hemlock deep-copies all arguments passed to `spawn()`. This means:
- **Objects, arrays, strings, buffers** are fully copied — mutations in the spawned task do NOT affect the caller.
- **Channels** are shared (retained, not copied) — they're the primary cross-task communication mechanism.
- **WebSocket handles** are shared (retained) — they can be used across tasks.

This fundamentally prevents the Node.js EventEmitter pattern where callbacks mutate shared state. The architecture differs from gn.js:

- **Server**: `listen()` blocks the calling thread and runs an internal event loop (reads from a channel fed by background accept/recv tasks). Spawn it in a background task. Server-side callbacks (on packet, on connect) run inside the server's event loop and CAN access the server's own state (connections, etc.) since they execute on the same thread.
- **Client**: Uses `recv_packet()` for synchronous blocking receive on the main thread. The `on()` + `run()` callback API also works but only within the client's own event loop.

### Typical Usage Pattern

```hemlock
// Server in background task (callbacks are self-contained)
let server = Server();
server.on("packet", fn(conn, packet) {
    conn.send(packet); // echo
});
async fn run_server(srv, p) { srv.listen(p); }
spawn(run_server, server, port);

// Client on main thread (synchronous send/recv)
let client = Client();
client.connect("127.0.0.1", port);
client.send(packet);
let response = client.recv_packet(); // blocks until packet arrives
```

## File Structure

```
src/
  index.hml              - re-exports Packet, Server, Client, Connection
  util/gm_convert.hml    - binary type detection and serialization
  network/
    packet.hml           - Packet build/load (binary protocol)
    connection.hml       - server-side connection wrapper
    server.hml           - WebSocket server (channel-based event loop)
    client.hml           - WebSocket client (sync recv + callback modes)
example/
  echo.hml               - echo server/client demo
  pingpong.hml           - ping-pong exchange demo
test/
  gm_convert_test.hml    - 45 tests for binary conversion
  packet_test.hml        - 13 tests for packet serialization
  server_client_test.hml - 5 integration tests
  server_advanced_test.hml - 4 tests: callbacks, multi-client, broadcast
  network_types_test.hml - 4 tests: floats, negatives, u32, client run()
```

## Running

Requires Hemlock 1.9.4+ (for WebSocket binary support and spawn fix).

```bash
hemlock example/echo.hml
hemlock test/gm_convert_test.hml
hemlock test/packet_test.hml
hemlock test/server_client_test.hml
hemlock test/server_advanced_test.hml
hemlock test/network_types_test.hml
```

## Hemlock Quirks Found During Development

- **`spawn()` without capturing the handle was broken before 1.9.4.** The task would get freed while the thread was still running (use-after-free). Fixed by having the worker thread hold a reference. Always use Hemlock >= 1.9.4.
- **`__sleep()` vs `sleep()` from `@stdlib/time`**: Both are the same function. Either works. Prefer the stdlib import for consistency.
- **Object method syntax**: Use `name: fn() {}` not `fn name() {}` inside object literals. The latter is a parse error.
- **`select([ch], 0)` works** for non-blocking channel poll when data is already buffered. Works correctly with channels shared across tasks.
- **WebSocket binary recv** requires `__lws_msg_binary` builtin (added in 1.9.3). Messages arrive as `{ type: "binary", binary: <buffer> }`.
- **Port reuse**: WebSocket servers need `LWS_SERVER_OPTION_ALLOW_LISTEN_SHARE` for rapid rebind after close (added in 1.9.4).
- **Float serialization**: Uses `ptr_write_f32`/`ptr_read_f32` via temporary `alloc()` buffers to get IEEE 754 bytes. Native byte order matches the LE protocol on x86/ARM.
- **String-to-bytes**: Use `str.to_bytes()` for UTF-8 buffer, `__string_from_bytes(array)` for reconstruction.

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
  multiclient.hml        - 3 concurrent clients demo
test/
  gm_convert_test.hml    - 49 tests for binary conversion
  packet_test.hml        - 14 tests for packet serialization
  server_client_test.hml - 6 integration tests
```

## Running

Requires Hemlock 2.7.0+ (uses `from_bytes` from `@stdlib/strings`, added in 2.7.0, and the `@stdlib/websocket` module which requires `make stdlib` during Hemlock build). Last verified against Hemlock 2.8.1: all tests and examples pass.

```bash
hemlock example/echo.hml
hemlock test/gm_convert_test.hml
hemlock test/packet_test.hml
hemlock test/server_client_test.hml
```

## Hemlock Quirks Found During Development

- **Hemlock 2.0.0 moved 63 builtins to `@stdlib` modules.** Math, signal, net, process, fs, atomic, debug, and ffi functions now require explicit imports. This project's builtins (`alloc`, `free`, `ptr_write_f32`, `__string_from_bytes`, etc.) were NOT moved and remain available globally.
- **`@stdlib/websocket` requires `make stdlib`** during Hemlock build to compile the libwebsockets C wrapper (`lws_wrapper.so`). Without it, WebSocket imports will fail at runtime.
- **Object method syntax**: Hemlock 2.7.0 added `fn name() {}` shorthand inside object literals; on 2.0.0–2.6.x it is a parse error. This codebase keeps the `name: fn() {}` spelling, which works everywhere.
- **`select([ch], 0)` works** for non-blocking channel poll when data is already buffered. Works correctly with channels shared across tasks.
- **WebSocket recv messages** arrive as `{ type: "binary", binary: <buffer> }` for binary data and `{ type: "close" }` for disconnections via `@stdlib/websocket`.
- **Float serialization**: Uses the typed buffer methods (`write_f32_le`/`read_f32_le` etc.) for IEEE 754 bytes in protocol byte order.
- **String-to-bytes**: Use `str.to_bytes()` for UTF-8 buffer, `from_bytes(src)` from `@stdlib/strings` for reconstruction (added in 2.7.0 as the documented replacement for the internal `__string_from_bytes` dunder).
- **Length-field overflow**: Hemlock's `write_u16_le`/`write_u8` silently wrap out-of-range values (Node's `Buffer` throws). The library validates string/buffer/packet sizes against their length fields and throws, matching gn.js behavior. The same applies to integer values: `determine_type` throws for values outside the u32/s32 range instead of letting `write_u32_le` wrap them.
- **Channel sharing requires pre-spawn creation**: nested channels inside an object are shared across a `spawn()` deep copy, but only if they exist at spawn time. `Server()` creates `_event_ch`/`_stop_ch`/`_stopped_ch`/`_done_ch` in the constructor for exactly this reason — it is what makes `close()` on the caller's copy able to reach the event loop running in the task's copy. (Before this, `close()` was silently a no-op: the caller's copy had null channels.)
- **`WebSocketServer.close()` stalls for seconds** (Hemlock 2.8.1): freeing the lws context takes ~4-5s for an idle server and ~20s+ once a client has connected (`lws_context_destroy` stall in the runtime, upstream issue). `Server.close()` therefore acknowledges once the server has *logically* stopped (event loop exited, clients kicked, accept loop stopped, ~1s) and lets the server task free the OS resources in the background. Rebinding the same port immediately after `close()` may fail.
- **u32 vs literal comparison** (Hemlock 2.8.1): a `u32` value read via `read_u32_le` compares `false` against an equal integer literal (e.g. `read_u32_le(...) == 4294967295` is false even though both print as `4294967295`). Compare via string rendering or another same-typed value.
- **f16 wire format**: values of type f16 (2 bytes, e.g. GameMaker's `buffer_f16`) are decoded manually in `parse_data_from_buffer` (Hemlock buffers have no `read_f16_le`). The library never *writes* f16 — `determine_type` picks f32/f64 — but decodes it for wire compat with gn.js/GameMaker peers.
- **Malformed input safety**: `parse_data_from_buffer` returns `{ data, size, valid }`; `valid: false` marks unknown types and truncated data so `Packet.load` stops cleanly instead of an out-of-bounds buffer read throwing through the server event loop. `valid: true` with `data: null` is a legitimate undefined value — parsing continues past it.

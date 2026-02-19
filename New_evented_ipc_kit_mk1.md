# New Evented IPC Kit (MK1)

## 1) Introduction

### Purpose
Evented IPC Kit (MK1) specifies a reusable, high-performance evented inter-process communication architecture that can be implemented in:
- Ruby
- Python
- JavaScript
- TypeScript
- Gleam
- Elixir
- PHP
- Rust
- C++
- Go

It is built from proven patterns in:
- `Asteria/src/Ipc.cpp`
- `Asteria/src/EventQueue.cpp`
- `Asteria/Run/RequestMapper.cpp`
- `Asteria/Run/MainLoop.cpp`
- conceptual overlap with continuation/event loop behavior in `PebbleScript/include/PebbleScript.h`

### Problem statement
Applications repeatedly need the same infrastructure:
1. robust message framing over streams/sockets,
2. safe concurrent event queues,
3. request/response correlation,
4. process/actor orchestration.

This spec standardizes those capabilities into one portable kit.

### Design goals
1. **Simple mental model**: event envelopes + queues + correlator + orchestrator.
2. **Fast by default**: O(1) enqueue/dequeue and linear I/O.
3. **Safe under failure**: bounded queues, timeouts, structured errors.
4. **Portable**: common contract across OO and non-OO ecosystems.
5. **Testable**: deterministic protocol, shared corpus, strict acceptance checks.

### Non-goals
- Not a full distributed consensus system.
- Not a durable message broker.
- Not tied to one serialization format (protobuf/json/msgpack all possible).

---

## 2) System Overview

## 2.1 Recommended split
1. **ipc-framing-core**
   - transport framing (magic/signature, length prefix, payload, checksum optional)
   - complete send/receive loops
2. **actor-event-queue**
   - thread-safe/event-loop-safe incoming/outgoing queues
   - blocking and non-blocking pop, notifications
3. **request-correlation-map**
   - request id generation
   - request/response channel correlation
   - pending map lifecycle (timeouts/cleanup)
4. **orchestrator layer (optional but strongly recommended)**
   - actor/process registry
   - event routing (spawn, kill, alive, data, ack)

## 2.2 Core message model

```text
EventEnvelope:
  version: u8
  event_type: enum
  data_type: enum|optional
  message_id: u64
  request_id: optional<u64>
  sender: EndpointRef
  receiver: EndpointRef
  timestamp_ms: u64
  flags: bitset
  payload: bytes

EndpointRef:
  id: string|u64
  name: optional<string>
```

### Required event types
- `DATA`
- `ACK`
- `ERROR`
- `SPAWN_REQUEST`
- `KILL_REQUEST`
- `ALIVE_REQUEST`
- `ALIVE_RESPONSE`
- `FINISHED`
- `KILLED`

---

## 3) Framing Protocol Specification

## 3.1 Frame layout (network byte order)

```text
| MAGIC(4) | VERSION(1) | FLAGS(1) | HEADER_LEN(2) | PAYLOAD_LEN(4) | HEADER(header_len bytes) | PAYLOAD(payload_len bytes) |
```

Default constants:
- `MAGIC = 0x45564B31` ("EVK1")
- `VERSION = 1`

### Header content (encoded independently)
Header holds metadata fields (event type, ids, sender/receiver, etc.) in stable encoded form (JSON/protobuf/msgpack/custom binary). Payload is opaque bytes.

## 3.2 Validation rules
1. Reject invalid magic/version.
2. Reject frame lengths above configured limits.
3. Reject malformed header decode.
4. Reject required-field violations (e.g., missing `event_type`, missing `sender`).

## 3.3 I/O behavior
- `send_frame` must write all bytes or return error.
- `recv_frame` must read exact byte counts; partial reads loop until complete or timeout/EOF/error.
- On protocol mismatch, return `ProtocolError` and close channel by policy.

---

## 4) Queue and Event Loop Semantics

## 4.1 Queue model
Each actor/process endpoint has:
- `in_queue` (received events)
- `out_queue` (events to send)
- optional `data_queue` for payload-oriented fast path

Queue policies:
- bounded capacity (required)
- backpressure mode: `block | drop_oldest | drop_newest | fail`

## 4.2 Loop model
Minimal loop behavior:
1. poll/read transport; push decoded events to `in_queue`.
2. pop from `out_queue`; encode/send.
3. notify waiters on incoming data.
4. process control events (kill/terminate) with priority.

### Scheduling
- Single-thread runtime: cooperative event loop.
- Multi-thread runtime: dedicated I/O thread + worker loop(s).
- Must avoid busy-spin; use blocking waits or timed waits.

---

## 5) Request-Response Correlation

## 5.1 Correlation model
Correlation key:
- primary: `request_id`
- fallback channel descriptor (for compatibility): `sender->receiver` directional channel queues

Structures:

```text
pending_by_request_id: Map<u64, PendingRequest>
pending_by_channel: Map<ChannelKey, Queue<u64>>

PendingRequest:
  request_id: u64
  sender: EndpointRef
  receiver: EndpointRef
  created_at_ms: u64
  timeout_ms: u64
  metadata: map<string, string>
```

## 5.2 Rules
1. New request:
   - assign unique `request_id`
   - insert pending entry
   - enqueue id in directional channel queue
2. Response:
   - must include `request_id` when available
   - resolve pending entry, remove from maps, emit completion
3. Timeout:
   - periodic sweep removes expired pending entries
   - emit timeout error event/callback

---

## 6) Orchestration Layer

## 6.1 Registry model

```text
actors_by_id: Map<ActorId, ActorHandle>
actors_by_name: Map<String, ActorHandle>
```

## 6.2 Required orchestrator operations
- spawn actor/process
- route event by id or name
- query liveness
- terminate actor/process
- clean up dead children/zombies (where relevant)

## 6.3 Routing algorithm (simplified)
1. Receive control/data event from actor A.
2. Resolve target actor B by name/id.
3. For `DATA`, set/validate request correlation and forward.
4. For control messages, perform operation and emit `ACK`/`ERROR`.
5. On child death, remove from registries and pending correlations.

---

## 7) Core Algorithms

## 7.1 Framing send algorithm
1. Serialize header.
2. Validate `header_len` and `payload_len` limits.
3. Write frame prefix then header then payload with full-write loop.
4. Return success or transport/protocol error.

## 7.2 Framing receive algorithm
1. Read fixed prefix.
2. Validate magic/version/lengths.
3. Read header and payload exact lengths.
4. Decode header and construct envelope.
5. Return envelope or structured error.

## 7.3 Queue push/pop algorithm
- Push:
  - lock or single-thread critical section
  - apply backpressure policy
  - append event
  - signal waiters
- Pop:
  - if non-blocking and empty: return none
  - else wait on condition/event with optional timeout
  - return event

## 7.4 Correlator algorithm
- `register_request`: generate id, store pending, return id
- `resolve_response`: locate by id; if found, remove + return pending
- `sweep_timeouts`: iterate pending map by deadline structure, emit timeout completions

## 7.5 Complexity targets
- enqueue/dequeue: O(1)
- register/resolve correlation: O(1) average
- timeout sweep:
  - O(n) simple map scan, or
  - O(log n) insertion + O(k log n) expiration using min-heap by deadline

---

## 8) Data Structures

Required:
1. `EventEnvelope` struct/class/record
2. bounded queue (ring buffer preferred)
3. pending map keyed by request id
4. optional min-heap for timeout deadlines
5. actor registry maps by id/name

Recommended queue implementation:
- ring buffer + mutex/condition (threads)
- mailbox process (actor runtimes like BEAM)
- channel/queue primitive if runtime provides bounded semantics

---

## 9) API Draft (OO style)

For Ruby, Python, JavaScript/TypeScript classes, PHP, C++, and Go methods on structs.

```text
class FrameCodec {
  encode(envelope: EventEnvelope) -> bytes
  decode(frame_bytes: bytes) -> EventEnvelope
  send(io, envelope, timeout_ms?) -> Result<void, IpcError>
  recv(io, timeout_ms?) -> Result<EventEnvelope, IpcError>
}

class EventQueue {
  push(event: EventEnvelope) -> Result<void, QueueError>
  pop(timeout_ms?) -> Option<EventEnvelope>
  try_pop() -> Option<EventEnvelope>
  size() -> int
  close() -> void
}

class RequestCorrelator {
  register_request(event: EventEnvelope, timeout_ms: int) -> u64
  resolve_response(event: EventEnvelope) -> Option<PendingRequest>
  sweep_timeouts(now_ms: u64) -> list<PendingRequest>
}

class Orchestrator {
  spawn(spec) -> ActorHandle
  route(event: EventEnvelope) -> Result<void, IpcError>
  kill(target) -> Result<void, IpcError>
  alive(target) -> bool
  loop_once() -> void
  run() -> void
  stop() -> void
}
```

Error base type:

```text
IpcError:
  code: enum/string
  message: string
  details: map
```

---

## 10) API Draft (Non-OO / Functional)

For Gleam, Elixir, Rust, Go (procedural style), functional JS/TS.

```text
type Envelope
type QueueState
type CorrelatorState
type OrchestratorState
type IpcError

encode_frame(envelope) -> Result(bytes, IpcError)
decode_frame(bytes) -> Result(envelope, IpcError)
send_frame(io, envelope, timeout_ms) -> Result(Nil, IpcError)
recv_frame(io, timeout_ms) -> Result(envelope, IpcError)

queue_new(capacity, policy) -> QueueState
queue_push(queue, envelope) -> { QueueState, Result(Nil, IpcError) }
queue_pop(queue, timeout_ms) -> { QueueState, Option(Envelope) }

corr_new() -> CorrelatorState
corr_register(corr, envelope, timeout_ms) -> { CorrelatorState, request_id }
corr_resolve(corr, envelope) -> { CorrelatorState, Option(PendingRequest) }
corr_sweep(corr, now_ms) -> { CorrelatorState, List(PendingRequest) }

orch_new(config) -> OrchestratorState
orch_route(orch, envelope) -> { OrchestratorState, Result(Nil, IpcError) }
orch_step(orch) -> { OrchestratorState, List(Effect) }
```

---

## 11) Security and Reliability Requirements

Mandatory safeguards:
1. **Frame length limits** (header and payload max bytes).
2. **Queue bounds** with explicit overflow policy.
3. **Timeouts everywhere** (send, recv, pending requests).
4. **Authentication hook** for trust boundaries (at least pluggable signature verifier).
5. **Integrity option** (checksum/MAC) for non-trusted channels.
6. **Replay mitigation option** for remote contexts (nonce/timestamp window).
7. **Graceful shutdown** that drains or rejects in-flight work by policy.
8. **No busy loops** without sleep/wait strategy.

Recommended defaults:
- max frame: 1 MiB payload, 64 KiB header
- max pending requests: configurable hard cap
- default request timeout: 30s

---

## 12) Language-specific Implementation Notes

## 12.1 Ruby
- Use `IO.select` for timeout-aware waits.
- Use `SizedQueue` or custom bounded queue for backpressure control.
- Avoid exception-heavy paths in hot I/O loops.

## 12.2 Python
- Support both sync (`socket`) and async (`asyncio`) adapters.
- Use `queue.Queue` or `asyncio.Queue` with explicit maxsize.
- Ensure thread-safe correlator updates (lock) in threaded mode.

## 12.3 JavaScript
- Node: implement stream framing with Buffer slices and full read loops.
- Browser/worker contexts: use MessagePort/WebSocket adapters where applicable.
- Keep event handlers non-blocking; heavy work offloaded.

## 12.4 TypeScript
- Use discriminated unions for `event_type` and `IpcError.code`.
- Provide strict typed envelope schema.
- Runtime validate decoded headers before casting.

## 12.5 Gleam
- Model orchestrator state transitions functionally.
- Use OTP/BEAM process mailboxes for actor-event-queue semantics.
- Keep correlation map updates pure and explicit.

## 12.6 Elixir
- Lean on GenServer/Supervisor for orchestration and lifecycle.
- Mailbox ordering is per-sender; document cross-sender ordering semantics.
- Use ETS for high-throughput correlation map when needed.

## 12.7 PHP
- Prefer long-running worker runtimes for evented behavior (Swoole/ReactPHP style adapters).
- For classic request/response PHP runtimes, scope kit to client role.
- Use binary-safe string operations for framing.

## 12.8 Rust
- Use `tokio` or std sync adapters behind traits.
- Strongly typed envelope and error enums; avoid panics on malformed input.
- Min-heap (`BinaryHeap` with reversed ordering) for timeout expiration.

## 12.9 C++
- Use `std::span`/`string_view` where possible; avoid unnecessary copies.
- Keep ownership explicit (`unique_ptr`/value types).
- Use condition variables for blocking queue operations.

## 12.10 Go
- Use goroutines + channels for loops, but enforce bounded buffering.
- Define context-aware operations (`context.Context`) for cancellation/timeouts.
- Avoid goroutine leaks on shutdown; close channels in strict order.

---

## 13) Testing Specification

## 13.1 Required test classes
1. **Framing tests**
   - valid frame round-trip
   - invalid magic/version
   - truncated frame
   - oversized frame rejection
2. **Queue tests**
   - FIFO correctness
   - bounded policy behavior
   - blocking pop timeout
3. **Correlation tests**
   - request registration uniqueness
   - response resolution correctness
   - timeout sweep correctness
4. **Orchestration tests**
   - spawn/route/kill/alive lifecycle
   - unknown target error handling
5. **Resilience tests**
   - abrupt socket close
   - malformed header payload
   - high-load burst with backpressure

## 13.2 Shared test corpus format
Use JSONL for cross-language portability:

```json
{"id":"frame_ok_001","kind":"frame","input_hex":"...","expect_ok":true}
{"id":"frame_bad_magic","kind":"frame","input_hex":"...","expect_ok":false,"error_code":"ProtocolError"}
{"id":"corr_timeout_001","kind":"corr","expect_expired":1}
```

## 13.3 Deterministic output contract
Each implementation should expose canonical JSON debug output for:
- envelope
- pending map snapshot
- orchestrator actor registry snapshot

---

## 14) AI/Human Acceptance Checklist

A generated implementation is acceptable only if all checks pass:

### Protocol correctness
- [ ] Frame encode/decode round-trips for valid fixtures.
- [ ] Partial read/write handling is correct.
- [ ] Length and version validation is enforced.

### Queue and loop behavior
- [ ] Queue is bounded with explicit overflow policy.
- [ ] Blocking operations honor timeout.
- [ ] Loop avoids busy-spin under idle conditions.

### Correlation and orchestration
- [ ] Request IDs are unique and collision-safe for configured domain.
- [ ] Responses correctly resolve pending requests.
- [ ] Timeouts remove stale pending entries.
- [ ] Actor registry updates are consistent on spawn/death.

### Security and reliability
- [ ] Max frame size and max pending limits are enforced.
- [ ] Malformed input never crashes runtime.
- [ ] Shutdown sequence is clean (no leaked threads/tasks/goroutines/processes).

### Performance
- [ ] Throughput and latency benchmarks exist for small/medium/large payloads.
- [ ] No avoidable per-byte allocations in hot framing path.

---

## 15) Reference Implementation Plan

1. Build `ipc-framing-core` with strict validation and tests.
2. Build bounded `actor-event-queue` with blocking/timeouts.
3. Build `request-correlation-map` with timeout sweeping.
4. Integrate orchestrator routing and lifecycle events.
5. Add corpus runner + stress benchmarks.
6. Add optional integrity/auth hooks.

---

## 16) Suggested Repository Layout

```text
evented-ipc-kit/
  spec/
    New_evented_ipc_kit_mk1.md
    corpus/
      framing_cases.jsonl
      correlation_cases.jsonl
      orchestration_cases.jsonl
  implementations/
    ruby/
    python/
    javascript/
    typescript/
    gleam/
    elixir/
    php/
    rust/
    cpp/
    go/
```

This keeps specification, validation corpus, and implementations aligned while preserving language autonomy.

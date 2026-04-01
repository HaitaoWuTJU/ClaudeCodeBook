# Summary of `cli/transports/`

## Purpose of `transports/`

Provides a **runtime-polymorphic transport layer** that bridges the CLI process and the remote execution runtime (CCR v2). Every transport shares the same `Transport` interface (`connect`, `write`, `writeBatch`, `flush`, `close`), but implementations vary in their read/write channel split and delivery guarantees, allowing a single call site to remain agnostic to the underlying protocol.

---

## Contents Overview

| File | Role |
|------|------|
| **`Transport.ts`** | Defines the `Transport` interface contract |
| **`getTransportForUrl.ts`** | Factory that inspects env vars and URL scheme, then instantiates the right transport |
| **`SSETransport.ts`** | Transport: HTTP SSE (long-poll) for reads; HTTP POST batched for writes |
| **`WebSocketTransport.ts`** | Transport: full WebSocket for reads and writes; automatic reconnect with exponential backoff; Bun + `ws` runtime support |
| **`HybridTransport.ts`** | Transport: WebSocket for reads; HTTP POST batched for writes (bridges Firestore-safe writes with real-time reads) |
| **`SerialBatchEventUploader.ts`** | Shared batching uploader: serializes writes, coalesces batches, retries with exponential backoff and jitter, enforces `maxQueueSize` backpressure |
| **`CCRClient.ts`** | HTTP-only transport for worker-to-runtime communication: heartbeats, worker state PUTs, internal event GET/POST, stream delta buffering/coalescing |
| **`WorkerStateUploader.ts`** | Specialized coalescing uploader: merges back-to-back PUT patches, sends one in-flight + one pending, absorbs intermediate updates |

---

## How Files Relate to Each Other

```
┌─────────────────────────────────────────────────────────────────────┐
│                      getTransportForUrl()                          │
│  (env-gated factory: SSE → Hybrid → WebSocket)                     │
└──────────────────────────┬────────────────────────────────────────┘
                           │ instantiates
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌────────────┐  ┌────────────┐  ┌─────────────────┐
    │SSETransport│  │HybridTrans │  │WebSocketTransport│
    └─────┬──────┘  └─────┬──────┘  └────────┬────────┘
          │               │                  │
          │               │                  │ full WebSocket
          │         WebSocket read           │ reconnect, buffering,
          │               │                   │ keepalive, ping/pong
          │               │                   │
          ▼               ▼                   ▼
    ┌──────────────────────────────────────────────────────────────┐
    │           SerialBatchEventUploader (shared)                 │
    │  Serializes writes; batches; retries w/ backoff; backpressure│
    └──────────────────────────────────────────────────────────────┘

    ┌──────────────────────────────────────────────────────────────┐
    │                       CCRClient                              │
    │  HTTP-only: heartbeat (POST), state PUT, internal event      │
    │  GET/POST, stream_event buffering + text_delta coalescing,   │
    │  delivery status uploads, and WorkerStateUploader for state   │
    └──────────────────────────────────────────────────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ WorkerStateUploader  │
                    │ (coalescing PUT)     │
                    └─────────────────────┘
```

**The `Transport` interface is the contract.** Both `SSETransport` and `HybridTransport` delegate their write path to `SerialBatchEventUploader`. `WebSocketTransport` bypasses it entirely — its writes are synchronous WebSocket sends (with local buffering for reconnect replay, not HTTP batching).

`CCRClient` is a separate HTTP-only client (not a `Transport` subclass) that uses `SerialBatchEventUploader` for its event writes and `WorkerStateUploader` for its state writes.

---

## Key Takeaways

### 1. Three transports, one interface, environment-gated selection

```
SSE  ← CLAUDE_CODE_USE_CCR_V2 && CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2
Hybrid ← CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2
WebSocket ← default
```

The factory pattern keeps call sites clean — every caller just calls `getTransportForUrl(url, headers, ...)` and gets the right transport.

### 2. Write serialization is mandatory to prevent Firestore collisions

All HTTP-based transports (`SSE`, `Hybrid`, `CCRClient`'s POST uploader) use `SerialBatchEventUploader`, which **serializes** all writes through a single queue. Concurrent POSTs to the same Firestore document would overwrite each other; the uploader prevents this by posting one batch at a time.

### 3. Stream event buffering collapses redundant text deltas

`HybridTransport` (100 ms buffer) and `CCRClient` (100 ms buffer) both coalesce `text_delta` SSE events into full-so-far snapshots before posting. This dramatically reduces per-delimiter POST count and ensures mid-stream reconnects see a consistent cursor rather than a burst of tiny deltas.

`CCRClient` additionally accumulates text deltas keyed by API message ID and clears them only when the complete `SDKAssistantMessage` arrives — a reliable end-of-stream signal that guards against spurious clears from abort/error events.

### 4. WebSocketTransport handles the long-running session problem

Browser and proxy idle timeouts kill idle WebSocket connections. `WebSocketTransport` counters this with:

- **Ping/pong** (10 s interval): detects dead connections and OS-level sleep/wake
- **Keep-alive data frames** (5 min interval): prevent proxy idle eviction (skipped in CCR remote sessions)
- **Exponential backoff reconnect** (1 s → 30 s, ±25% jitter): graceful recovery from transient outages
- **10-minute reconnect budget**: gives the server time to expire sessions during long sleeps
- **Message replay**: buffered messages are replayed on reconnect; server evicts confirmed ones via `x-last-request-id` (Node) or full replay (Bun)

### 5. CCRClient owns the worker lifecycle protocol

Unlike the other transports (which carry arbitrary events), `CCRClient` is purpose-built for the worker-to-runtime handshake:

- **Heartbeats** every 20 s (server TTL is 60 s), up to 10 consecutive auth failures before exiting
- **409 epoch mismatch** → immediate exit (or `onEpochMismatch` callback in `replBridge`)
- **JWT expiry** → immediate exit; non-expired 401/403 → count toward failure threshold
- **`WorkerStateUploader`** absorbs rapid state patches into a single PUT rather than flooding the endpoint

### 6. WorkerStateUploader is a specialized coalescer for a single endpoint

It holds at most **two slots**: one in-flight PUT and one pending payload. Any additional patches arriving while a PUT is in-flight are merged into the pending payload before the next attempt. This means rapid state churn collapses into one request rather than N.

### 7. Dual runtime support in WebSocketTransport

`WebSocketTransport` detects the runtime (`typeof Bun !== 'undefined'`) and uses:
- **Bun**: native `globalThis.WebSocket`; no upgrade response headers → always full replay on reconnect
- **Node.js**: `ws` package; server sends `x-last-request-id` header → targeted replay of unconfirmed messages only

### 8. Backpressure at multiple layers

| Layer | Mechanism | Bound |
|-------|-----------|-------|
| `SerialBatchEventUploader` | `enqueue()` blocks when `queue.length >= maxQueueSize` | 100,000 items |
| `SerialBatchEventUploader` | `maxBatchSize` | 500 items per POST |
| `WebSocketTransport` | `CircularBuffer` | 1,000 buffered messages |
| `WorkerStateUploader` | 2 slots max (1 in-flight + 1 pending) | 2 payloads |
| `HybridTransport` stream buffer | 100 ms delay | N/A — time-based, not size-based |

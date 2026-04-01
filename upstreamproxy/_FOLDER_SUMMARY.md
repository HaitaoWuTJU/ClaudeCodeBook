# Summary of `upstreamproxy/`

## Purpose of `upstreamproxy/`

Provides a **container-side MITM proxy layer** for CCR (Claude Code Remote) sessions. It intercepts outbound HTTPS traffic from all agent tools and subprocesses, transparently injects authentication credentials, and forwards requests through a relay that terminates TLS on the server side—enabling the CCR infrastructure to observe and operate on agent traffic without breaking the TLS model browsers and CLI tools expect.

## Contents Overview

| File | Role |
|------|------|
| `relay.ts` | **Runtime-agnostic WebSocket relay engine.** Listens on localhost, accepts HTTP `CONNECT` tunnels from tools (`curl`, `gh`, `kubectl`), encodes/decodes protobuf `UpstreamProxyChunk` messages, and multiplexes bidirectional bytes over a WebSocket connection to the CCR gateway. Supports both **Bun** and **Node.js** runtimes. |
| `upstreamproxy.ts` | **Container initialization & wiring.** Reads session tokens, configures security (`prctl` to disable ptrace), downloads and merges CA certificates, starts the relay, and exposes proxy environment variables (`HTTPS_PROXY`, `SSL_CERT_FILE`, etc.) to every subprocess the agent spawns. |

## How Files Relate to Each Other

```
upstreamproxy.ts                          relay.ts
─────────────────                         ────────
readToken()                               startUpstreamProxyRelay()
      │                                          │
      ▼                                          │
setNonDumpable() (security)                      │
      │                                          │
      ▼                                          │
downloadCaBundle() ──── writes CA PEM ──────────►│
      │                                          │
      ▼                                          │
startUpstreamProxyRelay() ──────────────────────►│
      │                                          │
      │◄─────────────────────────────────────────│  ← TCP CONNECT from
      │     127.0.0.1:port (LISTEN)              │    curl/gh/kubectl
      ▼                                          ▼
unlink session token                     WebSocket → CCR gateway
      │                                          │
      ▼                                          │
getUpstreamProxyEnv() ← env vars injected into subprocesses
```

**Key relationship**: `upstreamproxy.ts` is the **orchestrator**—it calls `startUpstreamProxyRelay()` from `relay.ts` and then rewrites all subprocess environment variables so every tool (Python `httpx`, `curl`, `npm`, `pip`, Go, Node.js) transparently routes through the relay. `relay.ts` handles only the raw byte tunnel; it knows nothing about tokens, CA bundles, or environment injection.

## Key Takeaways

1. **Fail-open by design.** If CA download fails, if the token is missing, if `prctl` errors—`upstreamproxy.ts` logs a warning and returns `enabled: false`. The session works without proxy interception; a broken proxy never breaks a session.

2. **Two-layer protocol.** Tool traffic is **not** proxied via standard HTTP `CONNECT` to an HTTP proxy daemon. Instead, `relay.ts` implements its own WebSocket transport with a hand-encoded `UpstreamProxyChunk` protobuf (`tag 0x0a` + varint length + raw bytes). The first chunk carries the raw `CONNECT` line and `Proxy-Authorization` header; subsequent chunks are raw encrypted bytes.

3. **Bun/Node duality lives in `relay.ts`.** The TCP server, WebSocket client, and write-backpressure logic differ between runtimes (`Bun.listen` + custom write queue vs. `net.createServer` + `ws` with proxy agent). `upstreamproxy.ts` is intentionally runtime-agnostic—Bun FFI (`bun:ffi`) is used only inside `setNonDumpable()`.

4. **NO_PROXY is a compatibility surface.** The proxy excludes Anthropic API, GitHub, package registries, loopback, and RFC1918 addresses because: the MITM CA is not trusted by Python `certifi`/Go standard library; these hosts are already authenticated; and some runtimes (Python `httpx`, Go `net/http`) cannot be forced through an `HTTPS_PROXY` env var.

5. **Security hardening.** `prctl(PR_SET_DUMPABLE, 0)` is called before any secrets are read, preventing a same-UID attacker from using `ptrace`/`/proc` to read the token or CA key out of the subprocess memory. The session token file is unlinked immediately after the relay starts listening.

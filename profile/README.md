<div align="center">

<img src="https://github.com/user-attachments/assets/ed11f537-b6b1-4d9e-92e4-d48ff3f4258e" width="200" height="200" alt="Rudy" />

# Rudy

**Telegram-compatible infrastructure, built in Elixir and Rust.**

</div>

Rudy provides the protocol, networking, and storage foundations for a Telegram-compatible server. Elixir runs the live system: connections, sessions, API handling, update delivery, and supervision. Rust handles the parts that need tight control over memory, latency, and correctness: storage, cryptography, and media processing.

## Summary

- **Elixir (BEAM)** owns the gateway, MTProto session orchestration, the application API, update routing, presence, and the lifecycle of the whole application.
- **Rust** owns application storage (Corestore with embedded Shardlog), media storage (Objectstore), cryptography, and media processing. These are exposed to the Elixir side through well-defined native interfaces.
- Protocol handling stays separate from application policy, and the public-facing edge does not depend directly on the private application database.

## Why Elixir

A messaging server is mostly a very large number of long-lived, mostly idle connections that occasionally need to do something quickly. That is the workload the BEAM was built for.

- **One process per connection and session.** Lightweight, isolated processes make per-client state simple to model and cheap to run at scale.
- **Fault isolation and supervision.** A crash in one session, handler, or subscription is contained and restarted without affecting others. Failure handling is structural, not ad hoc.
- **Built-in distribution.** Presence, pub/sub, and update fan-out across nodes use the runtime's native clustering instead of a custom transport layer.
- **Soft real-time scheduling.** Preemptive scheduling keeps latency predictable under load; one slow client cannot starve the rest.
- **Live operations.** Introspection, tracing, and hot code upgrades make it practical to inspect and change a running system.
- **Faster iteration on application logic.** API methods, permissions, and update semantics change often; Elixir makes that layer quicker to write, test, and evolve.

## What stays in Rust

| Area | Why Rust |
|---|---|
| **Storage** — Corestore, Shardlog, Objectstore | Sharded, Raft-replicated state on RocksDB and raft-engine, durable deliveries, chunked media storage, and integrity verification need predictable performance, explicit memory control, and strong correctness guarantees. |
| **Cryptography** | MTProto key exchange, encryption, and hashing are hot paths where constant-time behavior, zero-copy buffers, and audited low-level code matter. |
| **Media processing** | Thumbnailing, transcoding, hashing, and other CPU-heavy work belongs in native code, kept off the BEAM schedulers. |

## Components

| Component | Language | Purpose |
|---|---|---|
| Gateway | Elixir | TCP, WebSocket, and WSS edge; admission limits, backpressure, reply routing, and live updates. |
| MTProto | Elixir + Rust | Session state, ACKs, retransmission, and containers in Elixir; cryptographic primitives in Rust. |
| TL | — | Layer-aware schema codecs for Telegram API layers 225–233, with explicit cross-layer conversion. |
| Application API | Elixir | Method dispatch, validation, permissions, layer-correct replies, and difference recovery. |
| Corestore / Shardlog | Rust | Typed application storage over an embedded, sharded, Raft-replicated engine. |
| Objectstore | Rust | Chunked media storage with SeaweedFS and S3-compatible backends, upload sealing, ranged reads, and SHA-256 verification. |
| Media processing | Rust | CPU-intensive media work. |

**Repository access:** The repositories are temporarily private during internal development and restructuring. Public access will be restored as they are ready.

## Principles

- **Right tool per layer:** Elixir for concurrency, distribution, and fault tolerance; Rust for storage, cryptography, and compute.
- **Clear component boundaries:** Protocol handling, networking, application policy, structured storage, and media storage have separate responsibilities.
- **Bounded resources:** Connections, queues, buffers, outstanding calls, and background work have explicit limits.
- **Layer-aware compatibility:** Requests and results use the client's schema layer, with explicit conversion where supported.
- **Durable effects:** Application writes, operation receipts, cross-entity deliveries, and update recovery have explicit persistence contracts.
- **Independent validation:** Components are tested against independent implementations, including grammers, Telethon, and TDLib. Component interoperability does not imply complete Telegram API coverage.

## Status

**Active development. Rudy is not yet a complete Telegram server implementation.**

The move to Elixir for the live system is in progress. The Rust storage, cryptography, and media components continue to be developed and tested independently. Server composition, the application API, production provider adapters, and full end-to-end integration remain under development.

---

An independent project, unaffiliated with Telegram.

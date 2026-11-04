<div align="center">

<img src="https://github.com/user-attachments/assets/ed11f537-b6b1-4d9e-92e4-d48ff3f4258e" width="200" height="200" alt="Rudy" />

# Rudy

**Telegram-compatible infrastructure, built in Rust.**

</div>

Rudy provides the protocol, networking, and storage foundations for building a Telegram-compatible server.

The project is organized into independent components covering TL schema handling, MTProto, network transport, application data, and media storage. Each component has a clearly defined responsibility, allowing the application layer to implement business logic without coupling it to a specific networking or storage implementation.

## Repositories

| Repository | Purpose |
|---|---|
| [tl](https://github.com/rudy-chat/tl) | TL schema compiler and layer-aware codecs, supporting layers 225–233. |
| [mtproto](https://github.com/rudy-chat/mtproto) | Bounded, sans-I/O implementation of server-side MTProto 2.0. |
| [gateway](https://github.com/rudy-chat/gateway) | Linux network gateway with TCP, WebSocket, and TLS support. |
| [corestore](https://github.com/rudy-chat/corestore) | Storage engine for users, identities, messages, and application data. |
| [objectstore](https://github.com/rudy-chat/objectstore) | Storage engine for media, files, and other binary objects. |

**Note:** The repositories are temporarily private while internal development and restructuring are underway. Public access will be restored once they are ready.

## Architecture

```text
          Telegram-compatible clients
                       │
                TCP / WS / TLS
                       │
                ┌──────▼──────┐
                │   Gateway   │
                └──────┬──────┘
                       │
             ┌─────────▼─────────┐
             │ Application Layer │
             │  API & Business   │
             │      Logic        │
             └─────────┬─────────┘
                       │
            ┌──────────┴──────────┐
            │                     │
     ┌──────▼──────┐       ┌──────▼──────┐
     │  CoreStore  │       │ ObjectStore │
     │             │       │             │
     │ Users       │       │ Media       │
     │ Identities  │       │ Files       │
     │ Messages    │       │ Objects     │
     │ Metadata    │       │             │
     └─────────────┘       └─────────────┘


       Shared Protocol Foundations
       ┌────────────┬────────────┐
       │  MTProto   │     TL     │
       └────────────┴────────────┘
```

The gateway handles client connections, transport, and protocol delivery. The application supplies the API handlers and coordinates persistence through CoreStore and ObjectStore.

MTProto and TL are implemented as separate, reusable components rather than being embedded directly into the networking or storage layers.

## Components

### TL

A TL schema compiler and layer-aware codec implementation supporting Telegram API layers 225–233.

Provides schema-based serialization and deserialization while preserving compatibility with the client's negotiated API layer.

### MTProto

A bounded, sans-I/O implementation of server-side MTProto 2.0.

Protocol processing is independent of network I/O, allowing transport implementations to manage connections without embedding cryptographic or protocol state machines directly into their event loops.

### Gateway

A Linux network gateway supporting TCP, WebSocket, and TLS transports.

Responsible for connection lifecycle management, protocol delivery, and resource limits. Application-specific behavior is delegated to externally supplied API handlers.

### CoreStore

The persistent storage component for application data, including users, identities, messages, and associated metadata.

Separates application-level persistence from protocol handling and network transport.

### ObjectStore

The storage component for media, uploaded files, and other binary objects.

Keeps large object storage separate from structured application data, allowing the two storage workloads to be managed independently.

## Principles

- **Clear boundaries:** Protocol, networking, application logic, structured storage, and object storage have separate responsibilities.
- **Bounded resources:** Explicit limits on connections, queues, buffers, and outstanding calls.
- **Layer-aware compatibility:** Decode requests and encode responses using the client's negotiated schema layer.
- **Transport independence:** MTProto processing is separated from the underlying network transport.
- **Storage separation:** Structured application data and binary objects are handled by dedicated components.
- **Independent validation:** Interoperability testing with grammers, Telethon, and TDLib.

## Status

**Active development.**

Rudy is an infrastructure project, not yet a complete Telegram server implementation.

The protocol, gateway, and storage components provide foundations for a Telegram-compatible backend. Full Telegram API coverage, application services, and end-to-end functionality remain under development.

The repositories are temporarily private for a limited period and will be made publicly accessible again after the ongoing internal work is complete.

---

An independent project, unaffiliated with Telegram.

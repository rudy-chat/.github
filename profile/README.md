<img width="200" height="200" alt="image" src="https://github.com/user-attachments/assets/ed11f537-b6b1-4d9e-92e4-d48ff3f4258e" />


# Rudy


Telegram-compatible infrastructure, built in Rust.

Rudy provides the protocol and networking foundations for building a Telegram-compatible server. Schema handling, MTProto, and the network gateway are separate crates, while the application controls business logic and storage.

## Repositories

| Repository | Purpose |
|---|---|
| [tl](https://github.com/rudy-chat/tl) | TL schema compiler and layer-aware codecs, supporting layers 225–233. |
| [mtproto](https://github.com/rudy-chat/mtproto) | Bounded, sans-I/O implementation of server-side MTProto 2.0. |
| [gateway](https://github.com/rudy-chat/gateway) | Linux network gateway with TCP, WebSocket, and TLS support. |

## Architecture
```
Telegram-compatible clients
            │
      TCP / WebSocket / TLS
            │
         Gateway
            │
     Application services
            │
      Storage implementation

   Shared protocol foundations:
          MTProto + TL
```
The application supplies the gateway’s API handler and storage backend. The gateway handles connections and protocol delivery without depending on a particular database.

## Principles

- **Clear boundaries:** protocol, networking, application logic, and storage have separate responsibilities.
- **Bounded resources:** explicit limits on connections, queues, buffers, and outstanding calls.
- **Layer-aware compatibility:** decode requests and encode responses using the client’s negotiated schema layer.
- **Independent validation:** interoperability tests with grammers, Telethon, and TDLib.

## Status

Active development. These crates provide server infrastructure; they do not constitute a complete Telegram API implementation.

---

An independent project, unaffiliated with Telegram.

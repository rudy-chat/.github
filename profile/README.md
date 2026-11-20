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


## Notes
Rudy is an independent project, unaffiliated with Telegram.

<div align="center">

<img src="https://github.com/user-attachments/assets/ed11f537-b6b1-4d9e-92e4-d48ff3f4258e" width="200" height="200" alt="Rudy" />

# Rudy

**Telegram-compatible infrastructure, built in Rust.**

</div>

Rudy provides the protocol, networking, and storage foundations for a Telegram-compatible server: layer-aware TL codecs, server-side MTProto, a bounded network gateway, sharded application storage, and media storage.

The main application composes these components. It supplies API handlers, storage adapters, configuration, and lifecycle management. Protocol handling stays separate from application policy, and the public gateway does not depend on the private application database.

## Repositories

| Repository | Purpose |
|---|---|
| [tl](https://github.com/rudy-chat/tl) | TL schema generation, validation, borrowed codecs, and conversion across layers 225–233. |
| [mtproto](https://github.com/rudy-chat/mtproto) | Bounded, sans-I/O implementation of server-side MTProto 2.0. |
| [gateway](https://github.com/rudy-chat/gateway) | Linux network gateway for TCP, WebSocket, and secure WebSocket connections. |
| [corestore](https://github.com/rudy-chat/corestore) | Typed application storage over an embedded, sharded, replicated storage engine. |
| [objectstore](https://github.com/rudy-chat/objectstore) | Chunked media storage, upload sealing, ranged reads, and integrity verification. |

The application API, server composition, and private adapters connect these components into a server. Corestore embeds **Shardlog**, which supplies shard routing, Raft replication, persistent state, and durable work.

**Repository access:** The repositories are temporarily private during internal development and restructuring. Public access will be restored as they are ready.

## Architecture

These diagrams distinguish existing component contracts from unfinished server integration. **Solid arrows show implemented calls, dependencies, or storage paths. Dashed arrows show integration that remains to be completed.** They describe the code architecture, not a claim that a complete production deployment is running.

### Application, protocol, and media paths

Each application instance owns its connections and local runtime state. Its main binary supplies the API and backend implementations; the gateway never opens or configures Corestore itself.

```mermaid
flowchart LR
    CLIENT["Telegram-compatible clients<br/>Mobile · Desktop · Web · Third-party"]

    subgraph SERVER["Rudy application instances — repeated across nodes"]
        direction LR

        MAIN["Main binary — integration pending<br/>Configuration · Secrets · Certificates<br/>Dependency injection<br/>Startup · Drain · Shutdown"]

        GW["Gateway<br/>Per-core socket ownership<br/>TCP · WebSocket · WSS<br/>Admission limits · Backpressure<br/>Bounded buffers · Queues · Timers<br/>RPC completion · Update routing · Metrics"]

        MT["MTProto library<br/>Framing · Key exchange · Encryption<br/>Sessions · Replay protection<br/>ACKs · Retransmission · Containers<br/>Invocation wrappers and ordering"]

        TL["TL library<br/>Generated schemas: layers 225–233<br/>Borrowed decoding · Validation<br/>Encoding · Layer conversion<br/>Layer 233 uses a pinned preview"]

        API["Application API — in progress<br/>Constructor-ID dispatch<br/>Validation and permissions<br/>Account · Messaging · Media operations<br/>Layer-correct replies and updates<br/>Difference recovery"]

        ADAPTER["Private Gateway Backend adapter<br/>Integration pending<br/>Auth-key lookup and mutations<br/>Key-change watches<br/>User/channel subscriptions"]

        CS["Corestore<br/>Accounts · Keys · Sessions · Identities<br/>Profiles · Privacy · Contacts<br/>Dialogs · Messages · Groups · Channels<br/>Media metadata and permissions<br/>Updates · Settings · Business domains"]

        LOCAL["Node-local state<br/>Auth/profile caches<br/>Presence<br/>Subscription registrations"]

        FEEDS["Corestore update pump<br/>Committed user/channel updates<br/>Auth-key invalidation<br/>Bounded lag and resync notification<br/>Subscriptions to local replicas"]

        ENGINE["Embedded Shardlog node<br/>Group → partition → shard routing<br/>Local execution or leader forwarding<br/>Operation receipts · Committed reads<br/>Durable deliveries · Command timers"]

        PROVIDERS["Injected Corestore provider interfaces<br/>ObjectStore · SearchIndex<br/>SmsSender · PushSender<br/>PaymentGateway"]
    end

    CLIENT <-->|"TCP / WS / WSS"| GW
    GW <-->|"drives state machine; receives events"| MT
    GW -->|"uses codecs"| TL
    MT -->|"uses codecs"| TL
    API -->|"uses codecs"| TL

    MAIN -.->|"configure and start"| GW
    MAIN -.->|"provide Api implementation"| API
    MAIN -.->|"provide Backend implementation"| ADAPTER
    MAIN -.->|"open and own lifecycle"| CS
    MAIN -.->|"supply implementations"| PROVIDERS

    GW -.->|"API calls and update encoding"| API
    API -.->|"complete replies"| GW
    API -.->|"typed domain operations"| CS

    GW -.->|"Backend calls"| ADAPTER
    ADAPTER -.->|"keys and subscriptions"| CS
    FEEDS -.->|"committed changes"| ADAPTER
    ADAPTER -.->|"invalidate keys; deliver updates"| GW

    CS -->|"commands and queries"| ENGINE
    ENGINE -->|"locally applied committed records"| FEEDS
    CS --> LOCAL
    LOCAL -->|"local presence notifications"| FEEDS
    CS --> PROVIDERS

    PEERS["Peer Shardlog nodes<br/>Replicated shard groups<br/>Expanded in the cluster view below"]
    ENGINE <-->|"mTLS: forwarding, replication, deliveries"| PEERS

    OBJADAPTER["Corestore ↔ Objectstore adapter<br/>Integration pending<br/>File identity · Receipts · Hash readiness"]

    OBJROUTER["Media-shard owner routing<br/>Application responsibility<br/>Not built into Objectstore"]

    OBJECTS["Objectstore owners × N<br/>One exclusive namespace per owner<br/>Chunked uploads · Immutable generations<br/>Sealing · Ranged reads<br/>Range and whole-file hashes"]

    JOURNAL[("Local journal per owner<br/>Namespace key · Part pointers<br/>Seals · Locators · Hash progress")]

    MASTER["SeaweedFS master<br/>Volume assignment and lookup"]
    VOLUMES[("SeaweedFS volume servers<br/>Payload bytes<br/>Configured replica placement")]
    S3[("Alternative: S3-compatible service<br/>Payload bytes<br/>Service-configured durability")]

    SEARCH["Search provider implementation"]
    SMS["SMS provider implementation"]
    PUSH["Push provider implementation"]
    PAY["Payment provider implementation"]

    PROVIDERS -.->|"ObjectStore"| OBJADAPTER
    OBJADAPTER -.-> OBJROUTER
    OBJROUTER -.->|"select owning namespace"| OBJECTS

    OBJECTS -->|"persist storage bookkeeping"| JOURNAL
    OBJECTS <-->|"native backend: assign / lookup"| MASTER
    OBJECTS <-->|"native backend: payload I/O"| VOLUMES
    OBJECTS <-->|"alternative backend: signed HTTP"| S3

    PROVIDERS -.->|"SearchIndex"| SEARCH
    PROVIDERS -.->|"SmsSender"| SMS
    PROVIDERS -.->|"PushSender"| PUSH
    PROVIDERS -.->|"PaymentGateway"| PAY

    classDef pending fill:#fff3cd,stroke:#b8860b,color:#302400
    class MAIN,API,ADAPTER,OBJADAPTER,OBJROUTER,SEARCH,SMS,PUSH,PAY pending
```

Corestore invokes media storage through its injected `ObjectStore` provider. The adapter to the standalone Objectstore crate is a separate integration task. Media bytes do not travel through the replicated application-state log; Corestore persists the application metadata, references, and receipts needed to use them.

### Sharding, replication, and cluster traffic

Corestore assigns application state to **entity groups**: users, chats, channels, index buckets, media buckets, allocator groups, and the control group. The routing map places their partitions onto shards. **Each shard is a separate Raft group**, with its own leader and configured voters.

The following is an illustrative **two-shard, three-node configuration**. Leaders are spread across nodes; followers hold replicas of the same shards. These are the storage roles of Corestore/Shardlog nodes, not an additional mandatory service tier. Replica count and placement come from configuration.

```mermaid
flowchart LR
    CALL["Corestore operation<br/>Entity GroupId + operation ID<br/>Command or query"]

    ROUTE["Placement and leader routing<br/>Group → partition → shard<br/>Configured hash-range map<br/>Local execution or forwarding"]

    CONFIG["Configured cluster topology<br/>Node identities and peer endpoints<br/>Shard voter sets<br/>Preferred startup leaders<br/>mTLS certificates"]

    GRANTS["Corestore group provisioning<br/>Signed ownership grants<br/>Realm authority key<br/>Current integration: epoch 1"]

    subgraph N1["Node 1 — Corestore / embedded Shardlog"]
        direction TB
        A1["Shard A replica<br/>Leader in this example"]
        B1["Shard B replica<br/>Follower in this example"]
        D1[("Node 1 local disk<br/>RocksDB shard state<br/>raft-engine logs")]
        F1["Node 1 committed-change pump<br/>Local application subscribers"]
        A1 -->|"persist"| D1
        B1 -->|"persist"| D1
        A1 -->|"applied records"| F1
        B1 -->|"applied records"| F1
    end

    subgraph N2["Node 2 — Corestore / embedded Shardlog"]
        direction TB
        A2["Shard A replica<br/>Follower in this example"]
        B2["Shard B replica<br/>Leader in this example"]
        D2[("Node 2 local disk<br/>RocksDB shard state<br/>raft-engine logs")]
        F2["Node 2 committed-change pump<br/>Local application subscribers"]
        A2 -->|"persist"| D2
        B2 -->|"persist"| D2
        A2 -->|"applied records"| F2
        B2 -->|"applied records"| F2
    end

    subgraph N3["Node 3 — Corestore / embedded Shardlog"]
        direction TB
        A3["Shard A replica<br/>Follower in this example"]
        B3["Shard B replica<br/>Follower in this example"]
        D3[("Node 3 local disk<br/>RocksDB shard state<br/>raft-engine logs")]
        F3["Node 3 committed-change pump<br/>Local application subscribers"]
        A3 -->|"persist"| D3
        B3 -->|"persist"| D3
        A3 -->|"applied records"| F3
        B3 -->|"applied records"| F3
    end

    RESULT["Corestore result<br/>Operation receipt / read result<br/>Commit token where provided"]

    CALL --> ROUTE
    CONFIG -->|"placement and peer configuration"| ROUTE
    CONFIG -->|"voters and identities"| N1
    CONFIG -->|"voters and identities"| N2
    CONFIG -->|"voters and identities"| N3

    GRANTS -->|"provision through shard leader"| A1
    GRANTS -->|"provision through shard leader"| B2

    ROUTE -->|"Shard A operations"| A1
    ROUTE -->|"Shard B operations"| B2

    A1 <-->|"Shard A Raft traffic / mTLS"| A2
    A1 <-->|"Shard A Raft traffic / mTLS"| A3
    B2 <-->|"Shard B Raft traffic / mTLS"| B1
    B2 <-->|"Shard B Raft traffic / mTLS"| B3

    A1 <-->|"ordered, deduplicated cross-shard deliveries"| B2

    A1 -->|"committed write / completed read"| RESULT
    B2 -->|"committed write / completed read"| RESULT
```

Writes require a majority of the shard's voters. In this three-voter example, the majority is two. Raft manages elections and replication; leader positions in the diagram can change. Application results and committed-change notifications follow the relevant commit/application progress.

Cross-entity effects use durable, ordered, deduplicated deliveries. This does not make every multi-entity application operation one atomic distributed transaction.

### Storage and ownership boundaries

| Concern | Owner |
|---|---|
| Sockets, connection admission, send queues, backpressure, and reply routing | Gateway |
| Wire encryption, protocol sessions, replay checks, ACKs, and retransmission | MTProto, driven by Gateway |
| Schema validation, decoding, encoding, and layer conversion | TL |
| RPC permissions, method behavior, response construction, and difference recovery | Application API |
| Durable auth keys, identities, messages, permissions, media references, and update history | Corestore |
| Entity routing, shard replication, persistent state/logs, deliveries, and command timers | Embedded Shardlog |
| Media bytes, immutable parts, upload seals, range reads, and integrity hashes | Objectstore and its selected storage backend |
| Objectstore namespace ownership, owner routing, and journal recovery placement | Embedding application and deployment |
| Node configuration, provider implementations, secrets, and component lifecycle | Main application |

Objectstore sharding is independent of Corestore sharding. Each Objectstore owner exclusively opens one namespace journal; different owners can run on different nodes. The journal contains storage bookkeeping, not the payload bytes. SeaweedFS or the selected S3-compatible service stores those bytes and supplies its configured durability. The native SeaweedFS path uses the master and volume servers directly, without a filer.

## Components

### TL

A schema generator and bounded, layer-aware codec implementation for Telegram API layers **225–233**. Layer 233 uses a pinned preview schema; layers 225–232 have official client schema sources.

TL provides borrowed object views, checked encoding and decoding, typed RPC results, and explicit cross-layer conversion. It owns no connections, cryptographic keys, protocol sessions, or application storage.

### MTProto

A bounded, sans-I/O implementation of server-side **MTProto 2.0**.

It implements transport framing, cryptographic handshakes, encrypted messages, session protocol state, replay checks, acknowledgements, retransmission, and invocation wrappers. Callers supply I/O, time, entropy, storage interfaces, and scheduling. Protocol state is distinct from durable application state.

### Gateway

A Linux network edge supporting **TCP, WebSocket, and WSS**. Each socket stays on its accepting I/O core, while shared session ownership and bounded command queues coordinate work between cores.

Gateway enforces admission and resource limits, drives MTProto, delivers calls to the supplied API, and routes replies and live updates. Its `Backend` interface abstracts durable keys and subscriptions. The private application implements that interface; Gateway has no direct Corestore or Shardlog dependency.

### Corestore and Shardlog

Corestore exposes typed application operations for accounts, authentication, profiles, privacy, contacts, dialogs, messages, groups, channels, media metadata, and other application domains. It also exposes injected interfaces for media storage, search, SMS, push notifications, and payments.

Corestore embeds Shardlog. Entity groups route to shards; each replicated shard has its own Raft group. Shardlog persists application state in RocksDB and replication logs in raft-engine. Corestore uses operation receipts for retries, committed read tokens, durable cross-entity deliveries, and command timers.

Standalone mode runs local shards without machine redundancy. Cluster mode configures replica placement, peer identities, mTLS connections, and leader forwarding.

### Objectstore

An async media-storage component with **native SeaweedFS and S3-compatible backends**.

It supports chunked uploads, immutable part generations, durable upload sealing, ranged downloads, SHA-256 range verification, and background whole-file hashing. Each owner keeps an exclusive local journal of namespace identity, locators, part pointers, seals, and hash progress.

Corestore owns user-visible media metadata, access rights, message references, and stored receipts. Objectstore owns byte storage and the bookkeeping required to recover and serve those bytes.

## Principles

- **Explicit ownership:** The main application owns configuration, dependency injection, provider implementations, and lifecycle.
- **Clear component boundaries:** Protocol handling, networking, application policy, structured storage, and media storage have separate responsibilities.
- **Bounded resources:** Connections, queues, buffers, outstanding calls, storage admission, and background work have explicit limits.
- **Layer-aware compatibility:** Requests and results use the client's schema layer, with explicit conversion where supported.
- **Sharding with replication:** Routing distributes entity groups across shards; each shard's voter set provides its replication and majority requirement.
- **Durable effects:** Application writes, operation receipts, cross-entity deliveries, and update recovery have explicit persistence contracts.
- **Independent validation:** Component tests use independent implementations, including grammers, Telethon, and TDLib. Component interoperability does not imply complete Telegram API coverage.

## Status

**Active development. Rudy is not yet a complete Telegram server implementation.**

The protocol, gateway, and storage components are developed and tested independently. The application API, production provider adapters, server composition, and full end-to-end integration remain in progress.

The architecture diagrams intentionally preserve these current limitations:

- **Server wiring:** The main binary, application API handlers, private Gateway backend adapter, and Corestore-to-Objectstore adapter are not presented as completed integration.
- **Live state:** Gateway connections and live protocol sessions are in memory. Corestore presence is currently node-local. Its current committed-change subscriptions require a locally hosted replica; a general remote per-node feed is not wired into this integration.
- **Cluster management:** Corestore currently uses its pinned Shardlog integration, configured placement, and signed group provisioning. Newer meta-shard, balancer, and per-node feed integration are not shown as deployed features.
- **Media ownership:** Objectstore does not provide an automatic cross-node owner router or active-active access to a shared namespace journal. Payload replication does not replace preservation of that journal and its namespace key.
- **Operations:** Storage-engine backup and state-transfer capabilities are not a claim that production backup scheduling, restore orchestration, or automated rebalancing has been configured.

Full Telegram API coverage, application services, deployment configuration, and operational qualification remain under development. Public repository access will be restored after the ongoing internal work is ready.

---

An independent project, unaffiliated with Telegram.

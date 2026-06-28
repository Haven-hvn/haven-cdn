# Building a Web3-Native IPFS CDN and Indexer: Technical Proposal

## Executive Summary

This proposal outlines the architecture for a decentralized, Web3-native content indexing and peer-to-peer distribution system built on IPFS. The system consists of two tightly integrated layers: a custom server-side indexer and caching seedbox built on Boxo, and an embeddable browser-side P2P CDN built on Helia and Waku. 

The design explicitly maximizes IPFS network effects by using standard UnixFS data structures, allowing both public IPFS nodes and browser-based peers to participate in encrypted block routing. Access control is enforced entirely at the edge via Haven AOL and Web3 wallets, rather than at the network routing layer, creating a highly performant swarm that remains permissionless to route but cryptographically gated to consume.

## Problem Statement

Decentralized storage systems face a fundamental gap: proving that a node is actually serving content when requested is computationally and economically impractical. Filecoin proves storage, not transfer. IPFS has no built-in availability attestation. Cryptographic approaches (TEE attestation, payment channels) either don't solve the actual problem or impose overhead that exceeds the value of the bandwidth being proven.

Furthermore, browser-based IPFS clients are second-class citizens because they cannot accept inbound connections, effectively barring them from participating in the public DHT and fragmenting the network.

The practical question is not "can you cryptographically prove you served this content?" but "is the content available when someone requests it, and can browsers swarm it efficiently?" This architecture answers that by combining a demand-driven caching seedbox (availability is the proof) with a wallet-native browser CDN that bridges the signaling gap.

## Architectural Reference: bitmagnet and WebTorrent

The architecture draws from two proven P2P templates:

1. **bitmagnet (Indexing & Metadata):** Provides the template for the server-side indexer. It crawls content identifiers, enriches them with metadata, stores them in PostgreSQL, and exposes them via GraphQL. We adapt this to read from an on-chain registry rather than a DHT.
2. **WebTorrent (Browser CDN):** Proves that browser-to-browser file swarming works using WebRTC. We evolve this by replacing centralized WebSocket trackers with Waku (a decentralized, economically secured pub/sub network) and binding peer identity to Web3 wallets.

## Proposed Architecture: Hybrid IPFS CDN and Indexer

The system consists of two symbiotic components: the **Boxo Indexer/Seedbox** (server-side) and the **Helia/Waku CDN Package** (browser-side).

### Component A: The Boxo Indexer and Seedbox (Server-Side)

Built using `github.com/ipfs/boxo` rather than a full Kubo daemon, this is a custom, lightweight IPFS node designed for indexing, metadata storage, and demand-driven caching.

**1. Content Discovery Layer**
Subscribes to an on-chain registry (Arkiv L3). Each new CID posted represents encrypted content to be indexed. The discovery layer feeds new CIDs into the indexing pipeline but does not immediately pin the content.

**2. Demand-Driven Caching (Prewarming)**
The Boxo server acts as a seedbox. It uses the Synapse SDK to fetch encrypted blocks from Filecoin Onchain Cloud (FOC) on-demand or via operator-configured prewarming policies. Blocks are cached locally in a BadgerDB datastore. It manages storage allocation and LRU eviction policies. Crucially, it serves these blocks to the public IPFS network via standard Bitswap, acting as the "first seed" for the browser CDN.

**3. Metadata Store**
Reads associated metadata from the registry (title, description, content type, provenance) and persists it to PostgreSQL. The database holds the index, never the file content.

**4. Query Layer**
A GraphQL endpoint exposes the local index for programmatic access. A web UI provides human-friendly search. Both resolve CIDs to their cached status on the local node.

### Component B: The Helia/Waku CDN Package (Browser-Side)

An embeddable JavaScript/TypeScript package for disparate applications. It turns every user's browser into an IPFS node that fetches and shares encrypted blocks peer-to-peer.

**1. Web3 Wallet Identity**
The user's EVM wallet (via MetaMask, WalletConnect, etc.) acts as their network identity. The wallet address is mapped to their Helia `PeerId`.

**2. Waku Signaling Layer**
Because browsers cannot participate in the IPFS DHT (they cannot accept inbound connections), the package uses Waku for out-of-band discovery and WebRTC signaling.
- Peers subscribe to a Waku content topic based on the application's token contract address.
- Peers broadcast presence, multiaddrs, and active CIDs to this topic.
- WebRTC SDP offers and answers are exchanged as Waku relay messages on the public content topic.
- Waku's RLN (Rate Limiting Nullifiers) provides economic security, preventing signaling spam without requiring centralized servers.

**3. Helia Data Plane**
Once WebRTC handshakes are complete, Helia establishes direct browser-to-browser data channels. Helia uses the IPFS Bitswap protocol over these channels to request and serve encrypted UnixFS blocks. If no browser peers have the block, Helia falls back to fetching directly from the Boxo Seedbox via WebTransport.

**4. Edge Decryption (Haven AOL)**
The CDN only routes encrypted blocks. When the application needs to decrypt, the package uses the Web3 wallet to generate a SIWE (Sign-In with Ethereum) signature. Haven AOL verifies if the wallet meets the on-chain access conditions (e.g., ERC-20 balance, NFT membership) and releases the decryption key locally.

### Data Flow

```
[Arkiv Registry] → [Boxo Indexer] → [PostgreSQL Metadata]
                                        ↓ (Demand / Prewarm)
[Filecoin Onchain Cloud] → (Synapse SDK) → [Boxo BadgerDB Cache]
                                        ↓ (First Seed)
[Waku Signaling Layer] ←→ [Web3 Wallets] ←→ [Helia WebRTC Browser Swarm]
                                        ↓ (Encrypted Blocks)
[Application] ← [Haven AOL Decryption] ← (Local Key Release)
```

### Trust and Access Model

**Availability** is hybrid. FOC + Arkiv acts as the verifiable warm layer at the top of the funnel. The Boxo server acts as a seedbox, caching content on demand. Browser peers provide CDN scaling by swarming among themselves.

**Discovery** is public and decentralized. Anyone can read the on-chain registry. Anyone can query Waku topics to find online peers.

**Content Integrity** is mathematical. IPFS Merkle DAGs ensure that if a peer claims to have a CID but sends invalid data, Helia detects the hash mismatch instantly and drops the peer via Bitswap penalties. No economic slashing is required for content integrity.

**Content Access** is gated by encryption and Web3 economics. The network is open for routing encrypted blocks, but decryption keys are only released by Haven AOL to wallets satisfying on-chain token conditions.

### Storage Hierarchy

The system uses a three-tier thermal hierarchy to match storage economics to demand:
- **Warm Layer (FOC + Arkiv):** Top of the funnel. All content exists on Filecoin Onchain Cloud, verified by Proof of Data Possession (PDP). Retrieved on-demand via Filecoin Beam.
- **Hot Layer (Boxo Seedbox):** Caches encrypted blocks in BadgerDB based on demand or prewarming. Evicts content via LRU policies when demand drops.
- **Ephemeral Layer (Browser Swarm):** Free residential ISP bandwidth. Uses a sliding window pinning strategy based on user retention and storage capacity.
- **Cold Layer (Filecoin Standard):** Content with proven, long-term demand is promoted to Filecoin Pin for permanent persistence.

## What This Architecture Avoids

- **Bandwidth proof systems & TEE attestation.** Availability is demonstrated by nodes being online and responsive. Content integrity is verified by local hashing.
- **Centralized WebRTC Trackers.** Replaced by Waku's decentralized, economically secured pub/sub network.
- **Network-layer token-gating.** Routing is open to maximize IPFS network effects and CDN performance. Access control is pushed entirely to the edge (Haven AOL).
- **Protocol-level reputation systems.** Bitswap's built-in ledger handles malicious peers locally. Waku's RLN handles network spam cryptologically.
- **Erasure coding.** Standard UnixFS full-block replication is used instead. The Warm and Hot layers guarantee availability; browsers are ephemeral caches, not persistent storage arrays.

## Federation Path

The initial deployment is personal: one operator running the Boxo Indexer, applications embedding the Helia/Waku package. The architecture supports federation natively:
- The Boxo Indexer's GraphQL endpoints can be shared between trusted operators.
- The Helia/Waku browser swarm is inherently federated; any application embedding the package joins the global pool of browser peers.
- IPFS nodes can be configured to prefer fetching from trusted peers (server or browser) via multiaddr prioritization.

## Implementation Priorities

**Phase 1: Core Boxo Indexer**
- Arkiv registry event listener
- PostgreSQL metadata store and basic GraphQL endpoint

**Phase 2: FOC Integration & Seedbox**
- Synapse SDK integration for FOC retrieval
- Custom Boxo block fetching and BadgerDB caching
- Prewarming and LRU eviction policies

**Phase 3: Browser CDN Package (Helia + Waku)**
- Helia initialization with custom libp2p (WebRTC, no public DHT)
- Waku integration for presence broadcasting and SDP signaling
- Bitswap block fetching over established WebRTC channels

**Phase 4: Web3 Access Control**
- Haven AOL integration for decryption key gating
- SIWE wallet authentication flow
- Content encryption pipeline (encrypt before IPFS storage)

**Phase 5: Storage Tiering & Ecosystem**
- Filecoin Pin integration for cold storage promotion
- Standard IPFS Pinning Services API implementation on Boxo
- Web UI for search and browsing

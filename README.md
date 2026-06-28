# Building a Web3-Native IPFS CDN and Indexer: Technical Proposal

## Executive Summary

This proposal outlines the architecture for a decentralized, Web3-native content indexing and peer-to-peer distribution system built on IPFS. The system consists of two tightly integrated layers: a custom server-side indexer/pinning service built on Boxo (for guaranteed availability and cold storage), and an embeddable browser-side P2P CDN built on Helia and Waku (for client-side, token-gated video streaming). 

The design explicitly maximizes IPFS network effects by using standard UnixFS data structures, allowing both public IPFS nodes and browser-based peers to participate in encrypted block routing. Access control is enforced entirely at the edge via Haven AOL and Web3 wallets, rather than at the network routing layer, creating a highly performant swarm that remains permissionless to route but cryptographically gated to consume.

## Problem Statement

Decentralized storage systems face a fundamental gap: proving that a node is actually serving content when requested is computationally and economically impractical. Filecoin proves storage, not transfer. IPFS has no built-in availability attestation. Cryptographic approaches (TEE attestation, payment channels) either don't solve the actual problem or impose overhead that exceeds the value of the bandwidth being proven.

Furthermore, browser-based IPFS clients are second-class citizens because they cannot accept inbound connections, effectively barring them from participating in the public DHT and fragmenting the network. 

The practical question is not "can you cryptographically prove you served this content?" but "is the content available when someone requests it, and can browsers swarm it efficiently?" This architecture answers that by combining a personal pinning service (availability is the proof) with a wallet-native browser CDN that bridges the signaling gap.

## Architectural Reference: bitmagnet and WebTorrent

The architecture draws from two proven P2P templates:

1. **bitmagnet (Indexing & Metadata):** Provides the template for the server-side indexer. It crawls content identifiers, enriches them with metadata, stores them in PostgreSQL, and exposes them via GraphQL. We adapt this to read from an on-chain registry rather than a DHT.
2. **WebTorrent (Browser CDN):** Proves that browser-to-browser file swarming works using WebRTC. We evolve this by replacing centralized WebSocket trackers with Waku (a decentralized, economically secured pub/sub network) and binding peer identity to Web3 wallets.

## Proposed Architecture: Hybrid IPFS CDN and Indexer

The system consists of two symbiotic components: the **Boxo Indexer/Seeder** (server-side) and the **Helia/Waku CDN Package** (browser-side).

### Component A: The Boxo Indexer and Pinning Service (Server-Side)

Built using `github.com/ipfs/boxo` rather than a full Kubo daemon, this is a custom, lightweight IPFS node designed for indexing, metadata storage, and guaranteed availability.

**1. Content Discovery Layer**
Subscribes to an on-chain registry (Arkiv L3). Each new CID posted represents encrypted content to be indexed and pinned. 

**2. Custom Boxo Pinning Service**
When a new CID is discovered, the service uses Boxo's `blockservice` to fetch the blocks from the IPFS network and stores them locally in a BadgerDB datastore. It manages storage allocation, eviction policies, and Filecoin archival tiering. Crucially, it serves these blocks to the public IPFS network via standard Bitswap, acting as the "first seed" for the browser CDN.

**3. Metadata Store**
Reads associated metadata from the registry (title, description, content type, provenance) and persists it to PostgreSQL. The database holds the index, never the file content.

**4. Query Layer**
A GraphQL endpoint exposes the local index for programmatic access. A web UI provides human-friendly search. Both resolve CIDs to their pinned status.

### Component B: The Helia/Waku CDN Package (Browser-Side)

An embeddable JavaScript/TypeScript package for disparate media applications. It turns every user's browser into an IPFS node that fetches and shares encrypted video blocks peer-to-peer.

**1. Web3 Wallet Identity**
The user's EVM wallet (via MetaMask, WalletConnect, etc.) acts as their network identity. The wallet address is mapped to their Helia `PeerId`.

**2. Waku Signaling Layer**
Because browsers cannot participate in the IPFS DHT (they cannot accept inbound connections), the package uses Waku for out-of-band discovery and WebRTC signaling. 
- Peers subscribe to a Waku content topic based on the media application's token contract address.
- Peers broadcast presence and multiaddrs to this topic.
- WebRTC SDP offers and answers are exchanged via Waku direct messages.
- Waku's RLN (Rate Limiting Nullifiers) provides economic security, preventing signaling spam without requiring centralized servers.

**3. Helia Data Plane**
Once WebRTC handshakes are complete, Helia establishes direct browser-to-browser data channels. Helia uses the IPFS Bitswap protocol over these channels to request and serve encrypted UnixFS blocks. If no browser peers have the block, Helia falls back to fetching directly from the Boxo Indexer/Seeder via a public gateway or WebTransport.

**4. Edge Decryption (Lit Protocol)**
The CDN only routes encrypted blocks. When the video player needs to decrypt, the package uses the Web3 wallet to generate a SIWE (Sign-In with Ethereum) signature. Lit Protocol verifies if the wallet meets the on-chain access conditions (e.g., ERC-20 balance, NFT membership) and releases the decryption key locally.

### Data Flow

```
[Arkiv Registry] → [Boxo Indexer] → [PostgreSQL Metadata] + [BadgerDB Blockstore]
                                                              ↓ (First Seed)
[Waku Signaling Layer] ←→ [Web3 Wallets] ←→ [Helia WebRTC Browser Swarm]
                                                              ↓ (Encrypted Blocks)
[Video Player] ← [Lit Protocol Decryption] ← (Local Key Release)
```

### Trust and Access Model

**Availability** is hybrid. The Boxo server guarantees content always exists as a first seed. Browser peers provide CDN scaling by swarming among themselves.

**Discovery** is public and decentralized. Anyone can read the on-chain registry. Anyone can query Waku topics to find online peers. 

**Content Integrity** is mathematical. IPFS Merkle DAGs ensure that if a peer claims to have a CID but sends invalid data, Helia detects the hash mismatch instantly and drops the peer via Bitswap penalties. No economic slashing is required for content integrity.

**Content Access** is gated by encryption and Web3 economics. The network is open for routing encrypted blocks, but decryption keys are only released by Lit Protocol to wallets satisfying on-chain token conditions. 

### Storage Backstop

For content that should persist beyond the operator's local server capacity, Filecoin Onchain Cloud provides a long-term storage backstop. The Boxo pinning service implements tiered policies: pin hot content locally in BadgerDB, archive cold content to Filecoin, and retrieve from Filecoin on demand.

## What This Architecture Avoids

- **Bandwidth proof systems & TEE attestation.** Availability is demonstrated by nodes being online and responsive. Content integrity is verified by local hashing.
- **Centralized WebRTC Trackers.** Replaced by Waku's decentralized, economically secured pub/sub network.
- **Network-layer token-gating.** Routing is open to maximize IPFS network effects and CDN performance. Access control is pushed entirely to the edge (Lit Protocol).
- **Protocol-level reputation systems.** Bitswap's built-in ledger handles malicious peers locally. Waku's RLN handles network spam cryptographically.

## Federation Path

The initial deployment is personal: one operator running the Boxo Indexer, media applications embedding the Helia/Waku package. The architecture supports federation natively:
- The Boxo Indexer's GraphQL endpoints can be shared between trusted operators.
- The Helia/Waku browser swarm is inherently federated; any media app embedding the package joins the global pool of browser peers.
- IPFS nodes can be configured to prefer fetching from trusted peers (server or browser) via multiaddr prioritization.

## Implementation Priorities

**Phase 1: Core Boxo Indexer**
- Arkiv registry event listener
- Custom Boxo block fetching and BadgerDB storage
- PostgreSQL metadata store and basic GraphQL endpoint

**Phase 2: Browser CDN Package (Helia + Waku)**
- Helia initialization with custom libp2p (WebRTC, no public DHT)
- Waku integration for presence broadcasting and SDP signaling
- Bitswap block fetching over established WebRTC channels

**Phase 3: Web3 Access Control**
- Lit Protocol integration for decryption key gating
- SIWE wallet authentication flow
- Content encryption pipeline (encrypt before IPFS storage)

**Phase 4: Storage Tiering & Network Bridging**
- Filecoin Onchain Cloud integration for archival
- Boxo server WebTransport endpoint for browser fallback
- Tiered pinning policies (hot/cold)

**Phase 5: Ecosystem Integration**
- Standard IPFS Pinning Services API implementation on Boxo
- Web UI for search and browsing
- Cross-index search and trusted-peer routing configuration
The practical question is not "can you cryptographically prove you served this content?" but "is the content available when someone requests it?" A self-hosted pinning service answers this directly: the operator controls the node, pins the CIDs, and knows the content is present because they put it there. Availability is the proof.

## Architectural Reference: bitmagnet

bitmagnet provides a proven architectural template that maps cleanly to IPFS-based systems. Its core functions:

1. **Content discovery via DHT crawling.** No central tracker. The system listens to peer announcements and indexes discovered content. This is the BitTorrent equivalent of reading content identifiers from an on-chain registry.

2. **Metadata classification and enrichment.** Raw hashes are insufficient. The system attempts to identify what each content item actually is (media type, category, external database links) and stores enriched metadata.

3. **Search and API access.** A GraphQL endpoint enables programmatic queries. A web UI enables human search. Both operate against the local index.

4. **Ecosystem integration.** The system exposes standard APIs that downstream tools (Servarr stack, automation pipelines) can consume as if it were a content source.

## Proposed Architecture: IPFS Indexer and Pinning Service

### Component Overview

The system consists of five primary components, each mapped from the bitmagnet architecture:

**1. Content Discovery Layer**

Instead of crawling a DHT, the indexer reads content identifiers (CIDs) from an on-chain registry (Arkiv L3). Each new CID posted to the registry represents a content item that should be indexed and pinned. The discovery layer subscribes to registry events and feeds new CIDs into the indexing pipeline.

**2. Pinning Service**

When the discovery layer identifies a new CID, it instructs the local IPFS node to pin it. Pinned content remains available from the operator's node until explicitly unpinned. The pinning service manages storage allocation, eviction policies, and pin lifecycle.

**3. Metadata Store**

The indexer reads associated metadata from the registry (title, description, content type, provenance) and persists it to a local database. Optional enrichment pipelines can classify content further, link to external databases, or derive additional metadata.

**4. Query Layer**

A GraphQL endpoint exposes the local index for programmatic access. A web UI provides human-friendly search and browsing. Both operate against the metadata store and can resolve CIDs to their pinned status on the local node.

**5. Integration Layer**

The system exposes standard APIs that downstream tools can consume. While the IPFS ecosystem lacks a direct equivalent to the Servarr stack, a well-documented API enables integration with automation tools, content managers, and other IPFS-aware applications.

### Data Flow

```
Registry (Arkiv) → Discovery Layer → [Metadata Store, Pinning Service]
                                         ↓
                                    Query Layer (GraphQL + Web UI)
                                         ↓
                                    Integration Layer (API consumers)
                                         ↓
                                    IPFS Node (content retrieval)
```

### Trust and Access Model

The architecture separates three concerns that are often conflated:

**Availability** is personal. The operator controls the pinning node and is responsible for keeping it running. Trust in availability is trust in the operator.

**Discovery** is public. Anyone can read the on-chain registry and see what CIDs exist. The index is a public good that anyone can replicate.

**Content access** is gated by encryption. Content is encrypted before being stored on IPFS or Filecoin. The CID on-chain points to encrypted data. Decryption is gated by Haven AOL access conditions (token holdings, community membership, or whatever criteria the operator defines).

This separation means the indexer can build a public catalog without being able to read the content itself. Users query the indexer, fetch encrypted content from the IPFS node or Filecoin, and decrypt locally using credentials that satisfy the Haven AOL access condition.

### Storage Backstop

For content that should persist beyond the operator's local storage capacity, Filecoin Onchain Cloud provides a long-term storage backstop. The pinning service can implement tiered policies: pin hot content locally, archive cold content to Filecoin, and retrieve from Filecoin on demand.

## What This Architecture Avoids

The design explicitly omits several approaches that have been explored and found impractical:

- **Bandwidth proof systems.** No attempt to cryptographically attest that content was served. Availability is demonstrated by the node being online and responsive.
- **TEE attestation infrastructure.** No trusted execution environments. The operator trusts their own node.
- **Protocol-level reputation systems.** No on-chain reputation, staking, or slashing. The overhead of these systems exceeds their value for personal-scale deployments.
- **Payment channels for transfer attribution.** No micropayment infrastructure for content retrieval.

The bitmagnet author's observation on Tribler applies here: protocol-level trust and reputation systems carry too much overhead to be compelling alternatives to simpler models. Personal trust plus encryption is cheaper and works.

## Federation Path

The initial deployment is personal: one operator, one node, one index. The architecture supports federation with trusted peers without requiring protocol changes:

- Peers can share CID lists (the registry is already public).
- Peers can query each other's GraphQL endpoints to discover what content is available from whom.
- Peers can configure their IPFS nodes to prefer fetching from trusted peers.
- Trust is managed out-of-band (the operator decides which peers to trust).

This is not a decentralized reputation system. It is a federation of personal pinning services where trust is established socially, not cryptographically.

## Implementation Priorities

**Phase 1: Core indexing and pinning**
- Registry event listener (Arkiv subscription or polling)
- IPFS pinning integration
- Metadata store (PostgreSQL or similar)
- Basic GraphQL endpoint

**Phase 2: Search and UI**
- Web UI for search and browsing
- GraphQL schema refinement
- Pin lifecycle management (eviction policies, storage quotas)

**Phase 3: Access control and encryption**
- Haven AOL integration for decryption gating
- Content encryption pipeline (encrypt before IPFS storage)
- Client-side decryption flow

**Phase 4: Storage tiering**
- Filecoin Onchain Cloud integration for archival
- Tiered pinning policies (hot/cold)
- Retrieval-from-Filecoin fallback

**Phase 5: Federation**
- Peer discovery and GraphQL endpoint sharing
- Trusted-peer IPFS routing configuration
- Cross-index search

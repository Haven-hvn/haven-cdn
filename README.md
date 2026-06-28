# Building a Self-Hosted IPFS Indexer and Pinning Service: Technical Proposal

## Executive Summary

This proposal outlines the architecture for a self-hosted content indexing and availability system built on IPFS. The design draws inspiration from bitmagnet's BitTorrent indexer architecture, adapted for IPFS-based content distribution. The system addresses the core challenge of content availability without relying on bandwidth attestation, TEE infrastructure, or protocol-level reputation systems. The result is a pragmatic, personal-scale tool that can later federate with trusted peers.

## Problem Statement

Decentralized storage systems face a fundamental gap: proving that a node is actually serving content when requested is computationally and economically impractical. Filecoin proves storage, not transfer. IPFS has no built-in availability attestation. Cryptographic approaches (TEE attestation, payment channels) either don't solve the actual problem or impose overhead that exceeds the value of the bandwidth being proven.

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

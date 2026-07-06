# Haven

A modular, decentralized, community-scoped content delivery network. Haven combines IPFS content addressing, Arkiv blockchain-indexed metadata, Waku scoped pub/sub, and Haven-AOL cryptographic attestations to create token-gated, community-owned media swarms without centralized trackers or global DHTs.

---

## Design Principles

1. **Modularity by default.** Every actor is a standalone process with a well-defined interface.
2. **On-chain stores only what is static.** CIDs, metadata, and Lit/AOL access conditions live on-chain. PeerIDs, network state, and ephemeral data stay off-chain.
3. **No actor knows more than it needs to.** The CDN doesn't decrypt. Waku doesn't know who is authorized. The canister doesn't know who is hosting. Separation of concerns is enforced at the process boundary.
4. **The contract is the single source of truth for caching.** If it's on-chain, it's cached. If it's not on-chain, it's not.
5. **Encryption at the storage layer.** Content is encrypted before hitting IPFS. CDN nodes and public gateways cache encrypted blobs without access control at the storage layer.
6. **Routing is scoped, not global.** Discovery happens on Waku content topics scoped to communities, not on a global DHT. Only community members can find providers.
7. **Membership is proven cryptographically.** Holding proofs signed by the Haven-AOL canister prove a peer is part of the community. Unverified peers are ignored at the client level.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         ON-CHAIN LAYER                          │
│                                                                 │
│   ┌─────────────────────────┐    ┌──────────────────────────┐   │
│   │  Arkiv Content Registry │    │  Haven-AOL Canister (ICP)│   │
│   │  - CID + metadata       │    │  - Balance-checked gates │   │
│   │  - Lit/AOL conditions   │    │  - VetKD decryption keys │   │
│   │  - encrypted_cid        │    │  - Holding attestations  │   │
│   │  - expires_in           │    │  (t-Schnorr / Ed25519)   │   │
│   └─────────────────────────┘    └──────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ (entity create/update events)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        OFF-CHAIN ACTORS                         │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐                  │
│  │  CDN     │  │  Browser │  │   Client     │                  │
│  │  Nodes   │  │  Swarm   │  │  (Helia +    │                  │
│  │  (Boxo)  │  │  (Helia) │  │  Lit/AOL)    │                  │
│  └──────────┘  └──────────┘  └──────────────┘                  │
│       │             │               │                          │
│       │             │               │                          │
│       └─────────────┴───────────────┘                          │
│                     │                                           │
│                     ▼                                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Waku Content Topics (Scoped)                  │   │
│  │  - Presence + holding proofs + WebRTC SDP signaling        │   │
│  │  - Topic = community scope (e.g., per token contract)      │   │
│  │  - Unverified messages are ignored by clients              │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Actors

### 1. Arkiv Content Registry (On-Chain)

**Role:** The static source of truth for what content exists, how to decrypt it, and how long it should be cached.

**Stores in attributes (public, queryable):**
- `title`, `creator_handle`, `source_uri` — metadata for indexing
- `phash` — perceptual hash for content matching
- `mint_id` — NFT tracking
- `cid_hash` — SHA-256 of root CID for deduplication
- `is_encrypted` — int flag (0 or 1)
- `encrypted_cid` — the encrypted CID (safe for public indexing)
- `gate_chain`, `gate_token`, `gate_threshold` — Lit/AOL access control conditions
- `project` — always "haven"
- `type` — always "video"
- `created_at_ts` — numeric timestamp for range queries
- `created_at`, `updated_at` — ISO timestamps

**Stores in payload:**
- `filecoin_root_cid` — the actual root CID
- `piece_cid` — Filecoin piece CID (bafkzcib…) required for Synapse download
- `cid_encryption_metadata` — Lit/AOL gate metadata for decrypting the encrypted_cid
- `encryption_metadata` — Lit/AOL gate metadata for decrypting the content
- `content_mime_type`, `content_file_size` — original file info
- `original_hash` — hash of plaintext for verification
- `vlm_json_cid` — VLM analysis JSON CID
- `segment_metadata` — multi-segment recording info
- `attestation` — canister-signed holding proof (single-CID or Merkle batch)

**Does NOT store:**
- PeerIDs (ephemeral, high churn)
- Provider lists (handled by Waku content topics)
- Transfer metrics (not needed for caching decisions)

**Expiration:** Entities have an `expires_in` field (default 4 weeks). The CDN uses this to know when to evict content. Creators renew by calling `update_entity` with a new expiration.

---

### 2. Haven-AOL Canister (ICP)

**Role:** Cryptographic access management. Verifies wallet ownership and on-chain token balances, then issues decryption keys (via VetKD) and holding attestations (via t-Schnorr/Ed25519).

**Two functions in this architecture:**

1. **Decryption keys (VetKD):** When a client wants to decrypt content, it signs an EIP-712 `GateRequest` or `GateRequestV3`. The canister verifies the signature, checks the wallet's on-chain token balance via EVM RPC, and derives a VetKD key. The client decrypts locally.
   - v1: Per-CID derivation (unique key per file)
   - v3: Corpus + epoch derivation (one key unlocks all content in a 30-day epoch for a given token policy)

2. **Holding attestations (t-Schnorr/Ed25519):** When a client wants to prove community membership on Waku, it signs an EIP-712 `AttestRequest`. The canister verifies the signature, checks the token balance, and signs a canonical attestation. The client broadcasts this attestation on the Waku content topic. Other peers verify the attestation signature offline using the canister's public key.

**Attestation payload:** `evmAddress`, `chain`, `tokenAddress`, `threshold`, `balanceAtCheck`, `cidHash`, `timestamp`. Canonical signing preimage: `HAVEN_ATTEST_V1:{chain}:...`

**Why this matters for Waku:** The attestation is a portable, offline-verifiable proof that a wallet holds the required token. Peers don't need to query the canister or the blockchain to verify membership. They just check the t-Schnorr signature against the canister's public key. This makes community scoping cryptographically enforced without any central server.

---

### 3. CDN Nodes (Hot Layer)

**Role:** The "first seed" and on-chain-driven cache. Built on Boxo. Caches exactly what the Arkiv contract says to cache.

**How it works:**
1. Sidecar subscribes to Arkiv entity creation and update events
2. Pins encrypted CIDs for entities with `type = "video"` and `project = "haven"`
3. Uses Synapse SDK to fetch from Filecoin Onchain Cloud on-demand or via prewarming
4. Caches encrypted blocks locally in BadgerDB with LRU eviction
5. Serves encrypted blocks via standard Bitswap
6. **Announces presence on the community's Waku content topic** with a holding attestation
7. Participates in WebRTC signaling on the same topic if serving to browsers

**What it doesn't do:**
- Decrypt content
- Enforce access control at the network layer
- Make caching decisions based on demand
- Pin anything not registered on-chain
- Announce to any global DHT or public router

---

### 4. Browser CDN (Ephemeral Layer)

**Role:** Turn every viewer's browser into an IPFS node that fetches and shares encrypted blocks peer-to-peer within the community.

**How it works:**
1. **Web3 Wallet Identity:** The user's EVM wallet acts as their network identity. The wallet address is mapped to their Helia `PeerId`.
2. **Community Authentication:** Before joining the Waku topic, the browser client requests a holding attestation from the Haven-AOL canister:
   - User signs EIP-712 `AttestRequest` with their wallet
   - Canister verifies balance and signs attestation
   - Client caches the attestation locally
3. **Waku Content Topic:** The browser subscribes to the community's Waku content topic. The topic is scoped to the community (e.g., derived from the token contract address).
4. **Presence Broadcasting:** The browser broadcasts its presence on the topic, including:
   - Its PeerId and supported multiaddrs
   - The CIDs it currently has cached
   - Its holding attestation (proving community membership)
5. **Peer Verification:** When the browser receives presence messages from other peers, it verifies their holding attestations offline using the canister's public key. Unverified peers are ignored.
6. **WebRTC Signaling:** The browser uses the same Waku topic to exchange SDP offers/answers with other browsers for NAT traversal.
7. **Helia Data Plane:** Once WebRTC handshakes are complete, Helia establishes direct browser-to-browser data channels. Bitswap runs over these channels to request and serve encrypted UnixFS blocks. Fallback to CDN via WebTransport if no browser peers have the block.
8. **Edge Decryption:** The CDN only routes encrypted blocks. When the application needs to decrypt, the client requests a VetKD decryption key from the Haven-AOL canister (v1 or v3 depending on content). The canister verifies token balance and derives the key. The client decrypts locally.

**What it doesn't do:**
- Use the public DHT for content routing
- Accept connections from unverified peers
- Decrypt at the network layer
- Make caching decisions

---

### 5. Client

**Role:** The end-user application (browser-based via Helia, or desktop via Kubo).

**How it works:**
1. Reads the Arkiv content registry to display the available library
2. User selects content → client gets CID + gate conditions from Arkiv
3. Client requests holding attestation from Haven-AOL canister (if not cached)
4. Client subscribes to the community's Waku content topic
5. Client broadcasts presence with attestation
6. Client receives presence messages from other peers, verifies attestations
7. Client connects to verified peers:
   - CDN nodes via WebTransport (direct)
   - Browser peers via WebRTC (Waku SDP signaling)
8. Client fetches encrypted blocks from verified peers via Bitswap
9. Client requests VetKD decryption key from Haven-AOL canister
10. Client decrypts locally and streams to player
11. Client optionally serves encrypted blocks to other verified peers

---

## Data Flow

### Upload (Creator)

```
1. Creator encrypts content with Haven-AOL (v1 or v3)
   - Access condition: token balance > threshold
   - Both content and CID are encrypted
2. Encrypted content is stored on IPFS / Filecoin Onchain Cloud
3. Creator runs haven-cli pipeline:
   - Uploads to IPFS/Filecoin
   - Generates VLM analysis
   - Creates attestation
4. haven-cli syncs to Arkiv:
   - Creates entity with CID, metadata, gate conditions
   - Entity has expires_in (default 4 weeks)
5. CDN nodes pick up the Arkiv entity event:
   - Pin the encrypted CID
   - Pin the filecoin_root_cid
6. CDN nodes join the community Waku topic and announce presence with attestation
```

### Playback (Viewer)

```
1. Client reads Arkiv registry → queries for haven/video entities
2. Client displays library (title, creator, thumbnail)
3. User selects content → client extracts:
   - encrypted_cid from attributes
   - gate conditions from attributes
4. Client requests holding attestation from Haven-AOL canister:
   - Signs EIP-712 AttestRequest
   - Canister verifies balance, signs attestation
   - Client caches attestation locally
5. Client subscribes to community Waku content topic
6. Client broadcasts presence + attestation
7. Client receives presence from other peers, verifies attestations offline
8. Client connects to verified CDN nodes via WebTransport
9. Client exchanges WebRTC SDP with verified browser peers via Waku
10. Client fetches encrypted blocks from verified peers via Bitswap
11. Client requests VetKD decryption key from Haven-AOL canister (v1 or v3)
12. Client decrypts locally and streams to player
13. Client serves encrypted blocks to other verified peers
```

### Unverified Peer Handling

```
1. Unverified peer broadcasts presence on Waku topic without valid attestation
2. Verified peers receive the message
3. Verified peers check attestation signature against canister public key
4. Signature invalid or missing → message ignored
5. No WebRTC connection established
6. No blocks exchanged
7. Unverified peer cannot discover providers or fetch content
```

### CDN Expiration

```
1. CDN sidecar tracks expires_in for each pinned entity
2. When entity expires:
   - Unpins CID from Boxo node
   - Stops announcing presence for that CID on Waku
3. Creator can renew by calling update_entity with new expires_in
4. CDN sidecar picks up update event and extends pin
```

---

## Storage Hierarchy

| Tier | Technology | Role | Eviction |
|------|-----------|------|----------|
| Warm | Filecoin Onchain Cloud + Arkiv | Source of truth, all content exists here | Never (until creator requests) |
| Hot | Boxo CDN Nodes (BadgerDB) | First seed, caches what Arkiv says | When entity expires |
| Ephemeral | Browser Swarm (Helia) | Free residential bandwidth, scales CDN | When user closes browser |
| Cold | Filecoin Standard | Permanent persistence for proven content | Never |

---

## What Lives Where

| Data | Location | Why |
|------|----------|-----|
| CID (encrypted) | On-chain (Arkiv attributes) | Static, must be queryable |
| CID (plaintext) | On-chain (Arkiv payload) | Static, needed for restore |
| Metadata | On-chain (Arkiv attributes) | Static, public, queryable |
| Gate conditions | On-chain (Arkiv attributes + payload) | Static, enforcement layer |
| piece_cid | On-chain (Arkiv payload) | Static, required for Synapse |
| Attestation (on-chain record) | On-chain (Arkiv payload) | Static, verifiable holding proof |
| expires_in | On-chain (Arkiv entity) | Static, controls CDN eviction |
| PeerIDs | Off-chain (Waku presence messages) | Ephemeral, high churn |
| Provider lists | Off-chain (Waku presence messages) | Ephemeral, scoped to community |
| Encrypted content | IPFS / Filecoin Onchain Cloud | Content-addressed, cacheable |
| Holding attestations (live) | Off-chain (Waku messages, client-cached) | Ephemeral, periodically refreshed |
| WebRTC SDP offers/answers | Off-chain (Waku, ephemeral) | Transient, only needed for handshake |
| Transfer metrics | Not stored | Not needed for caching decisions |

---

## Key Design Decisions

### Why Waku content topics instead of a DHT or IPIP-0337 router?
A DHT—whether public or private—broadcasts provider records to the entire swarm. There's no way to scope discovery to a specific community. For a community model where members care about a small subset of videos and don't want to announce to the world what they're hosting, the DHT is the wrong primitive. Waku content topics provide scoped discovery: only subscribers to the topic see presence messages. The topic is the privacy boundary.

### Why Haven-AOL holding attestations on Waku?
Waku content topics are not inherently access-controlled. Anyone who knows the topic name can subscribe. Holding attestations solve this: every presence message includes a canister-signed proof that the peer's wallet holds the required token. Peers verify this offline using the canister's public key. Unverified peers are ignored. This makes community membership cryptographically enforced without any central server, and without Waku relay nodes needing to understand access control.

### Why two attestation types (VetKD keys + t-Schnorr attestations)?
They serve different purposes:
- **VetKD decryption keys** unlock content. The client requests a key, the canister checks balance, and the key is derived. This is for consumption.
- **t-Schnorr holding attestations** prove membership. The client requests an attestation, the canister checks balance, and signs a portable proof. This is for discovery and peer authentication. The attestation doesn't contain a decryption key—it's just a signed statement that the wallet held the token at a specific time.

### Why is the CDN a dumb pipe?
The CDN caches exactly what the Arkiv contract says to cache. No demand-driven caching, no popularity algorithms. Creator controls caching by choosing to sync to Arkiv. The contract is the single source of truth.

### Why encrypt before storing on IPFS?
Encryption at the storage layer means CDN nodes and public gateways can cache content without access control. The CDN is a dumb pipe for encrypted bytes. Access control is enforced at the Haven-AOL canister layer (VetKD key release) and at the peer layer (attestation verification). Routing is open within the community; decryption is gated.

### Why not use the public IPFS DHT for content routing?
The public DHT leaks that content exists at a given CID. Using Waku content topics keeps provider records scoped to the community. Only community members with valid attestations can discover providers.

---

## Current Status

This is a planning document. The architecture is designed to be modular so individual actors can be built and tested independently:

1. **Arkiv content registry** — already implemented in `arkiv_sync.py`
2. **Haven-AOL canister** — already implemented (v1 + v3 protocols, attestation flow)
3. **CDN nodes** — Boxo-based nodes with Arkiv-aware pinning sidecar + Waku presence
4. **Browser CDN** — Helia + Waku package with attestation verification
5. **Client** — Helia with Waku routing and Haven-AOL integration

Each actor has a clear interface and can be developed in parallel.

---

## Open Questions

- How often should holding attestations be refreshed? The canister signs with a timestamp, but what's the acceptable staleness window for peers?
- Should the Waku content topic be derived from the token contract address, or should it be a separate community identifier?
- Should CDN nodes also broadcast holding attestations, or are they implicitly trusted as "first seeds"?
- How does the client select between CDN edges and browser peers—latency-based, or preference for P2P to reduce CDN costs?
- Should the CDN pin both `encrypted_cid` and `filecoin_root_cid`, or just one?
- How does the system handle content that is uploaded to IPFS but not synced to Arkiv—is it invisible to the CDN and swarm?

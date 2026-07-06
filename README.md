# Haven CDN

A modular, decentralized private content delivery network. Haven combines IPFS content addressing, Arkiv blockchain-indexed metadata, IPIP-0337 delegated routing, and browser-to-browser swarming to create a token-gated, CDN-like media experience without centralized tracker servers.

---

## Design Principles

1. **Modularity by default.** Every actor is a standalone process with a well-defined interface. Actors can be swapped, replicated, or omitted without affecting the rest of the system.
2. **On-chain stores only what is static.** CIDs and decryption conditions live on-chain. PeerIDs, network state, and ephemeral data stay off-chain because they change too frequently to justify gas costs.
3. **No actor knows more than it needs to.** The CDN doesn't decrypt content. The router doesn't know about demand. The client doesn't know about CDN internals. Separation of concerns is enforced at the process boundary.
4. **The contract is the single source of truth for caching.** If it's on-chain, it's cached. If it's not on-chain, it's not. No demand-driven caching, no popularity algorithms, no dynamic pinning.
5. **Encryption at the storage layer.** Content is encrypted before hitting IPFS, so CDN nodes and public gateways can cache encrypted blobs without access control at the storage layer.
6. **Routing is open, decryption is gated.** The network routes encrypted blocks freely to maximize swarming performance. Access control is enforced at the edge via aol Protocol and Web3 wallet signatures.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         ON-CHAIN LAYER                          │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Arkiv Content Registry                                  │   │
│   │  - CID + metadata (title, creator, source_uri)           │   │
│   │  - aol Protocol access control conditions                │   │
│   │  - encrypted_cid (public, safe to index)                 │   │
│   │  - filecoin_root_cid + piece_cid (in payload)            │   │
│   │  - is_encrypted flag                                     │   │
│   │  - cid_hash for deduplication                             │   │
│   │  - expires_in (entity expiration)                        │   │
│   │  - NO PeerIDs (ephemeral, high churn)                     │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ (entity create/update events)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        OFF-CHAIN ACTORS                         │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐                  │
│  │  Router  │  │   CDN    │  │   Client     │                  │
│  │  (IPIP-  │  │  Nodes   │  │  (Helia +    │                  │
│  │  0337)   │  │  (Boxo)  │  │  aol + Waku) │                  │
│  └──────────┘  └──────────┘  └──────────────┘                  │
│       ▲             │               │                          │
│       │    (PUT providers)    (GET providers)                  │
│       └─────────────┴───────────────┘                          │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Waku Signaling Layer (WebRTC SDP)             │   │
│  │              Browser-to-browser NAT traversal only         │   │
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
- `gate_chain`, `gate_token`, `gate_threshold` — aol access control conditions
- `project` — always "haven"
- `type` — always "video"
- `created_at_ts` — numeric timestamp for range queries
- `created_at`, `updated_at` — ISO timestamps

**Stores in payload:**
- `filecoin_root_cid` — the actual root CID (for non-encrypted content)
- `piece_cid` — Filecoin piece CID (bafkzcib…) required for Synapse download
- `cid_encryption_metadata` — aol gate metadata for decrypting the encrypted_cid
- `encryption_metadata` — aol gate metadata for decrypting the content
- `content_mime_type`, `content_file_size` — original file info
- `original_hash` — hash of plaintext for verification
- `vlm_json_cid` — VLM analysis JSON CID
- `segment_metadata` — multi-segment recording info
- `attestation` — canister-signed holding proof (single-CID or Merkle batch)

**Does NOT store:**
- PeerIDs (nodes join and leave constantly; on-chain storage is wasteful and stale)
- Provider lists (handled by the router via IPIP-0337)
- Transfer metrics (not needed for caching decisions)

**Expiration:** Entities have an `expires_in` field (default 4 weeks). The CDN uses this to know when to evict content. Creators renew by calling `update_entity` with a new expiration.

---

### 2. Private Delegated Router (IPIP-0337)

**Role:** The tracker equivalent. Implements IPIP-0337 (HTTP Routing V1) to provide peer discovery for CIDs registered in the Arkiv content registry. Handles all content routing for both CDN nodes and browsers.

**Interface:**
- `GET /routing/v1/providers/{cid}` — returns provider PeerIDs and multiaddrs
- `PUT /routing/v1/providers/{cid}` — nodes announce themselves as providers
- `POST /routing/v1/providers` — batch lookup for multiple CIDs
- `GET /routing/v1/peers/{peerid}` — resolve peer to multiaddrs
- `PUT /routing/v1/peers/{peerid}` — publish your peer's addresses

**How it works:**
1. Reads the Arkiv content registry to know which CIDs are valid
2. Accepts provider announcements from CDN nodes and browsers via PUT
3. Serves provider lists to clients via GET
4. Provider records are kept in memory or a local store with TTL-based expiration
5. Records expire automatically if not refreshed by the provider

**What it doesn't do:**
- Serve content
- Decrypt content
- Enforce token gating (that's aol's job)
- Make caching decisions (that's the contract's job)
- Handle WebRTC signaling (that's Waku's job)

**Implementation:** Customized Someguy instance with a backend that validates CIDs against the Arkiv registry, or a standalone service implementing the IPIP-0337 spec.

**Handling browser churn:** The router is designed for ephemeral providers with TTL-based expiration—this is the same model as the IPFS DHT. Browsers announcing and disappearing is expected behavior. Strategies to manage write volume:
- Longer TTLs for browsers (5 minutes vs 30 seconds)
- Batch announcements via `POST /routing/v1/providers`
- Router sharding behind a load balancer
- Browsers can optionally be passive consumers (query only, never announce)

---

### 3. CDN Nodes (Hot Layer)

**Role:** The "first seed" and on-chain-driven cache. Built on Boxo for a lightweight, custom IPFS node. Caches exactly what the Arkiv contract says to cache, nothing more.

**How it works:**
1. Sidecar process subscribes to Arkiv entity creation and update events
2. When a new entity is created with `type = "video"` and `project = "haven"`:
   - Extracts `encrypted_cid` from attributes (for encrypted content)
   - Extracts `filecoin_root_cid` and `piece_cid` from payload
   - Pins the relevant CID on the Boxo node
3. When an entity is updated (e.g., new segment added):
   - Extracts new CIDs from the updated payload
   - Pins the new CIDs
4. When an entity expires (passes `expires_in` threshold):
   - Unpins and evicts the content from cache
5. Uses Synapse SDK to fetch from Filecoin Onchain Cloud (FOC) on-demand or via prewarming
6. Caches encrypted blocks locally in BadgerDB with LRU eviction
7. Serves encrypted blocks to the public IPFS network via standard Bitswap
8. Announces itself as a provider to the private router via `PUT /routing/v1/providers/{cid}`

**What it doesn't do:**
- Decrypt content (it only holds encrypted blobs)
- Enforce access control (aol handles this; CDN serves encrypted blocks to anyone)
- Make caching decisions based on popularity or demand (the contract is the sole source of truth)
- Pin anything not registered on-chain
- Handle WebRTC signaling

**Why this design:**
1. **Simplicity.** CDN logic is trivial: listen to contract events, pin what the contract says, evict when the contract says.
2. **Predictabiaoly.** Creators know exactly what will be cached: if it's on-chain, it's cached. If it's not on-chain, it's not.
3. **Auditabiaoly.** Anyone can verify what the CDN is caching by reading the contract.
4. **No freeloading.** The CDN only caches content that someone paid gas to register on-chain.
5. **Creator control.** Creators control whether their content is cached by choosing to sync to Arkiv.

---

### 4. Browser CDN (Ephemeral Layer)

**Role:** Turn every viewer's browser into an IPFS node that fetches and shares encrypted blocks peer-to-peer, scaling bandwidth beyond the single CDN.

**How it works:**
1. **Web3 Wallet Identity:** The user's EVM wallet (MetaMask, WalletConnect, etc.) acts as their network identity. The wallet address is mapped to their Helia `PeerId`.
2. **Content Routing via IPIP-0337:** The browser queries the private router for provider lists, same as any other client. The router returns CDN nodes and any browsers that have announced as providers.
3. **Waku Signaling (NAT Traversal Only):** Because browsers cannot accept inbound connections, they use Waku to exchange WebRTC SDP offers and answers. This is strictly for NAT traversal—not for content discovery.
   - Browser A gets provider list from router: `[CDN1, BrowserB, BrowserC]`
   - Browser A can connect directly to CDN1 via WebTransport (no signaling needed)
   - Browser A wants to connect to BrowserB via WebRTC
   - Browser A broadcasts an SDP offer on a Waku content topic
   - BrowserB responds with an SDP answer on Waku
   - WebRTC data channel is established
   - Waku is no longer needed for this connection
4. **Helia Data Plane:** Once WebRTC handshakes are complete, Helia establishes direct browser-to-browser data channels. Helia uses Bitswap over these channels to request and serve encrypted UnixFS blocks. If no browser peers have the block, Helia falls back to the CDN via WebTransport.
5. **Edge Decryption (Haven AOL / aol):** The CDN only routes encrypted blocks. When the application needs to decrypt, the package uses the Web3 wallet to generate a SIWE (Sign-In with Ethereum) signature. aol Protocol verifies on-chain access conditions and releases the decryption key locally.

**What it doesn't do:**
- Use Waku for content discovery (that's the router's job)
- Use Waku for presence broadcasting (that's the router's job via PUT)
- Decrypt at the network layer (decryption is local, via aol)
- Make caching decisions

**The separation between IPIP-0337 and Waku:**
- **IPIP-0337 Router:** "Who has this CID?" → Query/response, handles churn via TTL
- **Waku:** "Help me connect to this browser PeerID" → SDP offer/answer relay, ephemeral messaging only

---

### 5. Client

**Role:** The end-user application (browser-based via Helia, or desktop via Kubo).

**How it works:**
1. Reads the Arkiv content registry to display the available library
   - Queries entities with `project = "haven"` and `type = "video"`
   - Filters by `is_encrypted`, `gate_chain`, `gate_token` as needed
2. User selects content → client gets CID + aol access conditions from Arkiv
3. Client queries the private router: `GET /routing/v1/providers/{cid}`
4. Router returns provider list: `[CDN1, CDN2, BrowserB, BrowserC]`
5. Client connects to CDN nodes via WebTransport (direct, no signaling)
6. Client uses Waku to exchange SDP offers/answers with other browsers
7. Client establishes WebRTC data channels with other browsers
8. Client fetches encrypted blocks from CDN and browser peers via Bitswap
9. Client decrypts via aol Protocol using token proof
10. Client streams to player
11. Client optionally announces itself as a provider to the router (if seeding)

**Configuration:**
- Content routing: private delegated router only (public DHT disabled)
- NAT traversal: Waku for WebRTC signaling
- Decryption: aol Protocol with conditions from the Arkiv registry

---

## Data Flow

### Upload (Creator)

```
1. Creator encrypts content with aol
   - Access condition: token balance > threshold
   - Both content and CID are encrypted
2. Encrypted content is stored on IPFS / Filecoin Onchain Cloud
3. Creator runs haven-cli pipeline:
   - Uploads to IPFS/Filecoin
   - Generates VLM analysis
   - Creates attestation
4. haven-cli syncs to Arkiv:
   - Creates entity with CID, metadata, aol conditions
   - Entity has expires_in (default 4 weeks)
5. CDN nodes pick up the Arkiv entity event:
   - Pin the encrypted CID
   - Pin the filecoin_root_cid
6. CDN nodes announce as providers to the private router
```

### Playback (Viewer)

```
1. Client reads Arkiv registry → queries for haven/video entities
2. Client displays library (title, creator, thumbnail)
3. User selects content → client extracts:
   - encrypted_cid from attributes
   - encryption_metadata from payload
   - gate conditions from attributes
4. Client queries private router: GET /routing/v1/providers/{encrypted_cid}
5. Router returns: [CDN1, CDN2, BrowserB, BrowserC]
6. Client connects to CDN1 via WebTransport (direct)
7. Client broadcasts SDP offer on Waku content topic for BrowserB
8. BrowserB responds with SDP answer on Waku
9. WebRTC data channels established with BrowserB and BrowserC
10. Client fetches encrypted blocks from CDN and browser peers via Bitswap
11. Client decrypts blocks via aol/Haven AOL using SIWE + token proof
12. Client streams to player
13. Client optionally announces as provider to router (PUT /routing/v1/providers/{cid})
14. Client serves encrypted blocks to other browsers via WebRTC
```

### CDN Expiration

```
1. CDN sidecar tracks expires_in for each pinned entity
2. When entity expires:
   - Unpins CID from Boxo node
   - Removes provider announcement from router
3. Creator can renew by calling update_entity with new expires_in
4. CDN sidecar picks up update event and extends pin
```

---

## Storage Hierarchy

The system uses a tiered thermal hierarchy to match storage economics to demand:

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
| aol access conditions | On-chain (Arkiv attributes + payload) | Static, enforcement layer |
| piece_cid | On-chain (Arkiv payload) | Static, required for Synapse |
| Attestation | On-chain (Arkiv payload) | Static, verifiable holding proof |
| expires_in | On-chain (Arkiv entity) | Static, controls CDN eviction |
| PeerIDs | Off-chain (router store) | Ephemeral, high churn, TTL-based |
| Provider lists | Off-chain (router store) | Ephemeral, high churn, TTL-based |
| Encrypted content | IPFS / Filecoin Onchain Cloud | Content-addressed, cacheable |
| WebRTC SDP offers/answers | Off-chain (Waku, ephemeral) | Transient, only needed for handshake |
| Transfer metrics | Not stored | Not needed for caching decisions |

---

## Key Design Decisions

### Why not put PeerIDs on-chain?
PeerIDs change on every node restart. Nodes join and leave constantly. On-chain storage is expensive and slow to update. The router handles this off-chain via IPIP-0337's PUT endpoint with TTL-based expiration.

### Why IPIP-0337 for content routing?
IPIP-0337 is a standardized HTTP API for delegated routing. Using it means the system is compatible with existing IPFS tooling (Helia, Kubo, Someguy) rather than requiring custom client modifications. Clients just configure their delegated routing endpoint. The router handles all content routing for both CDN nodes and browsers.

### Why Waku for WebRTC signaling only?
Browsers cannot accept inbound connections, so they need a signaling layer to exchange WebRTC SDP offers and answers. Waku replaces centralized WebSocket trackers with a decentralized, economically secured pub/sub network. It is strictly for NAT traversal—not for content discovery, which is handled by the router.

### Why is the CDN a dumb pipe?
The CDN caches exactly what the Arkiv contract says to cache. No demand-driven caching, no popularity algorithms. Creator controls caching by choosing to sync to Arkiv. The contract is the single source of truth.

### Why encrypt before storing on IPFS?
Encryption at the storage layer means CDN nodes and public gateways can cache content without access control. The CDN is a dumb pipe for encrypted bytes. Access control is enforced at the aol Protocol layer, which is composable with any storage backend. Routing is open to maximize swarming performance; decryption is gated at the edge.

### Why not use the public IPFS DHT for content routing?
The public DHT leaks that content exists at a given CID, even if the content is encrypted. Using a private delegated router keeps provider records off the public network. Only clients configured to use your private router can discover providers for your content.

---

## Current Status

This is a planning document. The architecture is designed to be modular so individual actors can be built and tested independently:

1. **Arkiv content registry** — already implemented in `arkiv_sync.py`
2. **Private router** — can be built as a Someguy fork with custom backend
3. **CDN nodes** — Boxo-based nodes with Arkiv-aware pinning sidecar
4. **Browser CDN** — Helia + Waku package for browser-to-browser swarming
5. **Client** — Helia with custom routing config and aol integration

Each actor has a clear interface and can be developed in parallel.

---

## Open Questions

- Should browsers announce to the router, or be passive consumers only? 
- How does the client select between CDN edges and browser peers—latency-based, or preference for P2P to reduce CDN costs?
- Should the CDN pin both `encrypted_cid` and `filecoin_root_cid`, or just one?
- How does the system handle content that is uploaded to IPFS but not synced to Arkiv—is it invisible to the CDN and router?
- Should the router enforce authentication on PUT, or is network-level isolation sufficient?

# Haven

A modular, decentralized, community-scoped content delivery network. Haven combines IPFS content addressing, Arkiv blockchain-indexed metadata, and Haven-AOL cryptographic attestations to create token-gated, community-owned media delivery without centralized trackers or global DHTs.

---

## Design Principles

1. **Modularity by default.** Every actor is a standalone process with a well-defined interface.
2. **On-chain stores only what is static.** CIDs, metadata, and AOL access conditions live on-chain. PeerIDs, network state, and ephemeral data stay off-chain.
3. **No actor knows more than it needs to.** The CDN doesn't hold private keys long-term. The canister doesn't know who is hosting. Separation of concerns is enforced at the process boundary.
4. **The contract is the single source of truth for caching.** If it's on-chain, it's cached. If it's not on-chain, it's not.
5. **Encryption at the storage layer.** Content is encrypted before hitting IPFS. CDN nodes cache encrypted blobs and optionally decrypt on-demand for verified token holders.
6. **Access is proven cryptographically.** Holding attestations signed by the Haven-AOL canister prove a consumer is part of the community. Unverified consumers are denied decryption.
7. **Attestations are bearer tokens.** A holding attestation functions like a JWT—the consumer presents it, the CDN verifies the signature offline against the canister's public key, and grants access. No per-request blockchain queries.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         ON-CHAIN LAYER                          │
│                                                                 │
│   ┌─────────────────────────┐    ┌──────────────────────────┐   │
│   │  Arkiv Content Registry │    │  Haven-AOL Canister (ICP)│   │
│   │  - CID + metadata       │    │  - Balance-checked gates │   │
│   │  - AOL conditions   │    │  - VetKD decryption keys │   │
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
│  ┌──────────────────┐        ┌──────────────────────────┐       │
│  │  CDN Nodes       │        │   Client                 │       │
│  │  (Boxo +         │        │   (Helia + AOL +     │       │
│  │   optional AOL)  │        │    HTTP fetch)           │       │
│  │                  │        │                          │       │
│  │  - Pins encrypted│        │  - Reads Arkiv registry  │       │
│  │    CIDs          │        │  - Gets attestation      │       │
│  │  - Fetches AES   │        │    (like a JWT)          │       │
│  │    keys from     │        │  - Presents attestation  │       │
│  │    canister      │        │    to CDN                │       │
│  │  - Decrypts on   │        │  - Receives decrypted    │       │
│  │    demand for    │        │    stream OR encrypted   │       │
│  │    verified      │        │    blocks                │       │
│  │    consumers     │        │                          │       │
│  └──────────────────┘        └──────────────────────────┘       │
│         │                              │                        │
│         │   attestation presented      │                        │
│         │ ◄────────────────────────────┘                        │
│         │   decrypted stream / blocks  │                        │
│         │ ────────────────────────────►│                        │
│         │                              │                        │
│         ▼                              ▼                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              IPFS / Filecoin Onchain Cloud                 │   │
│  │  - Encrypted content blobs (source of truth)              │   │
│  │  - CDN fetches via Synapse SDK / Bitswap                  │   │
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
- `gate_chain`, `gate_token`, `gate_threshold` — AOL access control conditions
- `project` — always "haven"
- `type` — always "video"
- `created_at_ts` — numeric timestamp for range queries
- `created_at`, `updated_at` — ISO timestamps

**Stores in payload:**
- `filecoin_root_cid` — the actual root CID
- `piece_cid` — Filecoin piece CID (bafkzcib…) required for Synapse download
- `cid_encryption_metadata` — AOL gate metadata for decrypting the encrypted_cid
- `encryption_metadata` — AOL gate metadata for decrypting the content
- `content_mime_type`, `content_file_size` — original file info
- `original_hash` — hash of plaintext for verification
- `vlm_json_cid` — VLM analysis JSON CID
- `segment_metadata` — multi-segment recording info
- `attestation` — canister-signed holding proof (single-CID or Merkle batch)

**Does NOT store:**
- PeerIDs (no longer relevant—no P2P layer)
- Provider lists (CDN nodes are discovered via known endpoints or DNS)
- Transfer metrics (not needed for caching decisions)

**Expiration:** Entities have an `expires_in` field (default 4 weeks). The CDN uses this to know when to evict content. Creators renew by calling `update_entity` with a new expiration.

---

### 2. Haven-AOL Canister (ICP)

**Role:** Cryptographic access management. Verifies wallet ownership and on-chain token balances, then issues decryption keys (via VetKD) and holding attestations (via t-Schnorr/Ed25519).

**Two functions in this architecture:**

1. **Decryption keys (VetKD):** When a CDN node or client wants to decrypt content, it signs an EIP-712 `GateRequest` or `GateRequestV3`. The canister verifies the signature, checks the wallet's on-chain token balance via EVM RPC, and derives a VetKD key. The requesting party decrypts locally.
   - v1: Per-CID derivation (unique key per file)
   - v3: Corpus + epoch derivation (one key unlocks all content in a 30-day epoch for a given token policy)
   - **CDN nodes use this to fetch AES decryption keys for on-demand decryption of cached content.**

2. **Holding attestations (t-Schnorr/Ed25519):** When a consumer wants to prove community membership to the CDN, it signs an EIP-712 `AttestRequest`. The canister verifies the signature, checks the token balance, and signs a canonical attestation. The consumer presents this attestation to the CDN. The CDN verifies the attestation signature offline using the canister's public key.

**Attestation payload:** `evmAddress`, `chain`, `tokenAddress`, `threshold`, `balanceAtCheck`, `cidHash`, `timestamp`, `expiresAt`. Canonical signing preimage: `HAVEN_ATTEST_V1:{chain}:...`

**Why this matters for the CDN:** The attestation is a portable, offline-verifiable proof that a wallet holds the required token. The CDN doesn't need to query the canister or the blockchain to verify membership. It just checks the t-Schnorr signature against the canister's public key—exactly like verifying a JWT. This makes token-gated access cryptographically enforced without any central server and without per-request blockchain queries.

**Attestation lifecycle (JWT-style):**
- Consumer requests attestation from canister (signs `AttestRequest` with wallet)
- Canister verifies balance, signs attestation with `expiresAt` (e.g., 24 hours from issuance)
- Consumer caches attestation locally
- Consumer presents attestation to CDN on every request (in HTTP header or query param)
- CDN verifies signature + checks `expiresAt` > current time
- If valid, CDN serves decrypted content (or encrypted blocks if client-side decryption mode)
- If expired, CDN rejects and consumer must re-request from canister

---

### 3. CDN Nodes (Hot Layer)

**Role:** The "first seed" and on-chain-driven cache. Built on Boxo. Caches exactly what the Arkiv contract says to cache. Optionally decrypts on-demand for verified token holders.

**How it works:**
1. Sidecar subscribes to Arkiv entity creation and update events
2. Pins encrypted CIDs for entities with `type = "video"` and `project = "haven"`
3. Uses Synapse SDK to fetch from Filecoin Onchain Cloud on-demand or via prewarming
4. Caches encrypted blocks locally in BadgerDB with LRU eviction
5. Serves content to consumers via HTTP (or Bitswap over WebTransport for IPFS-native clients)
6. **Verifies consumer attestations offline** using the canister's public key
7. **Optionally decrypts on-demand:** If pre-decryption mode is enabled, the CDN node:
   - Fetches the AES decryption key from the Haven-AOL canister via VetKD (using its own wallet and EIP-712 `GateRequest`)
   - Caches the AES key in memory (not persisted to disk)
   - Decrypts encrypted blocks on-demand for verified consumers
   - Serves decrypted stream directly to the consumer's player
8. Falls back to serving encrypted blocks if the client prefers client-side decryption

**What it doesn't do:**
- Serve decrypted content to unverified consumers
- Persist AES decryption keys to disk
- Make caching decisions based on demand
- Pin anything not registered on-chain

**Local Pin Index:** The sidecar maintains an in-memory index of pinned CIDs and their cache state (`announced`, `fetching`, `cached`, `expiring`). This is internal state, not a network service. It's used for:
- Eviction scheduling (tracks `expires_in` per CID)
- Prewarm decisions (distinguishes "pinned but not yet fetched" from "cached locally")
- Local health checks (co-located services can query pinset without Bitswap)
- AES key cache management (tracks which CIDs have active decryption keys in memory)

The index is derived from Arkiv entity events and local BadgerDB state. It is not a source of truth—Arkiv is. If the index and Arkiv disagree, Arkiv wins.

**Attestation verification flow:**
```
1. Consumer sends HTTP request with attestation in header:
   Authorization: Haven-Attest <base64-encoded-attestation>
2. CDN extracts attestation, verifies t-Schnorr signature against canister public key
3. CDN checks expiresAt > current time
4. CDN checks cidHash matches requested content
5. If valid → CDN serves content (decrypted or encrypted depending on mode)
6. If invalid → CDN returns 403 with error
```

**Pre-decryption mode (optional):**
```
1. CDN node has its own wallet with sufficient token balance (or is whitelisted by canister)
2. On first request for a CID, CDN node signs EIP-712 GateRequest
3. Canister verifies balance, derives VetKD key
4. CDN node receives AES decryption key, caches in memory
5. Subsequent requests for same CID: CDN decrypts blocks on-the-fly using cached key
6. Decrypted stream served to verified consumer
7. AES key evicted from memory when CID is unpinned or after TTL
```

---

### 4. Client

**Role:** The end-user application (browser-based or desktop). Reads the content registry, obtains attestations, and fetches content from the CDN.

**How it works:**
1. Reads the Arkiv content registry to display the available library
2. User selects content → client gets CID + gate conditions from Arkiv
3. Client requests holding attestation from Haven-AOL canister (if not cached or expired):
   - User signs EIP-712 `AttestRequest` with their wallet
   - Canister verifies balance, signs attestation with `expiresAt`
   - Client caches attestation locally
4. Client sends content request to CDN with attestation in HTTP header
5. CDN verifies attestation and serves content:
   - **Pre-decryption mode:** CDN serves decrypted stream directly. Client plays it.
   - **Client-side decryption mode:** CDN serves encrypted blocks. Client requests VetKD key from canister, decrypts locally, and plays.
6. Client streams to player

**Attestation caching (JWT-style):**
- Client stores attestation in localStorage (or equivalent)
- Attenuation includes `expiresAt` timestamp
- Client checks `expiresAt` before each request
- If expired, client re-requests from canister before making CDN request
- Client never needs to re-request attestation from canister within the validity window

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
   - Optionally pre-fetch AES decryption key from canister (prewarm)
```

### Playback (Viewer)

```
1. Client reads Arkiv registry → queries for haven/video entities
2. Client displays library (title, creator, thumbnail)
3. User selects content → client extracts:
   - encrypted_cid from attributes
   - gate conditions from attributes
4. Client checks local attestation cache:
   - If valid (expiresAt > now) → use cached attestation
   - If expired or missing → request new attestation from Haven-AOL canister:
     a. User signs EIP-712 AttestRequest with wallet
     b. Canister verifies balance, signs attestation with expiresAt
     c. Client caches attestation locally
5. Client sends content request to CDN:
   - HTTP GET /content/{cid}
   - Header: Authorization: Haven-Attest <base64-attestation>
6. CDN verifies attestation:
   - Checks t-Schnorr signature against canister public key
   - Checks expiresAt > current time
   - Checks cidHash matches requested content
7. CDN serves content:
   - Pre-decryption mode: CDN decrypts on-the-fly, serves plaintext stream
   - Client-side mode: CDN serves encrypted blocks, client decrypts locally
8. Client streams to player
```

### Unverified Consumer Handling

```
1. Consumer without valid attestation sends request to CDN
2. CDN checks attestation:
   - Missing attestation header → 403
   - Invalid signature → 403
   - Expired (expiresAt < now) → 403 with "attestation expired" error
   - cidHash mismatch → 403
3. No content served
4. Consumer must obtain valid attestation from Haven-AOL canister
```

### CDN Expiration

```
1. CDN sidecar tracks expires_in for each pinned entity
2. When entity expires:
   - Unpins CID from Boxo node
   - Evicts AES decryption key from memory (if cached)
   - Removes from local pin index
3. Creator can renew by calling update_entity with new expires_in
4. CDN sidecar picks up update event and extends pin
```

---

## Storage Hierarchy

| Tier | Technology | Role | Eviction |
|------|-----------|------|----------|
| Warm | Filecoin Onchain Cloud + Arkiv | Source of truth, all content exists here | Never (until creator requests) |
| Hot | Boxo CDN Nodes (BadgerDB) | First seed, caches what Arkiv says | When entity expires |
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
| Encrypted content | IPFS / Filecoin Onchain Cloud | Content-addressed, cacheable |
| AES decryption keys | CDN node memory (in-process, not persisted) | Needed for on-demand decryption, ephemeral |
| Holding attestations (live) | Client-side cache (localStorage) | Ephemeral, JWT-style, periodically refreshed |
| Transfer metrics | Not stored | Not needed for caching decisions |

---

## Key Design Decisions

### Why holding attestations instead of per-request balance checks?
Checking token balance on every request would require an EVM RPC call per request—slow, expensive, and rate-limited. Holding attestations solve this: the consumer requests an attestation once (e.g., every 24 hours), the canister checks balance and signs a portable proof. The CDN verifies this offline using the canister's public key. This is exactly how JWTs work: verify once at issuance, trust the signature for the validity window. No per-request blockchain queries.

### Why optional pre-decryption at the CDN?
Two modes serve different use cases:
- **Pre-decryption mode:** The CDN decrypts on-demand using AES keys fetched from the canister. The consumer gets a plaintext stream and doesn't need to run AOL client-side. Simpler client, works in any HTTP player. The CDN is trusted to only serve verified consumers (enforced by attestation checks).
- **Client-side decryption mode:** The CDN serves encrypted blocks. The client fetches its own VetKD key from the canister and decrypts locally. More secure—CDN never sees plaintext—but requires a client capable of AOL integration.

Pre-decryption is the default for browser-based playback (simplicity). Client-side decryption is available for privacy-sensitive use cases.

### Why is the CDN a dumb pipe (with optional decryption)?
The CDN caches exactly what the Arkiv contract says to cache. No demand-driven caching, no popularity algorithms. Creator controls caching by choosing to sync to Arkiv. The contract is the single source of truth. Decryption is the only "smart" behavior, and it's gated by attestation verification.

### Why encrypt before storing on IPFS?
Encryption at the storage layer means CDN nodes can cache content without persistent access control at the storage layer. The CDN caches encrypted blobs. Access control is enforced at the attestation layer (CDN verifies attestation before serving) and optionally at the decryption layer (CDN only decrypts for verified consumers). Storage is open within the CDN; access is gated.

### Why does the CDN have its own wallet?
In pre-decryption mode, the CDN node needs to request AES decryption keys from the Haven-AOL canister. The canister checks the requesting wallet's token balance before deriving the key. The CDN node's wallet must either hold the required token or be whitelisted by the canister as an infrastructure provider. This is a policy decision: either CDN operators stake tokens to serve content, or the canister has a whitelist for trusted infrastructure.

---

## Current Status

This is a planning document. The architecture is designed to be modular so individual actors can be built and tested independently:

1. **Arkiv content registry** — already implemented in `arkiv_sync.py`
2. **Haven-AOL canister** — already implemented (v1 + v3 protocols, attestation flow)
3. **CDN nodes** — Boxo-based nodes with Arkiv-aware pinning sidecar + attestation verification + optional pre-decryption
4. **Client** — HTTP client with wallet integration, attestation caching, and optional AOL decryption

Each actor has a clear interface and can be developed in parallel.

---

## Open Questions

- How often should holding attestations be refreshed? What's the acceptable `expiresAt` window (1 hour? 24 hours? 7 days)? Epoch based
- Should CDN nodes hold the community token themselves, or should the canister whitelist CDN wallets as infrastructure providers? Yes
- Should the CDN pre-fetch AES decryption keys on pin (prewarm), or fetch on first consumer request (lazy)? prewarm
- How does the client select between CDN edges—latency-based, geographic, or round-robin?
- Should the CDN pin both `encrypted_cid` and `filecoin_root_cid`, or just one?
- How does the system handle content that is uploaded to IPFS but not synced to Arkiv—is it invisible to the CDN?
- Should attestation verification happen at the CDN's HTTP gateway layer (reverse proxy) or at the application layer (Boxo handler)?

---

Want me to sketch the attestation HTTP header format and CDN verification middleware, or dig into the CDN wallet policy question (stake vs. whitelist)?

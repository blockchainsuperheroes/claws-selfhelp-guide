# Pentagon NFT Data API

**The authoritative source for NFT ownership across Pentagon Games IP.**

Every Pentagon service — marketplace, mining, wallet, verifier, admin dashboard — reads ownership from this single backend. One API, one answer, always correct.

---

## What This Is

A unified NFT ownership tracking system with:

- **49 active contracts** across 14 blockchain networks
- **Three-layer sync engine** — real-time listener + block scanner + on-chain verifier
- **Authenticated API** with per-app rate limits and usage tracking
- **Sub-second ownership queries** backed by PostgreSQL

If you need to know who owns an NFT across Pentagon's ecosystem — this is where you ask.

---

## Quick Start

### 1. Get an App Key

App keys are managed through the Pentagon Admin Panel. Each key tracks:
- Which app is making requests
- Usage count and rate limits  
- Allowed origins (for web apps) or IPs (for servers)

Request a key from the Pentagon team. Keys look like: `pk_live_xxxxx` (production) or `pk_test_xxxxx` (development).

### 2. Make Your First Query

```bash
# Look up NFTs owned by a wallet address
curl -H "X-PG-App-Key: pk_live_YOUR_KEY" \
  "https://nft-data.pentagon.games/api/v1/nft/owner/0x37224cFD71347Da6f097c94B8649f9C552f803c9"
```

Response:
```json
{
  "total": 17,
  "results": [
    {
      "token_id": "4201",
      "name": "BCSH #4201",
      "owner": "0x37224cfd71347da6f097c94b8649f9c552f803c9",
      "image": "https://assets.pentagon.games/bcsh/4201.png",
      "contract": "0xcDAD57bFc48E8373280C6dc3039C5169353B6879",
      "chain_id": 3344,
      "collection": "BCSH OASYS"
    }
  ]
}
```

---

## API Reference

### Base URL

```
https://nft-data.pentagon.games
```

### Authentication

Pass your app key via the `X-PG-App-Key` header:

```
X-PG-App-Key: pk_live_your_key_here
```

Public endpoints work without a key but are rate-limited to **10 requests/minute**.  
Authenticated requests get **up to 100 requests/minute** (configurable per key).

### Endpoints

#### Health Check

```
GET /
```

Returns service status. No auth required.

```json
{
  "service": "nft-data-api",
  "status": "ok",
  "auth": "X-PG-App-Key (shared with pentagon-login-backend)",
  "timestamp": "2026-05-10T23:06:21.435905+00:00"
}
```

---

#### Look Up NFTs by Owner

```
GET /api/v1/nft/owner/<wallet_address>
```

Returns all NFTs owned by a wallet address across all tracked chains and collections.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `wallet_address` | path | Yes | Ethereum-style wallet address (0x...) |
| `chain_id` | query | No | Filter by chain ID (e.g., `3344` for Pentagon) |
| `contract` | query | No | Filter by contract address |
| `collection` | query | No | Filter by collection name |
| `page` | query | No | Page number (default: 1) |
| `limit` | query | No | Results per page (default: 50, max: 200) |

**Response:**

```json
{
  "owner": "0x37224cFD71347Da6f097c94B8649f9C552f803c9",
  "total": 17,
  "page": 1,
  "limit": 50,
  "nfts": [
    {
      "tokenId": "4201",
      "name": "BCSH #4201",
      "owner": "0x37224cfd71347da6f097c94b8649f9c552f803c9",
      "image": "https://images.pfpvault.com/3344/0xcdad.../4201.webp",
      "contractAddress": "0xcDAD57bFc48E8373280C6dc3039C5169353B6879",
      "chain": 3344,
      "collection": "BCSH OASYS",
      "updatedAt": "2026-05-11T04:15:39.874904+00:00"
    }
  ]
}
```

> **Image Resolution:** The `image` field returns the best available source — prefers per-token CDN (`images.pfpvault.com`) over on-chain metadata (`arweave.net`). This ensures upgraded NFTs (e.g. Dark Setsuko, Obelith variants) display their correct artwork.
```

---

#### Look Up NFT by Token ID

```
GET /api/v1/nft/<contract_address>/<token_id>
```

Returns details for a specific NFT including current owner and metadata.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `contract_address` | path | Yes | Contract address |
| `token_id` | path | Yes | Token ID |

**Response:**

```json
{
  "contract": "0xcDAD57bFc48E8373280C6dc3039C5169353B6879",
  "tokenId": "5555000000553",
  "collection": "BCSH OASYS",
  "chain": 3344,
  "owner": "0x6e28399aa9dd598caa49df9fa46d0ae87c0e7d57",
  "metadata": {
    "name": "Dark Setsuko",
    "description": "After shattering her katana...",
    "image": "https://images.pfpvault.com/3344/0xcdad.../5555000000553.webp",
    "imageSource": "cached",
    "attributes": [
      {"trait_type": "Background", "value": "Blue"}
    ]
  },
  "source": {
    "tokenUri": "https://api.bcsh.xyz/metadata/5555000000553",
    "lastSync": "2026-05-08T18:19:02+00:00"
  }
}
```

> **`imageSource` field:** Indicates where the image was resolved from:
> - `"cached"` — per-token CDN image from pfpvault (most accurate for upgraded NFTs)
> - `"onchain"` — image URL from on-chain metadata (tokenURI)
> - `"original"` — fallback original image URL
```

---

#### List Tracked Collections

```
GET /api/v1/collections
```

Returns all actively tracked NFT collections with contract addresses and chain info.

**Response:**

```json
{
  "total": 49,
  "collections": [
    {
      "name": "BCSH OASYS",
      "contract": "0xcDAD57bFc48E8373280C6dc3039C5169353B6879",
      "chain_id": 3344,
      "chain_name": "Pentagon",
      "type": "ERC721",
      "sync_status": "healthy",
      "total_supply": 10000,
      "last_synced": "2026-05-10T23:00:00Z"
    }
  ]
}
```

---

#### Collection Stats

```
GET /api/v1/collection/<contract_address>/stats
```

Returns ownership statistics for a collection.

**Response:**

```json
{
  "contract": "0xcDAD57bFc48E8373280C6dc3039C5169353B6879",
  "name": "BCSH OASYS",
  "total_supply": 10000,
  "unique_owners": 3421,
  "top_holders": [
    {"address": "0x...", "count": 150},
    {"address": "0x...", "count": 87}
  ]
}
```

---

#### Transfer History

```
GET /api/v1/nft/<contract_address>/<token_id>/transfers
```

Returns transfer history for a specific NFT.

**Response:**

```json
{
  "transfers": [
    {
      "from": "0x0000000000000000000000000000000000000000",
      "to": "0x37224cfd71347da6f097c94b8649f9c552f803c9",
      "tx_hash": "0xabc123...",
      "block_number": 1234567,
      "timestamp": "2026-01-15T12:30:00Z",
      "source": "scanner"
    }
  ]
}
```

---

### Error Responses

```json
// 401 — Invalid or missing app key
{"error": "Invalid or inactive app key"}

// 404 — NFT or collection not found
{"error": "NFT not found"}

// 429 — Rate limit exceeded
{"error": "Rate limit exceeded", "retry_after": 60}
```

---

## Supported Chains

| Chain | Chain ID | Native RPC | Status |
|-------|----------|------------|--------|
| Pentagon | 3344 | `rpc.pentagon.games` | ✅ Real-time + Scanner |
| Ethereum | 1 | Infura | ✅ Scanner |
| BSC | 56 | Ankr | ✅ Scanner |
| Polygon | 137 | Infura | ✅ Scanner |
| Arbitrum | 42161 | Infura | ✅ Scanner |
| Core | 1116 | CoreDAO | ✅ Scanner |
| OASYS | 248 | OASYS | ✅ Scanner |
| Monad | 143 | Infura | ✅ Scanner |
| Avalanche | 43114 | Public | ✅ Scanner |
| SKALE Nebula | 1482601649 | SKALE | ✅ Scanner |

Pentagon chain contracts get real-time WebSocket tracking (sub-second updates). All other chains use block-range scanning (2-6 minute intervals).

---

## Tracked Collections

### Pentagon Chain (3344)

| Collection | Contract | Type |
|------------|----------|------|
| BCSH OASYS | `0xcDAD57bFc48E8373280C6dc3039C5169353B6879` | ERC721 |
| BCSH Vaelion | `0x35A31E23FB1AAD207Ad4075C52e981dC9165059b` | ERC721 |
| BCSH No_5 | `0x0A5FE002F2eD146415A1f4865DE1c180a39D599E` | ERC721 |
| BCSH Tamago | `0x2444D26cC268848f2B1bd837456537510F1aac81` | ERC721 |
| BCSH Baiyi | `0xF8C869a5575f44fB9b68F670e6B158B30fB8Ccf5` | ERC721 |
| GCN NFT | `0x42D97d553Ee71deF76a131e48Fc42BCf8da3B141` | ERC721 |
| GCN Shards | `0xfd8276d1745761D3C5C55269fe4466FCD31D9dD3` | ERC1155 |
| Gunnies PFP | `0x7a8a3236e3783E7cC33b97729378e31Cf14d3Ebc` | ERC721 |
| PentaPets | `0xe6BdE156369D209C4d420E966541eE17093705B5` | ERC721 |
| RugPull Art | `0xd77f88ef51b2589d132d6eb61068079f61dfe4a3` | ERC721 |
| Gemry | `0xc05b96b89Ce46c306223E3f4c413891d17E1De70` | ERC721 |
| PEGNAMES | `0xf97EB9f8293D1FD5587a809Eb74518c300738d07` | ERC721 |
| EtherFantasy | `0x8F83c6122Dd4d275B53a7846B3D3dB29Cca1e698` | ERC721 |

### Ethereum (1)

| Collection | Contract | Type |
|------------|----------|------|
| BCSH ETH | `0x53b719422f427Fe158f480Bfc3Cf32201e416F89` | ERC721 |
| Moonbirds | See contract registry | ERC721 |
| Azuki | See contract registry | ERC721 |
| Doodles | See contract registry | ERC721 |
| CLONE X | See contract registry | ERC721 |
| + 11 more | — | — |

*Full list available via `GET /api/v1/collections`*

---

## Sync Engine Architecture

The NFT Data API is backed by a three-layer sync engine that guarantees ownership accuracy:

### Layer 1: Real-Time Listener
- **How:** WebSocket subscriptions to Transfer events
- **Speed:** ~2 seconds from on-chain transfer to DB update
- **Scope:** Pentagon chain contracts (we own the RPC — zero rate limits)
- **Recovery:** Saves block checkpoints; on restart, catches up any missed blocks via Layer 2

### Layer 2: Block-Range Scanner
- **How:** Periodic `eth_getLogs()` calls scanning from last saved block to chain head
- **Speed:** 2-6 minute intervals per contract
- **Scope:** All 14 chains, all 49 contracts
- **Reliability:** Never misses blocks — picks up from saved checkpoint even after downtime

### Layer 3: Periodic Verifier
- **How:** On-chain `ownerOf()` spot checks comparing DB state to chain truth
- **Speed:** Hourly sampling (10%), daily full sweep
- **Purpose:** Catches edge cases both layers might miss (contract upgrades, admin transfers, bugs)

### Triangulation Rule

When Layer 1 and Layer 2 disagree → **Scanner wins** (more reliable).  
If disagreement persists → Layer 3 calls `ownerOf()`.  
**Chain is always ground truth.**

---

## For Pentagon Internal Services

### Marketplace

Instead of running your own `TransferListenerService`, read ownership from:
```
GET /api/v1/nft/owner/<address>?chain_id=3344
```
Keep your local DB for listings, bids, cart, and stats. Let the master handle ownership.

### Mining / Wallet / Verifier

Same pattern — query the API for ownership, don't maintain your own sync.

### Internal Health Dashboard

```
GET /api/internal/sync/health    — Per-contract sync status
GET /api/internal/sync/gaps      — Listener/scanner gap detections
GET /api/internal/sync/mismatches — Ownership verification failures
```

Requires a `server`-type app key.

---

## For External Projects

### Getting Listed

To have your NFT collection tracked by Pentagon:

1. **Contact the Pentagon team** with your contract address and chain
2. We register your contract in our system
3. Sync begins automatically — all three layers
4. You receive an app key for API access

### What You Get

- ✅ **Bulletproof ownership tracking** — triple-verified, never stale
- ✅ **Fast API access** — sub-second queries
- ✅ **Cross-chain visibility** — tracked wherever your NFTs exist
- ✅ **Transfer history** — full log from deployment to present
- ✅ **Dashboard visibility** — see sync health for your collection
- ✅ **Pentagon ecosystem integration** — marketplace, mining, wallet, verifier

### Requirements

- ERC721 or ERC1155 contract
- Standard `Transfer` event emissions
- Contract deployed on a supported chain (see Supported Chains above)

---

## Rate Limits

| Auth Level | Limit | Scope |
|------------|-------|-------|
| No key (public) | 10/min | Per IP |
| App key (default) | 100/min | Per key |
| App key (custom) | Up to 500/min | Per key |

Rate limits are configurable per app key via the admin panel.

---

## Image Resolution

The API uses a **three-tier image fallback** to ensure NFTs always display the best available artwork:

1. **`cached_image_url`** (pfpvault CDN) — Per-token rendered images, includes upgraded variants. Preferred source.
2. **`image`** (on-chain metadata) — Image URL from the token's on-chain metadata (tokenURI → JSON → image). May be generic for collections with upgradeable metadata.
3. **`original_image_url`** — Original image captured at first sync.

The `image` field in API responses always returns the best available (`cached > onchain > original`).

For single-token lookups, the `imageSource` field tells you which tier was used.

### Why This Matters

Some collections (like BCSH OASYS / Setsuko) have upgradeable NFTs where the on-chain metadata returns a generic base image for all variants. The `cached_image_url` from pfpvault stores the correct per-token artwork including upgrades (Dark Setsuko, Obelith Setsuko, etc.).

### Metadata Refresh (Coming Soon)

A queued metadata refresh system is planned:
- **User-facing:** `POST /api/v1/nft/refresh/<contract>/<tokenId>` — rate-limited to 1 refresh per token per user per 10 minutes, queued via RabbitMQ
- **Admin:** Full collection refresh from the admin panel with progress tracking
- This handles the OpenSea-style "refresh metadata" workflow for upgradeable NFT collections

---

## Status & Monitoring

Service health: `GET /` — returns `200 OK` when operational.

Sync health per collection is available via the internal dashboard endpoints (requires server-type key).

---

## Support

- **Pentagon Games Discord:** [discord.gg/pentagongamesxp](https://discord.gg/pentagongamesxp)
- **Internal:** #nft-data-api channel
- **API Issues:** Contact Pentagon dev team

---

*Built and maintained by Pentagon Games infrastructure team.*

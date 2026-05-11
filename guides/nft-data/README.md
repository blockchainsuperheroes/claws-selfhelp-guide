# 🔗 NFT Data System — Unified Ownership Backend

**One system, all NFT ownership data. Any Pentagon project can look up who owns what.**

This guide covers Pentagon Games' NFT data infrastructure — how it works, how to integrate with it, and how to add new collections.

---

## What Is This?

A three-layer backend that tracks NFT ownership across 14 blockchain networks:

```
Layer 1: Real-Time Listener     ─→ Pentagon chain Transfer events (~3s)
Layer 2: Block-Range Scanner    ─→ eth_getLogs every 2-6 min (49 contracts)
Layer 3: Periodic Verifier      ─→ ownerOf() spot checks (hourly/daily)
                                         │
                                    ┌────┴────┐
                                    │ pg_nft_db │  ← single source of truth
                                    └────┬────┘
                    ┌────────────────────┼────────────────────┐
                    │                    │                    │
              nft-data-api        marketplace          wallet/mining
```

**Result:** 372K+ NFT records, always in sync with chain state. Any Pentagon service can query one database or one API for ownership data.

---

## Quick Start — I Just Want to Look Up NFT Owners

### Option 1: HTTP API (Recommended)

The NFT Data API runs on `nft-data.pentagon.games` (or internal `pg-oracle:8009`).

**Public (no auth, 10 req/min):**
```bash
# Get NFTs owned by a wallet
curl "https://nft-data.pentagon.games/api/v1/nft/owner/0x37224cFD71347Da6f097c94B8649f9C552f803c9"

# Get specific NFT details
curl "https://nft-data.pentagon.games/api/v1/nft/contract/0x8F83c6122Dd4d275B53a7846B3D3dB29Cca1e698/1"

# Get collection info
curl "https://nft-data.pentagon.games/api/v1/nft/collection/EtherFantasy"
```

**Authenticated (100 req/min with app key):**
```bash
curl -H "X-PG-App-Key: pk_live_your_key_here" \
  "https://nft-data.pentagon.games/api/v1/nft/owner/0x..."
```

Get an app key at `cli.pentagon.games` (Admin Panel → App Keys). The same key works for all Pentagon APIs (login, nft-data, etc).

### Option 2: Direct Database (Internal Services Only)

For backend services on the same VPC:

```
Host: 172.31.46.190 (AWS VPC internal)
Port: 5432
Database: pg_nft_db
User: cron_cg_nft  (read-only recommended for consumers)
```

Key tables:
- `nfts` — NFT records with `owner`, `token_id`, `name`, `image`, `hashrate`
- `nft_contract` — Contract registry (49 active across 14 chains)
- `nft_owner` — Additional owner tracking
- `nft_transfer_history` — Transfer event log

```sql
-- Who owns token #42 of EtherFantasy?
SELECT n.owner, n.name, n.image
FROM nfts n
JOIN nft_contract nc ON n.asset_contract_id = nc.id
WHERE nc.name = 'EtherFantasy Genesis'
  AND n.token_id = '42';

-- All NFTs owned by a wallet
SELECT n.token_id, n.name, n.image, nc.name AS collection
FROM nfts n
JOIN nft_contract nc ON n.asset_contract_id = nc.id
WHERE LOWER(n.owner) = LOWER('0x37224cFD71347Da6f097c94B8649f9C552f803c9');
```

---

## Supported Chains & Contracts

| Chain | Chain ID | Contracts | Sync Method |
|-------|----------|-----------|-------------|
| Ethereum | 1 | 25 | Scanner (every 2-30 min depending on contract) |
| Pentagon | 3344 | 13 | **Real-time listener + scanner + verifier** |
| BSC | 56 | 2 | Scanner |
| Polygon | 137 | 1 | Scanner |
| Core | 1116 | 2 | Scanner |
| Arbitrum | 42161 | 1 | Scanner |
| SKALE Nebula | 1482601649 | 2 | Scanner |
| Monad | 143 | 1 | Scanner |
| Oasys | 248 | 1 | Scanner |
| TRON | 728126428 | 1 | JSON-RPC scanner |

Pentagon chain gets all three layers — real-time detection within 3 seconds, scanner backup every 2-6 minutes, and verifier checks hourly.

---

## Adding a New Collection

### Step 1: Register the Contract

Insert into `nft_contract` table in `pg_nft_db`:

```sql
INSERT INTO nft_contract (
    name, address, "chainId", contract_type, sync_type,
    metadata_url, is_active, logo, display_order
) VALUES (
    'My New Collection',           -- display name
    '0xYOUR_CONTRACT_ADDRESS',     -- contract address (checksummed)
    3344,                          -- chain ID
    'ERC721',                      -- ERC721, ERC1155, TRC721, or TON
    'custom',                      -- sync_type: api, api-json, custom, pegnames, rugpull, gemry, etherfantasy
    'https://your-metadata-api/',  -- optional: metadata URL (token ID appended)
    true,                          -- is_active
    'https://your-logo-url.png',   -- collection logo
    100                            -- display order
);
```

**Sync types:**
- `api` — Fetches metadata from OpenSea API
- `api-json` — Fetches metadata from `{metadata_url}{tokenId}.json`
- `custom` — Uses built-in custom data handlers (for PG IP like BCSH, PentaPets, etc.)
- `etherfantasy`, `gemry`, `rugpull`, `pegnames` — Game-specific sync handlers

### Step 2: The Scanner Picks It Up

The scheduler auto-detects new active contracts. Within 5 minutes:
1. Creates a sync job entry in `nft_owner_sync_jobs`
2. Starts scanning from block 0 (or specified start block)
3. Processes Transfer events in batches of up to 999 blocks
4. Populates `nfts` table with owner, metadata, images

### Step 3: (Optional) Add Real-Time Listener

If it's a Pentagon chain contract, the listener will automatically pick it up on next restart. For other chains, listener support can be extended in `master/listener.py`.

### Step 4: (Optional) Add Custom Metadata Handler

If the collection needs special metadata handling (e.g., custom image URLs, game-specific traits):

1. Add a handler in `master/custom_nft_data.py`
2. Return a dict with: `name`, `image`, `description`, `hashrate`, `traits`, `tokenload`, `catch`
3. Set `sync_type = 'custom'` in the contract registration

---

## API Reference

### Endpoints

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/api/v1/nft/owner/<address>` | GET | Optional | Get all NFTs owned by wallet |
| `/api/v1/nft/contract/<address>/<tokenId>` | GET | Optional | Get specific NFT details |
| `/api/v1/nft/collection/<name>` | GET | Optional | Collection metadata + stats |
| `/api/v1/nft/collections` | GET | Optional | List all collections |
| `/api/v1/nft/search?q=<query>` | GET | Optional | Search NFTs by name |
| `/api/v1/nft/hashrate/<address>` | GET | Optional | Mining hashrate for wallet |

### Authentication

**No key (public):** 10 requests per minute per IP

**With `X-PG-App-Key` header:** Per-app rate limit (default 100/min, configurable per key)

```bash
# Example: authenticated request
curl -H "X-PG-App-Key: pk_live_36f496a8795c9d29b6ad037fe5905084" \
  "https://nft-data.pentagon.games/api/v1/nft/owner/0x..."
```

**Getting a key:**
1. Go to `cli.pentagon.games`
2. Admin Panel → App Keys
3. Create new key with `nft-data:read` scope
4. Same key works for login API, nft-data API, and future Pentagon services

### Response Format

```json
{
  "status": "ok",
  "data": {
    "nfts": [
      {
        "token_id": "42",
        "name": "EtherFantasy Hero #42",
        "collection": "EtherFantasy Genesis",
        "chain_id": 3344,
        "owner": "0x37224cFD71347Da6f097c94B8649f9C552f803c9",
        "image": "https://api.etherfantasy.com/images/...",
        "hashrate": 75,
        "traits": [
          {"trait_type": "Class", "value": "Warrior"},
          {"trait_type": "Level", "value": "5"}
        ]
      }
    ],
    "total": 1
  }
}
```

---

## Internal Health Dashboard

For operators and internal tools, the master service exposes health endpoints on port 9011 (VPC only):

```bash
# Overall system stats
curl http://localhost:9011/api/internal/sync/stats

# Per-contract sync health (block lag, layer status)
curl http://localhost:9011/api/internal/sync/health

# Gaps (transfers caught by scanner but missed by listener)
curl http://localhost:9011/api/internal/sync/gaps

# Ownership verification mismatches
curl http://localhost:9011/api/internal/sync/mismatches

# Trigger manual verification for a contract
curl -X POST http://localhost:9011/api/internal/sync/verify/42

# Force resync from a specific block
curl -X POST "http://localhost:9011/api/internal/sync/resync/42?from_block=1000000"
```

---

## Architecture Details

### How the Three Layers Work Together

**Layer 1 — Listener (Real-Time):**
- Polls Pentagon RPC every 3 seconds for new Transfer events
- Updates `nfts.owner` immediately
- Logs to `nft_transfer_log` with `source='listener'`
- Saves checkpoint in `listener_checkpoint`
- Fast but fragile — can miss events during RPC downtime

**Layer 2 — Scanner (Block-Range):**
- Runs on APScheduler intervals (1 min to 1.5 hours depending on contract activity)
- Fetches `eth_getLogs` for Transfer topic in configurable block ranges
- Updates `nfts.owner` and `nft_transfer_history`
- Logs to `nft_transfer_log` with `source='scanner'`
- Reliable — resumes from last processed block, never misses blocks

**Layer 3 — Verifier:**
- Hourly: samples 10% of NFTs, calls `ownerOf()` on-chain
- Daily (3 AM UTC): full sweep of all Pentagon chain NFTs
- If DB owner ≠ chain owner → auto-corrects + logs mismatch
- Logs to `ownership_verification` and `nft_transfer_log` with `source='verifier'`
- Ground truth — the chain is always right

**Triangulation rule:** If scanner and listener disagree, scanner wins. If anything disagrees with chain (`ownerOf()`), chain wins.

### Database Schema (New Tables)

```sql
-- Real-time listener tracking
CREATE TABLE listener_checkpoint (
    id SERIAL PRIMARY KEY,
    contract_id INTEGER REFERENCES nft_contract(id),
    chain_id INTEGER NOT NULL,
    last_block BIGINT NOT NULL DEFAULT 0,
    last_event_at TIMESTAMP WITH TIME ZONE,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(contract_id)
);

-- Unified transfer log (all layers)
CREATE TABLE nft_transfer_log (
    id BIGSERIAL PRIMARY KEY,
    contract_id INTEGER REFERENCES nft_contract(id),
    token_id VARCHAR(100) NOT NULL,
    from_address VARCHAR(100) NOT NULL,
    to_address VARCHAR(100) NOT NULL,
    tx_hash VARCHAR(100),
    block_number BIGINT,
    block_timestamp TIMESTAMP WITH TIME ZONE,
    source VARCHAR(20) NOT NULL,  -- 'listener', 'scanner', 'verifier'
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Verifier spot-check results
CREATE TABLE ownership_verification (
    id BIGSERIAL PRIMARY KEY,
    contract_id INTEGER REFERENCES nft_contract(id),
    token_id VARCHAR(100) NOT NULL,
    db_owner VARCHAR(100),
    chain_owner VARCHAR(100),
    is_match BOOLEAN NOT NULL,
    verified_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Per-contract health dashboard
CREATE TABLE sync_health (
    id SERIAL PRIMARY KEY,
    contract_id INTEGER REFERENCES nft_contract(id) UNIQUE,
    listener_status VARCHAR(20) DEFAULT 'stopped',
    listener_last_event TIMESTAMP WITH TIME ZONE,
    scanner_last_block BIGINT DEFAULT 0,
    scanner_last_ran TIMESTAMP WITH TIME ZONE,
    verifier_last_ran TIMESTAMP WITH TIME ZONE,
    verifier_mismatches INTEGER DEFAULT 0,
    chain_head_block BIGINT DEFAULT 0,
    block_lag INTEGER DEFAULT 0,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## Infrastructure

| Component | Server | Port | Notes |
|-----------|--------|------|-------|
| nft-ownership-master | pg-crons (18.143.147.51) | 9011 | All 3 layers + health API |
| nft-sync-cron (legacy) | pg-crons (18.143.147.51) | 8009 | Original scanner (still running, will be deprecated) |
| nft-data-api | pg-oracle (13.212.88.57) | 8009 | Public REST API |
| pg_nft_db | pg-db AWS (172.31.46.190) | 5432 | PostgreSQL database |

### Deployment

The master service runs under supervisor on pg-crons:

```bash
# Check status
sudo supervisorctl status nft-ownership-master

# Restart
sudo supervisorctl restart nft-ownership-master

# View logs
tail -f /var/log/nft-ownership-master/error.log

# Pull updates
cd /var/www/nft-sync/nft-ownership-master
git pull origin main
sudo supervisorctl restart nft-ownership-master
```

---

## For External Projects

If you're building on Pentagon Chain and want to integrate NFT ownership data:

1. **Get an app key** at `cli.pentagon.games`
2. **Use the API** at `nft-data.pentagon.games` — start with `/api/v1/nft/owner/<wallet>`
3. **Request your contract** be added — reach out via Pentagon Games Discord
4. **Rate limits:** 100 req/min with app key, sufficient for most integrations

### Integration Patterns

**Game Backend — Check if player owns an NFT:**
```python
import requests

def player_owns_nft(wallet, collection_contract, token_id):
    resp = requests.get(
        f"https://nft-data.pentagon.games/api/v1/nft/contract/{collection_contract}/{token_id}",
        headers={"X-PG-App-Key": "pk_live_your_key"}
    )
    data = resp.json()
    return data["data"]["owner"].lower() == wallet.lower()
```

**Wallet App — Show user's NFTs:**
```javascript
const response = await fetch(
  `https://nft-data.pentagon.games/api/v1/nft/owner/${walletAddress}`,
  { headers: { 'X-PG-App-Key': 'pk_live_your_key' } }
);
const { data } = await response.json();
// data.nfts = [{token_id, name, image, collection, chain_id, ...}]
```

**Discord Bot — Verify NFT ownership:**
```python
# Check if Discord user's linked wallet holds a specific collection
nfts = get_nfts_for_wallet(user_wallet)
has_collection = any(n["collection"] == "Gunnies PFP" for n in nfts)
if has_collection:
    await member.add_roles(holder_role)
```

---

## Repos

| Repo | What | Link |
|------|------|------|
| infra-nft-ownership-master | Three-layer sync engine (this) | `blockchainsuperheroes/infra-nft-ownership-master` |
| infra-nft-sync | Original scanner cron (legacy) | `blockchainsuperheroes/infra-nft-sync` |
| cg-nft-api | Old Laravel NFT API (deprecated) | `blockchainsuperheroes/cg-nft-api` |

---

## Troubleshooting

**Scanner shows 0 events for a contract?**
- Check `nft_owner_sync_jobs` — is the block number recent?
- The contract may just have no recent transfers
- Try `/api/internal/sync/resync/<id>?from_block=0` to resync from genesis

**Listener not detecting transfers?**
- Check `/api/internal/sync/health` — is listener_status "running"?
- Pentagon RPC might be down — check `curl https://rpc.pentagon.games`
- Restart: `sudo supervisorctl restart nft-ownership-master`

**Ownership mismatch?**
- Check `/api/internal/sync/mismatches` for recent corrections
- Trigger manual verify: `POST /api/internal/sync/verify/<contract_id>?full=true`
- The verifier auto-corrects — chain state always wins

**OOM on pg-crons?**
- The server has 3.7GB RAM with multiple services
- Scanner jobs are staggered (3s apart) to avoid memory spikes
- If OOM persists, increase stagger interval in `master/scheduler.py`

---

*Built by Cerise02 💜 — part of the Pentagon Games AI agent team.*

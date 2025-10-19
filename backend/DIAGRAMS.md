# Flexanon Architecture & Flow Diagrams

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         FRONTEND (Next.js)                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ Landing  │  │  Create  │  │  Public  │  │  Dashboard   │   │
│  │   Page   │  │   Flow   │  │ Profile  │  │   (Owner)    │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ↕ HTTP/JSON
┌─────────────────────────────────────────────────────────────────┐
│                      BACKEND (Express)                          │
│  ┌────────────────┐           ┌────────────────┐               │
│  │  Auth Routes   │           │  Share Routes  │               │
│  │  /api/auth/*   │           │  /api/share/*  │               │
│  └────────────────┘           └────────────────┘               │
│           ↕                            ↕                        │
│  ┌────────────────────────────────────────────────┐            │
│  │              Services Layer                     │            │
│  │  • Portfolio Service (SMT builder)             │            │
│  │  • Share Token Service (DB manager)            │            │
│  └────────────────────────────────────────────────┘            │
│           ↕                            ↕                        │
│  ┌──────────────┐           ┌─────────────────┐               │
│  │   Crypto     │           │   Zerion API    │               │
│  │   • SMT      │           │   Client        │               │
│  │   • Hashing  │           └─────────────────┘               │
│  │   • Signing  │                    ↕                         │
│  └──────────────┘                    │                         │
│           ↕                          │                         │
│  ┌──────────────────────────────────┘                         │
│  │           Database (PostgreSQL)                             │
│  │  • share_tokens                                             │
│  │  • sessions                                                 │
│  │  • wallet_nonces                                            │
│  └─────────────────────────────────────────────────────────────┘
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↕
                     External Services
                ┌─────────────────────────┐
                │    Zerion API           │
                │  (Portfolio Data)       │
                └─────────────────────────┘
```

## Create Share Link Flow

```
USER                    FRONTEND                BACKEND                ZERION
 │                         │                       │                      │
 │  1. Connect Wallet      │                       │                      │
 ├────────────────────────>│                       │                      │
 │                         │                       │                      │
 │  2. Sign Message        │                       │                      │
 │    (prove ownership)    │                       │                      │
 ├────────────────────────>│                       │                      │
 │                         │  3. POST /auth/verify │                      │
 │                         ├──────────────────────>│                      │
 │                         │    (signature)        │                      │
 │                         │                       │  4. Verify signature │
 │                         │                       │     ✓ Valid          │
 │                         │  5. Session Token     │                      │
 │                         │<──────────────────────┤                      │
 │                         │                       │                      │
 │  6. Set Privacy Options │                       │                      │
 │    [x] Total Value      │                       │                      │
 │    [x] PnL              │                       │                      │
 │    [x] Top 3 Assets     │                       │                      │
 │    [ ] Wallet Address   │                       │                      │
 ├────────────────────────>│                       │                      │
 │                         │                       │                      │
 │  7. Click "Generate"    │  8. POST /share/gen  │                      │
 ├────────────────────────>├──────────────────────>│  9. GET portfolio   │
 │                         │   + session_token     ├────────────────────>│
 │                         │   + preferences       │                      │
 │                         │                       │  10. Portfolio data  │
 │                         │                       │<─────────────────────┤
 │                         │                       │                      │
 │                         │                       │  11. Build SMT       │
 │                         │                       │      ┌──────────┐    │
 │                         │                       │      │ Leaves:  │    │
 │                         │                       │      │ • wallet │    │
 │                         │                       │      │ • value  │    │
 │                         │                       │      │ • PnL    │    │
 │                         │                       │      │ • SOL    │    │
 │                         │                       │      │ • ETH    │    │
 │                         │                       │      └──────────┘    │
 │                         │                       │           ↓          │
 │                         │                       │    12. Select reveal │
 │                         │                       │       • value ✓      │
 │                         │                       │       • PnL ✓        │
 │                         │                       │       • SOL ✓        │
 │                         │                       │       • wallet ✗     │
 │                         │                       │           ↓          │
 │                         │                       │    13. Gen proofs    │
 │                         │                       │        for revealed  │
 │                         │                       │           ↓          │
 │                         │                       │    14. Save to DB    │
 │                         │                       │        + merkle_root │
 │                         │                       │        + proofs      │
 │                         │                       │        + token_id    │
 │                         │                       │                      │
 │                         │  15. Share URL        │                      │
 │                         │<──────────────────────┤                      │
 │  16. Share URL          │  stealth.app/s/abc123 │                      │
 │<────────────────────────┤                       │                      │
 │                         │                       │                      │
```

## Public View & Verification Flow

```
PUBLIC USER             FRONTEND                BACKEND
 │                         │                       │
 │  1. Open share link     │                       │
 │    /s/abc123            │                       │
 ├────────────────────────>│                       │
 │                         │  2. GET /resolve      │
 │                         ├──────────────────────>│
 │                         │    ?token=abc123      │
 │                         │                       │  3. Read from DB
 │                         │                       │     • Check expired
 │                         │                       │     • Check revoked
 │                         │                       │     • Get revealed leaves
 │                         │                       │     • Get proofs
 │                         │                       │
 │                         │  4. Public data       │
 │                         │<──────────────────────┤
 │                         │    {                  │
 │                         │      total: "$50k"    │
 │                         │      pnl: "+143%"     │
 │                         │      assets: [...]    │
 │                         │      wallet: "hidden" │
 │                         │      proofs: [...]    │
 │                         │    }                  │
 │                         │                       │
 │  5. Display Profile     │                       │
 │<────────────────────────┤                       │
 │    ┌──────────────────┐ │                       │
 │    │ Total: $50,000   │ │                       │
 │    │ PnL: +143.25%    │ │                       │
 │    │ Top Assets:      │ │                       │
 │    │  • SOL: 10.5     │ │                       │
 │    │  • ETH: 2.3      │ │                       │
 │    │                  │ │                       │
 │    │ Wallet: [Hidden] │ │                       │
 │    │                  │ │                       │
 │    │ [Verify] button  │ │                       │
 │    └──────────────────┘ │                       │
 │                         │                       │
 │  6. Click "Verify"      │                       │
 ├────────────────────────>│                       │
 │                         │  7. For each revealed │
 │                         │     leaf, POST /verify│
 │                         ├──────────────────────>│
 │                         │    {                  │
 │                         │      merkle_root,     │
 │                         │      leaf,            │
 │                         │      proof            │
 │                         │    }                  │
 │                         │                       │  8. Verify proof
 │                         │                       │     hash(leaf)
 │                         │                       │       + siblings
 │                         │                       │       = root? ✓
 │                         │                       │
 │                         │  9. Valid ✓           │
 │                         │<──────────────────────┤
 │                         │                       │
 │  10. Show verification  │                       │
 │<────────────────────────┤                       │
 │    ✅ All data verified │                       │
 │    against merkle root  │                       │
 │                         │                       │
```

## Sparse Merkle Tree Structure

```
                           ROOT (committed publicly)
                         /                           \
                    HASH(L+R)                      HASH(L+R)
                   /          \                   /          \
              HASH(L+R)    HASH(L+R)         HASH(L+R)    HASH(L+R)
             /      \      /      \          /      \      /      \
         Leaf1   Leaf2  Leaf3  Leaf4     Leaf5   Leaf6  Leaf7  Leaf8
         -----   -----  -----  -----     -----   -----  -----  -----
       wallet   total    PnL   chain      SOL     ETH    BTC   USDC
       HIDDEN REVEALED REVEALED  ✓     REVEALED HIDDEN HIDDEN REVEALED

       🔒       💰      📈       🌐       ☀️      💎      🟠      💵
```

### Verification Example:

To verify "total_value" leaf:
```
1. Start with: hash(total_value)
2. Get sibling: hash(wallet)
3. Compute parent: hash(wallet + total)
4. Get uncle: hash(PnL + chain)
5. Compute grandparent: hash(parent + uncle)
6. Continue up the tree...
7. Final result = ROOT? ✅ VALID
```

If someone tries to fake the value:
```
1. Start with: hash(fake_value)
2. Get sibling: hash(wallet)
3. Compute parent: hash(wallet + fake)  ← Different!
4. This will never match the ROOT ❌ INVALID
```

## Privacy Levels

```
┌────────────────────────────────────────────────────────────┐
│                    FULL PRIVACY (Paranoid Mode)            │
│  Revealed: Chain only                                      │
│  Hidden: Everything else                                   │
│  Privacy Score: 95%                                        │
│                                                            │
│  Public sees: "Someone on Solana has a portfolio"         │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                    BALANCED PRIVACY (Default)              │
│  Revealed: Total value, PnL, Top 3 assets                 │
│  Hidden: Wallet address, other assets                     │
│  Privacy Score: 70%                                        │
│                                                            │
│  Public sees: "$50k portfolio, +143%, holds SOL/ETH/USDC" │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                    MINIMAL PRIVACY (Flex Mode)             │
│  Revealed: Everything including wallet                     │
│  Hidden: Nothing                                           │
│  Privacy Score: 0%                                         │
│                                                            │
│  Public sees: Full portfolio + wallet address              │
└────────────────────────────────────────────────────────────┘
```

## Database Relationships

```
┌─────────────────────┐
│   share_tokens      │
├─────────────────────┤
│ token_id (PK)       │◄─────┐
│ owner_hash          │      │
│ merkle_root         │      │
│ revealed_leaves     │      │
│ proof_data          │      │
│ expires_at          │      │
│ revoked             │      │
└─────────────────────┘      │
                             │
                             │ Foreign Key
                             │ (via token_id)
                             │
                    ┌────────┴──────────┐
                    │   share_views     │
                    ├───────────────────┤
                    │ id (PK)           │
                    │ token_id (FK)     │
                    │ viewer_ip         │
                    │ user_agent        │
                    │ viewed_at         │
                    └───────────────────┘

┌─────────────────────┐
│   sessions          │
├─────────────────────┤
│ session_id (PK)     │
│ wallet_address      │
│ expires_at          │
└─────────────────────┘

┌─────────────────────┐
│   wallet_nonces     │
├─────────────────────┤
│ wallet_address (PK) │
│ nonce               │
│ expires_at          │
└─────────────────────┘
```

## API Request/Response Examples

### Generate Share Link

**Request:**
```http
POST /api/share/generate
Authorization: Bearer abc123session
Content-Type: application/json

{
  "chain": "solana",
  "reveal_preferences": {
    "show_total_value": true,
    "show_pnl": true,
    "show_top_assets": true,
    "top_assets_count": 3,
    "show_wallet_address": false
  },
  "expiry_days": 30
}
```

**Response:**
```json
{
  "success": true,
  "share_url": "https://stealth.app/s/x7k2p9m1",
  "token_id": "x7k2p9m1",
  "merkle_root": "0x7f3e8a9b...",
  "revealed_count": 5,
  "hidden_count": 12,
  "privacy_score": 71,
  "expires_at": "2025-02-15T10:30:00Z"
}
```

### Resolve Public Profile

**Request:**
```http
GET /api/share/resolve?token=x7k2p9m1
```

**Response:**
```json
{
  "token_id": "x7k2p9m1",
  "merkle_root": "0x7f3e8a9b...",
  "committed_at": "2025-01-15T10:30:00Z",
  "revealed_data": {
    "total_value": "$50,000.00",
    "pnl_percentage": "+143.25%",
    "top_assets": [
      {
        "symbol": "SOL",
        "amount": "10.5",
        "value_usd": "2,100.00"
      },
      {
        "symbol": "ETH",
        "amount": "2.3",
        "value_usd": "5,750.00"
      },
      {
        "symbol": "USDC",
        "amount": "15000",
        "value_usd": "15,000.00"
      }
    ],
    "snapshot_time": "2025-01-15T10:30:00Z"
  },
  "proof_data": [...],
  "privacy": {
    "wallet_address": "hidden",
    "total_assets_count": 17,
    "revealed_count": 5
  }
}
```

## Technology Stack

```
Backend Stack:
├── Runtime: Node.js 18+
├── Framework: Express
├── Language: TypeScript
├── Database: PostgreSQL
├── Crypto: Custom SMT implementation
├── APIs: Zerion (portfolio data)
└── Auth: Wallet signatures (EVM + Solana)

Dependencies:
├── express - Web framework
├── pg - PostgreSQL client
├── axios - HTTP client
├── @ethersproject/wallet - EVM signatures
├── tweetnacl - Solana signatures
├── bs58 - Base58 encoding
├── dotenv - Environment config
└── tsx - TypeScript runner
```

## Security Considerations

```
✅ Wallet addresses hashed before storage
✅ Session tokens expire after 24 hours
✅ Nonces expire after 5 minutes
✅ Share links can be revoked
✅ Share links can expire
✅ Merkle proofs prevent data tampering
✅ CORS protection
✅ Rate limiting (TODO for production)
✅ Input validation
✅ SQL injection protection (parameterized queries)
```

---

**This completes the backend implementation!** 🎉

All diagrams show the complete flow from wallet connection to public verification.

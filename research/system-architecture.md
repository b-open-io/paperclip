# System Architecture: ClawNet Identity & Authorship

Visual reference for how AIP signatures, BAP identities, and profiles interconnect in ClawNet.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CLAWNET SKILL SYSTEM                             │
└─────────────────────────────────────────────────────────────────────────┘

┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   User CLI   │         │ BRC-100      │         │  Sigma Auth  │
│   (Signer)   │         │  Wallet      │         │  (Session)   │
│              │         │  (Identity)  │         │              │
└──────┬───────┘         └──────┬───────┘         └──────┬───────┘
       │                        │                        │
       │ privateKey             │ identityKey            │ bearerToken
       │ (signing key)          │ (identity key)         │
       │                        │                        │
       └─────────────┬──────────┴──────────┬─────────────┘
                     │                     │
              ┌──────▼─────────────────────▼──────┐
              │  PUBLISH SKILL TRANSACTION         │
              │  ─────────────────────────────────│
              │  • AIP.sign(data, privateKey)     │
              │  • Outputs: signerAddress         │
              │  • Send to server with payload    │
              └──────┬──────────────────────┬─────┘
                     │                      │
              ┌──────▼─────────┐  ┌────────▼──────────────┐
              │ aipSignature   │  │ signerAddress        │
              │ (base64)       │  │ (Bitcoin address)    │
              │ (stored)       │  │ (stored, not used)   │
              └────────────────┘  └─────────────────────┘

                     ▼

        ┌──────────────────────────────────┐
        │ POST /api/v1/skills               │
        │ ClawNet Server                    │
        ├──────────────────────────────────┤
        │ 1. Extract auth headers/token    │
        │ 2. Verify authentication         │
        │ 3. Resolve auth.bapId            │
        │    └─ From BRC-31: pubkeyToBapId │
        │    └─ From Sigma: session data   │
        │ 4. Assign authorBapId = auth.bapId
        │ 5. Store in Convex skillVersions │
        └──────────────────────────────────┘

                     ▼

        ┌──────────────────────────────────┐
        │ Convex Database                  │
        ├──────────────────────────────────┤
        │ skillVersions {                  │
        │   authorBapId,                   │
        │   aipSignature,                  │
        │   signerAddress,                 │
        │   ...                            │
        │ }                                │
        │                                  │
        │ profiles {                       │
        │   bapId,                         │
        │   identity: {                    │
        │     alternateName,               │
        │     image,                       │
        │     ...                          │
        │   }                              │
        │ }                                │
        └──────────────────────────────────┘

                     ▼

        ┌──────────────────────────────────┐
        │ Skill Detail Page Display         │
        ├──────────────────────────────────┤
        │ 1. Load skillVersions.authorBapId│
        │ 2. Lookup in profiles cache      │
        │    └─ Cache HIT: use cached data │
        │    └─ Cache MISS: fetch Sigma    │
        │ 3. Display author profile        │
        │    ├─ Name (alternateName)       │
        │    ├─ Avatar (image)             │
        │    └─ Bio (description)          │
        └──────────────────────────────────┘

                     ▼

        ┌──────────────────────────────────┐
        │ Sigma API (External)             │
        │ sigma.1sat.app/1sat/bap/...      │
        │                                  │
        │ GET /profile/{bapId}             │
        │ Returns: SigmaProfile            │
        └──────────────────────────────────┘
```

---

## Key Relationships

```
                    INDEPENDENT IDENTITY STREAMS

┌─────────────────────────────────────────────────────────────┐
│ STREAM 1: TRANSACTION SIGNING (AIP)                         │
│                                                             │
│ User's Signing Key                                         │
│         │                                                  │
│         ├─ AIP.sign(skillData)                             │
│         │                                                  │
│         └─ signerAddress (Bitcoin address)                 │
│            │                                               │
│            └─ Stored in skillVersions                      │
│               └─ PURPOSE: Cryptographic proof of sig       │
│               └─ USE: Optional verification/audit          │
│               └─ NOT USED for: Identifying author          │
└─────────────────────────────────────────────────────────────┘

                          ⚠️ DIFFERENT KEYS

┌─────────────────────────────────────────────────────────────┐
│ STREAM 2: USER AUTHENTICATION (BRC-31 / Sigma)              │
│                                                             │
│ User's Identity Key                                        │
│         │                                                  │
│         ├─ BRC-31 or Sigma validation                      │
│         │                                                  │
│         ├─ pubkeyToBapId(identityKey)                      │
│         │  OR                                              │
│         ├─ sigma.session.activeOrganizationId              │
│         │                                                  │
│         └─ authorBapId (BAP identity)                      │
│            │                                               │
│            ├─ Stored in skillVersions                      │
│            │   └─ PRIMARY IDENTITY SOURCE                  │
│            │                                               │
│            ├─ Stored in profiles table                     │
│            │   └─ Cache of Sigma profile data              │
│            │                                               │
│            └─ Resolved via Sigma API                       │
│                └─ Fetch author name, avatar, bio           │
└─────────────────────────────────────────────────────────────┘
```

---

## Authentication Context Derivation

```
OPTION A: BRC-31 (Authrite)
─────────────────────────────────────────────────────────────

Client Request:
├─ x-bsv-auth-identity-key: 03abc123def... (66 hex chars)
├─ x-bsv-auth-signature: base64sig
└─ x-bsv-auth-your-nonce: serverNonce

Server Processing:
├─ Extract identityKey from header
├─ Lookup session by serverNonce in Convex
├─ Verify signature using BRC-43 derived keys
├─ Extract identityKey as verified public key
└─ pubkeyToBapId(identityKey) → BAP ID

Result:
└─ AuthContext { bapId, pubkey }


OPTION B: Sigma Auth
─────────────────────────────────────────────────────────────

Client Request:
└─ Authorization: Bearer sigma_token_here

Server Processing:
├─ Extract bearer token
├─ Call auth.sigmaidentity.com/api/auth/get-session
├─ Validate token, extract user data
├─ Prefer session.activeOrganizationId (org context)
│  OR user.pubkey (personal context)
└─ Use as BAP ID

Result:
└─ AuthContext { bapId, pubkey }
```

---

## Data Flow: From User to Display

```
STEP 1: CLI - Signing
────────────────────

User $ clawnet publish --skill SKILL.md
           │
           ├─ Read user's private key from ~/.clawnet/keys
           │
           ├─ Read SKILL.md content
           │
           └─ buildSkillOpReturn(fields, privateKey)
              │
              ├─ AIP.sign(B|MAP|protocols, PrivateKeySigner(privateKey))
              │
              └─ Return {
                   signerAddress,      ← from AIP signature
                   aipSignature,       ← base64 signature
                   opReturnHex,        ← full OP_RETURN script
                   lockingScript
                 }


STEP 2: Network - Publish
──────────────────────────

POST /api/v1/skills
├─ Headers: BRC-31 auth headers (or Bearer token)
├─ Body: { slug, name, files, aipSignature, signerAddress, ... }
│
└─ Server receives request
   ├─ requireAuth(request)
   │  │
   │  ├─ Check BRC-31 headers first
   │  │  └─ Extract and verify identityKey
   │  │
   │  ├─ OR check Bearer token
   │  │  └─ Validate with Sigma Auth service
   │  │
   │  └─ Return: AuthContext { bapId, pubkey }
   │
   ├─ Extract skill payload
   │  ├─ aipSignature (stored as-is)
   │  ├─ signerAddress (stored as-is)
   │  └─ Other skill metadata
   │
   └─ Call Convex mutation: publishVersion
      ├─ authorBapId = auth.bapId  ← KEY ASSIGNMENT
      ├─ Save skillVersions record
      └─ Return success


STEP 3: Database - Storage
───────────────────────────

Convex skillVersions:
├─ slug: "my-skill"
├─ version: "0.1.0"
├─ authorBapId: "03xyz789..."      ← FROM AUTH.BAPID
├─ aipSignature: "base64abc..."    ← FROM PAYLOAD (not used)
├─ signerAddress: "1ABC123..."     ← FROM PAYLOAD (not used)
└─ ...


STEP 4: Display - Rendering
────────────────────────────

GET /skills/my-skill [Browser]
├─ Convex query: api.skills.getBySlug("my-skill")
├─ Load skillVersions with authorBapId
│
├─ Lookup profile for authorBapId:
│  ├─ SELECT FROM profiles WHERE bapId = "03xyz789..."
│  │
│  ├─ Cache HIT:
│  │  └─ Use cached { alternateName, image, description }
│  │
│  └─ Cache MISS:
│     ├─ Fetch from Sigma: GET /bap/profile/03xyz789...
│     ├─ Parse response: { alternateName, image, ... }
│     └─ Store in Convex profiles table
│
├─ Resolve image URLs (if inscribed on-chain)
│  └─ /txId_vout → https://ordfs.network/txId_vout
│
└─ Render author section:
   ├─ Avatar: <img src="https://ordfs.network/..." />
   ├─ Name: alternateName || truncated(bapId)
   ├─ Bio: description
   └─ Link: /profile/{authorBapId}
```

---

## Convex Data Model

```
skillVersions
─────────────

┌─ slug: string
├─ version: string
├─ authorBapId: string              ← PRIMARY IDENTITY
├─ aipSignature: optional<string>   ← For audit (not used)
├─ signerAddress: optional<string>  ← For audit (not used)
├─ opReturnHex: optional<string>
├─ content: string                  ← SKILL.md
├─ files: array<{ path, content }>
├─ publishedAt: number
├─ onChain: boolean
├─ txId: optional<string>           ← On-chain transaction
└─ ...


profiles (Cache)
────────────────

┌─ bapId: string                    ← Foreign key to skillVersions.authorBapId
├─ identity: object {
│  ├─ alternateName: optional<string>
│  ├─ description: optional<string>
│  ├─ image: optional<string>       ← Profile avatar
│  ├─ paymail: optional<string>
│  ├─ paymentAddress: optional<string>
│  ├─ type: optional<string>
│  └─ ...
├─ lastSeen: number                 ← Cache timestamp
└─ ...


Index: profiles.by_bapId
───────────────────────

Fast lookup: WHERE bapId = ?
Used for profile display on skill detail pages
```

---

## External Service Dependencies

```
┌────────────────────────────────────────────────────────────┐
│ Sigma Identity Service                                     │
│ https://sigma.1sat.app/1sat/bap                            │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ API Endpoint:                                             │
│ GET /profile/{bapId}                                      │
│                                                            │
│ Input:  bapId (path parameter)                            │
│ Output: SigmaProfile {                                    │
│           "@context": string,                             │
│           "@type": "Person",                              │
│           "alternateName": string,                        │
│           "image": string,          (relative path)       │
│           "banner": string,                               │
│           "description": string,                          │
│           "paymail": string,                              │
│           "url": string,                                  │
│           ...                                             │
│         }                                                 │
│                                                            │
│ Error: 404 if no profile found (gracefully handled)       │
│                                                            │
│ Caching:                                                  │
│ ├─ Convex profiles table stores results                  │
│ ├─ ISR revalidation: 3600s (1 hour)                       │
│ └─ Reduce redundant fetches                              │
│                                                            │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│ Sigma Auth Service (For Auth)                              │
│ https://auth.sigmaidentity.com                             │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ Endpoint: /api/auth/get-session                           │
│ Headers: Authorization: Bearer <token>                     │
│                                                            │
│ Returns:                                                  │
│ {                                                         │
│   "session": {                                            │
│     "activeOrganizationId": "03abc..."  (optional)        │
│   },                                                      │
│   "user": {                                               │
│     "pubkey": "03xyz...",                                 │
│     "sub": "user_id",                                     │
│     ...                                                   │
│   }                                                       │
│ }                                                         │
│                                                            │
│ Usage: Resolve BAP ID from token for auth context         │
│                                                            │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│ ORDFS Gateway                                              │
│ https://ordfs.network                                      │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ Purpose: Fetch on-chain inscribed media                   │
│                                                            │
│ Endpoint: /{txId}_{vout}                                  │
│                                                            │
│ Usage:                                                    │
│ └─ Resolve image URLs from Sigma profiles                │
│    └─ image: "/txId_0" → URL: "ordfs.network/txId_0"    │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Security Model

```
AUTHENTICATION & AUTHORIZATION
───────────────────────────────

┌─ BRC-31 (Authrite)
│  ├─ Cryptographic proof via ECDSA signature
│  ├─ No external dependency (local verification)
│  ├─ Requires session handshake in Convex
│  └─ Strong: Signature validated before accepting request

└─ Sigma Bearer Token
   ├─ Delegated to external auth service
   ├─ Rate-limited to prevent abuse
   ├─ Can support org contexts
   └─ Strong: Token validated by auth service

Both methods MUST resolve to valid { bapId, pubkey }
before authorBapId is assigned in database.


AUTHORIZATION CHECK
───────────────────

⚠️  signerAddress is NOT verified against auth.pubkey
    • They use different keys (intentionally)
    • No cryptographic link expected
    • Allows future delegated publishing

✓   Only authenticated users can publish
    • requireAuth() throws if not authenticated
    • skillVersions always has authorBapId
    • Convex enforces database schema


PROFILE PRIVACY
───────────────

◆ BAP profiles are PUBLIC (on-chain)
  • Anyone can query Sigma API
  • No auth required to fetch profile
  • HTTP caching enabled (1 hour TTL)

◆ skillVersions data is PUBLIC
  • Slug, version, authorBapId public
  • AIP signature proof visible
  • Convex query API allows listing
```

---

## Error Handling

```
Authentication Failures
───────────────────────

POST /api/v1/skills (missing/invalid auth)
└─ requireAuth() throws Error
   ├─ BRC-31: session not found, signature invalid, etc.
   ├─ Sigma: token invalid, expired, malformed
   └─ Response: 401 Unauthorized


Profile Fetch Failures
──────────────────────

Sigma API returns 404 (profile doesn't exist)
└─ Gracefully handled
   ├─ Fallback to truncated BAP ID for display
   ├─ Avatar: generated gradient or default
   └─ No error propagated to user


Profile Fetch Timeout
────────────────────

Sigma API unreachable/slow
└─ 3-second timeout
   ├─ Use cached profile if available
   ├─ Fallback to BAP ID display
   └─ Don't block skill loading


Cache Invalidation
──────────────────

Profiles stale after 1 hour (ISR)
└─ Next page build fetches fresh profile
   ├─ User sees updated name/avatar
   ├─ Minimal performance impact
   └─ No manual cache busting needed
```

---

## Scaling Considerations

```
PROFILE CACHING STRATEGY
────────────────────────

┌─ Convex Cache
│  ├─ 1 query = O(1) lookup by bapId
│  ├─ Stores computed data (reduced Sigma calls)
│  └─ Evicts after lastSeen > threshold

├─ HTTP Cache Headers
│  ├─ Sigma API responses cached for 1 hour
│  ├─ Browser and CDN cache enabled
│  └─ Reduces external API load

└─ Batch Fetching
   ├─ Skill detail page with 10 attestors
   ├─ Fetch all profiles in parallel
   ├─ Promise.allSettled() for fault tolerance
   └─ Cache locally to Convex


CONVEX INDEXES
──────────────

skillVersions:
├─ by_slug: Fast lookup for skill detail page
├─ by_authorBapId: All skills by author
└─ by_slug_and_version: Version history

profiles:
└─ by_bapId: Fast cache lookup for author info


API RATE LIMITING
─────────────────

Sigma API:
├─ No auth required (public endpoint)
├─ Rate limits likely per IP or token
└─ HTTP caching extends limits significantly

Convex:
├─ Built-in rate limiting
├─ Database write quotas
└─ Query volume scales with user base
```

---

## Summary Diagram

```
USER
  │
  ├─ Has BRC-100 wallet
  │  ├─ Identity key (for auth)
  │  └─ Could have Signing key (for AIP)
  │
  └─ Publishes skill
     │
     ├─ Authenticate with identity key (BRC-31 or Sigma)
     │  └─ Server resolves: authorBapId = pubkeyToBapId(identityKey)
     │
     ├─ Sign skill with signing key (AIP)
     │  └─ Produces: signerAddress (not used for identity)
     │
     └─ Create BAP profile (optional, future)
        └─ Update Sigma with name, avatar, bio

READER
  │
  └─ Views skill on ClawNet
     │
     ├─ Load skill → authorBapId
     ├─ Query profile cache → Sigma API
     ├─ Display author with name + avatar
     └─ Can click to view author's other skills
```


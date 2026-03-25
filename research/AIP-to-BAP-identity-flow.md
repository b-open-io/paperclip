# AIP Signer Address → BAP Identity Flow Research Report

## Executive Summary

This research documents the exact path from an AIP (Author Identity Protocol) signer address generated during skill publishing to a verified BAP (Bitcoin Attestation Protocol) identity on ClawNet. The flow involves three distinct transformations and operates across two separate authentication contexts.

**Key Finding**: `signerAddress` from AIP is **NOT** directly used to derive `authorBapId`. Instead, `authorBapId` comes from authenticated user context via BRC-31 or Sigma Auth.

---

## 1. Current Flow: Skill Publishing

### 1.1 AIP Signature Generation (CLI Layer)

**File**: `/Users/satchmo/code/clawnet/packages/cli/src/package/skill-tx.ts`

```typescript
// Lines 35-111: buildSkillOpReturn()
// Input: PrivateKey
// Output:
//   - lockingScript: LockingScript (OP_RETURN script)
//   - opReturnHex: string (hex-encoded)
//   - signerAddress: string     // <-- AIP produces THIS
//   - aipSignature: string
```

**What happens**:
1. User signs skill content (B protocol + MAP metadata) with their private key
2. AIP protocol creates signature using `PrivateKeySigner(privateKey)`
3. `aipData.data.address` is extracted as `signerAddress` (line 108)
4. This is a **Bitcoin address** derived from the signing private key

```typescript
const aipData = await AIP.sign(signatureData, new PrivateKeySigner(privateKey));
return {
  signerAddress: aipData.data.address,  // Bitcoin address
  aipSignature: Utils.toBase64(aipData.data.signature),
  ...
};
```

### 1.2 API Publish Request (CLI → Server)

**File**: `/Users/satchmo/code/clawnet/packages/cli/src/api.ts` (line 310-335)

The CLI sends:
```typescript
{
  slug: string;
  name: string;
  description: string;
  version: string;
  files: { path: string; content: string }[];
  aipSignature?: string;      // base64-encoded signature
  signerAddress?: string;     // Bitcoin address (NOT a BAP ID!)
  opReturnHex?: string;       // full OP_RETURN hex
}
```

### 1.3 Server-Side Publish Handler (Authentication)

**File**: `/Users/satchmo/code/clawnet/app/api/v1/skills/route.ts` (line 45-126)

```typescript
export async function POST(request: Request) {
  let auth: AuthContext;
  try {
    auth = await requireAuth(request);  // <-- USER AUTHENTICATION
  } catch {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  let payload: PublishPayload = /* ... parse request ... */;

  // KEY ASSIGNMENT - line 91:
  const result = await getConvex().mutation(api.mutations.publishVersion, {
    slug: payload.slug,
    // ... other fields ...
    authorBapId: auth.bapId,  // <-- FROM AUTHENTICATED USER, NOT signerAddress!
    aipSignature: payload.aipSignature,
    signerAddress: payload.signerAddress,
  });
}
```

**Critical Finding**: `authorBapId` is set from `auth.bapId` (line 91), NOT from the `signerAddress` sent in the request body.

### 1.4 Authentication Context Resolution

**File**: `/Users/satchmo/code/clawnet/lib/auth-middleware.ts`

The `requireAuth()` function supports two authentication mechanisms:

#### Path 1: BRC-31 (Authrite) - More Specific
```typescript
async function verifyBrc31Auth(request: Request, headers: Brc31AuthHeaders): Promise<AuthContext | null> {
  // ... verify signature against BRC-43 derived keys ...
  const bapId = pubkeyToBapId(identityKey);  // Line 111
  return { bapId, pubkey: identityKey };
}
```

Where `identityKey` is the client's **BRC-100 identity key** (compressed public key).

#### Path 2: Sigma Auth - Fallback
```typescript
async function verifySigmaSession(token: string): Promise<AuthContext | null> {
  const response = await fetch(`${SIGMA_AUTH_URL}/api/auth/get-session`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  const session = await response.json();
  const user = session?.user;

  // Prefer org, fall back to user pubkey
  const bapId = session.session?.activeOrganizationId || user.pubkey;  // Line 145
  return { bapId, pubkey: user.pubkey };
}
```

**Key Methods**:
- **BRC-31 → BAP ID**: `pubkeyToBapId(identityKey)` via `bsv-bap` library
  - Takes compressed pubkey (from BRC-100 wallet identity key)
  - Derives BAP ID using `bapIdFromPubkey()`

- **Sigma Auth → BAP ID**: From Sigma session directly
  - Can be org ID or user's pubkey (both are valid BAP identifiers)

### 1.5 Database Storage

**File**: `/Users/satchmo/code/clawnet/convex/schema.ts`

```typescript
skillVersions: defineTable({
  slug: v.string(),
  version: v.string(),
  authorBapId: v.string(),           // <-- From auth.bapId
  aipSignature: v.optional(v.string()),
  signerAddress: v.optional(v.string()),  // <-- From payload (stored but not used for identity)
  onChain: v.boolean(),
  // ...
})
```

Both are stored, but only `authorBapId` identifies the author.

---

## 2. BAP Identity Resolution (Display Layer)

### 2.1 Fetching BAP Profile Information

**File**: `/Users/satchmo/code/clawnet/lib/sigma.ts` (line 21-35)

```typescript
export async function fetchSigmaProfile(bapId: string): Promise<SigmaProfile | null> {
  try {
    const res = await fetch(`${SIGMA_API_BASE}/profile/${bapId}`, {
      next: { revalidate: 3600 },
    });
    if (!res.ok) return null;
    const data = await res.json();
    if (data.status !== "OK" || !data.result) return null;
    return data.result as SigmaProfile;
  } catch {
    return null;
  }
}
```

**Endpoint**: `https://sigma.1sat.app/1sat/bap/profile/{bapId}`

**Request**:
- **Method**: GET
- **Parameter**: BAP ID in path (not address)
- **Example**: `https://sigma.1sat.app/1sat/bap/profile/03a1b2c3d4...`

**Response** (on success):
```typescript
{
  "@context": string;
  "@type": string;
  alternateName?: string;      // Display name
  banner?: string;             // Image path
  description?: string;
  homeLocation?: { "@type": string; name: string };
  image?: string;              // Profile image path (can be relative like "_0")
  paymail?: string;
  url?: string;
}
```

### 2.2 Skill Detail Page Resolution

**File**: `/Users/satchmo/code/clawnet/app/skills/[slug]/page.tsx` (line 89-194)

**Flow**:
```
1. Load skill from Convex → skill.authorBapId
2. Fetch author profile from Convex profiles table
3. If not in Convex cache:
   → fetchSigmaProfile(skill.authorBapId)
   → Sigma API returns profile with alternateName, image, etc.
4. Resolve image URL via ORDFS gateway (if inscribed on-chain)
```

**Key Code** (line 186-193):
```typescript
let authorImageUrl: string | null = null;
if (author?.image) {
  authorImageUrl = resolveOnChainImageUrl(author.image);
} else {
  const sigmaProfile = await fetchSigmaProfile(skill.authorBapId);
  authorImageUrl = resolveOnChainImageUrl(sigmaProfile?.image);
}
```

---

## 3. Convex Profile Caching Layer

### 3.1 Profile Table Schema

**File**: `/Users/satchmo/code/clawnet/convex/schema.ts` (line 94-105)

```typescript
profiles: defineTable({
  bapId: v.string(),
  identity: v.object({
    alternateName: v.optional(v.string()),
    description: v.optional(v.string()),
    image: v.optional(v.string()),
    paymail: v.optional(v.string()),
    paymentAddress: v.optional(v.string()),
    type: v.optional(v.string()),
  }),
  lastSeen: v.number(),
}).index("by_bapId", ["bapId"]),
```

**Purpose**: Local cache of Sigma BAP profiles to reduce API calls.

### 3.2 Profile Lookup (Skill Detail Page)

**File**: `/Users/satchmo/code/clawnet/app/skills/[slug]/page.tsx` (line 155-166)

```typescript
const fetchedProfiles =
  auditorBapIds.length > 0
    ? await convex
        .query(api.profiles.getByBapIds, { bapIds: auditorBapIds })
        .catch(() => [])
    : [];

// Merge author profile into profiles array
const profiles = author
  ? [
      ...fetchedProfiles.filter(
        (p): p is NonNullable<typeof p> =>
          p !== null && p.bapId !== skill.authorBapId
      ),
      {
        bapId: skill.authorBapId,
        identity: {
          alternateName: author.alternateName,
          image: author.image,
        },
      },
    ]
  : fetchedProfiles;
```

---

## 4. The Three-Layer Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ LAYER 1: TRANSACTION SIGNING (CLI)                              │
│                                                                  │
│ Private Key → AIP Signature → signerAddress (Bitcoin address)   │
│                                 ↓                                │
│                         (not used for identity)                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ LAYER 2: AUTHENTICATION & IDENTITY (Server)                     │
│                                                                  │
│ BRC-31 Headers / Sigma Bearer Token                             │
│     ↓                                                            │
│ pubkeyToBapId() / Sigma Session                                 │
│     ↓                                                            │
│ authorBapId (BAP Identity) ← Stored in Convex                   │
│                                                                  │
│ This is the ONLY identity source for authorBapId!              │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ LAYER 3: PROFILE RESOLUTION (Display)                           │
│                                                                  │
│ authorBapId → Sigma API /bap/profile/{bapId}                   │
│     ↓                                                            │
│ Profile { alternateName, image, paymail, ... }                 │
│     ↓                                                            │
│ Render skill author with name, avatar, etc.                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Data Flow Mapping

### 5.1 What IS the `signerAddress`?

- **Type**: Bitcoin P2PKH address
- **Source**: Derived from AIP signature (public key of signing private key)
- **Current Use**: Stored in Convex but **not used for identity mapping**
- **Purpose**: Could be used to:
  - Verify signature authenticity (check if pubkey matches)
  - Track on-chain payment addresses
  - Attestation proof of authorship (future feature)

### 5.2 What IS the `authorBapId`?

- **Type**: BAP ID (typically 33-66 char hex string or Bitcoin address)
- **Source**:
  - From BRC-31: `pubkeyToBapId(identityKey)` where `identityKey` is user's identity key
  - From Sigma: Session or org ID
- **Current Use**: **Primary identity source** for author attribution
- **Storage**: `skillVersions.authorBapId` in Convex
- **Display**: Resolved via Sigma API for profile metadata

### 5.3 Relationship Between signerAddress and authorBapId

```
┌──────────────────┐
│  Private Key     │
└────────┬─────────┘
         │
         ├─→ AIP Signature → signerAddress (address of signer)
         │
         └─→ Public Key Component
              │
              └─→ (Different from identity key!)
                   │
                   └─→ NOT used in current flow
                        (Identity comes from auth layer instead)
```

**Problem**: The signing key (`privateKey` in `buildSkillOpReturn()`) is **NOT** the same as the user's BRC-100 identity key. These are separate keys:
- **Signing Key**: Used only for AIP transaction signature
- **Identity Key**: BRC-100 member key, used for BRC-31 auth or Sigma identity

---

## 6. Sigma Identity API Details

### 6.1 Endpoint Structure

```
GET https://sigma.1sat.app/1sat/bap/profile/{bapId}
```

**Parameters**:
- `bapId`: String (BAP identity key or address format)

**Returns**:
```json
{
  "status": "OK",
  "result": {
    "@context": "https://schema.org",
    "@type": "Person",
    "alternateName": "Display Name",
    "image": "/txId_vout",
    "banner": "/txId_vout",
    "description": "User bio",
    "paymail": "user@example.com",
    "url": "https://example.com"
  }
}
```

**Status Codes**:
- `200 OK + status: "OK"` → Profile found
- `404` or `status != "OK"` → Profile not found (no on-chain identity)

### 6.2 Image Path Resolution

**Relative Paths** (e.g., `_0`, `_1`):
- Expanded using attestation txId: `/{txId}_{vout}`
- Resolved to ORDFS gateway: `https://ordfs.network/{txId}_{vout}`

**Absolute Paths** (e.g., `/txId_vout`):
- Already expanded, just pass to ORDFS gateway

---

## 7. API Call Summary for Development

### To Go From `signerAddress` → `authorBapId`:

**Current Implementation: IMPOSSIBLE** (not designed this way)

The system doesn't have a reverse mapping from `signerAddress` → `authorBapId` because:
1. They come from different keys (signing key vs. identity key)
2. Multiple users could theoretically sign the same skill
3. The authentication layer is the source of truth for identity

### To Get Author Profile Given `authorBapId`:

```bash
# 1. Get BAP profile
curl -s "https://sigma.1sat.app/1sat/bap/profile/{bapId}"

# 2. Response (if profile exists):
{
  "status": "OK",
  "result": {
    "alternateName": "Author Name",
    "image": "/txId_vout",
    "description": "Bio"
  }
}

# 3. Resolve image URL
# If image exists, fetch from ORDFS:
curl -s "https://ordfs.network/{txId}_{vout}"
```

### To Verify AIP Signature Authenticity:

```typescript
// From skill version record:
const { aipSignature, signerAddress } = skillVersion;

// 1. Recover public key from signature
// 2. Derive address from pubkey
// 3. Compare with signerAddress to verify

// Note: This is an optional verification step, NOT required for identity
```

---

## 8. Proposed Improvements / Future State

### 8.1 Current Gap

The `signerAddress` is stored but never validated or displayed. This means:
- Users can't verify "who signed this skill"
- No cryptographic proof of authorship on-chain
- Skill could be published by someone else's account with custom signing key

### 8.2 Recommended Future Feature

**Add Signature Validation Display**:
```typescript
// On skill detail page:
1. If signerAddress exists:
   - Show badge: "Signed by: {signerAddress}"
   - Optionally verify signature matches claimed author

2. If signerAddress != authorBapId's payment address:
   - Show warning: "⚠️ Signer differs from claimed author"
   - Allow auditors to review
```

### 8.3 Blockchain Audit Trail

For fully decentralized verification:
1. Store `aipSignature` + `opReturnHex` + `signerAddress` on-chain
2. Sigma can verify: signature matches signer address
3. Convex tracks: authenticated user's `authorBapId` (who published)
4. Web of trust: auditors can attest to either/both

---

## 9. Code References Summary

| Component | File | Lines | Purpose |
|-----------|------|-------|---------|
| AIP Signature Generation | `skill-tx.ts` | 35-111 | Create transaction signature |
| CLI API Publish | `api.ts` | 310-335 | Send skill to server |
| Server Auth | `auth-middleware.ts` | 46-173 | Resolve authenticated user identity |
| Skill Publish Handler | `route.ts` (skills) | 45-126 | Assign authorBapId from auth context |
| Profile Fetch | `sigma.ts` | 21-35 | Get BAP profile from Sigma API |
| Skill Display | `page.tsx` (detail) | 89-194 | Render author info |
| Database Schema | `schema.ts` | 36-67 | Store authorBapId + signerAddress |

---

## 10. Testing the APIs

### Test BAP Profile Fetch

```bash
# Get profile for a known BAP ID (if exists on-chain)
curl -s "https://sigma.1sat.app/1sat/bap/profile/03abc123def456..." | jq .

# Expected response (if profile exists):
{
  "status": "OK",
  "result": {
    "@context": "https://schema.org",
    "@type": "Person",
    "alternateName": "Name",
    "image": "/txId_vout"
  }
}

# If no profile:
{
  "message": "Profile not found for BAP ID: ..."
}
```

### Test Skill API

```bash
# Get skill with author BAP ID
curl -s "https://api.clawnet.app/api/v1/skills/my-skill" | jq '.authorBapId'

# Get full skill list
curl -s "https://api.clawnet.app/api/v1/skills?limit=1" | jq '.[0] | {slug, authorBapId}'
```

---

## Conclusion

The path from AIP `signerAddress` to BAP identity is **not direct**. Instead:

1. **AIP Signature** produces a `signerAddress` (Bitcoin address)
2. **Authentication Layer** (BRC-31 or Sigma) produces the `authorBapId` (BAP identity)
3. **Sigma API** resolves `authorBapId` to profile data (name, image, bio)

The key insight: **`authorBapId` is derived from the authenticated user's identity credentials, not from the skill signing key**. This is intentional for security and audit purposes.


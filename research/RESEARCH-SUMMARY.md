# Research Summary: AIP Signer Address → BAP Identity

**Date**: 2026-03-21
**Scope**: ClawNet skill publishing flow
**Status**: COMPLETE

---

## Executive Summary

Investigated how skill author identity is established in ClawNet's skill publishing system. Found that **AIP (Author Identity Protocol) signer address and BAP (Bitcoin Attestation Protocol) identity are separate concerns using different cryptographic keys**.

### Key Finding

**No direct path from AIP `signerAddress` to BAP `authorBapId` exists** because:

1. **`signerAddress`** comes from the AIP signature (created with a skill-specific signing key)
2. **`authorBapId`** comes from user authentication (BRC-31 or Sigma Auth with identity key)
3. These are **intentionally different keys** for security and auditability

---

## Flow Overview

```
User publishes skill with signing key
    ↓
AIP creates signature, produces signerAddress (Bitcoin address)
    ↓
Skill data sent to ClawNet server with aipSignature + signerAddress
    ↓
Server validates user authentication (BRC-31 or Sigma token)
    ↓
Authentication layer produces authorBapId (from identity key, not signing key)
    ↓
authorBapId stored in Convex (PRIMARY IDENTITY SOURCE)
    ↓
On display: Query Sigma API with authorBapId to fetch profile
    ↓
Render skill with author name, avatar, bio
```

---

## Three Data Points

| Data Point | Source | Purpose | Current Use |
|-----------|--------|---------|-------------|
| **`signerAddress`** | AIP.sign() output | Cryptographic proof of skill transaction | Stored in Convex, not used for identity |
| **`aipSignature`** | AIP.sign() output | Signature bytes (base64) | Stored in Convex, never validated |
| **`authorBapId`** | BRC-31 or Sigma auth | User identity claim | PRIMARY - used for author attribution |

---

## Authentication Context (The Source of Truth)

### BRC-31 Path (Preferred)

```typescript
identityKey (from x-bsv-auth-identity-key header)
    ↓
pubkeyToBapId(identityKey)  [bsv-bap library]
    ↓
authorBapId
```

### Sigma Auth Path (Fallback)

```typescript
Bearer token
    ↓
Verify with auth.sigmaidentity.com
    ↓
session.activeOrganizationId || user.pubkey
    ↓
authorBapId
```

**Both paths must return `{ bapId, pubkey }` which becomes `auth.bapId` → `authorBapId`.**

---

## Profile Resolution

Once `authorBapId` is stored, the profile is resolved via:

```
authorBapId
    ↓
Check Convex profiles table
    ↓
Cache MISS → Call Sigma API
    ↓
GET https://sigma.1sat.app/1sat/bap/profile/{bapId}
    ↓
Returns: SigmaProfile { alternateName, image, description, ... }
    ↓
Cache result in Convex (lastSeen = now)
    ↓
Display on skill page
```

---

## What Each API Expects

### Skill Publish Endpoint

**Endpoint**: `POST https://clawnet.app/api/v1/skills`

**Authentication**:
- BRC-31 headers: `x-bsv-auth-identity-key`, `x-bsv-auth-signature`, `x-bsv-auth-your-nonce`
- OR Sigma Bearer token: `Authorization: Bearer <token>`

**Request Body**:
```json
{
  "slug": "my-skill",
  "name": "My Skill",
  "description": "Does something",
  "version": "0.1.0",
  "files": [{"path": "SKILL.md", "content": "..."}],
  "aipSignature": "base64-encoded-signature",
  "signerAddress": "1ABC123...",
  "opReturnHex": "6a..."
}
```

**Server Processing**:
```typescript
auth = await requireAuth(request);  // Validates auth headers
// → { bapId, pubkey }

authorBapId = auth.bapId;  // THIS is the identity source
// NOT from payload.signerAddress!
```

### Sigma BAP Profile Endpoint

**Endpoint**: `GET https://sigma.1sat.app/1sat/bap/profile/{bapId}`

**Request**:
```bash
curl "https://sigma.1sat.app/1sat/bap/profile/03abc123def456..."
```

**Response** (if profile exists):
```json
{
  "status": "OK",
  "result": {
    "@context": "https://schema.org",
    "@type": "Person",
    "alternateName": "Author's Display Name",
    "image": "/txId_vout",
    "banner": "/txId_vout",
    "description": "User bio",
    "paymail": "user@example.com",
    "url": "https://..."
  }
}
```

**Response** (if no profile):
```json
{
  "message": "Profile not found for BAP ID: 03abc123..."
}
```

---

## Database Schema

### skillVersions table

```typescript
{
  slug: string;
  version: string;
  authorBapId: string;           // ← PRIMARY IDENTITY
  aipSignature: optional<string>;
  signerAddress: optional<string>;
  opReturnHex: optional<string>;
  content: string;               // SKILL.md
  publishedAt: number;
  onChain: boolean;
}
```

### profiles table (Cache)

```typescript
{
  bapId: string;
  identity: {
    alternateName: optional<string>;
    description: optional<string>;
    image: optional<string>;
    paymail: optional<string>;
    // ... other fields from SigmaProfile
  };
  lastSeen: number;
}
```

---

## Critical Implementation Details

### 1. Do NOT Try to Derive `authorBapId` from `signerAddress`

The signing key and identity key are intentionally separate:
- Signing key: Used for AIP transaction signature
- Identity key: Used for user authentication

There's no reverse mapping and no relationship expected.

### 2. Do NOT Use `signerAddress` for User Identification

`signerAddress` is stored for optional audit purposes but is **never used** to identify the author. Always use `authorBapId`.

### 3. Do Validate Authentication Before Assigning `authorBapId`

```typescript
// CORRECT PATTERN:
const auth = await requireAuth(request);
if (!auth) throw new Error("Unauthorized");
const authorBapId = auth.bapId;  // ← From authentication, not request body
```

### 4. Do Handle Missing Profiles Gracefully

Not all users have published a BAP profile. Fallback logic:

```typescript
const authorName =
  author?.alternateName ||           // Convex cache
  sigmaProfile?.alternateName ||     // Sigma API result
  truncateId(skill.authorBapId);     // Fallback to BAP ID
```

### 5. Do Cache Profile Data

Sigma API is external and rate-limited. Always cache:
```typescript
// In Convex cache with 1-hour TTL
const profile = await convex.query(api.profiles.getByBapId, { bapId });
if (!profile) {
  const sigmaProfile = await fetchSigmaProfile(bapId);
  // Cache result
}
```

---

## Files Analyzed

| File | Purpose | Key Lines |
|------|---------|-----------|
| `skill-tx.ts` | AIP signature generation | 35-111 |
| `api.ts` | CLI publish endpoint types | 310-335 |
| `/api/v1/skills/route.ts` | Server publish handler | 45-126 |
| `auth-middleware.ts` | Authentication resolution | 46-173 |
| `sigma.ts` | Sigma API integration | 21-35 |
| `page.tsx` (skill detail) | Profile display | 89-194 |
| `schema.ts` | Database schema | 36-105 |

---

## Testing Commands

### Publish a Skill

```bash
# With BRC-31 auth headers
curl -X POST https://clawnet.app/api/v1/skills \
  -H "x-bsv-auth-identity-key: 03abc..." \
  -H "x-bsv-auth-signature: base64sig" \
  -H "x-bsv-auth-your-nonce: nonce" \
  -H "Content-Type: application/json" \
  -d '{
    "slug": "test-skill",
    "name": "Test Skill",
    "description": "A test",
    "version": "0.1.0",
    "files": [{"path": "SKILL.md", "content": "# Test"}],
    "aipSignature": "base64sig",
    "signerAddress": "1ABC..."
  }'
```

### Fetch BAP Profile

```bash
BAP_ID="03abc123def456..."
curl "https://sigma.1sat.app/1sat/bap/profile/${BAP_ID}" | jq .
```

### Check Convex Profile Cache

```bash
convex query api.profiles.getByBapIds --arg bapIds '["03abc..."]'
```

---

## Potential Future Enhancements

1. **Signature Verification Badge**: Display "Signed by: {signerAddress}" on skill page
2. **Cryptographic Proof**: Allow users to verify AIP signature authenticity
3. **Multi-Signer Support**: Track multiple signers for collaborative skills
4. **Signer Profile**: Fetch profile for `signerAddress` (if it's also a BAP ID)
5. **Audit Trail**: Store signer identity separately from author identity for dispute resolution

---

## Conclusion

The AIP signature process and BAP identity system operate at different layers:

- **AIP** proves the skill transaction was signed (cryptographic commitment)
- **BAP** identifies who published the skill (user authentication)

They are intentionally separate to support:
- **Security**: Different keys for different purposes
- **Auditability**: Can detect if signed key ≠ published key
- **Flexibility**: Supports delegated publishing in the future

The path to resolve author identity is:

```
skillVersions.authorBapId
    ↓ (lookup in Sigma API)
SigmaProfile { alternateName, image, ... }
    ↓ (display on page)
Author name & avatar
```

**Never** try to derive `authorBapId` from `signerAddress` — they use different keys by design.

---

## Resources

- **Full Report**: `AIP-to-BAP-identity-flow.md`
- **Data Flow Diagram**: `data-flow-diagram.txt`
- **Implementation Guide**: `implementation-reference.md`
- **Sigma API Docs**: https://sigma.1sat.app/
- **BRC Standards**: https://brc.dev/
- **Bitcoin Schema**: https://bitcoinschema.org/


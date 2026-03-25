# Research: AIP Signer Address to BAP Identity Flow

Complete investigation of how skill author identity is established in ClawNet's decentralized skill publishing system.

## Quick Answer

**There is no direct path from AIP `signerAddress` to BAP `authorBapId` because they come from different keys:**

- **`signerAddress`**: Output of AIP signature (signing key) → stored for audit
- **`authorBapId`**: Output of authentication layer (identity key) → used for identity

**The authentication layer is the sole source of truth for author identity.**

---

## Documents in This Research

### 1. **RESEARCH-SUMMARY.md** 🎯 START HERE
Executive summary of findings with key data points and testing commands.
- High-level overview
- Three-layer architecture
- Critical implementation details
- Potential enhancements
- ~500 lines, 20 minute read

### 2. **AIP-to-BAP-identity-flow.md** 📖 DETAILED GUIDE
Comprehensive documentation of the complete flow from AIP signing through BAP profile display.
- Current flow walkthrough (5 sections)
- Skill publishing to database storage
- Profile resolution via Sigma API
- Data flow mapping
- API call summary
- ~900 lines, 45 minute read

### 3. **data-flow-diagram.txt** 🎨 VISUAL REFERENCE
ASCII diagram showing the complete data transformation from user CLI to rendered profile.
- Phase 1: Skill creation (CLI)
- Phase 2: Authentication & identity
- Phase 3: Display & resolution
- Key transformations
- API endpoints
- ~400 lines, reference document

### 4. **system-architecture.md** 🏗️ ARCHITECTURE
Detailed system architecture with security model and scaling considerations.
- High-level architecture diagram
- Key relationships between streams
- Authentication context derivation
- Data flow from CLI to display
- Convex data model
- External service dependencies
- Security model & error handling
- ~600 lines, reference document

### 5. **implementation-reference.md** 👨‍💻 FOR DEVELOPERS
Practical guide for developers implementing or extending the system.
- Quick facts table
- How to verify signatures (optional feature)
- How to add profile fields
- Custom authentication methods
- Performance optimization
- Common mistakes to avoid
- ~500 lines, implementation guide

---

## Key Findings

### Finding 1: Separate Identity Streams

```
Stream 1: AIP Signing
├─ User's signing private key
├─ Produces: signerAddress (Bitcoin address)
└─ Used for: Cryptographic proof (optional audit)

Stream 2: User Authentication  ← PRIMARY
├─ User's identity key (BRC-100 or Sigma)
├─ Produces: authorBapId (via BRC-31 or Sigma Auth)
└─ Used for: Author attribution in database
```

### Finding 2: Authentication is the Source of Truth

```typescript
// What server does:
auth = await requireAuth(request);
authorBapId = auth.bapId;  // ← From authentication

// NOT from:
// authorBapId = payload.signerAddress;  // Wrong!
```

### Finding 3: Profile Resolution Path

```
authorBapId (from auth)
    ↓
Check Convex cache
    ↓
If miss: Query Sigma API
    ↓
Get profile { alternateName, image, description }
    ↓
Display on skill page
```

---

## File References

| Component | File | Purpose |
|-----------|------|---------|
| AIP signature generation | `skill-tx.ts` (lines 35-111) | Create transaction signature |
| Publish request handler | `route.ts` /api/v1/skills (lines 45-126) | Assign authorBapId from auth |
| Authentication layer | `auth-middleware.ts` (lines 46-173) | Resolve identity from BRC-31/Sigma |
| Profile fetch | `sigma.ts` (lines 21-35) | Query Sigma API |
| Skill display | `page.tsx` (lines 89-194) | Render author profile |
| Database schema | `schema.ts` (lines 36-105) | skillVersions & profiles tables |

---

## Testing the Flow

### Test 1: Publish a Skill

```bash
# With BRC-31 auth (via Yours Wallet or similar)
curl -X POST https://clawnet.app/api/v1/skills \
  -H "x-bsv-auth-identity-key: 03abc..." \
  -H "x-bsv-auth-signature: base64sig" \
  -H "x-bsv-auth-your-nonce: nonce" \
  -H "Content-Type: application/json" \
  -d '{
    "slug": "test-skill",
    "files": [{"path": "SKILL.md", "content": "# Test"}],
    "aipSignature": "base64sig",
    "signerAddress": "1ABC..."
  }'
```

### Test 2: Verify authorBapId Assignment

```bash
# Query the published skill
curl https://clawnet.app/api/v1/skills/test-skill | jq '.authorBapId'

# Should return BAP ID, NOT signerAddress
```

### Test 3: Fetch BAP Profile

```bash
# Get profile from Sigma for authorBapId
curl "https://sigma.1sat.app/1sat/bap/profile/03xyz789..." | jq .

# Returns profile or 404 if no on-chain profile
```

---

## Architecture Summary

### Three-Layer Stack

```
LAYER 1: TRANSACTION SIGNING
Private Key → AIP Signature → signerAddress (Bitcoin address)
(Stored in skillVersions, not used for identity)

                    ↓

LAYER 2: AUTHENTICATION & IDENTITY
BRC-31 headers / Sigma token
    ↓
pubkeyToBapId() / Sigma Session
    ↓
authorBapId ← STORED IN CONVEX (PRIMARY IDENTITY)

                    ↓

LAYER 3: PROFILE RESOLUTION
authorBapId → Sigma API /bap/profile/{bapId}
    ↓
SigmaProfile { alternateName, image, description }
    ↓
Display on skill page
```

---

## Critical Implementation Details

### ✓ DO

1. Use `auth.bapId` as `authorBapId` (not `signerAddress`)
2. Validate authentication before assigning identity
3. Cache profiles in Convex with TTL
4. Handle missing profiles gracefully
5. Verify auth context before storing data

### ❌ DON'T

1. Try to derive `authorBapId` from `signerAddress`
2. Use `signerAddress` to identify the author
3. Block skill publishing if profile not found
4. Assume signing key = identity key
5. Query Sigma API without caching

---

## API Reference

### Skill Publish Endpoint

```
POST https://clawnet.app/api/v1/skills

Authentication (one of):
├─ BRC-31 headers (x-bsv-auth-*)
└─ Bearer token (Authorization header)

Request body:
{
  "slug": "skill-name",
  "name": "Display Name",
  "description": "What it does",
  "version": "0.1.0",
  "files": [{"path": "SKILL.md", "content": "..."}],
  "aipSignature": "base64...",
  "signerAddress": "1ABC..."
}

Response:
{ "ok": true, "slug": "skill-name", "version": "0.1.0" }
```

### BAP Profile Endpoint (Sigma)

```
GET https://sigma.1sat.app/1sat/bap/profile/{bapId}

Parameters:
├─ bapId: BAP identity key (path parameter)

Response (success):
{
  "status": "OK",
  "result": {
    "alternateName": "Author Name",
    "image": "/txId_vout",
    "description": "Bio",
    ...
  }
}

Response (not found):
{
  "message": "Profile not found for BAP ID: 03abc..."
}
```

---

## Glossary

| Term | Definition |
|------|-----------|
| **AIP** | Author Identity Protocol — produces cryptographic signature |
| **BAP** | Bitcoin Attestation Protocol — identity system |
| **BRC-31** | Authrite protocol — message authentication with signatures |
| **BRC-100** | Wallet Interface standard — includes identity key |
| **signerAddress** | Bitcoin address derived from AIP signature (audit proof) |
| **authorBapId** | BAP ID from authenticated user (identity claim) |
| **identityKey** | User's BRC-100 member key (compressed pubkey) |
| **Sigma API** | Decentralized identity service (sigma.1sat.app) |
| **Convex** | Backend database service (stores skills & profiles) |

---

## Future Enhancements

1. **Signature Verification Badge**: Display "Signed by: {signerAddress}"
2. **Cryptographic Proof**: Allow users to verify AIP signature authenticity
3. **Signer Profiles**: If signerAddress is also a BAP ID, show signer profile
4. **Multi-Signer Support**: Track multiple signers for collaborative skills
5. **Audit Trail**: Store signer identity separately from author for disputes

---

## Related Resources

- **BSV Standards**: https://brc.dev/
- **Sigma Identity**: https://sigma.1sat.app/
- **1Sat**: https://1sat.app/
- **Bitcoin Schema**: https://bitcoinschema.org/
- **BRC-100 Spec**: https://brc.dev/100

---

## Report Metadata

- **Research Date**: 2026-03-21
- **Scope**: ClawNet skill publishing identity flow
- **Status**: Complete
- **Files Analyzed**: 7 TypeScript/schema files
- **APIs Tested**: Sigma BAP profile endpoint
- **Total Documentation**: ~3000 lines across 5 documents

---

## Reading Path Recommendation

**For Quick Understanding** (20 minutes):
1. RESEARCH-SUMMARY.md (overview)
2. data-flow-diagram.txt (visual)

**For Implementation** (60 minutes):
1. RESEARCH-SUMMARY.md (overview)
2. system-architecture.md (architecture)
3. implementation-reference.md (code guide)

**For Complete Mastery** (120 minutes):
1. Read all documents in order
2. Review code references
3. Test APIs locally
4. Implement extensions from implementation-reference.md

---

## Questions?

Refer to the relevant document:

- **"What is the relationship between signerAddress and authorBapId?"** → RESEARCH-SUMMARY.md
- **"How does the authentication layer work?"** → AIP-to-BAP-identity-flow.md (section 1.4)
- **"How do I implement signature verification?"** → implementation-reference.md
- **"What are the security implications?"** → system-architecture.md (Security Model)
- **"How should I cache profiles?"** → implementation-reference.md (Caching Strategy)


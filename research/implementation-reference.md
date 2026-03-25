# Implementation Reference: AIP Signature & BAP Identity

Quick reference for developers implementing or extending the AIP → BAP flow.

## Quick Facts

| Aspect | Details |
|--------|---------|
| **AIP → BAP relationship** | No direct mapping; they use different keys |
| **Who determines `authorBapId`** | Authentication layer (BRC-31 or Sigma), NOT the signer |
| **Where is `signerAddress` used** | Stored for reference/audit, not for identity |
| **Primary identity source** | `auth.bapId` from `requireAuth(request)` |
| **Profile API** | `GET https://sigma.1sat.app/1sat/bap/profile/{bapId}` |
| **Profile cache** | Convex `profiles` table, 3600s revalidation |

---

## For Developers: Adding Signature Verification

### If You Want to Verify AIP Signatures

Currently, ClawNet stores `aipSignature` and `signerAddress` but never validates them. To add verification:

```typescript
// 1. In skill publish handler, after storing the skill:
const { aipSignature, signerAddress } = payload;

if (aipSignature && signerAddress) {
  // Verify the signature
  const recovered = recoverPublicKeyFromSignature(aipSignature);
  const recoveredAddress = publicKeyToAddress(recovered);

  if (recoveredAddress !== signerAddress) {
    // Log warning or reject
    console.warn(
      `Signature verification failed: ${recoveredAddress} !== ${signerAddress}`
    );
  }
}

// 2. On display (optional - show "Signed by" badge):
if (skillVersion.signerAddress && skillVersion.signerAddress !== skill.authorBapId) {
  return (
    <div className="bg-yellow-50 p-2 text-xs">
      ⚠️ Signed by {truncateId(skillVersion.signerAddress)}
      <br />
      Published by {truncateId(skill.authorBapId)}
    </div>
  );
}
```

---

## For Developers: Adding BAP Profile Fields

### If You Want to Store More Profile Data

Currently, the `profiles` table stores minimal data. To add more fields:

```typescript
// In convex/schema.ts:
profiles: defineTable({
  bapId: v.string(),
  identity: v.object({
    // Existing fields
    alternateName: v.optional(v.string()),
    description: v.optional(v.string()),
    image: v.optional(v.string()),

    // New fields to consider:
    paymail: v.optional(v.string()),
    paymentAddress: v.optional(v.string()),
    type: v.optional(v.string()),
    banner: v.optional(v.string()),      // NEW: profile banner
    url: v.optional(v.string()),         // NEW: personal website
    txHash: v.optional(v.string()),      // NEW: attestation txid
    // ... other fields from SigmaProfile
  }),
  lastSeen: v.number(),
  lastSyncedAt: v.optional(v.number()), // Track when we last fetched from Sigma
}).index("by_bapId", ["bapId"]),
```

Then update the fetch logic:

```typescript
// In lib/sigma.ts, after fetchSigmaProfile():
export async function syncProfileToConvex(
  bapId: string,
  sigmaProfile: SigmaProfile
) {
  const convex = getConvex();

  await convex.mutation(api.mutations.upsertProfile, {
    bapId,
    identity: sigmaProfile, // Store full profile
    lastSyncedAt: Date.now(),
  });
}
```

---

## For Developers: Custom Authentication

### If You're Adding a New Auth Method

The `requireAuth()` function supports multiple auth methods. To add another:

```typescript
// In lib/auth-middleware.ts:
export async function verifyAuth(request: Request): Promise<AuthContext | null> {
  // Existing paths
  const authHeaders = extractAuthHeaders(request);
  if (authHeaders.identityKey && authHeaders.signature) {
    return verifyBrc31Auth(request, authHeaders);
  }

  const authHeader = request.headers.get("Authorization");
  if (authHeader?.startsWith("Bearer ")) {
    return verifySigmaAuth(authHeader.slice(7));
  }

  // NEW: Your custom auth method
  if (request.headers.has("x-custom-auth")) {
    return verifyCustomAuth(request);
  }

  return null;
}

async function verifyCustomAuth(request: Request): Promise<AuthContext | null> {
  const token = request.headers.get("x-custom-auth");
  if (!token) return null;

  try {
    // Your verification logic
    const payload = verifyToken(token); // e.g., JWT verification

    // IMPORTANT: Must resolve to { bapId, pubkey }
    const bapId = deriveOrLookupBapId(payload);

    return { bapId, pubkey: payload.pubkey };
  } catch {
    return null;
  }
}
```

Key requirement: **Must return `{ bapId, pubkey }`** which then becomes `auth.bapId` for `authorBapId` assignment.

---

## For Developers: Profile Caching Strategy

### Current: Lazy Loading with Convex Cache

```typescript
// In skill detail page:
const profiles = await convex.query(api.profiles.getByBapIds, { bapIds });
// If miss, call Sigma API and update Convex

// Pros: Works for most cases
// Cons: First load slow if profile not cached
```

### Alternative: Proactive Sync

```typescript
// On every skill version publication:
export async function publishVersionWithProfileSync(args) {
  // Publish skill
  const result = await convex.mutation(api.mutations.publishVersion, args);

  // Immediately fetch and cache author profile
  const sigmaProfile = await fetchSigmaProfile(args.authorBapId);
  if (sigmaProfile) {
    await convex.mutation(api.mutations.upsertProfile, {
      bapId: args.authorBapId,
      identity: sigmaProfile,
      lastSyncedAt: Date.now(),
    });
  }

  return result;
}
```

**Pros**: Profile always available when skill published
**Cons**: Extra API call per publish, might fail silently

---

## For Developers: Handling Profile Not Found

### When Sigma Returns 404

```typescript
// In sigma.ts:
export async function fetchSigmaProfile(bapId: string): Promise<SigmaProfile | null> {
  try {
    const res = await fetch(`${SIGMA_API_BASE}/profile/${bapId}`, {
      next: { revalidate: 3600 },
    });

    // 404 = No profile on-chain yet (user hasn't created BAP attestation)
    if (res.status === 404) {
      console.log(`No on-chain profile for ${bapId}`);
      return null; // Graceful fallback
    }

    if (!res.ok) {
      throw new Error(`HTTP ${res.status}`);
    }

    const data = await res.json();
    if (data.status !== "OK" || !data.result) return null;
    return data.result as SigmaProfile;
  } catch (error) {
    console.error(`Profile fetch error for ${bapId}:`, error);
    return null;
  }
}
```

### Fallback Display Logic

```typescript
// In skill detail page:
const authorName =
  author?.alternateName ||           // From Convex cache
  sigmaProfile?.alternateName ||     // From Sigma API
  truncateId(skill.authorBapId);     // Last resort: BAP ID

const authorImage =
  author?.image ||
  sigmaProfile?.image ||
  generateDefaultAvatar(skill.authorBapId); // Gradient avatar
```

---

## For Developers: Testing BAP Identity Flow

### Test Checklist

- [ ] BRC-31 Auth: Send signed request with `x-bsv-auth-*` headers
  ```bash
  curl -X POST https://clawnet.app/api/v1/skills \
    -H "x-bsv-auth-identity-key: 03abc123..." \
    -H "x-bsv-auth-signature: base64signature" \
    -H "x-bsv-auth-your-nonce: serverNonce" \
    -d '{"slug":"test",...}'
  ```

- [ ] Sigma Auth: Send Bearer token
  ```bash
  curl -X POST https://clawnet.app/api/v1/skills \
    -H "Authorization: Bearer sigma_token_here" \
    -d '{"slug":"test",...}'
  ```

- [ ] Verify `authorBapId` is stored correctly
  ```bash
  curl https://clawnet.app/api/v1/skills/test-skill | jq '.authorBapId'
  ```

- [ ] Fetch BAP profile
  ```bash
  curl "https://sigma.1sat.app/1sat/bap/profile/$(BAP_ID)" | jq .
  ```

- [ ] Check profile cache
  ```bash
  # Query Convex profiles table
  convex query api.profiles.getByBapIds --arg bapIds '["03abc..."]'
  ```

---

## For Developers: Common Mistakes

### ❌ Mistake 1: Using `signerAddress` as `authorBapId`

```typescript
// WRONG:
const bapId = payload.signerAddress; // This is a Bitcoin address, not BAP ID!

// RIGHT:
const bapId = auth.bapId; // From authentication layer
```

### ❌ Mistake 2: Assuming `signerAddress` and auth pubkey are related

```typescript
// WRONG:
if (payload.signerAddress !== auth.pubkey) {
  reject("Signer doesn't match authenticated user");
}

// RIGHT:
// These are separate concerns:
// - signerAddress: from AIP signature (signing key)
// - auth.pubkey: from authentication (identity key)
// No relationship expected
```

### ❌ Mistake 3: Blocking publish if profile not found

```typescript
// WRONG:
if (!sigmaProfile) {
  throw new Error("User has no BAP profile");
}

// RIGHT:
// Profile is optional - user can still publish
// Profile fetched on display, created later by user
```

### ❌ Mistake 4: Forgetting to await Sigma API

```typescript
// WRONG:
const profile = fetchSigmaProfile(bapId); // Returns Promise
return profile.alternateName; // undefined!

// RIGHT:
const profile = await fetchSigmaProfile(bapId);
return profile?.alternateName;
```

---

## For Developers: Performance Optimization

### Batch Profile Fetches

```typescript
// Instead of fetching one by one:
const profiles = await Promise.all(
  bapIds.map(id => fetchSigmaProfile(id)) // Parallel
);

// Use Promise.allSettled for fault tolerance:
const results = await Promise.allSettled(
  bapIds.map(id => fetchSigmaProfile(id))
);
const validProfiles = results
  .filter((r) => r.status === "fulfilled")
  .map((r) => r.value)
  .filter(Boolean);
```

### Cache Headers

```typescript
// Sigma API doesn't require auth, so HTTP caching works
fetch(`https://sigma.1sat.app/1sat/bap/profile/${bapId}`, {
  next: { revalidate: 3600 }, // ISR: 1 hour
  // or
  headers: {
    "Cache-Control": "public, max-age=3600",
  },
});
```

### Convex Queries

```typescript
// Index on bapId for fast lookups
profiles: defineTable({
  bapId: v.string(),
  // ...
}).index("by_bapId", ["bapId"]),

// Query efficiently:
const profile = await convex.query(
  api.profiles.getByBapId,
  { bapId }
);
```

---

## Key References

- **Skill TX Building**: `/Users/satchmo/code/clawnet/packages/cli/src/package/skill-tx.ts`
- **Auth Middleware**: `/Users/satchmo/code/clawnet/lib/auth-middleware.ts`
- **Sigma Integration**: `/Users/satchmo/code/clawnet/lib/sigma.ts`
- **Skill Detail Page**: `/Users/satchmo/code/clawnet/app/skills/[slug]/page.tsx`
- **Database Schema**: `/Users/satchmo/code/clawnet/convex/schema.ts`


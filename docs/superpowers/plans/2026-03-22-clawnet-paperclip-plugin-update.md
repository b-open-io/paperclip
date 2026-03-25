# ClawNet Paperclip Plugin — Registry Alignment Update

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bring the clawnet-paperclip-plugin up to date with the current ClawNet registry API — organizations, language, on-chain package fields, and search.

**Architecture:** The plugin syncs agents/skills/organizations from ClawNet into Paperclip's entity store, displays them in a marketplace UI, and links them to Paperclip agents. The API types and sync logic need updating for new fields and entity types.

**Tech Stack:** TypeScript, Paperclip Plugin SDK, React (plugin UI), ClawNet REST API

**Repo:** `~/code/clawnet-paperclip-plugin`

---

## File Structure

| File | Action | Responsibility |
|------|--------|----------------|
| `src/clawnet-api.ts` | Modify | API types + client methods |
| `src/worker.ts` | Modify | Sync logic, entity upsert, data handlers |
| `src/ui/index.tsx` | Modify | Marketplace UI — org tab, language badges, on-chain status |
| `src/constants.ts` | Modify | Entity type constants |

---

### Task 1: Update API types in clawnet-api.ts

**Files:**
- Modify: `src/clawnet-api.ts`

- [ ] **Step 1: Add language to ClawNetSkill**

```typescript
export interface ClawNetSkill {
  // ... existing fields ...
  language?: string; // BCP 47 tag
}
```

- [ ] **Step 2: Add package fields to ClawNetAgentVersion**

```typescript
export interface ClawNetAgentVersion {
  // ... existing fields ...
  manifestOutpoint?: string;
  packageOutputs?: string;
  manifestVout?: number;
  packageType?: string;
}
```

- [ ] **Step 3: Add ClawNetOrganization types**

```typescript
export interface ClawNetOrganizationAgent {
  slug: string;
  role?: string;
  reportsTo?: string;
}

export interface ClawNetOrganization {
  _id: string;
  slug: string;
  name: string;
  displayName: string;
  description: string;
  authorBapId: string;
  agents: ClawNetOrganizationAgent[];
  skills?: string[];
  color?: string;
  icon?: string;
  homepage?: string;
  tags?: string[];
  latestTxId?: string;
  deleted: boolean;
  starCount: number;
  downloadCount: number;
  downloadCountAllTime: number;
  createdAt: number;
  updatedAt: number;
}

export interface ClawNetOrganizationListResponse {
  organizations: ClawNetOrganization[];
  hasMore: boolean;
  cursor?: string;
}

export interface ClawNetOrganizationDetailResponse {
  organization: ClawNetOrganization;
  resolvedAgents: Array<{
    slug: string;
    name: string;
    displayName: string;
    description: string;
    model: string;
    color: string;
    version: string;
    role?: string;
    reportsTo?: string;
  }>;
  resolvedSkills: Array<{
    slug: string;
    name: string;
    description: string;
    version: string;
  }>;
  author: { bapId: string; pubkey: string } | null;
}
```

- [ ] **Step 4: Update ClawNetSearchResult type**

```typescript
export interface ClawNetSearchResult {
  type: "agent" | "skill" | "organization"; // was missing "organization"
  slug: string;
  displayName: string;
  description: string;
  score: number;
}
```

- [ ] **Step 5: Add organization client methods**

Add to `ClawNetClient` interface and `createClawNetClient`:

```typescript
listOrganizations(params?: ListOrganizationsParams): Promise<ClawNetOrganizationListResponse>;
getOrganization(slug: string): Promise<ClawNetOrganizationDetailResponse>;
```

Implementation follows the same pattern as `listAgents`/`getAgent`.

- [ ] **Step 6: Update searchAll type parameter**

```typescript
async searchAll(query: string, type?: "agent" | "skill" | "organization" | "all"): Promise<ClawNetSearchResponse>
```

- [ ] **Step 7: Commit**

```bash
git add src/clawnet-api.ts
git commit -m "OPL-1505: update API types — organizations, language, package fields"
```

---

### Task 2: Update sync logic in worker.ts

**Files:**
- Modify: `src/worker.ts`
- Modify: `src/constants.ts`

- [ ] **Step 1: Add organization entity type constant**

In `src/constants.ts`, add to the entity types:

```typescript
organization: "clawnet-organization",
```

- [ ] **Step 2: Add organization sync to performSync**

Find `performSync()` in `src/worker.ts`. After the skills sync loop, add organization sync:

```typescript
// Sync organizations
const orgs = await client.listOrganizations({ limit: 100, sort: "updated" });
for (const org of orgs.organizations) {
  await ctx.entities.upsert({
    externalId: org.slug,
    type: entityTypes.organization,
    data: org,
  });
}
```

- [ ] **Step 3: Add organization to data handlers**

Find the data handler that serves agent/skill lists to the UI. Add a handler for organizations:

```typescript
if (request.type === "organizations") {
  const entities = await ctx.entities.list({ type: entityTypes.organization });
  return { organizations: entities.map(e => e.data) };
}
```

- [ ] **Step 4: Commit**

```bash
git add src/worker.ts src/constants.ts
git commit -m "OPL-1505: sync organizations from registry, add entity type"
```

---

### Task 3: Update marketplace UI

**Files:**
- Modify: `src/ui/index.tsx`

- [ ] **Step 1: Add Organizations tab**

Find the marketplace tabs (Agents/Skills). Add a third tab for Organizations. Follow the exact same pattern as the existing tabs.

- [ ] **Step 2: Add organization list component**

Create an organization card that shows: name, description, agent count, skill count, author, stars. Follow the existing agent card pattern.

- [ ] **Step 3: Add language badge to agent/skill cards**

Where agent and skill cards are rendered, add a language badge if the item has a `language` field:

```tsx
{item.language && (
  <span className="text-xs bg-muted px-1.5 py-0.5 rounded font-mono">
    {item.language}
  </span>
)}
```

- [ ] **Step 4: Add on-chain indicator**

If the latest version has `manifestOutpoint`, show a small on-chain indicator (similar to how `latestTxId` might be shown).

- [ ] **Step 5: Update search to include organizations**

Update the search filter to allow searching across organizations when the Organizations tab is active.

- [ ] **Step 6: Commit**

```bash
git add src/ui/index.tsx
git commit -m "OPL-1505: organizations tab, language badges, on-chain indicators"
```

---

### Task 4: Build, test, and version bump

**Files:**
- Modify: `package.json`

- [ ] **Step 1: Type-check**

```bash
cd ~/code/clawnet-paperclip-plugin && bun run build
```

- [ ] **Step 2: Lint**

```bash
bun run lint
```

- [ ] **Step 3: Version bump**

Bump version in `package.json`.

- [ ] **Step 4: Commit and push**

```bash
git add -A
git commit -m "OPL-1505: release — registry alignment update"
git push
```

---

## Dependency Order

```
Task 1 (API types) → Task 2 (sync) → Task 3 (UI) → Task 4 (build/release)
```

Sequential — each task builds on the previous.

## Verification

1. `bun run build` — plugin compiles
2. `bun run lint` — no lint errors
3. Plugin loads in Paperclip — marketplace shows Agents, Skills, Organizations tabs
4. Sync pulls organizations from ClawNet
5. Language badges appear on skills that have them
6. Search includes organizations

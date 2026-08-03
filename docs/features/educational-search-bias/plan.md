# Feature: Search History Recording & Educational Content Biasing

## Overview

When a user searches in Vane, the system should record their request (query text, intent signals, and context) and use that information to bias the AI toward fetching more informational or educational content. This feature serves two purposes:
1. **Remember what users care about** so the search experience improves over time.
2. **Surface high-quality, authoritative, educational sources** when the user's intent suggests they are seeking to learn, research, or understand a topic deeply.

This creates a flywheel: the more someone uses Vane for learning, the better Vane gets at finding the right depth of content for them.

---

## Why This Feature Is Helpful

- **Research & learning is Vane's sweet spot**: Vane already supports academic search, quality mode, and citations. Users who come for deep answers benefit from a system that *recognizes* their intent and doubles down on authoritative sources.
- **The web search bias problem**: General web search is optimized for recency, popularity, and engagement, not *pedagogical quality*. By biasing the AI — both in the queries it generates and the sources it selects — we can surface better content without relying on users to manually toggle "academic mode" every time.
- **A nascent personalization system**: Vane has no auth today, but searches are saved in local SQLite. We can build a lightweight preference/profile layer on top of existing chat/message storage, creating a foundation for richer personalization later.
- **Differentiation**: Most AI search engines treat every query as a one-shot interaction. Building an implicit learning loop makes Vane feel smarter and more tailored.

---

## How the Bias Works

### What "Educational/Informational" Means
| Aspect | Examples |
|--------|----------|
| Source authority | Prefer `.edu`, academic journals, Wikipedia, high-quality explainer sites |
| Tone | Explainer, tutorial, reference, comparison, deep-dive |
| Depth | Multi-paragraph explanations with citations, not headlines or hot takes |
| User query patterns | "how does X work", "explain Y", "what is Z", "compare A and B", "history of", "tutorial" |

### Points of Intervention

The bias is injected at **four stages** of the existing SearchAgent pipeline:

```
User Query
    │
    ▼
┌─────────────────┐  ← 1. Classification: detect educational intent
│   Classifier    │     + enrich with user's historical preference
└────────┬────────┘
         │
    ├────┴────┬───────────────┐
    ▼         ▼               ▼
 Research  Widgets       Query Store
    │                        │
    ▼                        │
┌─────────────────────┐      │
│  Researcher Agent   │  ← 2. Search Query Formation:
│  (web/academic/etc) │     augment queries with educational
└──────────┬──────────┘     qualifiers when intent is high
           │
    ┌──────┴──────┐
    ▼             ▼
 Sources      Findings
    │
    ▼
┌─────────────────┐  ← 3. Source Ranking:
│   Writer LLM    │     pass educational bias hint
└─────────────────┘     to source selection / re-ranking
    │
    ▼
 Response (with citations)
```

---

## Implementation Plan

---

### Phase 1: Data Layer — Recording Search Requests

#### 1.1 New database table: `search_profiles`
```sql
CREATE TABLE search_profiles (
    id          INTEGER PRIMARY KEY,
    sessionId   TEXT NOT NULL,     -- browser session or local device ID
    query       TEXT NOT NULL,     -- raw query text
    standaloneQuery TEXT,          -- the classifier's standalone reformulation
    intentTags  TEXT,              -- JSON array: ["educational", "how_to", ...]
    sourceBias  TEXT,              -- 'neutral' | 'educational' | 'academic'
    createdAt   TEXT NOT NULL
);
```

> **Note**: Vane has no auth. We use a session/device fingerprint (e.g., a UUID stored in localStorage/cookie) as the user key. If authentication arrives later, `sessionId` can be migrated to `userId`.

#### 1.2 Extend `messages` table
Add a column so that every message can carry the detected educational bias:
```typescript
// In src/lib/db/schema.ts
export const messages = sqliteTable('messages', {
    ...existingColumns,
    detectedIntent: text('detectedIntent', { mode: 'json' })
        .$type<string[]>()
        .default(sql`'[]'`),
    educationalBiasLevel: text({ enum: ['none', 'low', 'medium', 'high'] })
        .default('none'),
});
```

#### 1.3 Classification enrichment
In `src/lib/agents/search/classifier.ts` and its Zod schema, add two new fields:
```typescript
educationalIntent: z.enum(['none', 'low', 'medium', 'high'])
    .describe('How likely this query is seeking educational or deeply informational content.'),
```

Update `classifierPrompt` to instruct the model on what counts as educational/informational intent — e.g., queries with "how to", "explain", "what is", "history of", "tutorial", "compare", "mechanism of", "deep dive", etc.

#### 1.4 Store on every search
In `src/lib/agents/search/index.ts`, after classification completes:
```typescript
// Pseudocode
await db.insert(searchProfiles).values({
    sessionId: session.id, // or device fingerprint
    query: input.followUp,
    standaloneQuery: classification.standaloneFollowUp,
    intentTags: JSON.stringify([
        ...(classification.educationalIntent !== 'none' ? ['educational'] : []),
        ...(classification.classification.academicSearch ? ['academic'] : []),
    ]),
    sourceBias: classification.educationalIntent === 'high' ? 'educational' : 'neutral',
    createdAt: new Date().toISOString(),
});
```

#### 1.5 Session preference aggregation (lightweight)
Add a small helper in `src/lib/searchProfile.ts` that, given a `sessionId`, computes:
- What fraction of recent searches (last N) were "educational"?
- Is the current session on a "learning streak"?

This produces a **Session Preference Score** (0–1). If it crosses a threshold, future searches are pre-biased even before the classifier runs.

---

### Phase 2: Biasing the Search

#### 2.1 Search query augmentation
In `src/lib/agents/search/researcher/actions/search/webSearch.ts` and other search actions, when `educationalIntent` is `medium` or `high`, automatically append educational qualifiers to generated queries:

```typescript
function augmentWithEducationalBias(queries: string[], intentLevel: string): string[] {
    if (intentLevel === 'none') return queries;

    const suffixMap = {
        low: ['explained', 'overview'],
        medium: ['comprehensive guide', 'in depth explanation'],
        high: ['site:.edu OR site:.ac.uk', 'research paper', 'textbook chapter'],
    };

    return queries.map(q => ...);
}
```

For academic search (`academicSearch.ts`), if educational intent is high, switch to broader scholarly queries or add `review article` / `survey` terms.

#### 2.2 Source re-ranking by authority
In `src/lib/scraper.ts` or the search result processing layer, add a lightweight **authority heuristic**:
- Boost domains known for educational content (`.edu`, `arxiv.org`, `wikipedia.org`, `coursera.org`, `khanacademy.org`, `nature.com`, etc.)
- Penalize sources flagged as "low depth" (e.g., short articles without citations, clickbait headlines).

This is a score multiplier applied during deduplication in the Researcher index step:
```typescript
// In Researcher result filtering step
const authorityBoost = computeAuthorityBoost(result.metadata.url, educationalBiasLevel);
result.score = result.score * authorityBoost;
```

#### 2.3 Writer prompt biasing
In `src/lib/prompts/search/writer.ts`, when `educationalBiasLevel` is `medium` or `high`, append an instruction:

```
<educational_bias>
The user appears to be seeking an in-depth, educational answer. Your response should:
- Prioritize clarity and structure: define terms before using them, use analogies where helpful.
- Cite authoritative sources and, where possible, explain *why* a source is credible.
- Go beyond a surface-level summary. If the topic has depth (history, mechanisms, trade-offs), include it.
- When sources disagree, present the nuance rather than collapsing it.
</educational_bias>
```

---

### Phase 3: Settings & Transparency

#### 3.1 UI toggle
In the Settings > Preferences panel, add:
- **"Auto-detect educational intent"** (default: ON) — lets users opt out of the system inferring their intent.
- **"Always prefer educational sources"** (default: OFF) — a manual override for users who consistently want this behavior.

#### 3.2 Inform the user
When the system activates high educational bias, show a small UI indicator (like "🎓 Educational mode active") so the bias is transparent, not invisible.

---

## File-by-File Changes

| File | Change |
|------|--------|
| `src/lib/db/schema.ts` | Add `search_profiles` table; extend `messages` with intent columns. Run a new Drizzle migration. |
| `src/lib/agents/search/types.ts` | Add `educationalIntent` to `ClassifierOutput` and `SearchAgentConfig`. |
| `src/lib/agents/search/classifier.ts` | Extend Zod schema, update prompt to request educational-intent classification. |
| `src/lib/prompts/search/classifier.ts` | Add educational-intent label instructions. |
| `src/lib/agents/search/index.ts` | After classification, persist profile + intent to DB. Read session preference before classifying as needed. |
| `src/lib/agents/search/researcher/actions/search/webSearch.ts` | Augment queries based on intent level. |
| `src/lib/agents/search/researcher/actions/search/academicSearch.ts` | Augment queries for educational depth. |
| `src/lib/agents/search/researcher/index.ts` | Pass educational bias into researcher config; apply authority boost during result deduplication. |
| `src/lib/prompts/search/writer.ts` | Conditionally inject `<educational_bias>` instructions. |
| `src/lib/agents/search/api.ts` | Mirror the persistence logic for the API route. |
| `src/lib/searchProfile.ts` | New: session preference scorer and helper. |
| `src/components/Settings/Sections/Preferences.tsx` | Add toggles for auto-detect and manual override. |
| `src/lib/config/types.ts` | Add preference keys to config registry if stored server-side. |
| `src/components/Chat.tsx` / `ChatWindow.tsx` | Show active-bias indicator when educational mode is engaged. |

---

## Open Questions

1. **User identity**: With no auth, do we tie profiles to `localStorage` session IDs or browser cookies? How do we handle privacy-conscious users who clear storage?
2. **Authority scoring whitelist**: Is there a maintained list of high-quality domains, or do we bootstrap one? Should it be user-editable?
3. **Negative bias**: Should we ever *reduce* educational bias if a user's recent queries are clearly shallow (e.g., "weather", "stock")? (Probably yes — the preference score should respond dynamically.)
4. **Cross-session warmth**: Should the first search of a new session be influenced by past sessions, or start cold each time?
5. **Scope of "educational"**: Does this include professional/technical docs (e.g., reading MDN, API docs), or strictly pedagogical content?

---

## Rollback / Iteration Strategy

- The feature is gated entirely by classification output. If the classifier mis-detects intent, the fallback behavior is the current default behavior (no bias). So the worst case is "no worse than today."
- Start with query augmentation and writer prompt injection only (no source re-ranking). Measure whether answers *feel* more educational without accidentally dropping relevant non-educational sources.
- Add authority scoring in a second pass once we have data on what users are actually clicking.

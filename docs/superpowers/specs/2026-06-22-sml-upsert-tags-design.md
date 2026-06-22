# SML Tags: Upsert Write + List Filter

**Issue:** [search-team#14968](https://github.com/elastic/search-team/issues/14968)
**Date:** 2026-06-22
**Plugin:** `x-pack/platform/plugins/shared/agent_context_layer`

## Problem

The SML upsert route (`PUT /internal/agent_context_layer/sml/:id`) does not accept a `tags` field.
The search route already filters by `tags`; the list route does not. There is no way to write tags
onto a KI at creation time, and no way to retrieve KIs by tag via the list route.

The driving use case: workflows that generate KIs from a specific source (e.g. Claude Code OTel
sessions) need to tag their entries so they can later be retrieved as a group — e.g. to find the
most recently written KI of that type and use its `updated_at` as a watermark for the next run.

## Scope

Two changes in one PR:

1. **Upsert write** — add `tags` to the upsert route body schema and thread it through to storage
2. **List filter** — add a `tags` query param to the list route, comma-delimited, filtered server-side

## Design

### Type contract

| Layer | Field | Type | Notes |
|---|---|---|---|
| `SmlDocumentInput` (input) | `tags?` | `string[] \| undefined` | Optional; absent = preserve existing tags on update |
| `SmlDocument` (stored) | `tags?` | `string[] \| undefined` | Already exists; absent = never tagged |
| `SmlHttpItem` (response) | `tags` | `string[]` | Always present; `doc.tags ?? []` normalizes absent |

### Upsert route changes (`PUT /internal/agent_context_layer/sml/:id`)

**`common/http_api/sml.ts`**
- Add `tags: string[]` to `SmlHttpItem` (required, always present in responses)

**`server/services/sml/types.ts`**
- Add `tags?: string[]` to `SmlDocumentInput`
- Add `tags?: string[]` to `SmlService.listDocuments` params signature

**`server/routes/upsert.ts`**
- Add to body schema:
  ```ts
  tags: schema.maybe(schema.arrayOf(schema.string({ maxLength: 200 }), { maxSize: 100 }))
  ```

**`server/services/sml/sml_service.ts`** — `upsertDocument()`
- On create: set `tags: document.tags ?? []` (normalize absent to empty array for new docs)
- On update: set `tags: document.tags ?? existing.tags` (preserve existing if caller omits field)

**`server/routes/common.ts`** — `toSmlHttpItem()`
- Add `tags: doc.tags ?? []` to the returned object

### List route changes (`GET /internal/agent_context_layer/sml`)

**`server/routes/list.ts`**
- Add to query schema:
  ```ts
  tags: schema.maybe(schema.string({ maxLength: 2000 }))
  ```
- Split on commas before calling `listDocuments`: `tags?.split(',').map(t => t.trim()).filter(Boolean)`

**`server/services/sml/sml_service.ts`** — `listDocuments()`
- Accept `tags?: string[]` param
- When present and non-empty, add a `terms: { tags: tags }` filter clause to the ES query
  (same pattern as the existing `type` and `originId` filter clauses)

### What is NOT changing

- `SmlDocument.tags` stays optional (no migration needed; existing docs without `tags` are unaffected)
- The search route (`POST /internal/agent_context_layer/sml/_search`) already handles `filters.tags` — no changes there
- No changes to the storage schema — `tags: types.keyword({ normalizer: 'lowercase' })` already exists

## Tests

**`server/routes/upsert.test.ts`**
- New: "forwards tags to sml.upsertDocument when provided"

**`server/routes/list.test.ts`**
- New: "passes tags filter to sml.listDocuments when provided (comma-delimited)"

## Out of scope (companion)

The issue notes a companion change: adding a `tags` filter to the list route is tracked here; adding
`tags` write support to the upsert route is the primary issue. Both are included in this PR given
their small combined size.

A separate discussion is warranted on whether the list route (`GET`) should be converted to `POST`
for consistency with the search route — deferred, not blocking.

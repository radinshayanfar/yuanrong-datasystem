# Codebase Map — Semantic Search Protocol

This document tells Legion commands (and any consumer) how to query the map dataset for
`yuanrong-datasystem` and then read real source before acting. Search is retrieval over map
metadata plus source reads — **no embeddings, vector DB, API keys, or network services**.

## Required Artifacts

- `.planning/CODEBASE.md` — human-readable architecture, risks, runbook, conventions.
- `.planning/codebase/index.jsonl` — one JSON object per retrievable chunk.
- `.planning/codebase/symbols.json` — entry points, apis (protos), modules, tests, config, dependencies, ownership, risk_areas.
- `.planning/config/directory-mappings.yaml` — directory category mappings.

> This repo also maintains authoritative human-curated guidance under `.repo_context/`
> (start at `.repo_context/index.md`). Prefer it for deep module design; use this map for fast retrieval.

## Query Planning (Section 18.1)

Normalize the query into:

```
query = {
  terms:        important nouns/verbs/feature names (e.g. "eviction", "scale down", "H2D", "slot recovery"),
  path_hints:   explicit files/dirs (e.g. src/datasystem/worker/object_cache),
  symbol_hints: classes/functions/protos (e.g. OcMetadataManager, worker_object.proto),
  domain_hints: object-cache | kv | stream | hetero-npu | cluster | etcd | l2cache-slot | rdma | logging | metrics | build | tests
}
```

## Retrieval Order (Section 18.2)

1. Match explicit **path hints** in `index.jsonl` (`path`, `related_files`) and `symbols.json`.
2. Match **symbol hints** in `symbols.json` (`symbols`, module/api names).
3. Match **terms/aliases** in `index.jsonl` (`keywords`, `aliases`, `summary`).
4. Scan `CODEBASE.md` section headings for broad architecture context.
5. **Read the original source files** for the top matches before writing plans, reviews, or code.

## Ranking (Section 18.3)

Rank by: exact path/symbol match > keyword/alias overlap > same domain as context >
risk level & fan-in relevance > git-hotspot recency. Return at most 5 primary chunks and 5
"read next" paths unless broader analysis is explicitly requested.

## Result Format

```markdown
## Map Search Results

| Rank | Chunk | Path | Lines | Kind | Why it matched |
|------|-------|------|-------|------|----------------|
| 1 | map:master-oc-metadata:001 | src/datasystem/master/object_cache/oc_metadata_manager.cpp | 1-4819 | module | alias "metadata manager"; domain object-cache |

### Read Next
- `src/datasystem/master/object_cache/oc_metadata_manager.cpp`
- `.repo_context/modules/runtime/object-cache-eviction/README.md`
```

## Example: `/legion:map --query "object cache eviction lifecycle"`

```markdown
## Map Search Results

| Rank | Chunk | Path | Lines | Kind | Why it matched |
|------|-------|------|-------|------|----------------|
| 1 | map:worker-eviction:001 | src/datasystem/worker/object_cache/worker_oc_eviction_manager.cpp | — | module | keyword "eviction"; alias "cache eviction" |
| 2 | map:master-oc-metadata:001 | src/datasystem/master/object_cache/oc_metadata_manager.cpp | 1-4819 | module | eviction-driven metadata cleanup |
| 3 | map:client-object:001 | src/datasystem/client/object_cache/object_client_impl.cpp | 1-4132 | module | object-cache domain, ref/lifecycle |

### Read Next
- `src/datasystem/worker/object_cache/worker_oc_eviction_manager.cpp`
- `src/datasystem/worker/object_cache/worker_oc_spill.cpp`
- `.repo_context/modules/runtime/object-cache-eviction/README.md`
```

## Consumer Safety Rules (Section 18.6)

- Do not treat chunk summaries as source of truth for code edits — read the file.
- Do not cite stale map data as current without checking freshness (header metadata in `CODEBASE.md`).
- Do not load the entire index into an agent prompt when a targeted query suffices.
- If query results conflict with current source, **current source wins** — refresh the map
  (`/legion:map --refresh`). This codebase is very active (~674 commits/90d), so re-check freshness.

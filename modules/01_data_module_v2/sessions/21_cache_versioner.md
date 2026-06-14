# Session 21 — Cache Manager + Versioner (`representation/cache_manager.py` + `versioner.py`)

**Date:** 2026-06-14
**Session number:** 21 of 36
**Mode:** Understand (119 + 100 lines — content-addressed caching, schema versioning)
**Estimated study time:** 20 min teaching + 10 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §1 Cache Manager (119 lines)

### Cache Key

```
(sha256_of_source, schema_version, extractor_version)
```

A change to ANY of these three invalidates the cache for that file.

### Cache Layout

```
data/representations/<source>/
    <sha256>.pt          — PyG Data graph
    <sha256>.tokens.pt   — windowed GraphCodeBERT tokens
    <sha256>.rep.json    — sidecar (the canonical cache record)
```

### Public API

| Function | Purpose |
|---|---|
| `is_cached(dir, sha256, schema_ver, extractor_ver) → bool` | Checks sidecar exists + versions match + both .pt files exist |
| `record_cache_hit(dir, sha256)` | No-op placeholder for future LRU tracking |
| `invalidate(dir, sha256)` | Removes all three files |
| `list_cached_sha256s(dir) → list[str]` | Returns sha256s with all three files present |
| `stale_entries(dir, schema_ver, extractor_ver) → list[str]` | Returns sha256s with mismatched versions |
| `evict_stale(dir, schema_ver, extractor_ver) → int` | Removes stale entries, returns count |

### How the Sidecar Works

```python
def is_cached(output_dir, sha256, schema_version, extractor_version):
    rep_path = output_dir / f"{sha256}.rep.json"
    if not rep_path.exists():
        return False
    try:
        sidecar = json.loads(rep_path.read_text())
    except (json.JSONDecodeError, OSError):
        return False
    return (
        sidecar.get("schema_version") == schema_version
        and sidecar.get("extractor_version") == extractor_version
        and (output_dir / f"{sha256}.pt").exists()
        and (output_dir / f"{sha256}.tokens.pt").exists()
    )
```

All three files MUST be present AND the versions MUST match. Missing `.pt` with existing sidecar → cache miss (partial write from a crashed run).

### Eviction Flow

1. Developer increments `FEATURE_SCHEMA_VERSION` from `"v9"` to `"v10"`
2. Orchestrator starts → calls `evict_stale(dir, "v10", extractor_ver)`
3. `stale_entries` scans all `.rep.json` files, finds ones with `schema_version="v9"`
4. `invalidate` removes all three files for each stale sha256
5. Orchestrator re-extracts all graphs with the new schema

---

## §2 Versioner (100 lines)

Manages the `extractor_version` string embedded in `.rep.json` sidecars. Format: `X.Y.Z` (major.minor.patch) following semantic versioning for the extraction pipeline.

### Version Policy

| Bump | When | Effect on cache |
|---|---|---|
| Major | Breaking schema change (new node type, feature removed) | Full re-extraction required |
| Minor | Non-breaking feature addition (new feature dim) | Re-extraction recommended |
| Patch | Bug fix that doesn't change feature shape | Re-extraction optional |

The orchestrator embeds the version in every `.rep.json` so extraction provenance is always traceable.

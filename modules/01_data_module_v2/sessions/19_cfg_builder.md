# Session 19 — CFG Builder (`representation/cfg_builder.py`)

**Date:** 2026-06-14
**Session number:** 19 of 36
**Mode:** Understand (309 lines — standalone per-function CFG artifact, opt-in via --emit-cfg)
**Estimated study time:** 30 min teaching + 12 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0.5 What You Already Know

- **CFG in graph_extractor** (Session 18): `_build_control_flow_edges` builds CFG nodes as part of the PyG graph (CONTAINS + CONTROL_FLOW edges). That's the PRIMARY CFG that feeds the GNN.
- **This CFG builder is SEPARATE**: it produces a standalone JSON artifact (`.cfg.json`) NOT merged into the training graph. Primary consumer: Stage 6's `complexity_proxy_risk` detector.
- **Slither IR surface**: uses the same `func.nodes` + `node.sons` API as `graph_extractor.py`.

---

## §1 Why a Standalone CFG Artifact? (lines 1–8, 23–33)

The graph_extractor's CFG is embedded in the PyG `Data` object — optimised for GNN message passing (tensor format, not human-readable). The standalone CFG artifact is:
- **JSON-serializable** — readable by any consumer without PyTorch
- **Per-function** — each function gets its own `CfgFunction` with `nodes` + `edges`
- **Self-describing** — node types stored as strings (e.g. `"CFG_NODE_WRITE"`), not integers
- **Opt-in** — only built when `--emit-cfg` flag is passed (Slither adds ~0.5s per contract)

**Design decisions (lines 23–33):**
- Node types use same `NODE_TYPES` string keys as `graph_schema.py` for consistency
- Keyed by `canonical_name` (Slither's stable cross-run identifier) for Stage 6 joining
- Loop detection uses DFS-back-edge counting — O(N+E) per function

---

## §2 Data Structures (lines 46–89)

```python
@dataclass
class CfgNode:
    index: int                # Positional index within function's sorted node list
    type: str                 # e.g. "CFG_NODE_WRITE", "CFG_NODE_CALL"
    source_lines: list[int]   # Slither source_mapping.lines
    expression: str           # str(slither_node) — human-readable label

@dataclass
class CfgEdge:
    source: int               # CfgNode.index (source)
    target: int               # CfgNode.index (target)
    edge_type: str            # "CONTROL_FLOW" | "CALL_ENTRY" | "RETURN_TO"

@dataclass
class CfgFunction:
    canonical_name: str       # Slither's stable cross-run identifier
    nodes: list[CfgNode]      # sorted by (source_line, node_id)
    edges: list[CfgEdge]      # control-flow edges between nodes

@dataclass
class CfgArtifact:
    contract_name: str
    source_file: str
    functions: list[CfgFunction]
    metadata: dict            # extraction stats: duration_s, n_nodes, n_edges, ...
```

All dataclasses are `JSON-serializable via dataclasses.asdict()`.

---

## §3 Key Differences from graph_extractor CFG

| Aspect | graph_extractor CFG | cfg_builder CFG artifact |
|---|---|---|
| Format | PyG Data tensor (`.pt`) | JSON serializable |
| Node types | Integer IDs (0–13) | String names |
| Scope | Single graph per contract | Per-function subgraphs |
| Edge types | All 12 types | CONTROL_FLOW + CALL_ENTRY + RETURN_TO |
| Loop detection | Binary `has_loop` feature | DFS back-edge count (detailed) |
| Consumer | GNN model | Stage 6 `complexity_proxy_risk` |

---

## §4 Integration Path

The orchestrator calls `build_cfg()` only when `emit_cfg=True` (default: False). The artifact is written alongside the graph `.pt` and tokens `.pt` files. The independent artifact means Stage 6 can run its analysis without re-extracting or depending on the GNN graph format.

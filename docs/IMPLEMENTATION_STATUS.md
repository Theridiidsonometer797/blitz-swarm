# Implementation Status

A clear inventory of what is implemented, partially implemented, and proposed.

## Fully Implemented

- Parallel agent execution via asyncio + subprocess
- Dynamic agent planning (LLM-based with heuristic fallback)
- 5 agent roles (researcher, critic, fact-checker, judge, synthesizer)
- Redis-backed blackboard with role-filtered context
- No-Redis in-memory fallback
- Consensus voting (unanimous "ready" required)
- Holdout override after 3 rounds
- Dissent extraction and formatting
- Convergence report generation
- Sentence-transformer embeddings (MiniLM-L6-v2)
- TOML-based configuration
- Per-run metrics logging (metrics.jsonl)
- Output generation (markdown documents)

## Partially Implemented

- **G-Memory Tier 1 (interaction traces):** Agent outputs are stored per round, but not in the documented three-tier schema. The storage is functional but not structured as described in the architecture docs.
- **SQLite persistence:** Database exists but the full retrieval pipeline (embed → expand → traverse) is not connected to the orchestrator.
- **LanceDB vector store:** Embedder exists but vector storage is not integrated into the run loop.

## Proposed (Not Yet Implemented)

- G-Memory Tier 2 (task-level semantic graphs)
- G-Memory Tier 3 (LLM-distilled cross-task insights)
- Memory retrieval pipeline (embed → expand → traverse)
- CRDT-based conflict resolution (LWW-Register, OR-Set, G-Counter)
- Arbiter agents for high-stakes memory conflicts
- Retention/eviction scoring (access frequency + reward + dependency + novelty)
- Temporal validity windows for stored knowledge
- Delta compression for iterative refinement
- Cross-run learning / compounding intelligence
- Web search capability

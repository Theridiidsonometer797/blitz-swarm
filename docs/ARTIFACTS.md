# Artifacts

## Core System

| File | Purpose | Status |
|------|---------|--------|
| `orchestrator.py` | Main lifecycle: spawn, blast, check consensus, finalize | Implemented |
| `agents.py` | 5 agent roles, dynamic planning, topic decomposition | Implemented |
| `consensus.py` | Convergence voting, holdout override, dissent preservation | Implemented |
| `blackboard.py` | Redis-backed shared memory, role-filtered context building | Implemented |
| `embedder.py` | Sentence-transformers wrapper (MiniLM-L6-v2) | Implemented |
| `config.py` | Configuration dataclasses, TOML loading | Implemented |

## Configuration

| File | Purpose |
|------|---------|
| `blitz.toml` | Runtime configuration (rounds, model, timeout, memory, Redis) |
| `pyproject.toml` | Package metadata and dependencies |

## Run Data

| File | Contents |
|------|----------|
| `metrics.jsonl` | 2 run entries with per-round metrics, quality scores, cost, timing |
| `output/` | Generated research documents (gitignored) |
| `memory.db` | SQLite database (gitignored) |

## Documentation

| File | Purpose |
|------|---------|
| `BLITZ-SWARM.md` | Detailed architecture specification (agent roles, memory tiers, convergence) |
| `Memory architecture for a parallel AI agent swarm.md` | Literature review: three-layer storage model, conflict resolution, retention scoring |
| `Building Hierarchical agent Memory from G-Memory's blueprint.md` | Literature review: G-Memory implementation guide, NeurIPS 2025 citations |

## Research Documents (docs/)

| File | Purpose |
|------|---------|
| `RESEARCH_QUESTIONS.md` | 5 research questions with status |
| `METHODOLOGY.md` | Experiment structure, variables, caveats |
| `EVALUATION.md` | Metric definitions and data status |
| `ABLATIONS.md` | 6 design decisions with rationale |
| `LIMITATIONS.md` | 10 known limitations |
| `ROADMAP.md` | 4 validated next steps, 4 speculative ideas |
| `CLAIMS_AND_EVIDENCE.md` | Claims mapped to evidence |
| `IMPLEMENTATION_STATUS.md` | What is implemented vs proposed |

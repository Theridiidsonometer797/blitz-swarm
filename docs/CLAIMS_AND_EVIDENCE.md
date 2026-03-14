# Claims and Evidence

## Architecture Claims

### "All agents fire simultaneously"
- **Evidence:** orchestrator.py uses `asyncio.to_thread()` for concurrent subprocess execution
- **Strength:** Strong — directly observable in code
- **Caveats:** "Simultaneous" is approximate — OS scheduling determines actual parallelism

### "Dissent is preserved, not suppressed"
- **Evidence:** consensus.py `extract_dissent()` and `format_dissent_section()` functions
- **Strength:** Strong — implemented in code
- **Caveats:** Whether preserved dissent adds value to output is not evaluated

### "G-Memory's three-tier hierarchical memory"
- **Evidence:** Tier 1 (interaction traces) stored in SQLite. Tiers 2-3 referenced in documentation only.
- **Strength:** Weak — only 1 of 3 tiers is implemented
- **Caveats:** Claiming "implements G-Memory" is an overstatement. More accurate: "implements G-Memory Tier 1 and proposes Tiers 2-3."

### "Compounding intelligence — each task makes the swarm smarter"
- **Evidence:** None. Memory pipeline is incomplete. Only 2 runs exist. No before/after comparison.
- **Strength:** Aspirational — not supported by data
- **Caveats:** This claim should be softened to "designed to enable compounding intelligence" until validated.

## Performance Claims

### "Quality score of 7.0 on SQLite WAL topic"
- **Evidence:** metrics.jsonl run 1
- **Strength:** Moderate — single run, single LLM evaluator
- **Caveats:** n=1. The same topic might score differently on a second run.

## Literature Claims

### "G-Memory achieves up to 20.89% improvement in task success"
- **Evidence:** Cited from Zhang et al. (NeurIPS 2025) in BLITZ-SWARM.md
- **Strength:** Strong (for the cited paper) — but this is their finding, not ours
- **Caveats:** This result was achieved in their system, not in Blitz-Swarm. Our system has not replicated this finding.

### "Google's blackboard MAS showed 13-57% relative improvement"
- **Evidence:** Cited in BLITZ-SWARM.md
- **Strength:** Strong (for the cited paper) — their result, not ours
- **Caveats:** Same as above. Literature references should be clearly distinguished from own findings.

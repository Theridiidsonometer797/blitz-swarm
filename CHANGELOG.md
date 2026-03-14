# Changelog

## [0.1.0] — 2026-03-12

### Initial Release

- Parallel multi-agent orchestration via asyncio + subprocess
- 5 agent roles: researcher, critic, fact-checker, quality judge, synthesizer
- Dynamic agent planning (LLM-based with heuristic fallback)
- Consensus convergence with holdout override
- Dissent preservation in final output
- Redis-backed blackboard with no-Redis fallback
- G-Memory Tier 1 (interaction traces)
- Sentence-transformer embeddings (MiniLM-L6-v2)
- TOML-based configuration
- Per-run metrics logging

### Research Documentation (same day)

- Added memory architecture literature review
- Added G-Memory blueprint documentation

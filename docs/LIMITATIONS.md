# Limitations

## Data Limitations

1. **n=2.** Two runs. One succeeded, one failed catastrophically. No statistical analysis, no trends, no aggregate conclusions. Everything in this repo is architectural, not empirical.

2. **No baseline comparison.** No runs of the same topics through a sequential pipeline or single-agent approach. Claims about parallel execution benefits are theoretical.

3. **No cost optimization.** Run 1 cost $2.26 for a single topic. At scale (5 rounds, 12 agents), costs could be substantial. No cost/quality tradeoff analysis exists.

## Architecture Limitations

4. **Incomplete memory pipeline.** G-Memory Tier 1 is stored but Tiers 2-3 (semantic graphs, distilled insights) are not implemented. The "compounding intelligence" claim is aspirational, not demonstrated.

5. **Cascading agent failures.** Run 2 demonstrates that errors in early rounds propagate — if Round 1 context is malformed, Rounds 2-3 fail entirely. No circuit breaker or isolation exists.

6. **Consensus never reached.** In neither run did all agents vote "ready." The convergence mechanism may need tuning, or the topics may have been too broad for 3-round convergence.

7. **No web search.** Unlike Research Swarm, Blitz-Swarm has no web search capability. All research is from the LLM's training data, which limits factuality.

## External Validity

8. **Claude-dependent.** All agents use Claude via CLI. Behavior with other models is unknown.

9. **Solo operator.** All runs, design decisions, and assessments by a single person.

10. **Referenced architecture files do not exist.** The BLITZ-SWARM.md references a memory/ subdirectory with schema.sql, models.py, writer.py, reader.py. These files do not exist in the repository. The memory pipeline is partially implemented in the main codebase but not as a separate module.

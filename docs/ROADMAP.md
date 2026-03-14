# Roadmap

## Validated Next Steps

1. **Run more experiments.** n=2 is insufficient for any analysis. Run at least 10 topics to establish baseline quality and convergence behavior.

2. **Fix cascading failures.** Run 2 showed that agent errors in Round 1 propagate to all subsequent rounds. Add error isolation (skip failed agents in later rounds rather than passing their malformed output).

3. **Direct comparison with Research Swarm.** Run identical topics through both parallel (Blitz-Swarm) and sequential (Research Swarm) pipelines to test RQ1.

4. **Implement memory Tiers 2-3.** The G-Memory architecture is documented but only Tier 1 is functional. Completing the pipeline is prerequisite for testing RQ4.

## Speculative Ideas

5. **Adaptive round limits.** Instead of fixed max_rounds, let the system estimate required rounds from topic complexity and agent count.

6. **Cross-model consensus.** Run researchers on different model families (Claude, GPT-4, open-source) to increase viewpoint diversity.

7. **Web search integration.** Port Research Swarm's MCP search server to give Blitz-Swarm agents real-time web access.

8. **CRDT-based conflict resolution.** The architecture documentation describes CRDT patterns (LWW-Register, OR-Set) for blackboard conflicts. These are proposed but not implemented.

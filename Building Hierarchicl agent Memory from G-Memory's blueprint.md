# Building hierarchical agent memory from G-Memory's blueprint

**G-Memory (NeurIPS 2025 Spotlight) is the first purpose-built hierarchical memory system for multi-agent swarms, and its three-tier graph architecture — Interaction Graph, Query Graph, Insight Graph — offers the most rigorous foundation for building shared memory into a subprocess-based agent system.** This guide reverse-engineers the paper's formal algorithms into concrete Python data structures, adapts them for a Redis + SQLite + LanceDB stack with 5–50 CLI-invoked agents, and maps out every implementation decision a solo developer needs to make. G-Memory achieves **up to 20.89% improvement** on embodied action tasks and up to 10.12% on knowledge QA, consistently outperforming every baseline across five benchmarks and three agent frameworks — making it the only memory system that never degrades multi-agent performance.

The repository at `github.com/bingreeky/GMemory` is research-grade code (16 commits, 2 contributors, ~14.6% Python), built atop the AgentSquare benchmark framework. It is not production-ready. What follows is a production-adapted implementation guide that preserves G-Memory's core algorithms while replacing its in-memory research infrastructure with durable, concurrent-safe storage.

---

## 1. G-Memory's three tiers as concrete data models

G-Memory models multi-agent memory as three nested graph structures, each at a different level of abstraction. Understanding their exact structure is the foundation for everything else.

### Interaction Graph (lowest tier — raw dialogue)

For each task query Q, the system records every agent utterance and causal link between them. Each node is an atomic utterance: a tuple of speaking agent and message content. Edges represent temporal or causal influence — utterance A "inspired" utterance B.

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional
import uuid, time

@dataclass
class Utterance:
    """Node in the Interaction Graph."""
    id: str = field(default_factory=lambda: str(uuid.uuid4()))
    agent_id: str = ""              # Which agent spoke
    content: str = ""               # Raw text of the utterance
    epoch: int = 0                  # Communication round number
    timestamp: float = field(default_factory=time.time)

@dataclass
class InteractionGraph:
    """Per-query record of all agent communication."""
    query_id: str = ""
    utterances: dict[str, Utterance] = field(default_factory=dict)  # id -> Utterance
    edges: list[tuple[str, str]] = field(default_factory=list)      # (source_id, target_id)

    def add_utterance(self, agent_id: str, content: str, epoch: int,
                      inspired_by: list[str] | None = None) -> str:
        u = Utterance(agent_id=agent_id, content=content, epoch=epoch)
        self.utterances[u.id] = u
        for src_id in (inspired_by or []):
            self.edges.append((src_id, u.id))
        return u.id
```

The paper constructs interaction graphs by tracing agent messages during the MAS communication epochs. In a subprocess architecture, **the orchestrator builds this graph** by logging each agent's output and the inputs it received.

### Query Graph (middle tier — task metadata and connections)

Each query node wraps a task attempt: the original query text, success/failure status, and a pointer to the full interaction graph. Edges encode semantic relationships between queries — connecting tasks that share similar themes or solution approaches.

```python
class TaskStatus(Enum):
    FAILED = "failed"
    RESOLVED = "resolved"

@dataclass
class QueryNode:
    """Node in the Query Graph."""
    id: str = field(default_factory=lambda: str(uuid.uuid4()))
    query_text: str = ""
    status: TaskStatus = TaskStatus.FAILED
    interaction_graph: InteractionGraph | None = None
    embedding: list[float] = field(default_factory=list)  # 384-dim from MiniLM
    created_at: float = field(default_factory=time.time)

@dataclass
class QueryGraph:
    """Cross-query semantic network."""
    nodes: dict[str, QueryNode] = field(default_factory=dict)
    edges: set[tuple[str, str]] = field(default_factory=set)  # (source_qid, target_qid)
```

### Insight Graph (highest tier — distilled wisdom)

Insight nodes contain **LLM-generated generalizations** extracted from interaction trajectories. Each insight tracks which queries support it (the Ω set). Edges are **ternary hyper-connections**: `(insight_A, insight_B, query_Q)` means insight A contextualizes insight B through the experience of query Q. This hyper-edge structure is why networkx is a poor fit — it requires a `MultiDiGraph` with awkward edge attributes.

```python
@dataclass
class InsightNode:
    """Node in the Insight Graph."""
    id: str = field(default_factory=lambda: str(uuid.uuid4()))
    content: str = ""                                           # The insight text
    supporting_queries: set[str] = field(default_factory=set)   # Ω_k: set of query IDs
    created_at: float = field(default_factory=time.time)

@dataclass
class InsightEdge:
    """Ternary hyper-edge: insight_a contextualizes insight_b via query."""
    source_insight_id: str = ""
    target_insight_id: str = ""
    via_query_id: str = ""

@dataclass
class InsightGraph:
    """Highest abstraction — distilled cross-task knowledge."""
    nodes: dict[str, InsightNode] = field(default_factory=dict)
    edges: list[InsightEdge] = field(default_factory=list)
```

**Key architectural insight:** G-Memory does not use networkx or any graph library. The paper's formal math translates to plain Python dicts, sets, and lists. The operations (hop expansion, set projection, membership testing) are simple enough that a graph library adds overhead without value.

---

## 2. How information flows between tiers

The flow has three phases: **retrieve** (before task execution), **execute** (agents work), and **update** (after completion). Understanding the exact triggers and algorithms is critical.

### Retrieval: coarse-grained then bi-directional

When a new query Q arrives, retrieval proceeds in four steps:

**Step 1 — Embedding similarity search.** Encode Q with `all-MiniLM-L6-v2`, compute cosine similarity against all stored query embeddings, return top-k (k=1 or k=2; **k=5 degrades performance by 7.71%**).

**Step 2 — 1-hop expansion.** Take the retrieved query set Q_S and add all 1-hop neighbors from the query graph (both in-edges and out-edges). The paper shows **1-hop is optimal** — 2-hop and 3-hop degrade performance as they pull in irrelevant context.

**Step 3 — Upward traversal to insights.** Collect all insight nodes whose supporting query set (Ω_k) overlaps with the expanded query set. In Python: `relevant_insights = [i for i in insights.values() if i.supporting_queries & expanded_query_ids]`.

**Step 4 — Downward traversal to interactions.** Use an LLM to score relevance of each expanded query against Q (the `R_LLM` function), select top-M most relevant (M ∈ {2,3,4,5}), then use an LLM graph sparsifier (`S_LLM`) to extract the essential dialogue subgraph from each interaction graph — filtering out dead-end actions and noise.

```python
def retrieve_memory(new_query: str, query_graph: QueryGraph,
                    insight_graph: InsightGraph, embedder, llm) -> dict:
    # Step 1: Embedding similarity
    q_emb = embedder.encode(new_query)
    scored = [(qid, cosine_sim(q_emb, qn.embedding))
              for qid, qn in query_graph.nodes.items()]
    scored.sort(key=lambda x: x[1], reverse=True)
    top_k_ids = {qid for qid, _ in scored[:2]}  # k=2

    # Step 2: 1-hop expansion
    expanded = set(top_k_ids)
    for src, tgt in query_graph.edges:
        if src in top_k_ids:
            expanded.add(tgt)
        if tgt in top_k_ids:
            expanded.add(src)

    # Step 3: Upward → insights
    relevant_insights = [
        ins for ins in insight_graph.nodes.values()
        if ins.supporting_queries & expanded
    ]

    # Step 4: Downward → sparsified interactions
    candidates = [query_graph.nodes[qid] for qid in expanded
                  if query_graph.nodes[qid].interaction_graph]
    # LLM scores relevance, selects top-M, sparsifies
    relevance_scores = [(c, llm.score_relevance(new_query, c.query_text))
                        for c in candidates]
    relevance_scores.sort(key=lambda x: x[1], reverse=True)
    top_m = relevance_scores[:3]  # M=3
    sparsified = [llm.sparsify_interaction(c.interaction_graph, new_query)
                  for c, _ in top_m]

    return {"insights": relevant_insights, "interactions": sparsified,
            "related_queries": expanded}
```

### What triggers promotion between tiers

There is **no explicit promotion** in G-Memory's design. Instead, every completed task simultaneously creates entries at all three tiers:

- **Interaction tier**: The orchestrator records all agent utterances into a new `InteractionGraph`
- **Query tier**: A new `QueryNode` is created, linked to semantically related historical queries and to queries referenced by the insights that were used
- **Insight tier**: The LLM summarization function `J(interaction_graph, status)` generates a new insight from the interaction trajectory and outcome

The critical update logic for the insight graph maintains **cumulative evidence**: when an existing insight is used to solve a new query, its `supporting_queries` set grows — `Ω_k = Ω_k ∪ {q_new}`. This makes frequently-validated insights retrievable from more query entry points, creating organic importance weighting without explicit scoring.

---

## 3. Adapting G-Memory for a subprocess-based swarm

G-Memory assumes agents live in a shared Python process. In a subprocess architecture where agents are invoked as `claude -p "..."`, four adaptations are necessary.

### The orchestrator owns the memory

Agents are stateless CLI invocations. The orchestrator process holds persistent connections to all storage backends, assembles per-agent context before invocation, and parses structured output for memory updates afterward.

```
┌──────────────────────────────────────────────────┐
│              Orchestrator (persistent)            │
│  ┌───────────┐ ┌──────────┐ ┌─────────────────┐  │
│  │  SQLite   │ │ LanceDB  │ │    Redis         │  │
│  │  (graphs) │ │ (vectors)│ │(blackboard+locks)│  │
│  └───────────┘ └──────────┘ └─────────────────┘  │
│  ┌───────────────────────┐                        │
│  │  Embedding Model      │  (loaded once)         │
│  │  all-MiniLM-L6-v2    │                        │
│  └───────────────────────┘                        │
└──────┬────────┬────────┬────────┬────────────────┘
       │        │        │        │
    agent_01 agent_02 agent_03  ...agent_50
    (subprocess, stateless, receives context in prompt)
```

### Context injection replaces shared memory

Instead of agents reading memory directly, the orchestrator builds a **role-specific context string** for each agent. This mirrors G-Memory's Φ operator, which filters insights and interaction fragments by agent role:

```python
def build_agent_context(agent_role: str, task: str,
                        memory: dict) -> str:
    sections = []

    # High-level insights (from Insight Graph)
    if memory["insights"]:
        sections.append("## Relevant Insights")
        for ins in memory["insights"]:
            sections.append(f"- {ins.content}")

    # Historical interaction patterns (from sparsified Interaction Graphs)
    if memory["interactions"]:
        sections.append("## Relevant Past Interactions")
        for interaction in memory["interactions"]:
            # Filter utterances relevant to this agent's role
            role_relevant = [u for u in interaction
                           if u.get("role_match", True)]
            for u in role_relevant[:10]:  # Cap at 10 utterances
                sections.append(f"  [{u['agent']}]: {u['content']}")

    return "\n".join(sections)
```

### Structured output captures memory updates

Agents emit memory-relevant information as a JSON block in their output. The orchestrator parses this and queues updates:

```python
MEMORY_OUTPUT_INSTRUCTION = """
When you complete your task, include a memory block with any
new knowledge worth preserving:
```memory
{
  "insight": "one-sentence lesson learned (or null)",
  "key_actions": ["list of critical steps taken"],
  "outcome": "resolved" or "failed",
  "confidence": 0.0-1.0
}
```
"""
```

### Write serialization through Redis Streams

With 5–50 agents completing tasks concurrently, all memory writes funnel through a Redis Stream consumed by a single dedicated writer process. This avoids SQLite write contention entirely:

```python
# Agent completion handler in orchestrator
def on_agent_complete(agent_id: str, task_id: str, output: str):
    memory_block = parse_memory_block(output)
    redis_client.xadd("memory:writes", {
        "type": "task_complete",
        "agent_id": agent_id,
        "task_id": task_id,
        "payload": json.dumps(memory_block)
    })
```

---

## 4. Embedding strategy for the memory tiers

G-Memory uses **`all-MiniLM-L6-v2`** (22M parameters, 384 dimensions) — confirmed in the paper's Section 5.1. This is the right choice for this use case.

### Why MiniLM and not something larger

The embeddings serve one purpose: coarse-grained retrieval of similar queries at the Query Graph level. The subsequent LLM-based relevance scoring (`R_LLM`) and graph traversal provide fine-grained filtering. There's no need for a larger model when **the embedding is just the first filter in a multi-stage pipeline**. MiniLM runs at **~5x the speed** of `all-mpnet-base-v2` with marginal quality loss on this task.

### Embedding generation and storage

Load the model **once** in the orchestrator. Never load it inside subprocess agents (2–5 second startup cost per invocation). Use LanceDB's native sentence-transformers integration for automatic embed-on-insert and embed-on-query:

```python
import lancedb
from lancedb.pydantic import LanceModel, Vector
from lancedb.embeddings import get_registry

embed_fn = get_registry().get("sentence-transformers").create(
    name="all-MiniLM-L6-v2"
)

class QueryVector(LanceModel):
    query_id: str
    text: str = embed_fn.SourceField()
    vector: Vector(384) = embed_fn.VectorField()
    status: str
    created_at: float

db = lancedb.connect("./memory_vectors")
query_table = db.create_table("queries", schema=QueryVector)

# Insert — embedding auto-generated
query_table.add([{
    "query_id": "q_001",
    "text": "Navigate to the kitchen and heat the mug",
    "status": "resolved",
    "created_at": time.time()
}])

# Search — query embedding auto-generated
results = query_table.search("find and warm the cup").limit(5).to_pandas()
```

For ONNX acceleration on CPU (recommended for a solo dev without GPU): `pip install sentence-transformers[onnx]` and use `SentenceTransformer("all-MiniLM-L6-v2", backend="onnx")` for 2–3x speedup.

### What gets embedded

Only **query texts** get embedded — not utterances, not insights. The paper embeds queries for top-k retrieval (Equation 4), then uses graph traversal and LLM scoring for everything else. Embedding all utterances would be wasteful given that the sparsifier (`S_LLM`) already handles relevance filtering at the interaction level.

---

## 5. Graph storage: SQLite property graph beats alternatives

### Why not networkx

NetworkX stores everything as Python dicts in memory. With subprocess-based agents, you'd need to serialize the entire graph, pass it through Redis or disk, and deserialize for every access. At 50 agents, this creates race conditions and stale-read bugs. NetworkX has no concurrency control, no persistence, and no transaction support.

### Why not a dedicated graph database

Neo4j and FalkorDB are excellent but violate the "solo-buildable, no managed services beyond Redis and SQLite" constraint. Neo4j requires a JVM, separate server process, and operational overhead. RedisGraph is **deprecated** as of Redis Stack 7.x — do not build on it.

### SQLite property graph: the right fit

SQLite in WAL mode supports concurrent reads from all 50 agents while serializing writes. The schema stores G-Memory's three tiers in relational tables with JSON flexibility:

```sql
PRAGMA journal_mode = WAL;
PRAGMA busy_timeout = 30000;

-- Insight Graph
CREATE TABLE insights (
    id TEXT PRIMARY KEY,
    content TEXT NOT NULL,
    supporting_queries TEXT NOT NULL DEFAULT '[]',  -- JSON array of query IDs
    created_at REAL NOT NULL,
    access_count INTEGER DEFAULT 0
);

CREATE TABLE insight_edges (
    source_insight_id TEXT NOT NULL REFERENCES insights(id),
    target_insight_id TEXT NOT NULL REFERENCES insights(id),
    via_query_id TEXT NOT NULL,
    created_at REAL NOT NULL,
    PRIMARY KEY (source_insight_id, target_insight_id, via_query_id)
);

-- Query Graph
CREATE TABLE queries (
    id TEXT PRIMARY KEY,
    query_text TEXT NOT NULL,
    status TEXT NOT NULL CHECK(status IN ('resolved', 'failed')),
    created_at REAL NOT NULL,
    access_count INTEGER DEFAULT 0,
    last_accessed REAL NOT NULL
);

CREATE TABLE query_edges (
    source_id TEXT NOT NULL REFERENCES queries(id),
    target_id TEXT NOT NULL REFERENCES queries(id),
    created_at REAL NOT NULL,
    PRIMARY KEY (source_id, target_id)
);

-- Interaction Graphs (stored per query)
CREATE TABLE utterances (
    id TEXT PRIMARY KEY,
    query_id TEXT NOT NULL REFERENCES queries(id),
    agent_id TEXT NOT NULL,
    content TEXT NOT NULL,
    epoch INTEGER NOT NULL,
    timestamp REAL NOT NULL
);

CREATE TABLE utterance_edges (
    source_id TEXT NOT NULL REFERENCES utterances(id),
    target_id TEXT NOT NULL REFERENCES utterances(id),
    query_id TEXT NOT NULL REFERENCES queries(id),
    PRIMARY KEY (source_id, target_id)
);

-- Indexes for retrieval
CREATE INDEX idx_utterances_query ON utterances(query_id);
CREATE INDEX idx_queries_status ON queries(status);
CREATE INDEX idx_queries_last_accessed ON queries(last_accessed);
```

### 1-hop expansion via SQL

The query graph hop expansion from G-Memory's Equation 5 translates to a single SQL query:

```sql
-- Given seed query IDs in a temp table or parameter list
SELECT DISTINCT neighbor_id FROM (
    SELECT target_id AS neighbor_id FROM query_edges
    WHERE source_id IN (:seed_ids)
    UNION
    SELECT source_id AS neighbor_id FROM query_edges
    WHERE target_id IN (:seed_ids)
);
```

For multi-hop traversal (not recommended by the paper, but available), SQLite's recursive CTEs handle it cleanly:

```sql
WITH RECURSIVE hops(qid, depth) AS (
    SELECT id, 0 FROM queries WHERE id IN (:seed_ids)
    UNION
    SELECT CASE WHEN qe.source_id = h.qid THEN qe.target_id
                ELSE qe.source_id END, h.depth + 1
    FROM query_edges qe
    JOIN hops h ON qe.source_id = h.qid OR qe.target_id = h.qid
    WHERE h.depth < 1  -- limit to 1-hop
)
SELECT DISTINCT qid FROM hops;
```

---

## 6. Redis blackboard integration with memory tiers

The Redis blackboard serves three functions in the memory architecture: **hot caching** of recently accessed memory, **write serialization** via Streams, and **event propagation** so agents can react to new knowledge.

### Redis data structure mapping

```python
# 1. BLACKBOARD — current agent state
"blackboard:agent:{agent_id}" → Hash {
    "status": "working|idle|complete",
    "current_task": "task_id",
    "last_heartbeat": "unix_timestamp"
}

# 2. MEMORY HOT CACHE — recent insights (avoid SQLite reads)
"memory:insight:{insight_id}" → Hash {
    "content": "insight text",
    "supporting_queries": "[q1, q2, q3]",  # JSON
    "access_count": "7"
}
# Auto-expire: EXPIRE memory:insight:{id} 3600

# 3. WRITE QUEUE — serialized memory updates
"memory:writes" → Stream [
    {"type": "task_complete", "agent_id": "...", "payload": "..."},
    {"type": "insight_update", "insight_id": "...", "payload": "..."}
]

# 4. EVENT BUS — real-time notifications
"memory:events" → Pub/Sub channel
    → {"event": "new_insight", "id": "...", "content": "..."}
    → {"event": "query_linked", "from": "...", "to": "..."}
```

### The writer process pattern

One persistent process consumes the write stream and applies all mutations to SQLite and LanceDB. This is the **only process that writes** to these stores:

```python
class MemoryWriter:
    """Single-writer daemon for all memory mutations."""
    def __init__(self):
        self.db = sqlite3.connect("memory.db")
        self.db.execute("PRAGMA journal_mode=WAL")
        self.db.execute("PRAGMA busy_timeout=30000")
        self.redis = redis.Redis(decode_responses=True)
        self.lance = lancedb.connect("./memory_vectors")

    def process_task_complete(self, payload: dict):
        """Called when an agent completes a task."""
        with self.db:
            # 1. Store interaction graph
            for utt in payload["utterances"]:
                self.db.execute(
                    "INSERT INTO utterances VALUES (?,?,?,?,?,?)",
                    (utt["id"], payload["query_id"], utt["agent_id"],
                     utt["content"], utt["epoch"], utt["timestamp"])
                )
            for src, tgt in payload["utterance_edges"]:
                self.db.execute(
                    "INSERT INTO utterance_edges VALUES (?,?,?)",
                    (src, tgt, payload["query_id"])
                )

            # 2. Create query node + edges
            self.db.execute(
                "INSERT INTO queries VALUES (?,?,?,?,0,?)",
                (payload["query_id"], payload["query_text"],
                 payload["status"], time.time(), time.time())
            )
            for related_qid in payload["related_query_ids"]:
                self.db.execute(
                    "INSERT OR IGNORE INTO query_edges VALUES (?,?,?)",
                    (related_qid, payload["query_id"], time.time())
                )

            # 3. Generate + store new insight
            if payload.get("insight"):
                ins_id = str(uuid.uuid4())
                self.db.execute(
                    "INSERT INTO insights VALUES (?,?,?,?,0)",
                    (ins_id, payload["insight"],
                     json.dumps([payload["query_id"]]), time.time())
                )

        # 4. Update LanceDB vector index
        self.lance.open_table("queries").add([{
            "query_id": payload["query_id"],
            "text": payload["query_text"],
            "status": payload["status"],
            "created_at": time.time()
        }])

        # 5. Notify via pub/sub
        self.redis.publish("memory:events", json.dumps({
            "event": "task_stored",
            "query_id": payload["query_id"]
        }))
```

---

## 7. Retention and eviction for growing memory

G-Memory's paper has **no explicit eviction** — memory grows indefinitely. For a production swarm, you need eviction. The system already has natural consolidation (interaction → insight distillation), but the underlying storage needs bounds.

### Three-tier eviction strategy

**Insights: never evict.** These are the highest-value, most-compressed knowledge. Even at 10,000 insights (each ~100 tokens), storage is trivial. Instead of evicting, merge overlapping insights periodically using an LLM.

**Queries: age-based archival.** After 90 days without access, archive query nodes (move to a separate table). Keep their IDs in the query graph for edge integrity but stop including them in embedding searches. Before archival, ensure any insights they support are well-connected to newer queries.

**Interactions: aggressive compression.** Interaction graphs are the largest tier by far. After insight extraction, the full trace serves only as supporting evidence. Apply the LLM sparsifier (`S_LLM`) at write time to create a compressed version, store that, and discard the full trace after 30 days.

### Concrete eviction logic

```python
def run_eviction_cycle(db: sqlite3.Connection, max_archive: int = 50):
    now = time.time()
    ninety_days_ago = now - (90 * 86400)
    thirty_days_ago = now - (30 * 86400)

    with db:
        # Archive old, unaccessed queries
        old_queries = db.execute("""
            SELECT id FROM queries
            WHERE last_accessed < ? AND access_count < 3
            ORDER BY last_accessed ASC LIMIT ?
        """, (ninety_days_ago, max_archive)).fetchall()

        for (qid,) in old_queries:
            db.execute("INSERT INTO queries_archive SELECT * FROM queries WHERE id=?", (qid,))
            db.execute("DELETE FROM queries WHERE id=?", (qid,))

        # Purge old interaction details (keep sparsified version)
        db.execute("""
            DELETE FROM utterances
            WHERE query_id IN (
                SELECT id FROM queries WHERE created_at < ?
            ) AND query_id NOT IN (
                SELECT id FROM queries ORDER BY access_count DESC LIMIT 100
            )
        """, (thirty_days_ago,))

        db.execute("""
            DELETE FROM utterance_edges
            WHERE query_id NOT IN (SELECT DISTINCT query_id FROM utterances)
        """)
```

Run this hourly via a scheduled task. Evict at most 50 nodes per cycle to prevent stampedes. Use soft deletion (archive table) with hard purge after 7 days.

---

## 8. G-Memory versus the alternatives: when to use what

The memory landscape in 2025–2026 splits into **production systems** (Letta, Zep, Mem0) and **research architectures** (G-Memory, A-MEM, MAGMA). Here's the decision matrix.

### Head-to-head comparison

| Dimension | G-Memory | Letta (MemGPT) | Zep/Graphiti | A-MEM | Mem0 |
|-----------|----------|-----------------|-------------|-------|------|
| **Architecture** | 3-tier graph hierarchy | OS-inspired tiers (core/recall/archival) | Temporal knowledge graph | Zettelkasten network | Flat + optional graph |
| **Multi-agent native** | ✅ Purpose-built | 🟡 Via shared sessions | 🟡 Group sessions | ❌ Single-agent | 🟡 User/agent levels |
| **Role-specific memory** | ✅ Per-agent filtering | ❌ | ❌ | ❌ | ❌ |
| **Temporal awareness** | ❌ Implicit only | ❌ Message timestamps | ✅ Bi-temporal model | ❌ | 🟡 Timestamps |
| **Production-ready** | ❌ Research code | ✅ Full stack | ✅ Full stack | ❌ Research | ✅ SOC2/HIPAA |
| **Storage backend** | In-memory Python | PostgreSQL + pgvector | Neo4j | ChromaDB | Vector DB + optional graph |
| **Self-hostable** | ✅ Python only | ✅ Docker | 🟡 Needs Neo4j | ✅ pip install | ✅ Docker |

### When to use each approach

**Build on G-Memory's architecture** (this guide's approach) when you need multi-agent swarm memory with role-specific recall, subprocess-based agents, and want full control over the implementation. Best for: teams of 5–50 specialized agents where collaboration patterns matter.

**Use Letta/MemGPT** when building single-agent systems that need long-running conversations with OS-style virtual memory management. Letta's self-editing memory model is elegant but assumes the agent manages its own memory — incompatible with stateless subprocess agents.

**Use Zep/Graphiti** when temporal accuracy matters — when facts change over time and you need to know what was true when. Zep's bi-temporal model (event time + ingestion time) with fact invalidation is unmatched. Best for: customer-facing agents tracking evolving user preferences.

**Use Mem0** when you want the fastest path to production memory with minimal engineering. One-line integration, managed hosting, automatic fact extraction. Best for: MVPs, products where memory is a feature not the core architecture.

**Watch MAGMA** (January 2026, arXiv 2601.03236) for its four orthogonal relation graphs (semantic, temporal, causal, entity) with intent-aware routing. Its dual-stream write path (fast synaptic ingestion + slow structural consolidation) is the most sophisticated architecture in the landscape, but it remains single-agent focused.

### Patterns the best systems share

Every successful memory system shares five patterns: **hierarchical organization** (multiple abstraction levels, never flat), **hybrid vector + graph retrieval** (embedding search for recall, graph traversal for reasoning), **LLM-in-the-loop management** (the LLM extracts, consolidates, and prunes memory), **role/intent-aware filtering** (not all memory is relevant to all consumers), and **write-time consolidation** (compress information at storage time, not just at retrieval time).

---

## 9. Recommended build order for a solo developer

Build in this sequence. Each phase produces a working system that the next phase extends.

**Phase 1 — Foundation (Days 1–3).** Set up the SQLite schema (all six tables above), Redis connection, and LanceDB vector table. Write the `MemoryWriter` class that consumes from the Redis write stream. Write a `MemoryReader` class that executes the retrieval pipeline (embedding search → 1-hop expansion → insight lookup). Test with synthetic data. This phase gives you persistent, concurrent-safe memory storage.

**Phase 2 — Orchestrator integration (Days 4–6).** Wire the memory system into your existing orchestrator. Before each agent invocation, call `retrieve_memory()` and inject the context string into the prompt. After each agent completion, parse the memory output block and queue it to the Redis write stream. At this point, agents benefit from memory but only at the Query and Insight tiers (no interaction graph yet).

**Phase 3 — Interaction graph recording (Days 7–9).** Instrument the orchestrator to build `InteractionGraph` objects as agents communicate. In a subprocess swarm, this means logging the input each agent received and the output it produced, with edges connecting outputs to the inputs of downstream agents. Store these interaction graphs via the writer process. Implement the LLM sparsifier — a simple prompt that takes a full interaction trace and returns the essential steps.

**Phase 4 — Insight generation (Days 10–12).** After each task completion, call the LLM summarization function `J(interaction_graph, status)` to generate an insight. Connect new insights to relevant existing insights via hyper-edges. This closes the full G-Memory loop. The prompt for insight generation follows the paper's approach: present the interaction trace and outcome, ask for a one-sentence generalizable lesson.

**Phase 5 — Eviction and hardening (Days 13–15).** Implement the eviction cycle. Add the hot cache layer in Redis for frequently accessed insights. Add monitoring (log memory size, retrieval latency, hit rates). Stress test with 50 parallel agents.

---

## 10. Known limitations and failure modes

**Unbounded memory growth without eviction.** G-Memory's paper has no eviction mechanism. Without the retention logic from Section 7, SQLite and LanceDB will grow indefinitely. At ~1,000 queries with full interaction graphs, expect ~100MB of storage and noticeably slower embedding searches (LanceDB scales well to millions of vectors, but the LLM sparsification calls become the bottleneck).

**LLM costs scale with memory operations.** Every task uses at least 3 LLM calls for memory management: relevance scoring (`R_LLM`), graph sparsification (`S_LLM`), and insight generation (`J`). At **$0.15/1M input tokens** with GPT-4o-mini, 100 tasks/day with 3 calls each costs roughly $0.50–2.00/day. Switching to a local model via Ollama (Qwen-2.5-7b) eliminates this cost at the expense of quality.

**Context window saturation.** The memory context injected into agent prompts can grow large. Cap the context at **2,000 tokens** maximum — the paper found that more context (higher k, more hops) actively degrades performance. This is counterintuitive but consistent across all benchmarks.

**Cold start problem.** G-Memory provides zero benefit until the system has accumulated enough query history for meaningful retrieval. Expect ~10–15 tasks before the memory system starts producing relevant recall. Mitigate by seeding the insight graph with domain knowledge from documentation or known best practices.

**Concurrent insight conflicts.** When two agents complete similar tasks simultaneously, they may generate near-duplicate insights. The writer process should deduplicate by checking embedding similarity of new insights against existing ones before insertion. If similarity exceeds **0.85**, merge rather than create a new node.

**Interaction graph fidelity in subprocess architectures.** In G-Memory's original design, the system traces every inter-agent message. In a subprocess architecture, you only see each agent's final output — not intermediate reasoning. The interaction graph will be sparser than the paper assumes. This is acceptable; the sparsifier already reduces full traces to essential steps, and subprocess outputs are typically more concise than multi-turn dialogues.

**SQLite write contention at scale.** Even with WAL mode and the single-writer pattern, SQLite becomes a bottleneck above ~100 writes/second. For 50 agents completing tasks every 30 seconds, you generate ~1.7 writes/second (well within limits). If you scale beyond 200 agents or need sub-second write latency, migrate the graph store to PostgreSQL with the same schema.

**The embedding model matters less than you think.** The paper's ablation shows that the graph structure (hop expansion, query-insight links) contributes more to retrieval quality than embedding precision. Don't over-invest in embedding model selection — `all-MiniLM-L6-v2` at 384 dimensions is genuinely sufficient. The LLM relevance scorer is the real quality filter.
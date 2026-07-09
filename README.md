# CodeLake

**Codebase intelligence powered by multi-agent AI, knowledge graphs, and a context lakehouse.**

AgentMesh continuously indexes any Git repository into a queryable knowledge graph and stores rich context in a local-first lakehouse. You can ask it architectural questions, trace change impact, understand unfamiliar codebases, and keep documentation in sync — all driven by a network of specialized agents that share a common world model.

> Status: **Design / Early Development** — architecture is defined, core agents are being built. Contributions welcome.

---

## The Problem

Understanding a large codebase is expensive. You switch to a new repo and spend days mapping what calls what, why a module exists, what breaks if you touch it. Existing tools are either:
- **Too shallow** — semantic search over embeddings answers "find code like this" but not "what does changing this function break?"
- **Too proprietary** — Sourcegraph, GitHub Copilot Workspace are closed; you can't extend them or run them locally
- **Stateless** — most AI coding assistants have no persistent memory of the codebase between sessions

AgentMesh solves this by treating a codebase as a **living knowledge graph**, with agents that incrementally maintain it and answer questions over it.

---

## How It Works

```
┌─────────────────────────────────────────────────────────┐
│                      Git Repository                      │
└────────────────────────┬────────────────────────────────┘
                         │ file changes / initial clone
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   Ingestion Pipeline                     │
│  Tree-sitter AST parsing → structured code events       │
└────────────────────────┬────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
  ┌───────────────┐ ┌──────────┐ ┌───────────────┐
  │  Knowledge    │ │ Context  │ │    Vector     │
  │    Graph      │ │Lakehouse │ │    Index      │
  │  (Kuzu DB)    │ │(DuckDB + │ │  (LanceDB)   │
  │               │ │ Parquet) │ │               │
  │ Files,funcs,  │ │ Events,  │ │  Embeddings   │
  │ classes,deps, │ │ snapshots│ │  for semantic │
  │ call edges    │ │ agent    │ │  search       │
  └───────┬───────┘ │ memory   │ └───────┬───────┘
          │         └──────────┘         │
          └──────────────┬───────────────┘
                         │ shared world model
          ┌──────────────┼───────────────────────────┐
          ▼              ▼              ▼             ▼
  ┌──────────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
  │ ParseAgent   │ │Lineage   │ │  Doc     │ │ Answer   │
  │              │ │Agent     │ │  Agent   │ │ Agent    │
  │ AST→graph    │ │          │ │          │ │          │
  │ nodes+edges  │ │ impact   │ │ generate │ │ Q&A over │
  │ incremental  │ │ analysis │ │ + sync   │ │ graph +  │
  │ updates      │ │ on change│ │   docs   │ │ lakehouse│
  └──────────────┘ └──────────┘ └──────────┘ └──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │  CLI / API / Web UI  │
              └─────────────────────┘
```

---

## Core Concepts

### Knowledge Graph
The knowledge graph is the primary index of your codebase. Every file, function, class, module, and import becomes a node. Relationships like `calls`, `imports`, `inherits`, `defines`, and `overrides` become edges.

This lets you ask graph-native questions:
- "What are all the callers of `process_payment`?"
- "Which modules have circular dependencies?"
- "What is the blast radius if I rename `UserSchema`?"

Built on [Kuzu](https://kuzudb.com/) — an embedded, high-performance graph database. No server to run.

### Context Lakehouse
Agents are stateful and their work should persist. The lakehouse stores:
- **Code events** — every parse result, diff, and agent observation, as immutable Parquet files
- **Agent scratchpads** — intermediate reasoning and annotations, versioned per run
- **Historical snapshots** — point-in-time views of the graph (useful for "what changed this week?")
- **Embeddings metadata** — chunk boundaries, model versions, source hashes

Built on [DuckDB](https://duckdb.org/) for in-process querying over Parquet. No warehouse needed.

The lakehouse is intentionally local-first: all data lives on disk, queryable with standard SQL.

### Multi-Agent Orchestration
Agents are small, focused, and communicate through the shared knowledge graph + lakehouse rather than passing messages directly. This makes the system observable and resumable.

| Agent | Trigger | Output |
|---|---|---|
| `ParseAgent` | File change or initial clone | Graph nodes + edges for that file |
| `LineageAgent` | User query or PR event | Impact report: what depends on changed symbols |
| `DocAgent` | Graph node has no docstring | Generated docstring or module summary, written back to graph |
| `AnswerAgent` | User question | Answer grounded in graph + semantic search |
| `ConsolidationAgent` | Background / scheduled | Prunes stale nodes, re-ranks embeddings, merges duplicates |

Orchestration is handled by a lightweight coordinator (no LangGraph dependency). Each agent declares its input schema and output schema; the coordinator routes work.

---

## Example Queries

```bash
# What breaks if I change this function?
agentmesh impact src/auth/token.py::validate_token

# Explain this module
agentmesh explain src/payments/

# Find all places a pattern is used
agentmesh find "calls:stripe.Charge.create"

# Ask a freeform question
agentmesh ask "Why does the billing module depend on the auth module?"

# Show the call graph for a function
agentmesh graph src/api/routes.py::handle_checkout --depth 3
```

---

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Language parsing | [Tree-sitter](https://tree-sitter.github.io/) | Fast, incremental, language-agnostic AST |
| Knowledge graph | [Kuzu](https://kuzudb.com/) | Embedded graph DB, no server, Cypher-compatible |
| Context lakehouse | [DuckDB](https://duckdb.org/) + Parquet | In-process OLAP, SQL over versioned files |
| Vector search | [LanceDB](https://lancedb.com/) | Embedded, co-located with the lakehouse |
| LLM | [Claude API](https://docs.anthropic.com/) | Tool use + long context for code understanding |
| Agent framework | Custom (lightweight) | No heavy dependencies; agents are just Python classes |
| CLI | [Typer](https://typer.tiangolo.com/) | Clean Python CLI with minimal boilerplate |
| API (optional) | [FastAPI](https://fastapi.tiangolo.com/) | REST interface for IDE integrations |

Languages supported at launch: Python, TypeScript/JavaScript. More via Tree-sitter grammars.

---

## Roadmap

### Phase 1 — Foundation (current)
- [ ] Tree-sitter ingestion pipeline for Python
- [ ] Kuzu graph schema: File, Function, Class, Module, Import nodes + edges
- [ ] DuckDB lakehouse schema: events, snapshots, agent runs
- [ ] `ParseAgent` — incremental AST → graph updates
- [ ] Basic CLI: `index`, `status`, `graph`

### Phase 2 — Intelligence
- [ ] LanceDB embedding store wired to graph nodes
- [ ] `AnswerAgent` — Q&A over graph + embeddings via Claude API
- [ ] `LineageAgent` — change impact analysis
- [ ] TypeScript/JavaScript parser support
- [ ] CLI: `ask`, `impact`, `explain`

### Phase 3 — Documentation & Collaboration
- [ ] `DocAgent` — generates and syncs docstrings and module-level docs
- [ ] `ConsolidationAgent` — background graph maintenance
- [ ] GitHub Actions integration (runs on PR, posts impact summary)
- [ ] Web UI — graph explorer + chat interface

### Phase 4 — Scale & Ecosystem
- [ ] Multi-repo indexing and cross-repo call graphs
- [ ] Plugin API for custom agents
- [ ] IDE extensions (VS Code first)
- [ ] MCP server — expose AgentMesh as a tool to other AI agents

---

## Getting Started

> Note: Early development. The setup below reflects the target experience; some steps are still being built.

```bash
# Install
pip install agentmesh  # not yet published — clone and install locally

# Clone and install from source
git clone https://github.com/your-username/agentmesh
cd agentmesh
pip install -e ".[dev]"

# Set your API key
export ANTHROPIC_API_KEY=sk-...

# Index a repository
agentmesh index /path/to/your/repo

# Ask a question
agentmesh ask "What is the entry point of this application?"
```

---

## Project Structure

```
agentmesh/
├── agentmesh/
│   ├── ingestion/          # Tree-sitter parsers, git watcher
│   ├── graph/              # Kuzu schema, graph client, query helpers
│   ├── lakehouse/          # DuckDB schema, Parquet writers, event log
│   ├── embeddings/         # LanceDB store, chunking, embedding calls
│   ├── agents/
│   │   ├── base.py         # Agent base class, schema contracts
│   │   ├── parse_agent.py
│   │   ├── lineage_agent.py
│   │   ├── doc_agent.py
│   │   ├── answer_agent.py
│   │   └── consolidation_agent.py
│   ├── coordinator.py      # Agent orchestration loop
│   └── cli.py              # Typer CLI
├── tests/
├── docs/
└── pyproject.toml
```

---

## Contributing

AgentMesh is early-stage and actively looking for contributors. Good first areas:

- **Tree-sitter grammars** — add support for a new language (Java, Go, Rust)
- **Graph schema** — improve the Kuzu node/edge model for a specific language
- **Agent logic** — improve the LineageAgent or DocAgent prompts and reasoning
- **Tests** — unit tests for the ingestion pipeline and graph queries
- **Docs** — architecture deep-dives, tutorials

See [CONTRIBUTING.md](CONTRIBUTING.md) for setup instructions and the coding conventions.

Open an issue before starting significant work so we can align on approach.

---

## Design Principles

1. **Local-first** — all data on disk, no cloud dependency required
2. **Incremental** — agents process diffs, not full re-indexes
3. **Observable** — every agent action is logged to the lakehouse; you can audit what happened
4. **Composable** — agents are independent; you can run one without the others
5. **Language-agnostic** — the graph schema and lakehouse are language-neutral; parsers are plugins

---

## Why Not Just Use Embeddings?

Vector search is great for "find code similar to this snippet." It breaks down for structural questions:

| Question | Embeddings | Knowledge Graph |
|---|---|---|
| Find functions that call `validate_user` | Approximate / misses indirect calls | Exact, via graph traversal |
| What is the blast radius of renaming `UserSchema`? | Cannot answer | Graph query over `uses` edges |
| Are there circular imports? | Cannot answer | Graph cycle detection |
| Which modules have no tests? | Guess from filenames | Cross-graph query |

AgentMesh uses embeddings for what they're good at (semantic similarity, freeform Q&A) and the knowledge graph for what it's good at (structure, relationships, traversal). The lakehouse ties them together with a consistent query interface.

---

## License

MIT

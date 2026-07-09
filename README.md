# AgentMesh

**A living architectural map of your entire codebase — across every repo, every commit, every decision.**

AgentMesh builds a unified knowledge graph across all your Git repositories and keeps it current. It answers structural questions that no AI assistant can answer from reading files alone, runs autonomously on every commit, and accumulates the *why* behind your architecture — not just the *what*.

> Status: **Design / Early Development** — architecture is defined, core agents are being built. Contributions welcome.

---

## Why AgentMesh exists

Modern AI coding assistants are good at reading code. What they can't do:

**They've never seen your other repos.**
You have 15 microservices. `shared-auth-lib` is getting a breaking change. Ask Claude which services will break — it can't tell you, because it has never seen the other 14 repos. AgentMesh indexes all of them into a single cross-repo knowledge graph. That question becomes one query.

**They have no memory of yesterday.**
Every session starts blank. The reasoning a senior engineer built up over two years — why `payments` bypasses the API gateway, why `UserSchema` has that nullable field, what that incident in 2023 taught the team — lives in PR descriptions, ADRs, Slack threads, and commit messages. AgentMesh ingests all of it alongside the code, so future answers are grounded in history, not just current structure.

**They run when you ask them to.**
A prompt or a skill fires when a human invokes it. AgentMesh hooks into your CI/CD and runs on every commit. It defines architecture rules ("no service should import directly from `infra/`") and flags violations on the PR — before review, without anyone having to remember to check.

**Your standards live in docs nobody reads.**
Your org has ADRs defining how services should be structured, how APIs should be versioned, what logging approach to use. They get written, merged, and forgotten. Six months later half your repos don't follow them and nobody knows. AgentMesh reads your ADRs and turns them into living, executable policies — enforced automatically across every repo, every PR. The ADR *is* the rule; no separate config to maintain or drift from.

These four gaps — **cross-repo intelligence**, **institutional memory**, **autonomous enforcement**, and **ADR-native governance** — are what no skill or prompt can fill.

---

## What you can do with it

```bash
# Cross-repo blast radius: which services break if this contract changes?
agentmesh impact shared-auth-lib::TokenSchema --across-repos

# Institutional memory: why does this dependency exist?
agentmesh ask "Why does the billing service import from the auth module directly?"
# → "Based on PR #312 (2023-08-14): the API gateway added latency that broke SLA for
#    billing's token refresh flow. The direct import was a deliberate tradeoff."

# Architecture enforcement: what's drifting from our rules?
agentmesh check --rules .agentmesh/rules.yaml
# → VIOLATION: payments/src/db.py imports infra.db directly (rule: no-direct-infra-import)
# → VIOLATION: user-service has no owner declared in CODEOWNERS

# Structural queries a language model cannot answer reliably
agentmesh graph src/api/routes.py::handle_checkout --depth 3
agentmesh find "circular-deps" --module payments
agentmesh find "callers:validate_token" --cross-repo
```

---

## How It Works

```
  Repos                       Ingestion                    Shared World Model
┌──────────┐               ┌────────────────┐
│  repo-1  ├──────────────►│  ParseAgent    │         ┌──────────────────────┐
│  repo-2  ├──────────────►│  (Tree-sitter  ├────────►│   Knowledge Graph    │
│  repo-3  ├──────────────►│   AST + diffs) │         │   (Kuzu DB)          │
│  ...     │               └────────────────┘         │                      │
└──────────┘                                          │  nodes: files, funcs,│
                           ┌────────────────┐         │  classes, modules,   │
  Git history  ───────────►│ MemoryAgent    ├────────►│  decisions, people   │
  PRs, ADRs    ───────────►│ (PRs, commits, │         │                      │
  Issues       ───────────►│  issues, docs) │         │  edges: calls,       │
                           └────────────────┘         │  imports, decided-by,│
                                                       │  broke-in, owns      │
                                                       └──────────┬───────────┘
                                                                  │
                                              ┌───────────────────┼──────────────────┐
                                              ▼                   ▼                  ▼
                                       ┌────────────┐    ┌─────────────┐   ┌──────────────┐
                                       │  Context   │    │   Vector    │   │  Rules +     │
                                       │ Lakehouse  │    │   Index     │   │  Policies    │
                                       │ (DuckDB +  │    │ (LanceDB)   │   │  Store       │
                                       │  Parquet)  │    │             │   │              │
                                       │            │    │ embeddings  │   │ architecture │
                                       │ all events,│    │ for semantic│   │ rules,       │
                                       │ snapshots, │    │ search      │   │ owner maps   │
                                       │ agent runs │    └─────────────┘   └──────────────┘
                                       └────────────┘
                                              │
                              ┌───────────────┼────────────────────┐
                              ▼               ▼                    ▼
                       ┌──────────┐   ┌──────────────┐   ┌──────────────────┐
                       │ Answer   │   │  Lineage     │   │  Enforcement     │
                       │ Agent    │   │  Agent       │   │  Agent           │
                       │          │   │              │   │                  │
                       │ Q&A over │   │ cross-repo   │   │ runs on commit,  │
                       │ graph +  │   │ blast radius │   │ checks rules,    │
                       │ history  │   │ analysis     │   │ posts PR comment │
                       └──────────┘   └──────────────┘   └──────────────────┘
                              │               │                    │
                              └───────────────┴────────────────────┘
                                              │
                              ┌───────────────┴───────────────┐
                              │       CLI / API / GitHub App   │
                              └───────────────────────────────┘
```

---

## Core Concepts

### Cross-Repo Knowledge Graph
The graph spans all your repositories. A function in `user-service` that calls a contract in `shared-auth-lib` is a single edge in the graph — regardless of which repo each node lives in. This enables blast radius analysis, circular dependency detection, and ownership mapping at the system level, not just per-repo.

Built on [Kuzu](https://kuzudb.com/) — embedded, no server, Cypher-compatible.

Node types: `File`, `Function`, `Class`, `Module`, `Decision`, `Incident`, `Person`
Edge types: `calls`, `imports`, `inherits`, `decided-by`, `broke-in`, `owns`, `references`

### Institutional Memory
Most tools index what your code *is*. AgentMesh also indexes why it got that way.

The `MemoryAgent` ingests git history, PR descriptions, ADR documents, and linked issues alongside the AST. A `decided-by` edge connects a code pattern to the PR that introduced it. A `broke-in` edge connects a function to the incident where it failed. When you ask "why does this exist?", the answer comes from the historical record — not a guess from the current code.

### Context Lakehouse
Every agent action is an immutable event written to Parquet files, queryable via DuckDB. This means:
- You can audit exactly what every agent did and why
- You can query "what changed in the graph this sprint" with plain SQL
- Agent state survives crashes and restarts — no re-indexing from scratch
- Historical snapshots let you ask "what did the graph look like before that refactor?"

### ADR-Native Governance
Most teams write ADRs and then manually translate them into linter rules or PR checklists — a process that drifts and breaks down at scale. AgentMesh treats the ADR itself as the source of truth.

The `PolicyAgent` reads an ADR, extracts enforceable constraints using an LLM, converts them into graph queries, and runs them across all indexed repos. The result is a compliance report showing exactly which services pass and which violate the decision — without anyone maintaining a separate rule file.

| ADR says | AgentMesh enforces |
|---|---|
| "All DB access must go through repository classes" | Flag any function outside `*repository*` files that imports a DB driver |
| "No service may access another service's DB directly" | Cross-repo edge: `service-A.db` imported by `service-B.*` → violation |
| "All public APIs must be versioned" | Check all route definitions for `/v{n}/` prefix |
| "Every service must have a designated owner in CODEOWNERS" | File presence check across all repos |
| "No bare `print()` in production code — use structured logging" | AST check: `print` calls outside test files |

Some ADRs are precise enough to automate fully; others need a human to confirm the extracted rule before it runs. AgentMesh surfaces the extracted rule for review before enforcing it, so the policy is always legible and auditable.

### Autonomous Enforcement
The `EnforcementAgent` runs on every commit via a GitHub Action and posts violations as PR comments before any human reviews the code. Policies can come from ADRs (via `PolicyAgent`) or be written directly as Cypher queries against the knowledge graph for cases that don't map to an ADR.

---

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Language parsing | [Tree-sitter](https://tree-sitter.github.io/) | Fast, incremental, language-agnostic AST |
| Knowledge graph | [Kuzu](https://kuzudb.com/) | Embedded graph DB, no server, Cypher-compatible |
| Context lakehouse | [DuckDB](https://duckdb.org/) + Parquet | In-process OLAP, SQL over versioned event log |
| Vector search | [LanceDB](https://lancedb.com/) | Embedded, co-located with the lakehouse |
| LLM | [Claude API](https://docs.anthropic.com/) | Tool use + long context for reasoning over graph |
| Agent framework | Custom (lightweight) | No heavy deps; agents are plain Python classes |
| CLI | [Typer](https://typer.tiangolo.com/) | Minimal, composable CLI |
| CI integration | GitHub Actions | Zero-config PR enforcement |

Languages supported at launch: Python, TypeScript/JavaScript. More via Tree-sitter grammars (pluggable).

---

## Why not just use an AI assistant or a Claude skill?

| Capability | AI assistant / skill | AgentMesh |
|---|---|---|
| Answer questions about one file | Yes | Yes |
| Answer questions about one repo | Partially (context limits) | Yes — indexed, instant |
| Answer questions across multiple repos | No | Yes — unified graph |
| Know why a decision was made | No — stateless | Yes — ingests PR/ADR history |
| Run without a human invoking it | No | Yes — hooks into CI/CD |
| Enforce architecture rules on every PR | No | Yes — EnforcementAgent |
| Structural queries (circular deps, blast radius) | Unreliable guesses | Exact graph traversal |
| State persists between sessions | No | Yes — lakehouse on disk |
| Enforce org-wide ADR compliance | No | Yes — PolicyAgent reads ADRs, enforces across all repos |
| Compliance report: which repos follow ADR-007? | No | Yes — one command, instant answer |

Skills and AI assistants are great for one-off questions. AgentMesh is infrastructure — it runs continuously, accumulates knowledge, and serves the whole team without anyone having to think about it.

---

## Roadmap

### Phase 1 — Foundation
- [ ] Tree-sitter ingestion pipeline for Python
- [ ] Kuzu graph schema: `File`, `Function`, `Class`, `Module` nodes + `calls`, `imports` edges
- [ ] DuckDB lakehouse schema: events, snapshots, agent run log
- [ ] `ParseAgent` — incremental AST → graph updates on file change
- [ ] Basic CLI: `index`, `status`, `graph`

### Phase 2 — Intelligence
- [ ] LanceDB embedding store wired to graph nodes
- [ ] `AnswerAgent` — Q&A over graph + embeddings via Claude API
- [ ] `LineageAgent` — single-repo change impact analysis
- [ ] TypeScript/JavaScript parser support
- [ ] CLI: `ask`, `impact`, `explain`

### Phase 3 — Memory, Governance & Enforcement
- [ ] `MemoryAgent` — ingest git log, PR descriptions, ADR docs into graph
- [ ] `Decision` and `Incident` node types + `decided-by`, `broke-in` edges
- [ ] `PolicyAgent` — parse ADR text → extract constraints → generate Cypher rules (with human confirmation step)
- [ ] `EnforcementAgent` — evaluate policies against graph, run on every commit
- [ ] GitHub Actions integration — posts violations as PR comments
- [ ] `agentmesh policy check <adr-file>` — check one ADR across all repos
- [ ] `agentmesh policy status` — org-wide compliance dashboard across all ADRs

### Phase 4 — Cross-Repo & Ecosystem
- [ ] Multi-repo indexing — unified graph across repos
- [ ] Cross-repo blast radius analysis
- [ ] `ConsolidationAgent` — background graph maintenance, deduplication
- [ ] Web UI — graph explorer + chat
- [ ] MCP server — expose AgentMesh as a tool to other AI agents
- [ ] IDE extension (VS Code)

---

## Getting Started

> Early development. The setup below reflects the target experience; some steps are still being built.

```bash
# Clone and install from source
git clone https://github.com/your-username/agentmesh
cd agentmesh
pip install -e ".[dev]"

# Set your API key
export ANTHROPIC_API_KEY=sk-...

# Index one or more repositories
agentmesh index /path/to/repo-1 /path/to/repo-2

# Ask a question
agentmesh ask "What is the entry point of the payments service?"

# Check blast radius of a change
agentmesh impact repo-1::src/auth/token.py::validate_token

# ADR governance: which repos are violating ADR-007?
agentmesh policy check docs/adrs/ADR-007-repository-pattern.md --across-repos
# → PASS  user-service      (12/12 DB calls go through repositories)
# → FAIL  payments-service  (3 violations: direct DB import in invoice.py, order.py, refund.py)
# → FAIL  notifications     (1 violation: direct DB import in email_queue.py)
# → PASS  auth-service      (8/8 DB calls go through repositories)

# Show all ADRs and their current compliance status across the org
agentmesh policy status
# → ADR-003  API versioning            ████████░░  18/22 services passing
# → ADR-007  Repository pattern        ██████████  20/22 services passing
# → ADR-011  Structured logging        ██████░░░░  14/22 services passing
# → ADR-015  CODEOWNERS required       ████████████ 22/22 services passing
```

To run enforcement on PRs, add the GitHub Action (see `docs/ci-setup.md`).

---

## Project Structure

```
agentmesh/
├── agentmesh/
│   ├── ingestion/            # Tree-sitter parsers, git watcher, PR/ADR ingestors
│   ├── graph/                # Kuzu schema, graph client, Cypher query helpers
│   ├── lakehouse/            # DuckDB schema, Parquet event writers, snapshots
│   ├── embeddings/           # LanceDB store, chunking, embedding calls
│   ├── agents/
│   │   ├── base.py               # Agent base class + schema contracts
│   │   ├── parse_agent.py        # AST → graph, incremental
│   │   ├── memory_agent.py       # git/PR/ADR → graph decisions + history
│   │   ├── policy_agent.py       # ADR text → extracted rules → Cypher queries
│   │   ├── lineage_agent.py      # blast radius, cross-repo impact
│   │   ├── enforcement_agent.py  # rule evaluation, PR comments
│   │   ├── answer_agent.py       # Q&A
│   │   └── consolidation_agent.py  # background graph maintenance
│   ├── coordinator.py        # Agent orchestration loop
│   ├── rules.py              # Rule DSL + Cypher rule evaluator
│   └── cli.py                # Typer CLI
├── tests/
├── docs/
│   └── ci-setup.md
└── pyproject.toml
```

---

## Contributing

AgentMesh is early-stage and actively looking for contributors. Good first areas:

- **Language parsers** — add Tree-sitter support for Java, Go, or Rust
- **Graph schema** — improve node/edge types for richer institutional memory
- **PolicyAgent** — improve ADR parsing and rule extraction accuracy
- **Rule DSL** — design the Cypher-based rule schema for manual policies
- **MemoryAgent** — ingest GitHub PR data and link it to code nodes
- **Tests** — unit tests for ingestion pipeline and graph queries

See [CONTRIBUTING.md](CONTRIBUTING.md) for setup and coding conventions.
Open an issue before starting significant work so we can align on approach.

---

## Design Principles

1. **Cross-repo first** — the graph spans repos; single-repo is a special case
2. **Memory, not just index** — the *why* matters as much as the *what*
3. **ADRs as executable specs** — standards live in one place and are enforced, not aspirational
4. **Autonomous by default** — runs on commits, not on human invocation
5. **Local-first** — all data on disk, no cloud required to run
6. **Observable** — every agent action is logged; the lakehouse is queryable with plain SQL
7. **Composable** — each agent is independent; run one without the others

---

## License

MIT

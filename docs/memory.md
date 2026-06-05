# Zoe Memory Layer — Columnar Retrieval Architecture

Flat markdown files loaded at session start do not work as agent memory at any useful scale. This document describes the pattern that does.

---

## The Problem

Every Zoe stores knowledge in memory files: behavioral rules, user preferences, domain context, anti-patterns. The standard approach is to load them all at startup and hope the LLM retains the relevant ones across a long session.

Three structural failures make this unreliable:

**1. No selective retrieval.** All memories are treated as equal weight. Context compression pushes critical rules out of the attention window long before the relevant task appears.

**2. No query at decision time.** When the user says a trigger word, no mechanism fires to check what it means. The LLM must spontaneously recall a rule it read thousands of tokens ago.

**3. Training priors override session context.** Even when a memory is retrieved, the model's training generates stronger logit pressure than a single line of session context. The prior wins.

These are not prompting problems. They are retrieval architecture problems.

---

## The Pattern

Replace flat file memory with a columnar embedded database. Add a hook-based preprocessor that runs before the LLM sees each message. Structure memory retrieval as three layers that fire in sequence.

### Layer 1: Hook-Based Trigger Preprocessor

A pre-LLM hook intercepts every user message. A lightweight script:

1. Tokenizes the message into keywords
2. Runs exact-match lookup against a `triggers` table
3. For mechanical triggers (LOOK, screenshot, prep the advisor): **executes the command directly** and injects the result. The LLM never decides whether to act — it receives the result.
4. For judgment triggers: injects the retrieved rule as high-priority context

This solves the override problem structurally. The model's training prior is irrelevant because the action was already taken before the model started thinking.

**Failure mode:** False negatives only (novel phrasing misses a keyword). Never false positives. That is the right failure mode for an always-on layer.

### Layer 2: Domain/Task Retrieval

When the LLM begins processing, it classifies the message by domain (work, farm, family, system) and task type (demo, email, deck, meeting). First action: query the memory DB for relevant rules.

```sql
SELECT id, name, summary, anti_pattern FROM memory
WHERE list_contains(tasks, 'deck')
   OR list_contains(keywords, 'slide')
ORDER BY anti_pattern DESC, last_verified DESC NULLS LAST;
```

Anti-patterns sort first — guardrails load before positive guidance. Results injected as tool output (privileged attention position).

### Layer 3: Semantic Search Fallback

Full vector similarity using an embedding model (Nomic Embed or equivalent) plus the Lance extension. Runs only when Layers 1 and 2 return zero results, or on novel entities.

Embeddings computed on a 1-2 sentence distilled summary per memory, not raw content. Raw text is too noisy for cosine similarity.

---

## Why DuckDB

DuckDB is the right backing store. The access pattern is unpredictable — what's relevant depends on a combination of keyword, domain, task type, entity, and time context. You cannot pre-index for every combination.

**Columnar over row-store:** Writes are rare (a few memories per session). Reads happen on every user message against unpredictable attribute combinations. Columnar scans any column independently — you can slice on any combination of (domain, task_type, entity, keyword) without having predicted that combination at schema design time.

**DuckDB + Lance convergence (May 2026):** Lance is now a core DuckDB extension. Single SQL statement combines columnar structured queries, vector similarity, and full-text search:

```sql
SELECT * FROM lance_hybrid_search('memories.lance', 'embedding', query_vec,
    'text', 'demo screen', k=10, alpha=0.5)
WHERE domain = 'work'
ORDER BY _hybrid_score DESC;
```

No separate vector store. No ETL between systems. One query, one file, one dependency.

**Embedded, no server:** DuckDB runs in-process. The entire memory layer is a single `.duckdb` file that travels with the user's Zoe repo.

---

## Schema

```sql
CREATE TABLE memory (
    id              VARCHAR PRIMARY KEY,
    name            VARCHAR NOT NULL,
    summary         VARCHAR NOT NULL,        -- 1-2 sentences for embedding
    content         VARCHAR NOT NULL,        -- full text
    type            VARCHAR NOT NULL,        -- user | feedback | project | reference
    domains         VARCHAR[] NOT NULL,      -- work | farm | family | system
    tasks           VARCHAR[] NOT NULL,      -- demo | deck | email | meeting | ...
    keywords        VARCHAR[] NOT NULL,
    entities        VARCHAR[] NOT NULL,      -- named accounts, people, products
    products        VARCHAR[] NOT NULL,
    anti_pattern    BOOLEAN DEFAULT FALSE,   -- sort these first
    permanent       BOOLEAN DEFAULT TRUE,
    source_file     VARCHAR NOT NULL,
    created_at      TIMESTAMP NOT NULL,
    last_verified   TIMESTAMP,
    embedding       FLOAT[384]              -- nomic-embed-text-v1.5
);

CREATE TABLE triggers (
    keyword         VARCHAR PRIMARY KEY,
    memory_id       VARCHAR REFERENCES memory(id),
    action          TEXT NOT NULL,           -- what to do when this phrase appears
    context         TEXT,
    source_file     VARCHAR NOT NULL,
    case_sensitive  BOOLEAN DEFAULT FALSE
);
```

**Design rationale:** Single denormalized table with array columns. At this scale (hundreds of rows), normalization adds join overhead for zero benefit. Array columns (`domains[]`, `tasks[]`, `keywords[]`) let the columnar engine scan every dimension in one pass without pre-built indexes.

---

## Key Decisions

1. **Curate, don't auto-extract.** Mem0's 97.8% junk rate at 10K entries proves automated extraction does not work yet. Hand-written memories, reliably retrieved, beat voluminous memories retrieved unreliably.

2. **Bypass the LLM for mechanical triggers.** Do not ask the model to overcome its own training. Route around it. The hook executes the command before the model starts thinking.

3. **Embeddings on summaries, not content.** Raw markdown is too noisy for cosine similarity. A 1-2 sentence distilled summary produces tighter vector clusters.

4. **Anti-patterns sort first.** Guardrails must arrive before positive guidance or they are ignored. The schema enforces this at the query layer, not the prompt layer.

---

## Reference Implementation

Chloe (Jim O'Donnell's Zoe instance, running on the Hermes Agent SDK) ships a working implementation:

- **Database:** `~/chloe/STATE/chloe-memory.duckdb`
- **Migration:** `~/chloe/tools/memory/migrate.py` — ingests flat markdown memory files into the DB
- **Query CLI:** `~/chloe/tools/memory/recall.py` — trigger lookup, entity search, domain/task filtering, free-text search

```bash
recall.py --trigger "LOOK"                          # fires before LLM sees the message
recall.py --domain work --task demo                 # load guardrails before building
recall.py --entity "JLR"                            # all memories mentioning an account
recall.py --query "building a demo deck for Publix" # free-text scored search
```

The implementation is specific to Claude Code hooks. The pattern is not.

---

## References

- Mem0 junk audit: github.com/mem0ai/mem0/issues/4573
- MemGPT/Letta archival failures: github.com/letta-ai/letta/issues/381
- Memori (arXiv 2603.19935, March 2026): LLM-agnostic persistent memory
- Event-centric representation (arXiv 2511.17208, Nov 2025)
- DuckDB Lance extension: duckdb.org/docs/current/core_extensions/lance
- LanceDB hybrid search: docs.lancedb.com/search/hybrid-search

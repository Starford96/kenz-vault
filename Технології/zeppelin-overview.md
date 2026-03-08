# Zeppelin — Overview

## What is it

Open-source, self-hostable vector + full-text search engine.
S3-native: object storage (MinIO / Amazon S3 / Cloudflare R2) is the source of truth.
Nodes are stateless — restart anytime, data stays in S3.

GitHub: https://github.com/zepdb/zeppelin

---

## What problem it solves

Finding semantically similar content across large knowledge bases — fast.

Naive approach: compare query vector against every stored vector → too slow at scale.
Zeppelin uses IVF indexing, BM25, bitmap pre-filters to do this in milliseconds even on millions of documents.

---

## When it's worth it

**Worth it:**
- Many repos / services — need cross-repo semantic search
- Multiple knowledge sources: Jira + Slack + Confluence + code + docs — all indexed together
- Agent "memory" across incidents: past errors, how they were resolved, patterns
- Codebase search: find where a specific error type is handled, without reading everything

**Not worth it:**
- Single repo with good docs — agent can just read files directly
- Small scale with few documents

---

## Real use cases

**1. AI agent for automated bug fixing**
- Error log hits Slack → triggers agent
- Agent queries Zeppelin via MCP: gets relevant service context (architecture, known bugs, business logic)
- Agent builds fix strategy → opens PR on GitHub

**2. Personal knowledge base**
- Index Obsidian / Notion notes as .md files
- Ask questions in natural language → get semantically relevant results even if keywords don't match

**3. RAG for LLMs**
- Upload documents (PDFs, wikis, runbooks)
- LLM answers questions grounded in your actual content, not hallucinations

**4. Cross-service search**
- Unified search across all microservice docs, ADRs, post-mortems, runbooks

---

## Architecture

```
[Knowledge sources]          [Indexing — one-time / on update]
  .md files, Jira,
  Confluence, Slack   ──►   Python service
                              │  reads + parses content
                              │  splits into chunks
                              │  runs embedding model → vectors
                              ▼
                           Zeppelin  ──►  MinIO (S3)
                           stores & indexes vectors


[Query flow]
  User / Agent query
        │
        ▼
  Python service (MCP server)
        │  converts query → vector via embedding model
        ▼
     Zeppelin
        │  finds nearest vectors → returns chunk IDs
        ▼
  Python service
        │  retrieves original text chunks
        │  (optionally) sends to LLM with context
        ▼
     Answer
```

---

## Components needed

| Component | Role | Example |
|---|---|---|
| Zeppelin | Vector storage + search | Self-hosted, Docker |
| MinIO | S3-compatible object storage | Self-hosted, Docker |
| Embedding model | Text → vector conversion | `paraphrase-multilingual-MiniLM-L12-v2` |
| Python service | Orchestration: indexing + search + MCP | Custom, FastAPI or similar |

---

## Key rules

- **Same model for indexing and querying** — different models produce incompatible vectors
- **Chunking required** — models have token limits; split documents into 200–500 word chunks with overlap
- **Multilingual content** — use a multilingual model (e.g. `multilingual-e5-large`), not English-only
- **Garbage in, garbage out** — quality of results depends entirely on quality of indexed content

---

## MCP integration note

Zeppelin exposes REST API only. To use it with AI agents via MCP:
wrap Zeppelin calls inside a Python MCP server — that's the bridge between the agent and Zeppelin.
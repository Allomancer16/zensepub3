# GitTimewarp — Phase 1 Implementation Plan

## Overview

**GitTimewarp** is an AI agent that "drip-feeds" code from a pre-existing private GitHub repository to a public hackathon repository at realistic time intervals — simulating authentic, real-time development. The system:

1. Clones a private repo and analyses every file (LLM estimates hours-to-write per file)
2. Orders files intelligently using dependency graphs + git history (R1)
3. Clusters files into semantically coherent commit batches using ChromaDB + k-means (R5)
4. Pushes the batches at timed intervals to the public repo, starting with an AI-generated README (R8)

All eight refinements from the spec doc (R1–R8) are included.

---

## Architecture

```
git_timewarp/
├── main.py                    # CLI entry point
├── config.yaml                # All tuneable parameters
├── .env                       # API keys (GROQ_API_KEY, GITHUB_TOKEN)
├── requirements.txt
│
├── analyzer/
│   ├── clone.py               # Clone private repo via PyGitHub + gitpython
│   ├── walker.py              # Walk & filter files → manifest list
│   ├── dependency.py          # Topological sort + R1 git-history blend
│   ├── llm.py                 # Groq LLM calls: R2 few-shot, R6 streaming, R7 confidence, R3 cross-file
│   ├── few_shot_examples.py   # R2: pre-built few-shot examples for system prompt
│   └── hash_cache.py          # R4: SHA-256 content hashing helpers
│
├── rag/
│   ├── embedder.py            # sentence-transformers embeddings + R4 hash gating
│   ├── vectorstore.py         # ChromaDB collection wrapper
│   ├── retriever.py           # Semantic retrieval + R8 semantic_diff()
│   └── writer.py              # Write analysis.json, query_log.json, cross_file_analysis.json
│
├── pipeline/
│   ├── orchestrator.py        # Runs stages 1–4 in sequence
│   ├── ingest_stage.py        # Stage 1: validate manifest, apply R1 + R3 corrections
│   ├── embed_stage.py         # Stage 2: R4 hash-gated embedding
│   ├── retrieve_stage.py      # Stage 3: k-means + R5 coherence scoring loop
│   └── reason_stage.py        # Stage 4: batch reasoning + R3 cross-file + R8 README gen
│
├── pusher/
│   ├── setup.py               # Clone public repo, R8 semantic diff pre-flight
│   ├── committer.py           # Git commit/push via gitpython
│   └── loop.py                # Timed commit loop (README first, then batches)
│
└── utils/
    ├── logger.py              # Rich-based logging to activity.log + terminal
    └── audit.py               # Schedule validation + R5 coherence threshold check
```

---

## Proposed Changes

### Layer 0 — Project scaffold

#### [NEW] `requirements.txt`
- `groq`, `gitpython`, `PyGithub`, `chromadb`, `sentence-transformers`, `scikit-learn`, `rich`, `python-dotenv`, `pyyaml`, `networkx`, `ast` (stdlib)

#### [NEW] `config.yaml`
```yaml
embedding:
  model: all-MiniLM-L6-v2
  hash_cache: ./embedding_cache.json
llm:
  model: llama3-70b-8192
  confidence_threshold: 0.65
  max_tokens: 2048
pipeline:
  k_batches: 8
  coherence_low: 0.55
  coherence_high: 0.85
  max_coherence_iterations: 5
pusher:
  default_interval_minutes: 45
  public_repo_dir: ./public_clone
private_repo_dir: ./private_clone
output_dir: ./output
```

#### [NEW] `.env.example`
```
GROQ_API_KEY=your_key_here
GITHUB_TOKEN=your_token_here
```

---

### Layer 1 — Analyzer

#### [NEW] `analyzer/clone.py`
- Accepts `--private-repo` (SSH/HTTPS URL) and optional `--public-repo` URL
- Uses `gitpython` to clone into `./private_clone`

#### [NEW] `analyzer/walker.py`
- Walks `private_clone/`, filters binary files, `.git/`, node_modules, etc.
- Returns manifest: `[{path, content, size, extension, relative_path}]`


#### [NEW] `analyzer/few_shot_examples.py`
**R2:** Returns a formatted string of 3 hand-written examples (config file → low, utility module → medium, service class → high) for injection into the system prompt.

#### [NEW] `analyzer/hash_cache.py`
**R4:** `load_cache()`, `get_hash(content)`, `is_changed(path, content, cache)` — under 30 lines.

#### [NEW] `analyzer/llm.py`
All four refinements land here. Implement in order:
- **R6 Streaming:** `stream=True` on Groq calls, `Rich.Live` display, buffer accumulation before `json.loads()`
- **R2 Few-shot:** Inject `few_shot_examples.get_examples()` into system prompt
- **R7 Confidence:** Add `confidence` field to schema; if `< threshold`, make corrective second call with full content
- **R3 Cross-file:** `run_cross_file_pass(manifest, collection)` — retrieves top-2 similar pairs from ChromaDB, runs dedicated pair-reasoning prompt, returns `cross_file_relationships` list

---

### Layer 2 — RAG

#### [NEW] `rag/embedder.py`
- `sentence-transformers` (`all-MiniLM-L6-v2`) for embedding
- **R4:** Calls `hash_cache.is_changed()` before embedding; skips unchanged files; uses ChromaDB `upsert()`

#### [NEW] `rag/vectorstore.py`
- ChromaDB persistent client; exposes `get_collection()`, `upsert()`, `query()`
- Metadata schema includes `content_hash` (R4), `layer`, `hours`, `blended_score`

#### [NEW] `rag/retriever.py`
- `get_similar(summary, n)` — semantic nearest-neighbour query
- **R8:** `semantic_diff(public_manifest)` — embeds public repo files, queries private collection, returns duplicate pairs above 0.92 cosine similarity

#### [NEW] `rag/writer.py`
- Writes `output/analysis.json` (all fields including R1, R4, R7 additions)
- Writes `output/query_log.json`
- Writes `output/cross_file_analysis.json` (R3)

---




#### [NEW] `pipeline/retrieve_stage.py`
- K-means clustering of embeddings
- **R5:** Coherence scoring loop: computes intra-batch cosine similarity, splits weak batches (< 0.55), merges near-duplicate batches (> 0.85), repeats up to 5 iterations, logs each iteration to `query_log.json`

#### [NEW] `pipeline/reason_stage.py`
- Per-batch LLM reasoning for commit messages and timing
- **R3:** Runs `llm.run_cross_file_pass()` before batch loop
- **R8A:** `generate_readme(state)` — queries ChromaDB for 8 central files, calls Groq, writes `README.md` to public clone dir

#### [NEW] `pipeline/orchestrator.py`
- Runs Stage 1 → 2 → 3 → 4 sequentially; passes shared `state` dict between stages

---

### Layer 4 — Pusher

#### [NEW] `pusher/setup.py`
- Clones public repo; runs walker on it
- Calls `retriever.semantic_diff()` → stores duplicates in `state['duplicates']` (R8B)

#### [NEW] `pusher/committer.py`
- `gitpython`-based: stage files, commit with message, push to remote

#### [NEW] `pusher/loop.py`
- Push `README.md` as commit #0 with message `"initial project documentation"` (R8A)
- Iterate batches: check `state['duplicates']`, skip flagged files, sleep `interval_minutes`, push

---

### Layer 5 — Utils & Entry Point

#### [NEW] `utils/logger.py`
- Rich-based logger writing to `activity.log` and coloured terminal output

#### [NEW] `utils/audit.py`
- Validates schedule: checks total hours, push_order monotonicity
- **R5 extension:** Warns if any batch has `coherence_score < 0.5`

#### [NEW] `main.py`
CLI flags:
- `--private-repo` (required), `--public-repo` (required)
- `--interval` (default: 45 min)
- `--skip-embed`, `--no-stream`, `--skip-cross-file`, `--confidence-threshold`, `--dry-run`

Full 12-step execution sequence as defined in spec Step 10.

---

## Key Dependencies

| Package | Purpose |
|---|---|
| `groq` | LLM API (streaming + non-streaming) |
| `gitpython` | Clone repos, run git log, commit & push |
| `PyGithub` | GitHub API for token-auth repo access |
| `chromadb` | Persistent vector store |
| `sentence-transformers` | Local embeddings (no API key needed) |
| `scikit-learn` | K-means + cosine similarity (R5) |
| `networkx` | Topological sort for dependency graph |
| `rich` | Streaming terminal display (R6), progress bars |
| `python-dotenv` | Load `.env` |
| `pyyaml` | Load `config.yaml` |

---

## Verification Plan

### Automated
- Run `python main.py --dry-run --private-repo <url> --public-repo <url>` after each phase and verify:
  - `output/analysis.json` contains all required fields
  - `output/cross_file_analysis.json` has ≥ 5 entries
  - `output/query_log.json` shows ≥ 3 coherence iterations
  - `embedding_cache.json` exists; a second `--skip-embed` run hits 100% cache

### Manual
- Run on a real (test) private repo, inspect the public GitHub repo's commit history
- Verify README is the first commit
- Verify commit timestamps are spaced by `interval_minutes`
- Check for streaming output in terminal during LLM analysis phase

---

## Open Questions

> [!IMPORTANT]
> **Do you already have a Groq API key and GitHub personal access token?** These are required for the LLM calls and authenticated git pushes. If not, I'll add setup instructions to the README.

> [!NOTE]
> The spec uses `sentence-transformers` (local model) for embeddings to avoid a second API dependency. Should I keep this, or would you prefer to use Groq/OpenAI embeddings instead?

> [!NOTE]
> The default push interval is 45 minutes. During development/testing, you'd run with `--interval 1` for fast testing. Do you want a special `--test-mode` flag that uses 5-second intervals?


# Stack Research

**Domain:** Bilingual (Thai/English) AI-assisted IT incident triage + early-warning web app (UP IT Pulse)
**Researched:** 2026-08-18
**Confidence:** HIGH on infrastructure/versions, HIGH on pgvector guidance, MEDIUM-HIGH on embedding model choice for Thai, MEDIUM on frontier-LLM naming (fast-moving)

---

## TL;DR — The Prescription

1. **Keep the spec's stack.** Next.js 16.3 + FastAPI + Postgres 18 + pgvector 0.8.6 is exactly the 2026 standard for this shape of app. Nothing in the research contradicts §14 of `docs/SPEC.md`.
2. **Canonical embedding dimension = 1024.** Default model: `BAAI/bge-m3` (self-hosted, 1024-dim, MIRACL-Thai nDCG@10 **83.7**). Hosted alternative: `gemini-embedding-001` truncated to 768 (must L2-normalize manually). Both fit pgvector's `vector` HNSW limit.
3. **Do NOT build a vector index at MVP scale.** At ≤100k tickets with a mandatory 24-hour `created_at` filter, exact scan is faster *and* 100% recall. Adding HNSW here actively hurts (post-filter recall collapse). Add HNSW only when measured p95 misses the 1s NFR.
4. **Structured output = `response_format: {"type":"json_schema", "strict": true}`** via the `openai` SDK pointed at `AI_BASE_URL`, wrapped in `instructor` for Pydantic validation + retry. Three-tier degradation ladder (strict schema → tool-call → json_object + repair). Never regex-parse prose.
5. **PII masking = deterministic regex, not ML.** Presidio has no usable Thai NER model; the spec's five entity classes are all pattern-based anyway. Must handle Thai digits `๐-๙` and `+66` phone forms.
6. **Thai keyword search needs `pg_trgm`, not `tsvector`.** Postgres has no Thai text-search parser (Thai has no inter-word spaces).
7. **Replace three spec-adjacent defaults:** `passlib` → `pwdlib[argon2]`, `python-jose` → `PyJWT`, `next-intl` → plain Thai copy module (no locale switcher needed for MVP).

---

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| **PostgreSQL** | **18.6** | Primary datastore | Latest stable minor (released 2026-08-13). pgvector officially supports PG18 since 0.8.1. One database for relational + vector + trigram search = one Docker service, no separate vector DB to explain to judges. |
| **pgvector** | **0.8.6** | Vector similarity search | Latest stable (2026-08-13). 0.8.0 added **iterative index scans**, which is *the* feature that makes filtered vector search (your 24h lookback) viable when you do eventually index. Docker image `pgvector/pgvector:0.8.6-pg18`. |
| **FastAPI** | **0.141.1** | Backend API | Pydantic-native request *and* AI-output validation in one type system; auto-generates OpenAPI → frontend types. `BackgroundTasks` covers FR-002's "background job" requirement with zero extra services. |
| **Pydantic** | **2.13.4** | Schema validation (API + AI output) | `model_json_schema()` feeds directly into LLM strict JSON-schema mode — the same class is your validator *and* your prompt contract. This is the single highest-leverage choice in the whole stack for FR-004. |
| **SQLAlchemy** | **2.0.52** | ORM | 2.0 async style; required by `pgvector-python` ≥0.5.0. Typed `Mapped[]` declarative models pair well with Pydantic DTOs. |
| **Alembic** | **1.19.1** | Migrations | Needed for the `CREATE EXTENSION vector` / `pg_trgm` step and for the embedding-dimension change path (Edge Case 14). |
| **psycopg** | **3.3.4** | Postgres driver | **Choose psycopg3 over asyncpg.** One driver serves both async app (`postgresql+psycopg://`) and sync Alembic; `pgvector.psycopg` returns `Vector` objects; wheels available for Python 3.13/3.14. asyncpg is marginally faster but forces two driver URLs and its last release predates Python 3.14. |
| **Python** | **3.13.x** | Runtime | All stack deps ship 3.13 wheels today. 3.14 is supported by FastAPI/SQLAlchemy/Pydantic but some transitive C-extension deps (e.g. `asyncpg` 0.31, older `psycopg` binaries) lag — not worth the hackathon risk. |
| **Next.js** | **16.3.1** | Frontend | App Router + Server Components. Turbopack is default in 16, so dev startup is fast on a laptop demo. |
| **React** | **19.2.x** | UI runtime | Required peer for Next 16 and shadcn/ui's current components. |
| **TypeScript** | **5.9.3** (recommended) / 7.0.2 (optional) | Types | **Pin 5.9.3 for the hackathon.** TS 7.0 (Go-native, GA 2026-07-08, ~10x faster) is supported by Next 16.3 via `experimental.useTypeScriptCli`, but **ships without a stable programmatic API until 7.1** — meaning typed-lint plugins and some editor/test tooling can misbehave. Upgrade after the demo, not before. |
| **Tailwind CSS** | **4.3.3** | Styling | v4 is the shadcn/ui default; CSS-first `@theme` config, no `tailwind.config.js` needed. |
| **shadcn/ui** | CLI **4.18.0** | Component library | Meets the spec's accessibility requirement (Radix primitives → keyboard nav, ARIA, focus management for free). Components are copied into your repo, so no version-lock risk mid-hackathon. Ships a `Chart` wrapper around Recharts. |
| **Docker Compose** | v2 (compose spec) | Deployment | Spec-mandated. Services: `web`, `api`, `postgres`, `redis` (optional), plus optional `embeddings` (TEI) if self-hosting. |

### AI Layer

| Component | Recommendation | Version | Why |
|-----------|---------------|---------|-----|
| **LLM client** | `openai` Python SDK against `AI_BASE_URL` | **3.2.0** | The OpenAI wire format is the de-facto standard: OpenAI, Azure, OpenRouter, Together, Groq, vLLM, Ollama, and SCB10X Typhoon (via vLLM) all speak it. This satisfies §14's "provider must be swappable via env vars" with **zero adapter code beyond a base_url**. ⚠️ v3.0.0 (2026-08-12) switched to **httpx2** as default HTTP client and no longer installs `httpx` — see Version Compatibility. |
| **Structured output** | `instructor` | **1.15.4** | Wraps any OpenAI-compatible client, takes a Pydantic model, and does validation + automatic re-ask on failure. ~3M downloads/month. Gives you FR-004's "retry then manual queue" ladder almost for free. |
| **Structured output mode** | `json_schema` with `strict: true` | — | OpenAI reports 100% schema adherence with strict mode vs. best-effort for JSON mode. vLLM (XGrammar backend, default since 0.7) and Ollama both implement grammar-constrained decoding through the same `response_format` field — so this works self-hosted too. |
| **Embedding model (default)** | `BAAI/bge-m3` | 1024-dim | See Embedding Model Decision below. |
| **Embedding server (self-host)** | HuggingFace **text-embeddings-inference** (TEI) or **Infinity** | latest | Exposes `/embeddings` in OpenAI format → same adapter, no code change. CPU-only is fine: your embedded text is `summary_th + category + affected_service + location`, i.e. <100 tokens, and the corpus is ~10k rows. |
| **LLM model (chat)** | Any current frontier model with strict JSON schema + strong Thai | — | Named models churn monthly; the adapter is the point. For Thai quality the Gemini family has consistently led translation/multilingual evals, and OpenAI's models have the most mature strict-schema implementation. **Local/sovereign option worth having in `.env.example`: `scb10x/typhoon2.1-gemma3-12b` (Thai/English bilingual, Gemma3-based) served via vLLM — a Thai-university-appropriate story for judges.** Confidence: MEDIUM (blog-sourced model rankings). |

### Supporting Libraries — Backend

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `pgvector` (Python) | **0.5.0** | `Vector`/`HALFVEC` SQLAlchemy types, driver codecs | Always. ⚠️ Breaking in 0.5.0: SQLAlchemy/SQLModel now return **`list`**, not `numpy.ndarray`; NumPy dependency dropped; requires Python ≥3.10 + SQLAlchemy ≥2. |
| `pydantic-settings` | **2.15.0** | Typed env-var config | Always — this is how §16's env vars (`SIMILARITY_THRESHOLD`, `INCIDENT_WINDOW_MINUTES`, `AI_*`) become validated typed settings instead of scattered `os.getenv`. |
| `pwdlib[argon2]` | **0.3.1** | Demo-account password hashing | Always. **Use instead of `passlib`** (last release 2020, breaks against modern bcrypt). Argon2id is the current OWASP recommendation. |
| `PyJWT` | **2.13.0** | Demo-login session tokens | Always. **Use instead of `python-jose`** (effectively unmaintained). Bind role server-side into the token claim — PROJECT.md flags client-selectable roles as a privilege-escalation risk. |
| `cryptography` | **50.0.0** | Field-level encryption (Fernet) for `description_original_encrypted`, `email_encrypted` | Always. Wrap in a SQLAlchemy `TypeDecorator` so encryption is invisible to business logic. Key from `FIELD_ENCRYPTION_KEY`. |
| `regex` | latest | PII masking patterns | Always. Better Unicode property support than stdlib `re` — needed for Thai script classes and Thai digits. |
| `pythainlp` | **5.3.7** | Thai word segmentation | **Only if** you build the tokenized-tsvector search path. Not needed for PII masking or embeddings. |
| `slowapi` | **0.1.10** | Rate limiting on `POST /tickets` (NFR + Edge Case 8) | Always. In-memory backend is fine for a single-worker demo; point it at Redis if you scale workers. |
| `structlog` | **26.1.0** | Structured JSON logs + correlation ID | Always — NFR "Observability" requires structured logs with a correlation ID per request/background job, and explicitly forbids logging raw PII. Structlog processors are the clean place to enforce a PII-redaction filter. |
| `arq` | **0.28.0** | Real async job queue | **Only if** `BackgroundTasks` proves insufficient (i.e. you need retry-across-restart or job status). `BackgroundTasks` loses jobs on crash. Requires the optional `redis` service. |
| `httpx2` | (installed by `openai` 3.x) | HTTP client | Transitively. Don't pin `httpx` 0.28 alongside it without reading the migration note. |
| `pytest` + `pytest-asyncio` | **9.1.1** / latest | Tests | Always — §21 mandates unit tests for masking, rules, similarity, thresholds, transitions, authz. |
| `polyfactory` | **3.3.0** | Pydantic model factories for the 60-ticket seed set + 100-example eval set | Optional, but pays for itself on §19/§21 data generation. |

### Supporting Libraries — Frontend

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `@tanstack/react-query` | **5.101.4** | Server state, cache, polling | Always. Polling is how the Suspected-Incident alert appears live on the Incident Monitor without WebSockets. |
| `@tanstack/react-table` | **9.1.2** | Ticket queue table (sort/filter/paginate) | Always for §13.2. Headless — pairs with shadcn/ui `Table`. |
| `react-hook-form` + `zod` + `@hookform/resolvers` | **7.85.0** / **4.4.3** / **5.9.1** | Step-by-step ticket form with per-field validation | Always for §13.1. Mirror the FastAPI Pydantic constraints (title 5–150, description 10–5000) in Zod so the user sees errors before submit. |
| `recharts` | **3.10.1** | Dashboard charts | Always. shadcn/ui's `chart` component is a Recharts wrapper — you get themed, accessible charts with almost no code. Prefer over ECharts (see Alternatives). |
| `openapi-typescript` + `openapi-fetch` | **7.13.0** / **0.17.0** | Generate TS types + typed fetch client from FastAPI's `/openapi.json` | Always. This *is* the `packages/shared-types` package in §15 — generated, not hand-maintained. Prevents frontend/backend schema drift on the 13-field AI analysis object. |
| `lucide-react` | **1.31.0** | Icons | Always (shadcn/ui default). |
| Thai webfont via `next/font/google` | — | `IBM Plex Sans Thai` or `Noto Sans Thai` | Always. **Thai-specific gotcha:** Latin-first fonts fall back to system Thai rendering that clips tone marks; also increase `line-height` (e.g. `leading-relaxed`) because Thai stacks vowels/tones above and below the baseline. |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| **uv** 0.12.5 | Python dependency + venv management | Use `uv` in the API Dockerfile (`uv sync --frozen`) — 10–100x faster than pip, and `uv.lock` gives the reproducible builds §22 implies. |
| **ruff** 0.16.3 | Lint + format (replaces black/isort/flake8) | Single tool, one config block in `pyproject.toml`. |
| **mypy** 2.3.1 | Type checking | mypy 2.x is current. Alternative: `ty` (Astral, v0.0.72) — much faster but still pre-1.0; not for a graded deliverable. |
| **ESLint** 10.8.1 | JS/TS lint | Note: Next 16 removed `next lint`; the codemod migrates you to the ESLint CLI. |
| **Node.js** 24 LTS | Frontend build/runtime | Node 24 is Active LTS as of Aug 2026; Node 26 is Current and becomes LTS in Oct 2026. Use `node:24-alpine` in the `web` image. |
| **`@next/codemod`** | Next 16 migration | Only relevant if you scaffold from a Next 15 template — it renames `middleware.ts` → `proxy.ts` and converts sync `params`/`searchParams` to async. |

---

## Embedding Model Decision (Thai)

### The core finding: Thai is the hard case

Thai is the single worst-performing language for most multilingual embedding models — Jina's own analysis identifies Thai as producing "the largest performance drop for most models," attributing it to the script and the absence of word delimiters. MIRACL classes Thai as low-resource. **Therefore: do not assume "multilingual" means "good at Thai," and do not pick a model on English MTEB score.** (Confidence: HIGH)

### Evidence table

| Model | Dims | Thai evidence | Confidence |
|-------|------|---------------|------------|
| **`BAAI/bge-m3`** | **1024** (native, not MRL) | **MIRACL Thai dev nDCG@10 = 83.7** (BGE M3-Embedding paper, arXiv 2402.03216) | HIGH — primary-source paper |
| `Qwen3-Embedding-8B` | 4096 (MRL, custom range) | **Best Thai score among 13 open models on SEA-BED: 81.49** across all Thai tasks | HIGH — peer-reviewed benchmark (arXiv 2508.12243) |
| `Qwen3-Embedding-0.6B` | up to 1024 (MRL 32–1024) | Same family; MTEB-multilingual mean 64.33 vs 70.58 for 8B. Needs `Instruct:\nQuery:` prefix on the query side — omitting it costs 1–5% retrieval | MEDIUM for Thai specifically |
| `multilingual-e5-large-instruct` | **1024** | **SEA-BED Thai = 81.11**; best overall across the 10 SEA languages. 512-token input limit (irrelevant here) | HIGH |
| `gemini-embedding-001` | 3072, MRL to 128–3072 (768/1536/3072 recommended) | Held #1 on MTEB(Multilingual); 100+ languages. No published Thai-specific figure found. 2048-token input | MEDIUM |
| `gemini-embedding-2` | MRL 128–3072, **auto-normalizes truncated output**, 8192 tokens, multimodal | Newer; no Thai-specific figure found | MEDIUM |
| `cohere embed-v4` | 256/512/1024/1536 | Vendor + secondary sources claim materially better non-Latin-script retrieval than OpenAI small; no primary Thai number found | LOW–MEDIUM |
| `text-embedding-3-large` | 3072 (`dimensions` param shortens) | MTEB 64.6%; OpenAI cites improved MIRACL average but publishes no Thai breakdown. Not represented as a Thai leader in any benchmark found | LOW for Thai |
| `all-MiniLM-L6-v2`, `paraphrase-multilingual-MiniLM` | 384 | Not evaluated as competitive on Thai in any source found; commonly used by default in tutorials | Avoid |
| `WangchanBERTa`, `PhayaThaiBERT` | 768 | Thai-specific *encoders*, strong on token-level/classification tasks — but **not contrastively trained for sentence similarity**. Would need fine-tuning before they beat bge-m3 at retrieval | Avoid for MVP |

### The prescription

**Set `AI_EMBEDDING_DIM=1024` as the project's canonical dimension** and pick one of:

| Scenario | Model | Config |
|----------|-------|--------|
| **Default / offline demo / zero API cost** (recommended) | `BAAI/bge-m3` via TEI container | 1024-dim native, CPU-viable, best primary-source Thai number of any option |
| **No GPU, no extra container, API budget exists** | `gemini-embedding-001` @ `output_dimensionality=768` | ⚠️ You must **L2-normalize manually** for any dim ≠ 3072 on `-001`. `gemini-embedding-2` normalizes automatically. Use `task_type=RETRIEVAL_DOCUMENT` / `RETRIEVAL_QUERY` |
| **GPU available, want best measured Thai OSS quality** | `Qwen3-Embedding-0.6B` @ 1024 | Must prepend the `Instruct:` prefix on queries |
| **Already committed to OpenAI-only** | `text-embedding-3-large` with `dimensions=1024` | Acceptable fallback; do **not** use the 3072 default (see pitfall below) |

**Mandatory implementation rules regardless of model:**
1. Store `embedding_model` **and** the dimension per row (`ticket_embeddings` already has `embedding_model` — add `embedding_dim`). Edge Case 14 requires re-indexing on model change; you cannot detect the need without this.
2. Use **cosine** distance (`vector_cosine_ops`, `<=>`), and convert to the spec's `semantic_similarity ∈ [0,1]` as `1 - (a <=> b)`.
3. Normalize vectors on write. Then `<#>` (negative inner product) is mathematically equivalent to cosine and cheaper — a free optimization if you want it.
4. **Do not tune `SIMILARITY_THRESHOLD=0.82` until you've measured it against your own model.** Cosine score distributions differ substantially between bge-m3 and gemini; 0.82 is calibrated for neither. Build the eval-set harness (§21) *before* freezing the threshold.

---

## PostgreSQL + pgvector at This Scale (tens of thousands of tickets)

### The counter-intuitive headline: don't index yet

Your similarity query is **always** filtered by a 24-hour window (FR-007). pgvector applies `WHERE` clauses **after** the index scan, so a restrictive time filter against an HNSW index returns far fewer than `k` rows — the classic "my top-5 came back with 1 result" failure. (Confidence: HIGH — stated in pgvector's own README and reproduced across production write-ups.)

At your scale the math favours exact search decisively:
- 10,000 rows × 1024 dims × 4 bytes = **~40 MB** of vector data — fits in `shared_buffers`.
- A sequential scan + exact cosine over that is single-digit-to-low-tens of milliseconds, against an NFR budget of **1 second for 10,000 tickets**.
- Exact search gives **100% recall**, which matters because §3 grades you on "≥80% of known duplicates appear in Top 5." An approximate index can only lose you points here.

**Phase 7 (Embedding + Similar Ticket search) recommendation:**

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE TABLE ticket_embeddings (
  ticket_id       uuid PRIMARY KEY REFERENCES tickets(id) ON DELETE CASCADE,
  embedding       vector(1024) NOT NULL,
  embedding_model text        NOT NULL,
  embedding_dim   int         NOT NULL,
  created_at      timestamptz NOT NULL DEFAULT now()
);

-- Index the FILTER, not the vector, at MVP scale.
CREATE INDEX ix_ticket_embeddings_created_at ON ticket_embeddings (created_at DESC);
CREATE INDEX ix_tickets_created_at          ON tickets (created_at DESC);
```

Keeping embeddings in a **separate table** (as the spec already does) is also correct for a second reason: it keeps the `tickets` table narrow, so the ticket-queue query in §13.2 never drags 4 KB of vector per row through the buffer cache.

### When and how to add HNSW

Add it only when a measured p95 breaches budget — realistically past ~200k rows.

```sql
SET maintenance_work_mem = '2GB';        -- build is far faster if the graph fits
SET max_parallel_maintenance_workers = 7;
CREATE INDEX ON ticket_embeddings USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);   -- pgvector defaults; fine to start
```

Then, per session/query:
```sql
SET hnsw.ef_search = 100;                -- default 40; higher = better recall, slower
SET hnsw.iterative_scan = 'relaxed_order';  -- REQUIRED with your 24h filter
SET hnsw.max_scan_tuples = 40000;        -- default 20000
```
- `strict_order` guarantees exact distance ordering; `relaxed_order` gives better recall for the same work. Use `strict_order` if the UI shows scores as a ranked list you're claiming is exact.
- Recall verification recipe: run each k-NN query twice — once normally, once with `SET enable_seqscan = off` disabled/reversed to force exact — and diff the ID sets.

### `halfvec` and quantization: not yet

`halfvec` halves storage and raises the HNSW indexable ceiling to 4,000 dims. **You don't need it at 1024 dims / 50k rows.** It only becomes necessary if you switch to a 3072-dim model (see pitfall) or index millions of rows.

### Hard dimension limits (memorize these)

| Type | Storable dims | **HNSW-indexable dims** | IVFFlat-indexable |
|------|---------------|-------------------------|-------------------|
| `vector` | 16,000 | **2,000** | 2,000 |
| `halfvec` | 16,000 | **4,000** | 4,000 |
| `bit` | — | 64,000 | 64,000 |
| `sparsevec` | 16,000 non-zero | 1,000 non-zero | — |

**This is why `text-embedding-3-large`'s 3072 default is a trap:** you can store it in `vector(3072)` but you can never HNSW-index it. Either pass `dimensions=1024`, or cast to `halfvec(3072)` in the index expression.

### Thai keyword search (FR-014) — a real gotcha

PostgreSQL ships **no Thai text-search configuration**, because `to_tsvector` tokenizes on whitespace and Thai does not put spaces between words. `to_tsvector('simple', 'เข้าอีเมลไม่ได้')` yields one useless token. (Confidence: HIGH — confirmed by the pgsql-hackers thread proposing ICU tokenization precisely because Thai/CJK are unsupported.)

**Recommended:** trigram search — zero extra services, works on Thai *and* English substrings, and covers the "search by title/description" requirement literally.
```sql
CREATE INDEX ix_tickets_title_trgm
  ON tickets USING gin (title gin_trgm_ops);
CREATE INDEX ix_tickets_desc_masked_trgm
  ON tickets USING gin (description_masked gin_trgm_ops);
-- query: WHERE description_masked ILIKE '%' || :q || '%'
```
**Only if you need ranked BM25-style relevance:** segment with PyThaiNLP into a space-joined `search_text` column, then `to_tsvector('simple', search_text)`. Costs you a denormalized column, a trigger/task to maintain it, and a Thai-tokenizer dependency. Defer past MVP. (`pg_bigm` is the third option — bigram FTS designed for non-space-delimited languages — but it's an extra extension in the Postgres image for little gain over `pg_trgm` here.)

### Incident detection: avoid the O(n²) trap

FR-008 needs "≥5 tickets, ≥3 reporters, 15-min window, avg similarity ≥0.82." Do **not** compute pairwise similarity across the window on every insert. Run **one** k-NN query for the newly-created ticket restricted to the window, then aggregate over the returned neighbours. One vector query per ticket, not N².

### Postgres config for the demo container

`shared_buffers` ≈ 25% of container RAM, `work_mem` 32–64MB, `maintenance_work_mem` 512MB–2GB (only matters if/when you build HNSW). Everything else default. Do not over-tune a hackathon demo.

---

## LLM Structured Output: The Reliability Ladder

FR-004 demands schema-valid JSON with 13 fields, `null`/`UNKNOWN` instead of guesses, and a defined failure path. Implement **exactly this ladder** and log which rung produced the result (feeds the "JSON schema success rate" metric in §21):

| Rung | Mechanism | Reliability | Notes |
|------|-----------|-------------|-------|
| **1** | `response_format={"type":"json_schema","json_schema":{...,"strict":true}}` | ~100% schema adherence (OpenAI's published eval) | Grammar-constrained decoding. Same field works on vLLM (XGrammar) and Ollama, so it survives a provider swap. |
| **2** | Strict **tool/function calling** (`strict: true`) | Very high | Fallback for providers that implement strict tools but not strict `response_format`. |
| **3** | `{"type":"json_object"}` + Pydantic validate + **one** re-ask with the validation error | Good | `instructor` does this loop for you. |
| **4** | Free text + JSON extraction | Do not ship | Only as a last-resort log-and-fail path. |

**Strict-mode schema constraints you will hit** (HIGH confidence — these are documented OpenAI limitations, and XGrammar-based servers have similar restrictions):
- Every property must be listed in `required`. Optional fields are expressed as **nullable unions**: `"type": ["string", "null"]` — which happens to match the spec's "unknown → `null`" rule perfectly.
- `additionalProperties: false` is mandatory on every object.
- Numeric/string *validators* (`minimum`, `maximum`, `format`) are generally not enforced by the decoder. So `confidence: 0.0–1.0` must be range-checked in Pydantic **after** decoding, not trusted from the schema.
- Enums are enforced — use them for `category`, `priority_suggestion`, `impact_scope`, `team_suggestion`. But **`team_suggestion` values must be injected into the prompt/schema from the DB at call time**, because §6 requires team names to be configurable, not hardcoded.
- `$defs`/nesting depth and total property count are capped; your 13-field flat object is comfortably inside all limits.

**Prompt-version discipline:** FR-004 requires storing `prompt_version`. Derive it as a hash of (system prompt text + `model_json_schema()` output) so a schema change automatically bumps the version. This makes §24's "prompt version comparison" possible later for free.

**Prompt injection (NFR + Edge Case 4):** put ticket text inside a clearly delimited, labelled user-content block and state in the system prompt that its contents are data. Structured decoding is itself a strong defense: a constrained decoder physically cannot emit "ignore previous instructions" prose — it can only emit tokens matching your schema. That's an under-appreciated security argument for rung 1.

---

## PII Masking Stack

**Recommendation: hand-rolled deterministic regex, ordered longest-first, with a reversible token map. No ML.** (Confidence: HIGH)

Why not Presidio (`presidio-analyzer` 2.2.364)? It's the obvious-looking answer and it's wrong here:
- Its NER engine is spaCy/Stanza/transformers-based, and **there is no production-grade Thai spaCy pipeline** to plug in. Presidio's own docs say non-English requires you to supply a model and rewrite the context-word lists per language.
- All five entity classes the spec requires (`EMAIL`, `PHONE`, `PERSON_ID`, `IP`, `SECRET`) are **pattern-detectable**. Presidio's regex recognizers would be doing the work anyway, wrapped in ~200MB of dependency.
- The grading bar is "≥95% of emails/phones masked" — a recall target that well-written regex + a test corpus hits deterministically, and that you can *prove* in a unit test.

Add Presidio only if scope later expands to Thai **person names** (which genuinely needs NER).

**Thai-specific masking gotchas to encode as test cases** (these are where naive regex fails):
1. **Thai digits `๐๑๒๓๔๕๖๗๘๙`** — normalize to ASCII digits before phone/ID matching, or phone numbers written in Thai numerals slip through.
2. **Thai phone formats** — `08X-XXX-XXXX`, `0XXXXXXXXX`, `+66 8X XXX XXXX`, `66-8X-XXX-XXXX`, and internal 4–5 digit extensions (`ต่อ 1234`).
3. **No word boundaries** — `\b` is unreliable adjacent to Thai script. Use explicit lookarounds on `[^\d]` / Unicode property classes via the `regex` package rather than `\b`.
4. **Student/staff IDs** — must be a *configured* pattern, and must not swallow years (`2569`), room numbers, or error codes (`500`, `404`). Require a length/prefix anchor.
5. **Secrets** — layer keyword-adjacency (`password:`, `token=`, `api[_-]?key`) on top of high-entropy detection. Per FR-003 these must be destroyed, not just tokenized — never persist the original.
6. **Ordering** — mask emails *before* phones, or the digits inside `user1234@up.ac.th` get partially phone-masked and corrupt the email match.

**Token map:** store `{"[EMAIL_1]": <ciphertext>}` alongside the ticket so an authorized, audit-logged unmask is possible (FR-003) without ever sending originals to the model. Encrypt with the same `cryptography` Fernet key as `description_original_encrypted`.

---

## Rule Engine: Build, Don't Import

FR-006 needs versioned, admin-editable, audit-logged, explainable rules. **Do not pull in a rules-engine library.** (`json-logic-py`, `business-rules`, `durable-rules` are all low-activity, and none give you versioning or audit trails.)

Instead: model rules as **Pydantic classes persisted as JSONB rows** with a `version` column, and write a ~100-line evaluator.
- The spec's YAML example (`when: {impact_scope: MANY_USERS} then: {priority: P2}`) is a flat conjunctive match — trivially expressible as a Pydantic model, and directly renderable in an Admin form.
- **Do not store rules in a YAML file**, despite §10's illustration: §6/FR-006 require Admin-UI editing plus audit logging, which means database rows.
- Every evaluation must emit the list of matched rule IDs → that *is* the "Rules explain" half of the product principle, and it's what the §13.3 three-way diff (user / AI / rule / officer) renders.
- Guardrail as code, not prompt: PROJECT.md flags a conflict — §10.2 makes `P1` human-only, yet the `confirmed-major-critical` rule suggests `P1`. Resolve by typing the output field as `priority_suggestion`, never `priority`, and enforcing at the API layer that only an authenticated Admin can write `priority = P1`.

---

## Installation

```bash
# ── Backend (uv) ────────────────────────────────────────────────
uv init apps/api && cd apps/api
uv add "fastapi[standard]==0.141.1" "uvicorn[standard]==0.52.3" \
       "pydantic==2.13.4" "pydantic-settings==2.15.0" \
       "sqlalchemy==2.0.52" "alembic==1.19.1" "psycopg[binary,pool]==3.3.4" \
       "pgvector==0.5.0" \
       "openai==3.2.0" "instructor==1.15.4" \
       "pwdlib[argon2]==0.3.1" "pyjwt==2.13.0" "cryptography==50.0.0" \
       "regex" "slowapi==0.1.10" "structlog==26.1.0" "python-multipart"
uv add --dev "pytest==9.1.1" pytest-asyncio "ruff==0.16.3" "mypy==2.3.1" \
             "polyfactory==3.3.0" httpx2

# Optional, only if you outgrow BackgroundTasks:
#   uv add "arq==0.28.0" "redis==8.1.0"
# Optional, only for the tokenized-tsvector search path:
#   uv add "pythainlp==5.3.7"

# ── Frontend ────────────────────────────────────────────────────
npx create-next-app@16.3.1 apps/web --typescript --tailwind --app --eslint
cd apps/web
npm pkg set devDependencies.typescript=5.9.3     # pin: see Version Compatibility
npx shadcn@4.18.0 init
npx shadcn@4.18.0 add button card table badge dialog form input textarea \
                      select tabs sonner chart skeleton alert
npm i @tanstack/react-query@5.101.4 @tanstack/react-table@9.1.2 \
      react-hook-form@7.85.0 zod@4.4.3 @hookform/resolvers@5.9.1 \
      recharts@3.10.1 openapi-fetch@0.17.0
npm i -D openapi-typescript@7.13.0

# Generate the shared-types package from the live FastAPI schema:
npx openapi-typescript http://localhost:8000/api/v1/openapi.json \
  -o ../../packages/shared-types/api.d.ts

# ── Data ────────────────────────────────────────────────────────
# docker-compose.yml
#   postgres:   image: pgvector/pgvector:0.8.6-pg18
#   redis:      image: redis:8-alpine            # optional
#   embeddings: image: ghcr.io/huggingface/text-embeddings-inference:cpu-latest
#               command: --model-id BAAI/bge-m3   # optional, self-hosted path
```

---

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| pgvector in Postgres | Qdrant / Weaviate / Milvus | Millions of vectors, or you need native filtered-HNSW. At 10k–100k tickets a dedicated vector DB adds a service, a consistency problem (tickets and vectors in different stores, no transactions), and a story you have to justify to judges. |
| Exact scan (no vector index) | HNSW from day one | >200k rows, or measured p95 breaches the 1s NFR. Revisit *with numbers*, not on principle. |
| `pg_trgm` for Thai search | PyThaiNLP-segmented tsvector; `pg_bigm`; `pg_textsearch` (Timescale BM25) | You need ranked relevance rather than substring matching, or Thai-word-accurate hit highlighting. Post-MVP. |
| `psycopg` 3 | `asyncpg` 0.31 | Measured DB-driver-bound throughput problem. Costs you a second driver URL for Alembic and lags on new Python releases. |
| `openai` SDK + `instructor` | `litellm` 1.97.0 | You must call a provider with **no** OpenAI-compatible endpoint, or you want built-in cost tracking / load balancing / a proxy. LiteLLM is a large dependency with a very fast release cadence — extra breakage surface during a hackathon, and unnecessary since Gemini, Anthropic, Together, Groq, vLLM, Ollama and Typhoon all expose OpenAI-compatible routes. |
| `openai` SDK + `instructor` | `pydantic-ai` | You're building a genuinely agentic, multi-step tool-calling flow. Overkill for one-shot extraction, and the spec's guardrails deliberately forbid autonomous action. |
| `openai` SDK + `instructor` | `outlines` / raw XGrammar | You self-host and want to own the grammar. Redundant — vLLM/Ollama already expose constrained decoding through `response_format`. |
| `FastAPI BackgroundTasks` | `arq` + Redis | You need retry-across-process-restart or job status polling. §17 says "background job must retry" — 2 in-process retries with exponential backoff satisfies §10.3 literally; `arq` is the honest answer if you want durability. Note PROJECT.md flags an unresolved retry-count conflict (§10.3 says 2, Edge Case 5 says 1) — resolve before implementing. |
| `Recharts` 3.10 | `ECharts` 6.1 | Very large datasets, or exotic chart types. Recharts wins here because shadcn/ui ships a themed wrapper for it — meaningfully less code for §13.5. |
| Plain Thai copy module | `next-intl` 4.13.7 | You genuinely need a runtime TH/EN UI switcher. §17 says "UI หลักเป็นภาษาไทย" with English terms in parentheses — that's one language with bilingual *labels*, not i18n. The bilingual requirement is about **ticket input text**, which is the model's problem, not the UI's. Skipping this saves routing/middleware complexity. |
| TypeScript 5.9.3 | TypeScript 7.0.2 | After the demo. 8–12x faster typecheck is real, and Next 16.3 supports it — but no stable programmatic API until 7.1 means typed-lint and some editor tooling can break. |
| Regex PII masking | Presidio | Scope expands to Thai person names or free-form addresses (needs NER). |
| Hand-rolled rule evaluator | Any rules-engine library | Never, for this spec. Versioning + audit + Admin UI are the actual requirements, and no library provides them. |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| `passlib` 1.7.4 | Last release **2020**; known incompatibility with modern `bcrypt` releases; effectively unmaintained | `pwdlib[argon2]` 0.3.1 |
| `python-jose` 3.5.0 | Sparse maintenance, historical CVEs; still the default in old FastAPI tutorials | `PyJWT` 2.13.0 |
| `vector(3072)` with an HNSW index | **Silently impossible** — HNSW caps `vector` at 2,000 dims. You'll get an error at index creation, after you've already embedded everything | `dimensions=1024` from the API, or `halfvec(3072)` in the index expression |
| `to_tsvector('english'/'simple', thai_text)` | Produces one giant token — Thai has no inter-word spaces and Postgres has no Thai parser. Search will appear to work in English tests and silently fail in Thai | `pg_trgm` GIN, or PyThaiNLP-segmented text |
| JSON mode (`{"type":"json_object"}`) as the primary path | Guarantees *valid JSON*, not *your schema*. OpenAI now treats it as legacy | `json_schema` + `strict: true` |
| `all-MiniLM-L6-v2` / `paraphrase-multilingual-MiniLM-L12-v2` | The default in most RAG tutorials; 384-dim, and Thai is the worst-case language for small multilingual models. Will tank your Recall@5 grade | `bge-m3` (1024) |
| Raw `WangchanBERTa` / `PhayaThaiBERT` embeddings | Thai-specific but **not** contrastively trained for sentence similarity — mean-pooled BERT embeddings underperform purpose-built retrieval models badly | `bge-m3`, or fine-tune these post-MVP |
| `numpy`-assuming code around pgvector | `pgvector-python` **0.5.0 dropped NumPy** and now returns plain `list` from SQLAlchemy | Plain lists, or convert explicitly |
| Pinning `httpx` 0.28.1 next to `openai` 3.x | `openai` 3.0.0 (2026-08-12) switched to **httpx2** and no longer installs `httpx`; mixing them breaks custom client/transport config | Let `openai` bring `httpx2`; migrate any custom transport |
| Storing rules in a YAML file | FR-006 requires Admin-UI editing + versioning + audit logging | JSONB rows with a `version` column |
| Client-selectable role at demo login | Direct privilege escalation — already flagged in PROJECT.md | Bind role **server-side** per credential; put it in the JWT claim and re-verify per request |
| Sending unmasked text to *any* AI call | FR-003 is absolute, and a leak here fails a graded criterion | A single choke-point function every AI/embedding call must pass through — enforce with a test that fails if any provider call site bypasses it |
| Rendering AI draft text as HTML | NFR: "ไม่ Render HTML จากข้อความ AI โดยไม่ Sanitize" | Render as plain text / React children only. Never `dangerouslySetInnerHTML`. |
| `next lint` | Removed in Next 16 | ESLint 10 CLI (the `@next/codemod` migrates this) |

---

## Stack Patterns by Variant

**If demo machine has no GPU and no reliable internet (likely for an on-site hackathon):**
- Embeddings: `BAAI/bge-m3` in a TEI CPU container. ~10k short texts is minutes of CPU work.
- LLM: still needs a hosted call, so build `Simulate AI outage` (Scenario D) as a first-class feature — it doubles as your offline fallback demo.
- Because: an offline-capable similarity demo is worth more than a marginally better embedding.

**If you have any GPU (even 8GB):**
- Add `scb10x/typhoon2.1-gemma3-4b` on vLLM as a second `AI_PROVIDER` entry. It's Thai/English bilingual and vLLM gives you the same OpenAI-compatible `response_format` strict decoding.
- Because: a fully-local, PII-never-leaves-campus demo is a differentiating story for a *university IT centre* audience, and it directly exercises the spec's provider-independence requirement.

**If the demo dataset stays at the seeded ~60–100 tickets:**
- Skip Redis entirely. Skip HNSW entirely. Skip `arq`.
- Because: every service in `docker-compose.yml` is another thing that can fail during a live demo, and §22 grades "runs with a single command."

**If ticket volume ever exceeds ~200k (post-hackathon):**
- Add HNSW + `hnsw.iterative_scan = relaxed_order`, partition `tickets`/`ticket_embeddings` monthly (which also makes the 24h filter a partition prune instead of a post-filter), and move to `arq` + Redis for durable jobs.

---

## Version Compatibility

| Package A | Compatible With | Notes |
|-----------|-----------------|-------|
| `pgvector-python` 0.5.0 | SQLAlchemy **≥2.0** only; Python **≥3.10** | Breaking: SQLAlchemy/SQLModel/Django/Peewee now return `list` instead of `numpy.ndarray`; NumPy dependency removed. Psycopg drivers still return `Vector` objects. |
| pgvector 0.8.6 | PostgreSQL 13–18 | PG18 support landed in 0.8.1. Use image `pgvector/pgvector:0.8.6-pg18`. PG12 dropped in 0.8.0. |
| `openai` 3.2.0 | **httpx2** (not `httpx`) | v3.0.0 made httpx2 the default and stopped auto-installing `httpx`. `InputAudio` removed from `ResponseInputContent`. A runtime legacy-httpx escape hatch exists but is temporary. |
| Next.js 16.3.1 | React 19.2.x, TypeScript 5.9 **or** 7.0 | TS 7 requires `experimental.useTypeScriptCli` (Next invokes the project-local `tsc` CLI because TS 7.0 has no stable JS API until 7.1). |
| Next.js 16 | — | Breaking from 15: `middleware.ts` → **`proxy.ts`** (silent failure — redirects just stop), sync `params`/`searchParams` access **fully removed**, `experimental.dynamicIO` → `cacheComponents`, `next lint` removed. Run `npx @next/codemod@canary upgrade latest` if starting from a 15 template. |
| shadcn/ui (CLI 4.18) | Tailwind 4.3.x + React 19 | New components install as Tailwind-v4/React-19 flavour. A v3 project keeps working but new components arrive in v3 flavour — don't mix; start on v4. |
| TanStack Table 9.x | React 19 | Major bump from v8; check the v9 migration notes if copying v8 examples from tutorials. |
| Python 3.13 | FastAPI 0.141, SQLAlchemy 2.0.52, Pydantic 2.13 | All three also declare 3.14 support; 3.13 chosen for broadest transitive C-extension wheel coverage. |
| PostgreSQL 18.6 | — | Latest minor, 2026-08-13. **18.5 was never shipped** (pulled for a regression) — don't pin `18.5`. |
| Node 24 LTS | Next 16.3 | Node 24 is Active LTS; Node 26 is Current and becomes LTS Oct 2026. |
| `pythainlp` 5.3.7 | Python 3.13 | Only needed for the optional segmented-tsvector path. |

---

## Open Questions / Gaps

1. **No primary-source Thai retrieval benchmark exists for `gemini-embedding-001`, `embed-v4`, or `text-embedding-3-large`.** All three publish only aggregate multilingual scores. The bge-m3 (MIRACL 83.7) and SEA-BED (e5-large-instruct 81.11 / Qwen3-8B 81.49) numbers are the only Thai-specific figures found. → **Roadmap implication: the Phase 7 plan must include a small Recall@5 bake-off on your own seed data before the embedding model is frozen.** This is cheap (60 tickets, minutes) and is the only way to resolve it.
2. **Frontier chat-model rankings are blog-sourced and volatile.** Model names surfaced in search (GPT-5.x, Gemini 3.x, Claude 4.5/5.x) could not be corroborated against primary vendor docs within this research pass. → Mitigated by design: the adapter makes this a config value, and `evaluation-set.json` (§21) makes it a measurable one. Do not hardcode a model name anywhere but `.env`.
3. **`SIMILARITY_THRESHOLD = 0.82` is unvalidated for any specific model.** Cosine distributions are model-dependent. → Flag Phase 7/8 as needing calibration against the eval set, and keep the threshold in config (which the spec already does).
4. **`pg_trgm` relevance quality on Thai substrings is untested here.** It will find substrings; whether the *ranking* is acceptable for FR-014 is unmeasured. → Acceptable MVP risk, since FR-014 only requires search + filter, not ranked relevance.
5. **Thai student/staff-ID pattern is not specified** anywhere in PROJECT.md or SPEC.md. → Needs a config-driven regex plus a decision from CITCOMS; block Phase 4 (PII masking) on getting the actual pattern shape, or make it an admin-editable pattern list.

---

## Sources

**Primary / HIGH confidence**
- `/pgvector/pgvector` (Context7 via ctx7 CLI) — HNSW halfvec ops, `hnsw.iterative_scan`, `max_scan_tuples`, `scan_mem_multiplier`, filtering semantics
- `/pgvector/pgvector-python` (Context7 via ctx7 CLI) — SQLAlchemy `HALFVEC`, index expressions, asyncpg `register_vector`
- https://raw.githubusercontent.com/pgvector/pgvector/master/CHANGELOG.md — 0.8.6 (2026-07-29), 0.8.1 PG18 support, 0.8.0 iterative scans
- https://raw.githubusercontent.com/pgvector/pgvector/master/README.md — dimension limits (vector 2,000 / halfvec 4,000 HNSW-indexable), `m`/`ef_construction`/`ef_search` defaults, `maintenance_work_mem` guidance, binary quantization
- https://raw.githubusercontent.com/pgvector/pgvector-python/master/CHANGELOG.md — 0.5.0 breaking changes (list return, NumPy dropped, SQLAlchemy 2 required)
- https://arxiv.org/html/2402.03216v3 — BGE M3-Embedding paper: **MIRACL Thai dev nDCG@10 = 83.7**
- https://arxiv.org/html/2508.12243v2 — SEA-BED: 17 models × 10 SEA languages; Thai = 81.11 (multilingual-e5-large-instruct), **81.49 (Qwen3-Embedding-8B, best OSS on Thai)**; machine-translated Thai STS within 2 Spearman points of human-crafted
- https://ai.google.dev/gemini-api/docs/embeddings — `gemini-embedding-001` (2,048 tok, MRL 128–3072, **manual normalization required below 3072**, task types) and `gemini-embedding-2` (8,192 tok, auto-normalizes, no `task_type`)
- https://developers.openai.com/api/docs/guides/embeddings — `text-embedding-3-small` 1536/$0.02, `-3-large` 3072/$0.13, `dimensions` param supported
- https://huggingface.co/Qwen/Qwen3-Embedding-0.6B — dims 1024/2560/4096, MRL ranges, 32k context, 100+ languages, MTEB-multilingual 64.33/69.45/70.58, **1–5% retrieval loss without query `Instruct:` prefix**
- https://openai.com/index/introducing-structured-outputs-in-the-api/ — strict mode 100% schema adherence vs. best-effort JSON mode
- https://developers.openai.com/api/docs/guides/function-calling — `strict: true` on tools; recommendation to always enable
- https://nextjs.org/docs/app/guides/upgrading/version-16 — `middleware.ts` → `proxy.ts`, async request APIs, `cacheComponents`, `next lint` removal
- https://ui.shadcn.com/docs/tailwind-v4 + https://ui.shadcn.com/docs/installation/next — Tailwind v4 + React 19 + Next 16 support
- https://www.postgresql.org/about/news/postgresql-186-1711-1615-1519-1424-and-19-beta-3-released-3365/ — PG 18.6 (2026-08-13); 18.5 never shipped
- https://github.com/microsoft/presidio/blob/main/docs/analyzer/languages.md + `docs/faq.md` — non-English requires supplying your own NLP model and rewriting context words
- https://docs.vllm.ai/en/latest/features/structured_outputs/ + https://docs.ollama.com/capabilities/structured-outputs — `response_format` / XGrammar constrained decoding on self-hosted servers
- https://www.postgresql.org/message-id/CAEV3FNPU8hU_hi=0+QNAbEkc-uO8-K9PB3aAChdmcCyPfWX6rg@mail.gmail.com — pgsql-hackers: PostgreSQL FTS lacks Thai/CJK tokenization; ICU parser proposed
- PyPI + npm registry APIs, queried 2026-08-18 — all pinned version numbers and release dates
- Docker Hub `pgvector/pgvector` tag list, queried 2026-08-18 — `0.8.6-pg18` available

**MEDIUM confidence**
- https://jina.ai/news/jina-embeddings-v3-a-frontier-multilingual-embedding-model/ — "Thai yields the largest performance drop for most models" (vendor blog, but a specific and widely-corroborated technical claim)
- https://python.useinstructor.com/ — `instructor` provider coverage and download figures (project docs)
- https://developers.googleblog.com/gemini-embedding-available-gemini-api/ — MTEB(Multilingual) #1 claim (vendor)
- https://www.infoq.com/news/2026/08/typescript-7-released/ + https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-rc/ — TS 7.0 GA 2026-07-08, no stable programmatic API until 7.1
- https://github.com/openai/openai-python/blob/main/CHANGELOG.md — openai-python 3.0.0 httpx2 default (2026-08-12)
- https://ollama.com/scb10x + https://huggingface.co/scb10x/typhoon2.1-gemma3-12b — Typhoon 2.1 Thai/English models, vLLM OpenAI-compatible serving
- https://endoflife.date/nodejs — Node 24 Active LTS, Node 26 Current → LTS Oct 2026
- https://github.com/pgvector/pgvector/issues/980 + https://www.dbi-services.com/blog/pgvector-a-guide-for-dba-part-2-indexes-update-march-2026/ — filtered-HNSW limitation, partial-index/partition workarounds

**LOW confidence (flagged in text, not relied upon)**
- Aggregated "best embedding model 2026" listicles (mixpeek, ailog, pecollective, tensoria) — used only for cross-checking that Qwen3/Gemini/Cohere are in the current top tier
- Frontier-LLM ranking blogs (teamai, alconost, iternal) — model names not corroborated against vendor docs; deliberately not hardcoded into any recommendation

---
*Stack research for: bilingual Thai/English AI-assisted IT incident triage web app*
*Researched: 2026-08-18*

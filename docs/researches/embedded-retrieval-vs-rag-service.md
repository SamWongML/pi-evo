# Embedded retrieval vs. a lightweight RAG service

_Findings for [issue #7](https://github.com/SamWongML/pi-evo/issues/7) · July 2026_

## Recommendation, up front

**None. Run `sqlite-vec` + FTS5 inside the SQLite file the fleet architecture already has, and do not add a service process.**

At 1–3 machines and a store in the low thousands of short entries, every serious open-source RAG service surveyed below either (a) fails one of the two hard requirements outright, (b) is architecturally disqualified before requirement-testing even starts (multi-container stacks, GPU-recommended, heavy Python dependency trees), or (c) satisfies both requirements but buys nothing an embedded engine doesn't already give you at this scale, while adding a process to run, monitor, back up, and upgrade. This is not a close call. Full reasoning and citations below.

---

## The two hard requirements, and a finding that cuts across every candidate

1. **Metadata filtering must gate candidate generation**, not post-filter or blend into the similarity score.
2. **Hybrid lexical + vector retrieval** — either native, or fusable outside the store with RRF.

Before scoring individual products, one thing became clear while verifying requirement 1 against primary sources: **the 43.9% cross-scope leak the prior research document cites was almost certainly an application-layer bug, not a property of the underlying engine.** In every system checked — SQLite, Postgres, DuckDB, Qdrant, Weaviate, Meilisearch, Typesense, and Chroma — the filter predicate and the similarity/rank computation are exposed as **architecturally separate parameters** (a SQL `WHERE` clause vs. an `ORDER BY embedding <=> $q`; a Qdrant `Filter` object vs. the vector query; a Weaviate `where` filter vs. `nearVector`; Meilisearch's `filter` vs. `hybrid`; Typesense's `filter_by` vs. `vector_query`). None of these APIs make it natural to *blend* scope into a score — you would have to deliberately write `score = similarity + scope_bonus` instead of `WHERE scope = X`. That is exactly the anti-pattern the reference implementation fell into. It is a schema/query design discipline, not a product selection problem, and it is enforceable in code review regardless of which of these engines you pick.

What *does* vary legitimately between engines is **recall under filtering with an approximate (ANN) index** — a selective filter can starve an HNSW/IVF traversal of enough in-scope candidates, so you get fewer results, not wrong-scope results. That's a performance question, not a leak. At a few thousand rows, none of these engines need an approximate index at all — exact brute-force search is fast enough that this distinction is close to moot (see the SQLite section below for numbers).

---

## The embedded baseline

| Engine | Req. 1 (gating) | Req. 2 (hybrid) | Honest ceiling | Ops burden |
|---|---|---|---|---|
| **SQLite + `sqlite-vec` + FTS5** | ✅ SQL `WHERE`, always exclusionary; `vec0` is exact brute-force by default so there is no ANN-recall caveat at all | ✅ FTS5 BM25 (native) + `sqlite-vec` cosine distance (native), fused with app-level RRF | Brute-force KNN reported at **~500k vectors in ~500ms** ([demo](https://www.youtube.com/shorts/Xs7uZ60xBmM)); vec0 has no ANN index yet by design ([tracking issue #25](https://github.com/asg017/sqlite-vec/issues/25)) — irrelevant at low-thousands scale | **Zero new processes.** Already installed (sqlite3 3.51). `better-sqlite3` — which the existing fleet design already uses for the per-device local mirror — loads `sqlite-vec` directly via `sqliteVec.load(db)` ([docs](https://alexgarcia.xyz/sqlite-vec/js.html)) |
| **Postgres + `pgvector` + `tsvector` + `pg_trgm`** | ✅ same SQL `WHERE` guarantee; `pgvector` 0.8+ adds iterative index scans specifically to fix recall under heavy filtering with HNSW ([pgvector README](https://github.com/pgvector/pgvector), current 0.8.6) | ✅ `tsvector`/`ts_rank_cd` (native BM25-like ranking) + `pgvector` cosine, `pg_trgm` for fuzzy error-signature matching ([pg_trgm docs](https://www.postgresql.org/docs/current/pgtrgm.html)), fused with app-level RRF | Effectively unbounded for this workload — the constraint is operational, not algorithmic | **No local Postgres installed** — this is a new container/service to run, back up, and patch. Same query-level guarantees as SQLite, at strictly higher ops cost for this scale |
| **LanceDB** | ✅ pre-filtering is the *default*: "Pre-filtering means LanceDB applies the metadata `where(...)` condition before running vector search, so the search only considers rows that already match the filter" ([docs](https://docs.lancedb.com/search/filtering)) | ✅ native — Tantivy-backed BM25 full-text index plus vector search, with a **default `RRFReranker()`** doing exactly `Σ 1/(k+rank)` for you ([hybrid search docs](https://docs.lancedb.com/search/hybrid-search)) | Designed for far larger corpora (embedded, disk-backed columnar format); no ceiling concern at this scale | Embedded library (Rust core, Python/Node/Rust bindings), no server process — but it's a new dependency and file format the team doesn't already have, versus SQLite which is already on the machine |
| **DuckDB + VSS + FTS extension** | ⚠️ conditional — the core `vss` HNSW index **does not combine with a `WHERE` clause**: "you can't combine it with a WHERE clause for filtering immediately" ([DuckDB VSS docs](https://duckdb.org/docs/current/core_extensions/vss)); the community `hnsw_acorn` extension adds prefiltering but is third-party and less mature ([community ext](https://duckdb.org/community_extensions/extensions/hnsw_acorn)). **Workaround: skip the HNSW index and use exact `ORDER BY array_distance(...)` with a plain `WHERE`** — this is exact and trivially satisfies req. 1, and is fast enough at this row count | ✅ separate `fts` extension implements Okapi BM25 natively ([docs](https://duckdb.org/docs/current/core_extensions/full_text_search)), combine with brute-force vector distance and fuse with app-level RRF | Fine at this scale in brute-force mode | The `vss` extension is explicitly labeled **experimental**, and its docs warn: "we still recommend that you do not use this feature in production environments" due to WAL/persistence risk on crash. Usable, but as brute-force-only, which erases its main advantage over SQLite |

**Bottom line on the baseline:** SQLite and Postgres give an identical correctness guarantee for requirement 1 (a relational `WHERE` clause cannot leak by construction) and both provide native hybrid legs. LanceDB is a legitimate embedded alternative with the nicest native hybrid fusion (RRF built in) but is a new dependency. DuckDB's vector index is explicitly not production-ready and doesn't compose with filtering — usable only in the same brute-force mode SQLite already handles for free. **SQLite wins on ops burden alone**: nothing to install, nothing new to run, and it reuses the `better-sqlite3` dependency the existing fleet-memory design (`evolve-recall.ts`) already has for the local mirror — the lesson store and the mirror become the same technology.

---

## The service field

### Architecture-first cut

Per the issue's filter (single container/binary, no GPU, no heavy Python dependency stack), several candidates are eliminated before requirement-testing is even relevant:

| Service | Why it's out | Source |
|---|---|---|
| **RAGFlow** | Docker Compose stack of Elasticsearch + MinIO + MySQL + Redis + Nginx + app; minimum **16 GB RAM**, GPU recommended (RTX 3080 min / RTX 4090 recommended) for its document-parsing pipeline; no ARM64 support | [system requirements](https://ragflow.io/docs/) |
| **Onyx (formerly Danswer)** | Docker Compose brings up **eleven containers** (Postgres, OpenSearch, two model servers, worker, Redis, MinIO, web app, API, nginx); recommended for enterprise inference is a 4–8× A100/H100 cluster | [architecture](https://github.com/av/harbor/wiki/2.1.14-Frontend-Onyx) |
| **Vespa** | Powerful hybrid engine (BM25 + ANN + phased ranking, filtered HNSW), but a self-managed single-node deployment still needs a **config server + container cluster as separate JVM processes**, minimum 4 GB just for the container and 8 GB recommended to "make sure the application is functionally correct" ([docs](https://docs.vespa.ai/en/operations/self-managed/node-setup.html)), plus schema/application-package deployment workflow — an operable-UI-grade system built for web-scale, not a fit for a single operator |
| **R2R** | Its full Docker deployment *is* Postgres + `pgvector` (plus a graph-clustering service and dashboard) under the hood — [architecture](https://r2r-docs.sciphi.ai/self-hosting/configuration/retrieval/overview) confirms it uses `pgvector` for vectors and `ts_rank`/`websearch_to_tsquery` for the lexical leg. It inherits exactly the same filtering semantics as embedded pgvector, but adds a graph-clustering service, dashboard, and its own API surface on top. There is nothing here embedded Postgres doesn't already give you, at strictly more moving parts |
| **Morphik** | Multi-modal RAG platform; self-hosting requires Docker Compose plus (per its own repo) the maintainers state limited support capacity for OSS self-hosting; aimed at multimodal document ingestion, not a scope-filtered short-entry lesson store | [self-hosting docs](https://www.morphik.ai/docs/self-hosting) |
| **LightRAG** | Knowledge-graph RAG framework (dual-layer KG + vector store, `local`/`global`/`hybrid`/`mix` query modes) aimed at entity/relationship extraction over long documents — not designed around per-request scope predicates at all; "hybrid" here means graph-vs-vector fusion, not lexical+vector, so it does not even address requirement 2 as posed | [GitHub](https://github.com/HKUDS/lightrag) |

### Candidates that pass the architecture gate: Qdrant, Weaviate, Meilisearch, Typesense, Chroma

All five run as a single container or single binary with no GPU requirement, and all five verified true on requirement 1 and requirement 2:

- **Qdrant** — filtering is not a post-step: the payload index **extends the HNSW graph itself**, so "filtering criteria [are] applied during the semantic search phase" ([filtering guide](https://qdrant.tech/articles/vector-search-filtering/)). A query planner chooses payload-index-only vs. filtered-HNSW based on filter cardinality. This is the most rigorous gating guarantee of any candidate surveyed, embedded or service. Hybrid: native sparse+dense fusion. Single container: `docker run -p 6333:6333 -v $(pwd)/data:/qdrant/storage qdrant/qdrant` ([install docs](https://qdrant.tech/documentation/installation/)); the docs note Qdrant itself is light and RAM is dominated by vector count — trivial at a few thousand entries.
- **Weaviate** — resolves the metadata filter into a **Roaring Bitmap allow-list before HNSW search runs**, then the adaptive ACORN algorithm traverses only within (or around) that allow-list depending on selectivity — "the inverted index is queried first, that filter resolves into an allow-list of matching object IDs, and then the HNSW vector search runs against that constrained set" ([filtering concepts](https://docs.weaviate.io/weaviate/concepts/filtering)). Native hybrid search (BM25 + vector) with a documented `alpha` fusion parameter and a choice of `rankedFusion` or the default `relativeScoreFusion` ([hybrid search docs](https://docs.weaviate.io/weaviate/concepts/search/hybrid-search)). Runs as a single standalone container via `docker-compose` with no mandatory external services. Minimum ~4 GB RAM for production, 8–16 GB recommended for local embedding models ([forum](https://forum.weaviate.io/t/memory-requirement/1424)).
- **Meilisearch** — single self-contained binary; filters are declared via `filterableAttributes` and applied as a query-time boolean predicate, structurally separate from `hybrid`/`semanticRatio` ([hybrid search docs](https://www.meilisearch.com/docs/capabilities/hybrid_search/overview)). Native hybrid (keyword + vector), with the caveat that Meilisearch's own blog states it deliberately does **not** default to plain RRF, using a proprietary score-based blend instead because "[RRF] assumes that documents from all the lists of results are of comparable relevance" ([why traditional hybrid search falls short](https://www.meilisearch.com/blog/fixing-hybrid-search)) — a native fusion mechanism either way, satisfying requirement 2. ~1 GiB RAM comfortably indexes hundreds of thousands of documents ([storage docs](https://www.meilisearch.com/docs/learn/engine/storage)).
- **Typesense** — single self-contained binary, `docker run -p 8108:8108 -v/tmp/data:/data typesense/typesense --data-dir /data --api-key=...` ([GitHub](https://github.com/typesense/typesense/blob/main/README.md)). `filter_by` and `vector_query` are separate query parameters; hybrid fusion is a documented explicit formula, `0.7*K + 0.3*S` by default (keyword rank vs. semantic rank), adjustable via `alpha` ([vector search docs](https://typesense.org/docs/30.2/api/vector-search.html)). RAM scales ~2–3× indexed data size — trivial at this entry count.
- **Chroma** — as of 2026 ships native BM25 + SPLADE sparse search alongside dense vectors, explicitly marketed as first-class hybrid ("Chroma has built-in support for vector search, full-text (BM25 + SPLADE), regex, metadata filtering, and hybrid capabilities" — [sparse vector search announcement](https://www.trychroma.com/project/sparse-vector-search)). Filtering restricts the candidate ID set searched by the underlying `hnswlib` fork before scoring rather than being blended into score, though — unlike Qdrant/Weaviate's graph-aware filtering — Chroma's own docs stop short of a formal statement on filter/traversal interaction, so treat its gating guarantee as good-but-less-rigorously-documented than Qdrant/Weaviate's. Not recommended below 2 GB RAM ([resources docs](https://cookbook.chromadb.dev/core/resources/)).

**None of these five loses on the two hard requirements.** The reason not to adopt any of them is the section below: at this entry count, they don't win anything measurable over the embedded baseline, and each is a new process to operate.

---

## Embeddings: local model vs. API

The read path has an ~80ms p50 budget for the whole retrieval round trip, which makes network-hop embedding latency a real constraint, not a rounding error.

- **Local (ONNX/fastembed/Ollama) is faster in the case that matters — the query-time embed.** A cross-provider benchmark found local open-source models on CPU sub-50ms per single query, versus roughly 150–300ms round-trip for the OpenAI embeddings API, with OpenAI's p95 measured at "almost a minute from GCP and almost 600ms from AWS" in adverse conditions ([survey of embedding models](https://blog.getzep.com/text-embedding-latency-a-semi-scientific-look/)). `fastembed` specifically is built on ONNX Runtime with quantized models and is reported at 10–100ms per document on CPU depending on model size (same source). Either way, a same-host or same-datacenter local model removes the external network hop entirely, which is the dominant term in the 80ms budget.
- **Quality is no longer a compromise for short technical text at this scale.** `nomic-embed-text` (274MB, 768-dim) is reported to "surpass OpenAI text-embedding-ada-002 and text-embedding-3-small performance on short and long context tasks" ([Ollama model card](https://ollama.com/library/nomic-embed-text)); `mxbai-embed-large` (670MB, 1024-dim) claims SOTA for BERT-large-sized models on MTEB, "outperform[ing] commercial models like OpenAI's text-embedding-3-large" (Ollama model card, [collabnix summary](https://collabnix.com/ollama-embedded-models-the-complete-technical-guide-to-local-ai-embeddings-in-2025/)). These numbers should be read with normal MTEB skepticism (benchmark-vs-real-workload gap), but the direction is unambiguous: local small models are not a quality downgrade for short lesson/error-signature text.
- **Cost and offline behavior both favor local outright.** Ollama serves embeddings at `$0` per token with no API key over `localhost:11434` ([Ollama docs](https://ollama.com/library/nomic-embed-text)). An agent fleet that must keep working when the network flaps (already a stated design constraint in the prior research doc) cannot depend on an embeddings API for its recall hot path regardless of latency — a local model is the only option that doesn't add "the embeddings provider is down" to the failure surface of every turn.
- **Recommendation:** embed locally via `fastembed` (Python/uv, ONNX-backed, no GPU) or an Ollama-served model (`nomic-embed-text` is the pragmatic default: smallest footprint at 0.5GB memory, "V.Fast" speed per benchmarking, and documented to match or beat OpenAI's small embedding model on short text). API embeddings (OpenAI/Voyage/Cohere) remain reasonable for the *offline curator* pass (nightly, latency-insensitive, can batch), but should not be on the per-turn read path.

---

## What a service would actually buy — and is any of it load-bearing here?

Per the issue, be blunt if the answer is no. Going through each thing a service offers that embedded doesn't:

- **Reranking (cross-encoder, Cohere, ColBERT).** Real, but reranking earns its cost when the first-stage candidate set is large and noisy (tens of thousands+ of candidates). At a few thousand entries, RRF over two clean legs (BM25 + cosine) is already a strong signal, and the existing v2 research document's own scoring formula already layers recency/occurrence/verification weighting on top. A reranker here is solving a precision problem that doesn't exist at this scale. **Not load-bearing.**
- **Chunking and ingestion pipelines.** These solve the "turn a 50-page PDF into retrievable units" problem. The lesson store's rows are already short, structured, single-purpose records (`trigger`/`guidance`/`error_sig`) produced by the curator — there is no long-document ingestion problem to solve. **Not load-bearing.**
- **Incremental indexing.** Real infrastructure question, but SQLite/Postgres/LanceDB all support row-level insert/update with the vector and FTS indexes updated transactionally as part of the same write — there's no batch-reindex step being avoided by a service here. **Not load-bearing.**
- **An operable UI.** This is the one genuine, non-manufactured win a service like Qdrant, Weaviate, or Meilisearch has over raw SQL: a dashboard to browse collections, run ad hoc filtered queries, and inspect payloads without writing SQL. For a **single operator**, this has real value during development and debugging. It is also fully replaceable by a 20-line `sqlite3` CLI habit or a tiny local Datasette instance (`pip install datasette`, point at the same SQLite file) that gives browsing/querying for free with zero new write-path infrastructure. **Marginally load-bearing, but cheaply substitutable without adopting a service.**

Nothing here clears the bar of "worth running and operating a process for," at this entry count, for a single operator.

---

## Comparison table — scored against the two hard requirements

| Candidate | Req. 1: hard gate | Req. 2: hybrid | Passes architecture gate | Verdict |
|---|:---:|:---:|:---:|---|
| SQLite + sqlite-vec + FTS5 | ✅ (SQL WHERE, exact) | ✅ (native BM25 + native cosine, app RRF) | ✅ already installed | **Recommended** |
| Postgres + pgvector + tsvector + pg_trgm | ✅ (SQL WHERE) | ✅ (native) | ⚠️ new container (no local Postgres) | Viable fallback only if concurrent multi-writer becomes a real need |
| LanceDB | ✅ (prefilter is default) | ✅ (native, RRF built in) | ✅ | Viable; new dependency vs. already-installed SQLite |
| DuckDB + VSS + FTS | ⚠️ (only in brute-force mode; native HNSW doesn't compose with WHERE, and is marked not-production-ready) | ✅ (separate fts extension) | ✅ | Viable in brute-force mode only; no advantage over SQLite then |
| Qdrant | ✅ (filter gates HNSW traversal) | ✅ (native sparse+dense) | ✅ single container | Passes everything; no measurable win at this scale |
| Weaviate | ✅ (roaring-bitmap allow-list gates before HNSW) | ✅ (native, alpha fusion) | ✅ single container | Passes everything; no measurable win at this scale |
| Meilisearch | ✅ (filter is a separate boolean predicate) | ✅ (native hybrid, proprietary fusion) | ✅ single binary | Passes everything; no measurable win at this scale |
| Typesense | ✅ (filter_by separate from vector_query) | ✅ (native, documented 0.7/0.3 fusion) | ✅ single binary | Passes everything; no measurable win at this scale |
| Chroma | ✅ (allowed-ID gating, less formally documented) | ✅ (native BM25+SPLADE+dense, 2026) | ✅ single container/embedded | Passes everything; no measurable win at this scale |
| RAGFlow | — not evaluated | — not evaluated | ❌ 16GB+RAM, GPU-recommended, 5-service stack | Eliminated at architecture gate |
| Onyx | — not evaluated | — not evaluated | ❌ 11-container stack | Eliminated at architecture gate |
| Vespa | ✅ (filtered HNSW is real) | ✅ (native, phased ranking) | ❌ multi-process JVM, 4-8GB minimum, schema/app-package workflow | Eliminated at architecture gate — web-scale tool, wrong shape for 1 operator |
| R2R | ✅ (inherits pgvector's) | ✅ (inherits pgvector's) | ❌ is Postgres+pgvector plus a graph service and dashboard | Adds nothing over embedded pgvector |
| Morphik | not verified | not verified | ❌ Docker Compose, limited OSS support stated by maintainers | Eliminated at architecture gate |
| LightRAG | not designed for scope predicates | "hybrid" = graph-vs-vector, not lexical+vector | ⚠️ Python-heavy, KG-oriented | Wrong tool — solves document/entity extraction, not a scope-filtered lesson store |

---

## Exact install commands (for the record, in case the operator wants a service anyway)

If the decision is made to run a service despite the above, the two that pass every gate cleanly with the least ops burden are Qdrant and Meilisearch:

```bash
# Qdrant — single container, ~6333/6334 ports, RAM dominated by vector count (trivial at this scale)
docker run -p 6333:6333 -v "$(pwd)/qdrant_data:/qdrant/storage" qdrant/qdrant
```
Source: [Qdrant installation docs](https://qdrant.tech/documentation/installation/). Disk footprint is the image (~200MB) plus raw vector storage; RAM footprint for a few thousand 768-dim vectors is a few MB, well under the 1–2GB the docs recommend allocating as a safety margin.

```bash
# Meilisearch — single binary, one container
docker run -p 7700:7700 -v "$(pwd)/meili_data:/meili_data" getmeili/meilisearch:v1.11 meilisearch --master-key="CHANGE_ME"
```
Source: [Meilisearch storage docs](https://www.meilisearch.com/docs/learn/engine/storage) — "1 GiB RAM indexes and serves hundreds of thousands of documents comfortably," so a few thousand short entries is a rounding error. Disk footprint is dominated by the LMDB-backed index, again negligible at this entry count.

**But the recommendation stands: don't run either.** Reuse `sqlite-vec` + FTS5 in the SQLite file the fleet design already has, embed locally with `fastembed` or Ollama's `nomic-embed-text`, and fuse BM25 + cosine + exact error-signature match with the same RRF formula (`Σ 1/(60+rank)`) the existing five-stage retrieval pipeline design already specifies.

---

## What this recommendation gives up

- No operable browsing UI out of the box (mitigated: point Datasette or a `sqlite3` shell at the file).
- No built-in reranking, sparse-vector fusion tuning, or adaptive filter-strategy switching (Qdrant/Weaviate's ACORN) — irrelevant at low-thousands scale where brute-force/simple-index recall is already ~exact.
- No horizontal scale-out story — if the store ever needs to hold millions of rows or serve very high QPS across many machines, this recommendation should be revisited; nothing here forecloses migrating to Postgres+pgvector or Qdrant later, since the query shape (WHERE-gated scope, RRF-fused hybrid) is identical across all of them.
- Cross-machine sync for the 1–3 machine fleet still needs to be solved (e.g., a single authoritative SQLite file plus [Litestream](https://litestream.io/)-style replication, or the hot-channel API design the prior research document already specifies) — this is unchanged by the embedded-vs-service decision, since even a service would need the same authoritative-store-plus-sync answer.

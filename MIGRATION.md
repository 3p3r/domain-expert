## Plan: Replace PostgreSQL with DynamoDB (PartiQL) + S3 Vectors

**Goal**: Remove PostgreSQL entirely so the system can run on **AWS Lambda** and cost **$0 at rest**. No always-on database servers.

**TL;DR**: The `documents` table and chunk metadata move to **DynamoDB** (queried via PartiQL through `pydynamodb`). Embeddings + vector search move to **S3 Vectors** (`boto3.client('s3vectors')`), which natively supports cosine similarity ANN search with metadata filtering. BM25 full-text search moves to a **contentless SQLite FTS5** database — the index file is stored on regular S3 and downloaded at runtime on cold start. The FastAPI app is deployed behind **Mangum** on Lambda + API Gateway (or Lambda function URL). All services are serverless/pay-per-request: DynamoDB on-demand, S3 Vectors, S3, Lambda. All boto3 calls wrapped in `asyncio.to_thread()`.

**Steps**

### Phase 1: S3 Vectors store

1. **Create [src/storage/s3_vector_store.py](src/storage/s3_vector_store.py)** — new class `S3VectorStore` using `boto3.client('s3vectors')`:
   - `connect()`: Create client, ensure vector bucket and index exist (idempotent via `create_vector_bucket()` + `create_index(dimension=1024, distanceMetric='cosine', dataType='float32')`)
   - `store_embedded_chunks(chunks)`: Call `put_vectors(vectors=[{key: chunk_id, data: {float32: embedding}, metadata: {document_id, chunk_index, section_path, content, token_count}}])` in batches
   - `search(embedding, limit, document_filter=None)`: Call `query_vectors(queryVector={float32: embedding}, topK=limit, filter={"document_id": {"$eq": doc_id}}, returnMetadata=True, returnDistance=True)`. Convert distance to similarity: `similarity = 1 - distance` (cosine). Return `SearchResult` objects.
   - `search_with_doc_ids(embedding, limit, document_ids)`: For metadata-filtered search — if S3 Vectors `filter` supports `$in` operator, use `filter={"document_id": {"$in": [...]}}`. Otherwise, run unfiltered query with higher topK and post-filter in Python.
   - `get_chunks_by_document(document_id)`: Use `list_vectors` paginator (filtered by metadata) or query DynamoDB chunks table as fallback
   - `delete_vectors(keys)`: Call `delete_vectors(keys=[...])`
   - `count_chunks()`: Use `list_vectors` paginator to count (or maintain count in DynamoDB)
   - Env vars: `S3V_VECTOR_BUCKET` (vector bucket name), `S3V_INDEX_NAME` (default `eth_chunks`), `AWS_REGION`
   - `metadataConfiguration.nonFilterableMetadataKeys`: `["content", "token_count", "section_path", "git_commit", "created_at"]` — these are returned but not used as query filters
   - Default filterable metadata: `document_id` (used for pre-filtered vector search)
   - All methods async via `asyncio.to_thread()` wrapping sync boto3 calls
   - No `reindex_embeddings()` needed — S3 Vectors manages its own index

### Phase 2: DynamoDB store

2. **Create [src/storage/dynamodb_store.py](src/storage/dynamodb_store.py)** — new class `DynamoDBStore` using `pydynamodb` (DB-API 2.0, PartiQL):
   - Single DynamoDB table (env var `DYNAMODB_TABLE`, default `eth_protocol`), single-table design:
     - Documents: PK = `document_id` (S), with all document metadata attributes
     - Schema tracking: PK = `_SCHEMA_VERSION`, version attribute
   - GSIs: `status-index` (status → document_id), `document_type-index` (document_type → document_id), `eip_number-index` (eip_number → document_id)
   - `connect()`: Create pydynamodb connection (wraps boto3 DynamoDB)
   - `store_document(...)`: PartiQL `INSERT INTO "table" VALUE {document_id: ?, title: ?, ...}` — or `UPDATE` for upsert semantics
   - `store_generic_document(...)`: Same pattern for non-EIP documents
   - `get_document(document_id)`: PartiQL `SELECT * FROM "table" WHERE document_id = ?`
   - `count_documents()`: PartiQL `SELECT COUNT(*) FROM "table" WHERE begins_with(document_id, 'eip-') OR ...` — or use `DescribeTable` for approximate count
   - `get_document_ids_by_filter(status, type, category, author, ...)`: Query GSIs or scan with filters, return list of document_ids for metadata filtering
   - `enrich_sources(document_ids)`: PartiQL `SELECT document_id, title, author, metadata FROM "table" WHERE document_id IN [...]`
   - `delete_document(document_id)`: PartiQL `DELETE FROM "table" WHERE document_id = ?`
   - All methods async via `asyncio.to_thread()`

### Phase 3: Unified facade

3. **Create [src/storage/store.py](src/storage/store.py)** — `Store` class composing `S3VectorStore` + `DynamoDBStore` + `FTS5Store`:
   - Same public API as current `PgVectorStore` so all consumers see one object
   - `store_document()` → `DynamoDBStore`
   - `store_embedded_chunks()` → `S3VectorStore` (vectors + chunk metadata in S3V metadata) **+ `FTS5Store.index_chunks()`** (insert into FTS5)
   - `search()` → `S3VectorStore.search()`, then return `SearchResult` objects with content from S3V metadata
   - `get_document()` → `DynamoDBStore`
   - `delete_document()` → delete from all three stores
   - `initialize_schema()` → create vector bucket/index + ensure DDB table exists + download FTS5 DB from S3
   - `upload_fts5()` → upload the local FTS5 DB file to S3 (called after ingestion)

### Phase 4: BM25 replacement — contentless SQLite FTS5

4. **Create [src/storage/fts5_store.py](src/storage/fts5_store.py)** — contentless FTS5 index backed by S3:
   - **Schema**: `CREATE VIRTUAL TABLE chunks_fts USING fts5(chunk_id UNINDEXED, document_id, content, content='', contentless_delete=1)` — stores only the inverted index, not the original text
   - **S3 lifecycle**:
     - `download()`: On cold start, `s3.download_file(FTS5_S3_BUCKET, FTS5_S3_KEY, local_path)`. If the file doesn't exist in S3 yet (first run), create an empty DB with the schema. Local path: `/tmp/bm25.db` (or configurable `FTS5_LOCAL_PATH`)
     - `upload()`: After ingestion completes, `s3.upload_file(local_path, FTS5_S3_BUCKET, FTS5_S3_KEY)` to persist the updated index back to S3
   - **Write methods** (called during ingestion):
     - `index_chunks(chunks)`: `INSERT INTO chunks_fts(chunk_id, document_id, content) VALUES (?, ?, ?)` — incremental, no rebuild
     - `delete_chunks(chunk_ids)`: `DELETE FROM chunks_fts WHERE chunk_id = ?`
     - `delete_by_document(document_id)`: `DELETE FROM chunks_fts WHERE document_id = ?`
     - `rebuild()`: `INSERT INTO chunks_fts(chunks_fts) VALUES('rebuild')` — for rare full reindex
   - **Read methods** (called at query time):
     - `search(query, limit, document_ids=None)`: `SELECT chunk_id, document_id, bm25(chunks_fts) AS rank FROM chunks_fts WHERE chunks_fts MATCH ? ORDER BY bm25(chunks_fts) LIMIT ?`. If `document_ids` provided, add `AND document_id IN (...)`. Returns `list[(chunk_id, document_id, rank)]`
     - No `highlight()` or `snippet()` available (contentless table) — highlighting done in Python post-retrieval using the chunk text from S3 Vectors metadata
   - **Size estimate**: ~2.5 GB for 100 GB of PDFs; for this project's Ethereum corpus, likely **< 200 MB**
   - **Cold start cost**: Download from S3 — ~1-3s for 200 MB over VPC, ~10-15s for 2.5 GB. One-time per container start.
   - Env vars: `FTS5_S3_BUCKET` (S3 bucket for the FTS5 file), `FTS5_S3_KEY` (default: `fts5/bm25.db`), `FTS5_LOCAL_PATH` (default: `/tmp/bm25.db`)
   - All methods sync (sqlite3 is not async) — wrap in `asyncio.to_thread()` for the async interface

5. **Update [src/retrieval/bm25_retriever.py](src/retrieval/bm25_retriever.py)**: Replace PostgreSQL `tsvector`/`tsquery` SQL with calls to `FTS5Store`:
   - `search()` calls `fts5_store.search(query, limit, document_ids)` to get `(chunk_id, document_id, rank)` tuples
   - Hydrate chunk content from S3 Vectors via `s3v_store.get_vectors(keys=[...], returnMetadata=True)` to build `StoredChunk` objects
   - Implement `_highlight(content, query)` in Python: split query into terms, wrap matches in `<mark>...</mark>` — replaces `ts_headline`
   - Keep same public `search()` interface returning `BM25Result`

### Phase 5: Update retrieval consumers

6. **Update [src/retrieval/hybrid_retriever.py](src/retrieval/hybrid_retriever.py)**:
   - Replace `_vector_search_with_filter()` SQL with `Store.search()` + optional `document_ids` pre-filter
   - For metadata filtering: call `DynamoDBStore.get_document_ids_by_filter()` first, then pass IDs to `S3VectorStore.search_with_doc_ids()`
   - RRF fusion logic unchanged

7. **Update [src/retrieval/simple_retriever.py](src/retrieval/simple_retriever.py)**: Replace `PgVectorStore` with `Store`

8. **Update [src/retrieval/code_retriever.py](src/retrieval/code_retriever.py)**: Replace `PgVectorStore` with `Store`. Fix `source_filter` → `document_filter` bug.

9. **Update [src/retrieval/graph_augmented.py](src/retrieval/graph_augmented.py)**: Replace `PgVectorStore` with `Store`

10. **Update [src/retrieval/staged_retriever.py](src/retrieval/staged_retriever.py)**: Imports `SearchResult` from `pg_vector_store` — update to import from `src.storage`

11. **Update [src/retrieval/reranker.py](src/retrieval/reranker.py)**: Imports `StoredChunk` from `pg_vector_store` — update to import from `src.storage`

12. **Update [src/filters/metadata_filter.py](src/filters/metadata_filter.py)**:
    - Replace SQL WHERE clause builder with a method that returns a list of matching `document_id`s from DynamoDB
    - `status IN (...)` → query `status-index` GSI
    - `type IN (...)` → scan/filter
    - `author ILIKE` → scan with `contains()` (case-insensitive done in app by lowercasing stored + queried values)
    - `requires @> ARRAY[N]` → `contains(requires, N)` in PartiQL
    - Returns `list[str]` of document_ids instead of a SQL subquery string

### Phase 6: Update type imports across codebase

13. **Rewire `SearchResult` / `StoredChunk` imports** — these dataclasses currently live in `pg_vector_store.py`. Move them to `src/storage/store.py` (or a `src/storage/types.py`) and update imports in all consumers:
    - [src/agents/retrieval_tool.py](src/agents/retrieval_tool.py) — imports `SearchResult`
    - [src/structured/timeline_builder.py](src/structured/timeline_builder.py) — imports `SearchResult`
    - [src/structured/argument_mapper.py](src/structured/argument_mapper.py) — imports `SearchResult`
    - [src/structured/comparison_table.py](src/structured/comparison_table.py) — imports `SearchResult`
    - [src/retrieval/staged_retriever.py](src/retrieval/staged_retriever.py) — imports `SearchResult`
    - [src/retrieval/reranker.py](src/retrieval/reranker.py) — imports `StoredChunk`
    - All retrieval modules that use these types
    - Re-export from `src/storage/__init__.py` for backward compatibility

### Phase 7: Update API & ingestion

14. **Update [src/api/main.py](src/api/main.py)**:
    - Replace `PgVectorStore` with `Store`
    - `enrich_sources()` raw SQL (`conn.fetch("SELECT ... FROM documents WHERE document_id = ANY($1)")`) → `DynamoDBStore.enrich_sources(doc_ids)`
    - Remove `store.pool is not None` health check → replace with DynamoDB/S3V connectivity check
    - Update startup/shutdown lifecycle

15. **Update all ingestion scripts** in [scripts/](scripts/) — every one of these imports `PgVectorStore`:
    - [ingest_eips.py](scripts/ingest_eips.py), [ingest_ercs.py](scripts/ingest_ercs.py), [ingest_rips.py](scripts/ingest_rips.py), [ingest_consensus_specs.py](scripts/ingest_consensus_specs.py), [ingest_execution_specs.py](scripts/ingest_execution_specs.py), [ingest_beacon_apis.py](scripts/ingest_beacon_apis.py), [ingest_execution_apis.py](scripts/ingest_execution_apis.py), [ingest_builder_specs.py](scripts/ingest_builder_specs.py), [ingest_portal_specs.py](scripts/ingest_portal_specs.py), [ingest_devp2p.py](scripts/ingest_devp2p.py), [ingest_clients.py](scripts/ingest_clients.py), [ingest_arxiv.py](scripts/ingest_arxiv.py), [ingest_research.py](scripts/ingest_research.py), [ingest_acd_transcripts.py](scripts/ingest_acd_transcripts.py), [ingest_magicians.py](scripts/ingest_magicians.py), [ingest_ethresearch.py](scripts/ingest_ethresearch.py), [ingest_all.py](scripts/ingest_all.py)
    - Replace `PgVectorStore()` → `Store()`
    - Remove all `reindex_embeddings()` calls
    - Add `await store.upload_fts5()` at end of each script (persist FTS5 to S3)
    - **Fix raw SQL in [ingest_ethresearch.py](scripts/ingest_ethresearch.py)**: `store.pool.acquire()` + `conn.fetch("SELECT DISTINCT (metadata->>'topic_id')::int ...")` → `DynamoDBStore` PartiQL query
    - **Fix raw SQL in [scripts/ingest_all_clients.sh](scripts/ingest_all_clients.sh)**: embedded Python `store.pool.fetch('''SELECT...''')` → `DynamoDBStore` query
    - **Fix direct asyncpg in [scripts/demo_e2e.py](scripts/demo_e2e.py)**: `asyncpg.connect(DATABASE_URL)` + raw SQL → `Store` API
    - **Fix raw SQL in [scripts/validate_corpus.py](scripts/validate_corpus.py)**: `store.connection()` + `conn.fetch()` / `conn.fetchrow()` → `Store` API
    - **Fix [scripts/test_e2e.py](scripts/test_e2e.py)** and **[scripts/test_full_rag.py](scripts/test_full_rag.py)**: build connection from `POSTGRES_PASSWORD` + `postgresql://` → remove, use `Store()`
    - **Fix [scripts/query_cli.py](scripts/query_cli.py)**: `PgVectorStore` → `Store`

16. **Update [src/storage/__init__.py](src/storage/__init__.py)**: Export `Store`, `S3VectorStore`, `DynamoDBStore`, `FTS5Store`, `SearchResult`, `StoredChunk`. Remove `PgVectorStore`, `SchemaMigrator`, `run_migrations`.

### Phase 8: Schema & migrations

17. **Delete [src/storage/pg_vector_store.py](src/storage/pg_vector_store.py)** — fully replaced by `Store` facade

18. **Delete [src/storage/pg_schema.py](src/storage/pg_schema.py)** — the `SchemaMigrator` + `run_migrations` + 6 PostgreSQL DDL migrations are no longer needed. Replace with `dynamodb_schema.py`:
    - `ensure_table()`: boto3 `create_table()` with key schema, GSIs, PAY_PER_REQUEST billing
    - `ensure_vector_index()`: `create_vector_bucket()` + `create_index()`
    - Simple version check item in DDB: `{document_id: "_SCHEMA_VERSION", version: 1}`

### Phase 9: Config & dependencies

19. **Update [pyproject.toml](pyproject.toml)**:
    - Add: `pydynamodb`, `boto3`, `mangum` (Lambda adapter for FastAPI)
    - Remove: `asyncpg>=0.29.0`, `pgvector>=0.2.4`
    - Note: no new dep for BM25 — `sqlite3` is Python stdlib

20. **New env vars** (update [.env.example](.env.example)):
    - `DYNAMODB_TABLE` — DynamoDB table name (default: `eth_protocol`)
    - `S3V_VECTOR_BUCKET` — S3 Vectors bucket name (required)
    - `S3V_INDEX_NAME` — vector index name (default: `eth_chunks`)
    - `FTS5_S3_BUCKET` — S3 bucket storing the FTS5 database file (required)
    - `FTS5_S3_KEY` — S3 object key for the FTS5 file (default: `fts5/bm25.db`)
    - `FTS5_LOCAL_PATH` — local path for the downloaded FTS5 file (default: `/tmp/bm25.db`)
    - `AWS_REGION` — (default: `us-east-1`)
    - `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` — optional, falls back to boto3 default chain
    - `AWS_ENDPOINT_URL` — optional, for local dev
    - Remove: `DATABASE_URL`, `POSTGRES_PASSWORD`

21. **Update [docker-compose.yml](docker-compose.yml)**:
    - Remove `postgres` service entirely
    - Remove `pgdata` volume
    - Remove `DATABASE_URL` and `POSTGRES_PASSWORD` from `api` service env
    - Remove `depends_on: postgres` from `api` service
    - Add `dynamodb-local` service for local dev (image: `amazon/dynamodb-local`)
    - Note: S3 Vectors has no local emulator yet — use real AWS or mock in tests

22. **Update [Dockerfile.api](Dockerfile.api)**: No direct PG references, but verify the image has sqlite3 with FTS5 (Python 3.12-slim includes it by default)

### Phase 10: Lambda deployment

23. **Add Lambda entry point** to [src/api/main.py](src/api/main.py):
    - Add `from mangum import Mangum` + `handler = Mangum(app)` for Lambda
    - Move `store` initialization into a lazy singleton (Lambda reuses across warm invocations)
    - FTS5 download happens once per cold start into `/tmp/bm25.db` (Lambda provides 512MB–10GB `/tmp`)
    - The `lifespan` context manager must handle Lambda's single-invocation lifecycle — use Mangum's `lifespan="auto"` or `lifespan="off"` with lazy init

24. **Update [justfile](justfile)**: Remove PostgreSQL reference comment (`PostgreSQL + pgvector`)

### Phase 11: Documentation

25. **Update [CLAUDE.md](CLAUDE.md)**:
    - Architecture section: PostgreSQL → DynamoDB + S3 Vectors + SQLite FTS5
    - Infrastructure section: remove `postgresql://` connection string, add new env vars
    - Key commands: remove `docker compose down -v` postgres reset
    - Module summary: update storage module description

26. **Update documentation files**:
    - [README.md](README.md): Remove `PostgreSQL (pgvector)` references, update env vars table
    - [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md): Replace schema diagrams, SQL examples, `asyncpg.create_pool`, `<=>` operator, `VECTOR(1024)` column
    - [docs/GETTING_STARTED.md](docs/GETTING_STARTED.md): Remove `POSTGRES_PASSWORD` setup, `docker compose logs postgres`, add AWS credential setup
    - [docs/CONCEPTS.md](docs/CONCEPTS.md): Update pgvector glossary entry
    - [docs/QUERIES.md](docs/QUERIES.md): Replace `from src.storage import PgVectorStore` examples with `Store`

### Phase 12: Tests

27. **Update [tests/conftest.py](tests/conftest.py)**: Add mock fixtures for `Store`, `S3VectorStore`, `DynamoDBStore`, `FTS5Store`. Update `MockStoredChunk` if the dataclass changes.

28. **Update [tests/test_phase3_hybrid_search.py](tests/test_phase3_hybrid_search.py)**: Uses `MockStoredChunk` mirroring pg-based `StoredChunk` — verify compatibility

29. **Update [tests/eval/run_eval.py](tests/eval/run_eval.py)**: Imports `PgVectorStore` directly — update to `Store`

30. **Update all test files** that reference `PgVectorStore` — update imports and mock targets

**Verification**

- `uv run pytest tests/ -v` — all tests pass with updated mocks
- `uv run ruff check . --fix && uv run ruff format .` — lint clean
- `grep -rn "asyncpg\|pgvector\|pg_vector_store\|PgVectorStore\|DATABASE_URL\|POSTGRES" src/ scripts/ tests/` — zero matches (confirms full removal)
- Manual: `uv run python scripts/ingest_eips.py` against real AWS
- Manual: `uv run python scripts/query_cli.py "What is EIP-1559?" --mode agentic`
- Verify `docker compose up -d` starts without postgres
- Verify Lambda: deploy + cold start + query cycle works end-to-end

**Decisions**
- **$0 at rest**: All services are serverless pay-per-request — DynamoDB on-demand, S3 Vectors, S3, Lambda. No always-on servers.
- **S3 Vectors** replaces pgvector entirely (managed ANN search, cosine distance, 1024-dim float32) — no in-memory index needed
- **Chunk metadata stored in S3 Vectors metadata**: `document_id` (filterable), `content`, `token_count`, `section_path`, `chunk_index`, `git_commit` (non-filterable). This avoids a DynamoDB round-trip after vector search.
- **BM25**: Contentless **SQLite FTS5** on disk. The `.db` file lives on regular S3 and is downloaded to local disk at runtime on cold start. During ingestion, chunks are inserted incrementally and the file is uploaded back to S3 when done. No full-corpus memory load. Contentless means only the inverted index is stored (~4x smaller than full-content FTS5); chunk text is hydrated from S3 Vectors metadata post-search.
- **DynamoDB single-table**: One table for documents + schema version. PartiQL via `pydynamodb`.
- **Upsert**: `put_vectors` is idempotent on key (overwrite). DynamoDB uses unconditional PutItem.
- **Metadata-filtered vector search**: `query_vectors(filter={...})` for `document_id` filtering. For complex multi-field filters, resolve document_ids from DynamoDB first, then pass as filter or post-filter.
- **No reindex needed**: S3 Vectors manages its index automatically.
- **Async**: All boto3/pydynamodb/sqlite3 sync calls wrapped in `asyncio.to_thread()`.
- **Lambda**: FastAPI served via Mangum adapter. FTS5 DB downloaded to `/tmp` on cold start. Store initialized lazily and reused across warm invocations.
- **FalkorDB**: Not part of this migration — remains as a separate service. For Lambda, graph features are optional (degrade gracefully if FalkorDB is unreachable).

**Files deleted**
- `src/storage/pg_vector_store.py`
- `src/storage/pg_schema.py`

**Files created**
- `src/storage/s3_vector_store.py`
- `src/storage/dynamodb_store.py`
- `src/storage/fts5_store.py`
- `src/storage/store.py`
- `src/storage/dynamodb_schema.py`

**Total files to modify**: ~55 (17 ingestion scripts, 7 retrieval modules, 4 structured modules, 1 agent module, 1 filter module, 1 API module, 1 storage __init__, 3 test files, 1 eval file, 6 docs files, pyproject.toml, docker-compose.yml, Dockerfile.api, .env.example, justfile, CLAUDE.md, README.md)

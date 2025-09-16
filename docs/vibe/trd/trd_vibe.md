# TRD: Local Files to Markdown

## 1. Technical Goals

- Deterministic PDF→Markdown pipeline; reproducible, resumable runs.
- High-fidelity hierarchy reconstruction (ToC + heuristics).
- Operationally robust CLI; clear YAML config; Windows-first path handling.

## 2. Architecture

- Layered pipeline: Ingest → Parse (Adapter) → Normalize → Structure → Emit.
- Functional core, imperative shell (CLI orchestrator).
- Caching at parse and normalize stages; bounded concurrency via config.

## 3. Components & Contracts

- **Ingest**: discover PDFs (file/dir), validate input, expand page ranges.
- **ParserAdapter**:
  - Method: `parse(pdf_path: Path, options: ParserOptions) -> ParserResult`.
  - Primary: Upstage DocumentParse; additional adapters post-MVP.
- **Normalizer**: `ParserResult → InternalDoc`; stable IDs; pydantic models.
- **StructureBuilder**: ToC alignment; heuristics (font tiers, heading regex, whitespace/layout anchors).
- **MarkdownEmitter**: write tree-structured Markdown with YAML front matter; canonical slugify; streaming writes.
- **CacheStore**: read/write parser responses and normalized artifacts; keying and invalidation.
- **CLI**: Typer-based; commands `convert`, `validate`.

## 4. CLI Specification

- `convert`:
  - `--config PATH` (required)
  - `--input PATH|DIR`, `--output DIR`
  - `--concurrency INT`, `--page-range START:END`, `--overwrite`, `--mock`
- `validate`:
  - `--input DIR` (output tree) — verifies structure, determinism, and cache integrity.

## 5. Config Schema (YAML)

- `job.name`, `input.path`, `output.dir`, `source_type`, `language`, `concurrency`, `page_ranges`.
- `parser.options` (API keys, endpoints, timeouts, table/figure flags).
- `emit.front_matter: bool`, `emit.images: bool`.
- `cache.dir`, `cache.mode: read_write|read_only|disabled`.
- `retry.max_attempts`, `retry.initial_backoff_ms`.

## 6. Internal Models (pydantic)

- `Document`: `doc_id`, `title`, `sections: list[Section]`, `tables: list[Table]`, `figures: list[Figure]`.
- `Section`: `section_id`, `level`, `title`, `blocks: list[Block]`.
- `Block`: `block_id`, `text`, `spans?: list[Span]`.
- `Table`: `table_id`, `markdown`, `caption?`.
- `Figure`: `figure_id`, `ref`, `caption?`.

## 7. Determinism Rules

- Stable sorting (by page, position, IDs), canonical slugification, seeded operations.
- No timestamps in content; optional YAML front matter contains only stable fields.
- Byte-identical outputs for fixed inputs/config.

## 8. Caching Strategy

- Location: `outputs/.cache/{parser,normalized}` (configurable).
- Keys: SHA256 of (PDF bytes + parser_version + relevant config subset).
- Invalidate on parser/config version change; `cache.mode` governs reads/writes.

## 9. Concurrency & Retries

- Worker pool for file-level parallelism; default `concurrency=4` (configurable).
- Exponential backoff on API calls; jittered retries up to `retry.max_attempts`.
- Respect API rate limits; fall back to serial on persistent 429/5xx.

## 10. Logging & Observability

- Console INFO by default; file logging to `log/YYYY-MM-DD_HHMMSS_pipeline.log`.
- Fields: `run_id`, `stage`, `pdf_path`, `source_type`, `elapsed_ms`, `item_count`; errors include exception and stack trace.
- Use `pytest` + `caplog` to assert key INFO/DEBUG events.

## 11. Testing Strategy

- Unit tests for adapters, normalization, slugify; mocks for API.
- Golden fixtures (≥3 PDFs) for acceptance: hierarchy ≥95%, tables/figures preserved.
- CLI E2E smoke tests with cached re-runs for determinism.

## 12. Non-Functional Constraints

- Python 3.11+, Poetry, Windows-friendly paths and filename length guards.
- Streaming emission to cap memory; page-range filtering supported.

## 13. Milestones

- Phase 1: Config, CLI, Upstage client, logging, cache skeleton.
- Phase 2: Normalization, structure heuristics, deterministic emitter, acceptance tests.
- Phase 3: Performance hardening, concurrency tuning, docs and example configs.

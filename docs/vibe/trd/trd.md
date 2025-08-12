## 1. Executive Technical Summary

### Product Overview

A mono-repo, pure Python package and CLI that converts PDFs into hierarchy-preserved Markdown, optimized for LLM ingestion. It orchestrates parsing via Upstage DocumentParse, reconstructs document structure (ToC + heuristics), and emits deterministic, tree-structured Markdown with optional YAML front matter.

### Core Technology Stack

- Python 3.12+
- Upstage DocumentParse API (primary parser)
- `pydantic`, `pydantic-settings` for config and models
- `dependency-injector` for pluggability and adapters
- `langchain`, `langchain-core`, `langchain-upstage` for downstream integration and optional graph workflows
- `pymupdf` and `markitdown` as optional fallbacks for rendering/fidelity
- `tqdm`, `pandas`, `numpy` for operations and progress

### Key Technical Objectives

- Deterministic, repeatable pipeline from PDF to Markdown trees
- High-fidelity hierarchy reconstruction (ToC + heuristic detection)
- Throughput at scale with parallelism and caching
- Clean, composable CLI with clear YAML configuration
- Seamless integration hooks for LangChain/LangGraph-based pipelines

### Critical Technical Assumptions

- Primary parsing quality is delegated to Upstage DocumentParse; tables/figures are available from the API.
- Language is provided via YAML config; no language auto-detection.
- Cloud, SaaS UI, and enterprise security are out of scope for MVP.
- Outputs must be byte-identical given fixed inputs and config.

## 2. Tech Stack

- Runtime: Python 3.12+
- Packaging: Poetry
- CLI: `typer` or `click` (final selection in implementation), entry point `local-files-to-md`
- Config and Models: `pydantic`, `pydantic-settings`, YAML via `pyyaml`
- Parsing: Upstage DocumentParse; optional adapters for `pymupdf`, `pdfminer.six` (post-MVP)
- Processing: `pandas`, `numpy`, `tqdm`
- Integration: `langchain`, `langchain-core`, `langchain-upstage`
- Testing: `pytest`
- Logging: `logging` stdlib (console INFO by default; file logging to `log/` with timestamped filenames)

## 3. System Architecture Design Review

### Option A: Layered Orchestrator with Parser Adapter Interface (Recommended)

- Ingest → Parse (adapter) → Normalize → Structure → Emit → Orchestrate (CLI)
- Pros: clean separation of concerns, easy to swap parsers, deterministic outputs, testable boundaries.
- Cons: requires careful internal document model and versioning.
- Why: aligns with PRD’s needs (extensibility, determinism, Upstage-first, future pluggability).

### Option B: Event-Driven Pipeline with Task Queue

- Steps emitted as events (e.g., parse_completed → normalize → structure → emit) with local queue.
- Pros: resilient to failures, easy resumability and metrics; natural concurrency control.
- Cons: heavier runtime footprint and operational complexity for a local CLI package; overkill for MVP.
- Why: viable for future scale or distributed runs, but not necessary at MVP.

### Option C: Functional Core, Imperative Shell

- Pure functions for each stage (stateless, deterministic) wrapped by thin CLI shell.
- Pros: maximal testability and determinism; easy to reason about.
- Cons: integration points (API retries, caching, IO) need careful side-effect orchestration.
- Why: complements Option A; can be combined (functional internal modules within layered architecture).

## 4. Performance & Optimization Strategy

- Concurrency: parallel file-level processing with configurable worker pool; bound by API rate limits and local CPU.
- Caching: persist parser responses and normalization artifacts to enable idempotent re-runs and partial resumes.
- Streaming Emission: write Markdown incrementally to avoid memory spikes on large PDFs.
- Heuristic Efficiency: pre-compute heading tiers and page anchors; avoid N^2 scans.
- Determinism: stable sorting, seeded operations, canonical slugification for filenames/paths.
- Benchmarks: per-source_type golden PDFs with page/time/accuracy metrics tracked.

### Logging & Observability

- Console logging: INFO by default; use DEBUG only inside tight loops or verbose paths to avoid noise. Consistent module-level logger names.
- File logging: write logs to `log/` at project root with timestamped filenames like `YYYY-MM-DD_HHMMSS_pipeline.log`; ensure `log/` exists and is gitignored. One file per run.
- Structured fields: include `run_id`, `stage` (ingest/parse/normalize/structure/emit), `pdf_path`, `source_type`, `elapsed_ms`, and `item_count` where applicable. Errors include exception type, message, and stack trace.
- Correlation: propagate `run_id` across stages to tie events to a single job; progress bars via `tqdm` remain enabled for human feedback.
- Testability: use `pytest` with `caplog` to assert key INFO/DEBUG messages at each stage (e.g., start/complete events, counts, cache hits).

## 5. Implementation Roadmap & Milestones

### Phase 1: Foundation (MVP Implementation)

- Config schema (pydantic) and YAML loader; logging baseline (console INFO + file logging to `log/` with timestamped filenames, structured key=value fields, `run_id` correlation)
- CLI skeleton (`convert`, `validate`) and file discovery
- Upstage DocumentParse client with retries, backoff, and response caching
- Internal document model (blocks/spans/tables/figures)
- Normalization pipeline to internal model; deterministic transforms
- Structure builder: ToC extraction, heading heuristics, page-range mapping
- Markdown emitter: tree-structured output, YAML front matter
- E2E CLI for bulk conversion; unit tests and golden fixtures

### Phase 2: Feature Enhancement

- Interactive structure preview (optional UI or TUI)
- Pluggable parser adapters (pymupdf, pdfminer) and adapter conformance tests
- Optional image extraction/embedding
- LangChain integration shims and metadata alignment
- Performance hardening: profiling, concurrency tuning, cache invalidation policy

## 6. Risk Assessment & Mitigation Strategies

### Technical Risk Analysis

- Parser dependency risk (API changes/quality drift): adapter interface, version pinning, conformance tests.
- Heuristic brittleness for unusual PDFs: rule sets by source_type, override patterns via YAML, preview tooling.
- Throughput limits (API rate, compute): caching, concurrency limits, page-range filtering.
- Markdown fidelity for complex tables/figures: fallback rendering, asset referencing, regression tests.
- File path and naming constraints on Windows: canonical slugify, length guards, collision checks.

### Project Delivery Risks

- Scope creep beyond MVP: enforce PRD boundaries and stage features into Phase 2.
- Dependency delays (API access/keys): early stub/mocks; toggle to mock mode for development tests.
- Insufficient test coverage on representative PDFs: curate golden set early; automate benchmark runs.
- Team bandwidth and timeline slippage: milestone gates with acceptance criteria from PRD; timeboxed iterations.

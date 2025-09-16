# PRD: Local Files to Markdown

## 1. Product Goal

Convert local PDFs into clean, hierarchy-preserving Markdown trees for LLM ingestion via a deterministic, scriptable CLI.

## 2. Users & Jobs-to-be-Done

- **Researcher/Student**: turn lecture notes and papers into organized Markdown for study/RAG.
- **Archivist**: bulk convert large PDF folders into a navigable Markdown corpus.
- **ML/Platform engineer**: produce consistent Markdown chunks with metadata for pipelines.

## 3. Inputs, Outputs, Interfaces

- **Inputs**:
  - PDF files or a directory of PDFs
  - YAML config with source_type, language, parser options, output paths, concurrency, page ranges

- **Outputs**:
  - Tree-structured Markdown files with optional YAML front matter
  - Optional assets for tables/figures (if enabled)

- **CLI**:
  - Entry: `local-files-to-md`
  - Subcommands: `convert`, `validate`

## 4. MVP Scope (Must-haves)

- File discovery (directory and single-file), input validation, skip/continue on failure
- Upstage DocumentParse integration (text, tables, figures) with retries and bounded concurrency
- Normalization to an internal document model (blocks, sections, tables, figures)
- Structure reconstruction: ToC alignment when present; heuristics when absent (font tiers, heading patterns, whitespace/layout cues)
- Deterministic Markdown emission: stable ordering, canonical slugification, optional YAML front matter
- Config-driven behavior via a single YAML file (global and per-source overrides)
- Response caching to enable resumable runs and byte-identical outputs with fixed inputs/config
- Logging to console (INFO) and `log/` file per run with timestamps

## 5. Non-Goals (MVP)

- Cloud storage integrations, SaaS UI, desktop GUI
- Language auto-detection; enterprise security features
- Interactive structure editor (post-MVP)

## 6. Constraints

- Windows-first path handling and filename length guards
- Determinism: fixed ordering, stable slugify, seeded operations
- Reasonable memory footprint via streaming emission and page-range filtering

## 7. Success Metrics

- Determinism: given fixed inputs/config, outputs are byte-identical
- Hierarchy quality: ≥95% section ordering/nesting correct on golden PDFs
- Operational reliability: zero unhandled exceptions on a batch of ≥100 PDFs; actionable error logs
- Usability: bulk conversion with one CLI command and one YAML file

## 8. Acceptance Criteria

- Running `local-files-to-md convert --config <job.yaml>` produces a tree of Markdown files whose content and paths are byte-identical across runs with the same inputs/config
- For the golden set (lecture notes, textbook chapter, academic paper):
  - Headings/sections are ordered and nested correctly (≥95%)
  - Tables become valid Markdown tables or referenced assets with stable links
  - Figures are referenced with stable filenames; optional embedding works when enabled
- Failures are logged with file path, stage, and exception; partial runs are resumable from cache

## 9. Milestones

- **M1**: Config schema, CLI skeleton (`convert`, `validate`), Upstage client with retries, basic logging, golden PDFs
- **M2**: Normalization + structure heuristics, deterministic emitter, caching, YAML front matter
- **M3**: Hardening: concurrency tuning, Windows path guards, docs, example configs, end-to-end tests

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Git Commit Convention

- All git commit messages made on behalf of the user must be written in **Chinese**.
- Always end with `# AI Commit 100% #`.

Example: `chore: 添加 uv 镜像源配置 # AI Commit 100% #`

## Project Overview

graphify turns any folder of code, docs, papers, images, or videos into a queryable knowledge graph. PyPI package name is `graphifyy` (double-y); CLI command is `graphify`. Python 3.10+, built on NetworkX + tree-sitter.

## Development Setup

```bash
uv sync --all-extras          # install all deps (core + optional + dev)
uv run graphify --version     # verify
```

## Commands

```bash
# Tests
uv run pytest tests/ -q                    # full suite
uv run pytest tests/test_extract.py -q     # single module
uv run pytest tests/ -q -k "python"        # filter by keyword

# Lint (conservative config — only E9,F63,F7,F82)
uv run ruff check graphify/ tests/

# Type check
uv run pyright

# Security audit
uv run bandit -r graphify/
```

CI runs on Python 3.10 and 3.12 (ubuntu-latest): `uv sync --all-extras` → `pytest` → `graphify --help` → `graphify install`.

## Architecture

### Pipeline (7 stages)

```
detect() → extract() → build_graph() → cluster() → analyze() → report() → export()
```

Each stage is a standalone function in its own module (`detect.py`, `extract.py`, `build.py`, `cluster.py`, `analyze.py`, `report.py`, `export.py`). They communicate via plain dicts and NetworkX graphs — no shared state, no side effects outside `graphify-out/`.

### Extraction schema

Every extractor returns `{nodes, edges}` where:
- `nodes`: `[{id, label, source_file, source_location}]`
- `edges`: `[{source, target, relation, confidence}]` with confidence `EXTRACTED` | `INFERRED` | `AMBIGUOUS`

`validate.py` enforces this schema before `build_graph()` consumes it.

### Key modules

| Module | Role |
|--------|------|
| `__main__.py` | CLI entry point — all subcommands and 20+ platform installers (~4100 lines) |
| `__init__.py` | Lazy-import facade via `__getattr__` — lets `graphify install` work without heavy deps |
| `extract.py` | AST extraction for 25+ languages via tree-sitter |
| `detect.py` | File discovery, type classification, `.graphifyignore`, incremental change detection |
| `build.py` | Assembles node+edge dicts into NetworkX graph with dedup |
| `cluster.py` | Community detection — Leiden (graspologic) or Louvain fallback |
| `analyze.py` | God nodes, surprising connections, suggested questions |
| `export.py` | Outputs: graph.json, graph.html (pyvis), graph.svg, graphml, Obsidian vault, Neo4j Cypher |
| `security.py` | URL validation, safe fetch, path guards, label sanitization |
| `serve.py` | MCP stdio server (query_graph, get_node, get_neighbors, shortest_path, list_prs, get_pr_impact, triage_prs) |
| `cache.py` | Per-file extraction cache (stat-based + content hashing) |
| `dedup.py` | Entity dedup via MinHash/LSH blocking + Jaro-Winkler verification + union-find |
| `symbol_resolution.py` | Deterministic symbol indexing and cross-file resolution |
| `manifest.py` | Re-exports from `detect.py` for backwards compat |

### Security boundary

All external input flows through `security.py`: URL validation (http/https only, blocks private IPs), response size caps (50MB/10MB), path traversal prevention, label sanitization (strip control chars, HTML-escape, 256 char cap). See `SECURITY.md` for the full threat model.

## Testing

- One test file per module in `tests/`, sample fixtures in `tests/fixtures/`
- All tests are pure unit tests — no network calls, no filesystem side effects outside `tmp_path`
- `conftest.py` suppresses noisy warnings from graspologic/umap/numba transitive deps
- `norecursedirs` in pyproject.toml excludes several top-level directories from test discovery

## Adding a New Language Extractor

1. Add `extract_<lang>(path) -> dict` in `extract.py` following existing tree-sitter patterns
2. Register file suffix in `extract()` dispatch and `collect_files()`
3. Add suffix to `CODE_EXTENSIONS` in `detect.py` and `_WATCHED_EXTENSIONS` in `watch.py`
4. Add tree-sitter package to `pyproject.toml` dependencies
5. Add fixture to `tests/fixtures/` and tests to `tests/test_languages.py`

## Git Workflow

- Active branch: `v8`
- Commit style: conventional commits (`fix:`, `feat:`, `docs:`)
- Run `uv run pytest tests/ -q` and confirm pass before opening PRs

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `GRAPHIFY_OUT` | Override output directory (default: `graphify-out`) |
| `GRAPHIFY_MAX_WORKERS` | AST parallelism |
| `GRAPHIFY_DEBUG` | Verbose error output |

## Notable Quirks

- `uv.toml` points to Tsinghua PyPI mirror — may need changing for non-China networks
- `__main__.py` is ~4100 lines containing all CLI subcommands and platform install logic
- `scip_ingest.py` is a skeleton not wired to CLI yet
- `multigraph_compat.py` is a runtime probe for future MultiDiGraph mode (no call sites yet)
- macOS has case-insensitive filesystems, so `sample.f90` and `sample.F90` fixtures can collide

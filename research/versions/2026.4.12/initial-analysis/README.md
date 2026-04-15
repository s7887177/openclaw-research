# Initial analysis (source-repo version: 2026.4.12)

This document starts a new research series that is explicitly tied to the authoritative source code in `source-repo/` (version 2026.4.12 as found in package.json).

Goals:
- Use `source-repo/` as the primary source of truth for all code-level facts.
- Produce structured research under `research/versions/2026.4.12/` using the format in skills/copilot-new-research/new-research.md.

Quick code-architecture notes (initial pass):
- Entrypoints: `openclaw.mjs` appears to be a top-level runner/CLI entry.
- Core directories: `src/` contains main application code; `skills/` contains skill definitions; `packages/` and `apps/` contain modular components.
- Configs: `package.json`, `pnpm-workspace.yaml`, and various Dockerfiles define runtime and developer environment.

Next steps:
1. Crawl `source-repo/src/` and identify module boundaries, public interfaces, and main classes/functions.
2. For each research topic, produce a file under `research/versions/2026.4.12/<level>/<topic>/` that cites exact source files and line ranges when asserting facts.
3. Produce architecture diagrams and call graphs where we can reliably infer structure from code (avoid speculative diagrams).

(Expanded analysis and per-file parsing will follow in subsequent commits.)


# CLAUDE.md — AI Cluster Network Stack Book

## Project

Technical book: *The AI Cluster Network Stack — Building High-Performance Fabrics for GPU Clusters from First Principles*

- **Scope:** 30 chapters, 7 parts, ~600 pages, appendices A–G
- **Audience:** Senior software/infrastructure engineers; Linux + TCP/IP assumed, no prior RDMA/HPC required
- **Lab environment:** Containerlab on a single laptop (SR Linux + FRR + SONiC-VS); soft-RDMA via `rdma_rxe` where needed
- **License:** CC BY 4.0

## Authoring Workflow

All new content follows this four-phase process:

### 1. Research
- Search primary sources: RFCs, IEEE standards, official project docs, GitHub repos, academic papers
- Use `arxiv-survey` skill for literature surveys on a topic
- Collect version-specific details (API names, CLI flags, config syntax) — these must be accurate
- Note any areas of uncertainty for expert review

### 2. Plan
- Outline section structure against the chapter plan in `book-outline.md`
- Identify hands-on lab exercises: must be runnable on a single laptop with Containerlab
- Flag cross-references to other chapters
- Confirm coverage matches the chapter's page budget (see `book-outline.md`)

### 3. Write
- Draft prose section by section, following the plan
- Include worked code examples, config snippets, and CLI walkthroughs for every major concept
- Write each lab exercise as a numbered step sequence with expected output
- Cross-reference related chapters inline (e.g., *see Chapter 7*)

### 4. Review
- **Correctness:** Verify CLI commands, config snippets, API names, and version numbers against primary sources
- **Cohesiveness:** Check that terminology is consistent with other chapters; cross-references resolve; the narrative flows from prior chapters
- **Lab validity:** Commands and topology YAML must be executable in the stated environment

## File Conventions

- One file per chapter: `chNN-slug.md` (e.g., `ch07-ebpf-xdp.md`)
- Appendices: `appendix-a-lab-topology-library.md`, etc.
- Supporting docs: `book-outline.md` (detailed plan), `book-plan.md`, `reference-map.md`
- Front matter: `front-matter.md`

## Content Standards

- Bold first use of every major technology name
- All labs must be self-contained and runnable without proprietary hardware
- Code blocks must specify language and include comments for non-obvious steps
- Each chapter ends with a References section containing full documentation URLs
- Footer: `© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).`

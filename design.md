# LLM Wiki — Operations Design

## Component Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Human (User)                      │
│  - Curates sources    - Asks questions               │
│  - Directs analysis   - Reviews wiki in Obsidian     │
│  - Commits raw/       - Browses git history          │
└──────────┬──────────────────────┬───────────────────┘
           │ drop source / query  │ browse wiki
           ▼                      ▼
┌─────────────────┐    ┌─────────────────────────────┐
│   LLM Agent     │◄──►│   Wiki Layer (wiki/)         │
│  (Claude Code,  │    │  - Entity pages              │
│   Codex, etc.)  │    │  - Concept pages             │
│                 │    │  - Source summaries           │
│  Governed by:   │    │  - Comparisons / analyses    │
│  Schema file    │    │  - index.md (catalog)        │
│  (CLAUDE.md)    │    │                              │
└────────┬────────┘    └─────────────────────────────┘
         │ read-only          ▲
         ▼                    │
┌─────────────────┐    ┌──────┴──────────────────────┐
│  Raw Sources    │    │  Git (operation log)         │
│  (raw/)         │    │  - One commit per op         │
│  Immutable      │    │  - Path filter separates     │
│                 │    │    AI (wiki/) vs user (raw/) │
└─────────────────┘    └─────────────────────────────┘
```

## Components

### 1. Raw Sources (`raw/`)
- Immutable source-of-truth layer
- User-curated: articles, papers, transcripts, images, data
- LLM reads but never modifies; **never stages `raw/` in AI commits**
- Subdirectory `raw/assets/` for downloaded images

### 2. Wiki (`wiki/`)
- LLM-owned markdown files
- Page types: **entity**, **concept**, **source-summary**, **comparison**, **synthesis**, **overview**
- All pages use YAML frontmatter (tags, dates, source refs)
- Cross-references via `[[wikilinks]]`
- One special file:
  - **`index.md`** — content catalog, organized by category, updated on every ingest

### 3. Git (operation log)
- Commit history replaces any `log.md` file
- One commit per wiki-modifying operation (ingest, query-filed, lint)
- Subject format: `<op>(<scope>): <subject>`
- Structured trailers: `Pages-Touched`, `Sources-Added`, `Contradictions`, `Query`
- Path filter invariant: AI commits touch `wiki/` only, user commits touch `raw/` only
- Schema changes (`CLAUDE.md`, `design.md`) committed separately with `schema:` prefix

### 4. Schema (`CLAUDE.md` / `AGENTS.md`)
- Governance doc: conventions, page formats, workflows, git commands
- Co-evolved by human + LLM over time

### 5. LLM Agent
- Executes operations (session-resume, ingest, query, lint)
- Reads schema for behavioral rules
- Uses `git log` for history queries (session resume, dup check, recency, staleness)
- Reads index.md for navigation
- Optionally uses CLI tools (e.g. `qmd`) for search at scale

### 6. Optional: Search Engine
- Small scale: `index.md` suffices
- Scale: hybrid BM25/vector search (e.g. `qmd`)
- Exposed via CLI or MCP server

---

## Git Operation Schema

### Commit message format

```
<op>(<scope>): <subject>

<optional short body>

Pages-Touched: <comma-separated paths>
Sources-Added: <raw paths, or "none">
Contradictions: <note, or "none">
Query: <original user question, query-filed only>
```

- `<op>` ∈ {`ingest`, `query-filed`, `lint`, `schema`, `fix`}
- `<scope>` = source slug / page slug / `wiki`
- Subject ≤72 chars, imperative mood

### Path separation invariant

| Author | Commits touch | Subject prefixes |
|--------|---------------|------------------|
| LLM    | `wiki/` only  | `ingest`, `query-filed`, `lint`, `fix` |
| User   | `raw/` only   | free-form |
| Either | repo root     | `schema:` (own commit, never mixed) |

### Git commands by purpose

| Purpose | Command |
|---------|---------|
| Session resume (AI activity) | `git log --oneline -20 -- wiki/` |
| Session resume (pending user sources) | `git log --oneline -10 -- raw/` |
| Duplicate detection | `git log --format="%H %s" -- "wiki/sources/<slug>.md"` |
| Recency context | `git log --since="2 weeks ago" -- wiki/` |
| Prior queries on topic | `git log --grep="^query-filed" -- wiki/` |
| Staleness (per page) | `git log -1 --format=%cI -- <page>` |
| Staleness (newer ingests) | `git log --grep="^ingest" --since=<date> -- wiki/` |
| User audit | `git log --grep="^<op>" -- wiki/` |

---

## Operations

### Operation 0: Session Resume

**Trigger:** LLM starts a new conversation/session.

**Steps:**
1. Read schema (`CLAUDE.md`) for rules
2. `git log --oneline -20 -- wiki/` — recent AI activity
3. `git log --oneline -10 -- raw/` — new user sources awaiting ingest
4. `git status` — abort if working tree dirty
5. Read `wiki/index.md` for page inventory
6. Brief status: last ingest, pending sources, page counts, last lint findings

**No commit.** Read-only.

### Operation 1: Ingest

**Trigger:** User drops new source in `raw/` and tells LLM to process it.

**Steps:**
1. `git log --format="%H %s" -- "wiki/sources/<slug>.md"` — duplicate check
2. If duplicate → warn user, skip or confirm re-ingest
3. Read source from `raw/`
4. Discuss key takeaways with user
5. Read `wiki/index.md` — find existing pages this source touches
6. Write source summary → `wiki/sources/<slug>.md`
7. For each entity/concept:
   - Exists → read, update, flag contradictions, add source, bump `updated`
   - Missing → create new page with frontmatter
8. Update cross-references on touched pages
9. Update `wiki/index.md`
10. `git add <paths in wiki/> && git commit` with `ingest(<slug>): ...` and trailers
11. Report commit hash + summary to user

**Scope:** Single ingest touches 10-15 pages. All in one commit.

**Variants:**
- Interactive ingest (default)
- Batch ingest (multiple sources, one commit per source)

### Operation 2: Query

**Trigger:** User asks a question.

**Steps:**
1. `git log --since="2 weeks ago" -- wiki/` — recency context
2. `git log --grep="^query-filed" -- wiki/` — prior queries on topic (optional)
3. Read `wiki/index.md` — identify candidate pages
4. Optional: shell out to `qmd` for search
5. Read relevant pages
6. Synthesize answer with citations
7. Present in appropriate format (markdown, table, Marp slides, chart)
8. If high-value → offer to file
9. If user accepts: write page, add backlinks, update `index.md`, `git commit` with `query-filed(<slug>): ...` + `Query:` trailer
10. If user declines: no commit

### Operation 3: Lint

**Trigger:** User says "lint" or periodic maintenance.

**Steps:**
1. `git log --grep="^ingest" -- wiki/` — ingest timeline
2. `git log --grep="^lint" -1 -- wiki/` — last lint date
3. Read `wiki/index.md` — page inventory
4. Per suspect page: `git log -1 --format=%cI -- <page>` → last-touch timestamp; compare to newer ingest commits
5. Run checks:
   - **Contradictions** — conflicting claims
   - **Stale pages** — not updated since newer sources arrived
   - **Orphans** — no inbound wikilinks
   - **Missing pages** — concepts mentioned but lacking page
   - **Missing cross-references** — related pages not linked
   - **Data gaps** — topics worth deeper sourcing
6. Produce lint report
7. Await user approval (per-fix or batch)
8. Apply approved fixes
9. Update `index.md` if pages added/removed
10. `git commit` with `lint: ...` and trailers
11. Optional: `git tag lint/YYYY-MM-DD`

---

## Data Flow Summary

| Operation | Reads | Writes | Commit | Human |
|-----------|-------|--------|--------|-------|
| Session Resume | `git log -- wiki/`, `git log -- raw/`, index, schema | — | No | None |
| Ingest | **git log (dup check)**, raw, index, existing pages | summary, entity/concept pages, index | Yes: `ingest(<slug>): ...` | Reviews, guides emphasis |
| Query | **git log (recency, prior queries)**, index, wiki, (search) | answer; optionally new page + index | If filed: `query-filed(<slug>): ...` | Asks, decides filing |
| Lint | **git log (timestamps)**, index, all wiki pages | fixed pages, index | Yes: `lint: ...` | Approves fixes |

---

## File Naming Conventions

```
raw/
  articles/
  papers/
  transcripts/
  assets/          # downloaded images
wiki/
  index.md         # content catalog
  sources/         # one summary per ingested source
  entities/        # people, orgs, places
  concepts/        # ideas, theories, patterns
  comparisons/     # side-by-side analyses
  synthesis/       # cross-cutting themes
  overview.md      # high-level project summary
CLAUDE.md          # schema / governance doc
design.md          # this file
```

## Page Frontmatter Schema

```yaml
---
title: "Page Title"
type: entity | concept | source-summary | comparison | synthesis | overview
created: 2026-04-15
updated: 2026-04-15
sources: ["raw/articles/article-name.md"]
tags: [tag1, tag2]
---
```

## Git Rules (summary)

- One commit per operation. Atomic. No mixed-op commits.
- No commit for read-only ops (session-resume, un-filed queries).
- Working tree clean between ops.
- Never stage `raw/` in AI commits.
- Never mix `wiki/` + schema in same commit.
- No `--amend` on published history. No force-push without explicit user OK.

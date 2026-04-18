# Personal Knowledge Base — Specification

A reusable spec for turning an empty directory into a Claude-operated personal knowledge base. Apply it by dropping the companion `CLAUDE.md` into the directory root; the layout below is created on demand the first time each path is needed.

## Purpose

A long-lived librarian. The user pastes meeting transcripts, code snippets, command-line output, and asks questions over time. Sessions are short-lived and interrupted often, so every meaningful piece of state lives on disk — Claude's conversation context is not durable. A fresh Claude session in the directory reads `CLAUDE.md`, bootstraps from index files, and is productive immediately.

## Directory layout (hybrid — type-first with project hubs)

```
<base>/
├── CLAUDE.md                         # rules (stable, rarely edited)
└── knowledge/
    ├── TAGS.md                       # freeform-but-tracked tag vocabulary
    ├── todos.md                      # open todos, grouped by project (single source of truth)
    ├── todos-archive.md              # completed todos, grouped by project + done-date
    ├── inbox/                        # unclassified pastes, triaged manually
    ├── raw/                          # verbatim original pastes (ground truth)
    │   └── YYYY-MM-DD-HHMM-<slug>.md
    ├── meetings/
    │   └── YYYY-MM-DD-<slug>.md      # distilled meeting notes, links to raw/
    ├── snippets/
    │   └── <slug>.md                 # evergreen code snippets
    ├── commands/
    │   └── <slug>.md                 # evergreen command-line recipes
    ├── notes/
    │   └── <slug>.md                 # evergreen concept notes
    ├── projects/
    │   ├── INDEX.md                  # Active / Archived project registry
    │   └── <slug>.md                 # one hub per project
    └── journal/
        ├── INDEX.md                  # pointer to current week + open threads
        ├── YYYY-MM-DD.md             # one file per day (append-only within day)
        └── weekly/
            └── YYYY-Www.md           # Monday–Sunday summaries, on request
```

Rationale for hybrid: type-first storage keeps ingestion deterministic (a transcript always goes to `meetings/`). Project hub files provide a readable per-project view without duplicating data. The two axes meet through forward links.

## Frontmatter schema (minimal)

Every entry begins with:

```yaml
---
title: <human-readable>
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: meeting | snippet | command | note | raw | project | journal
project: <slug> | none | [slug-a, slug-b]    # single preferred; list only when genuine
tags: [freeform, lowercase-hyphenated]
source: raw/<file>.md                        # only for distilled entries
status: active | archived                    # project hubs only
---
```

Extra fields (attendees, priority, etc.) go in the body, not the frontmatter. Frontmatter is for routing and retrieval only.

## Behavioral rules

### 1. Bootstrap on every session
Read order: `CLAUDE.md` → `knowledge/projects/INDEX.md` → `knowledge/journal/INDEX.md` → current-week journal files. Then ready.

### 2. Ingestion — verbatim + distilled
- Every paste is saved verbatim to `knowledge/raw/YYYY-MM-DD-HHMM-<slug>.md`.
- A distilled companion is written to the appropriate type folder (`meetings/`, `snippets/`, `commands/`, `notes/`) with `source:` pointing to the raw file.
- The relevant project hub gets a forward link to the distilled entry (not the raw).

### 3. Project assignment — explicit registry + ask when unclear
- `knowledge/projects/INDEX.md` is authoritative. Projects only exist if listed there.
- Obvious match → file directly, state what was done.
- Substantial content, ambiguous project → ask once, then file.
- Low-value or ambiguous small paste → `knowledge/inbox/` with timestamped filename; mention it. User runs `triage inbox` periodically.
- `new project: <name>` is the only way a project gets added to `INDEX.md`.

### 4. Updates — new-file vs update-in-place by folder
- Time-bound folders (`meetings/`, `journal/`, `raw/`): always new file.
- Evergreen folders (`snippets/`, `commands/`, `notes/`, `projects/`): update in place, bump `updated:`, state explicitly which file is being overwritten (never silent).
- If an evergreen update meaningfully changes meaning (not just refinement), ask whether to update or create a sibling.

### 5. Linking — relative paths, forward-only
- Standard markdown: `[text](../path/to/file.md)`. No wikilinks.
- Forward links only. Backlinks are not maintained; Claude greps on demand when asked.
- Project hubs link to their entries; entries link back to their hub.

### 6. Journal — append immediately, one file per day
Every interaction that touches the knowledge base (ingest, answer-from-KB, librarian action) appends an entry to `knowledge/journal/YYYY-MM-DD.md` *before* replying to the user.

Entry format:
```markdown
## HH:MM — <short verb-phrase title>
- project: <slug> | none
- added/updated: <file path(s)>    # omit if none
- searched: <folders or files>     # only for queries
- note: <one-line summary of what happened>
```

Purely generic questions (no reference to stored content) are not journaled. For todos: writes (`add`, `complete`, `reopen`) journal; reads (`todos`, `todos for …`, `done todos …`) do not.

### 7. Weekly summary — user-requested only
- Week = Monday to Sunday, ISO 8601 (`YYYY-Www`).
- Triggered by `weekly summary` (current week) or `weekly summary <Www>` / `weekly summary for <date>`.
- Claude reads all `journal/YYYY-MM-DD.md` files in the target week, synthesizes into `journal/weekly/YYYY-Www.md` with sections: *projects touched*, *key decisions*, *artifacts added*, *completed this week*, *open questions / threads*.
- "Completed this week" is built by filtering `todos-archive.md` entries by `(done YYYY-MM-DD)` within the target week, grouped by project. Omit the section if nothing was completed. Open todos are **not** snapshotted — they would go stale; use live `todos` queries for current state.
- Never auto-generated.

### 8. Query heuristic — search KB when plausibly personal
Search `knowledge/` first if the question:
- references a known project name (from `projects/INDEX.md`),
- uses possessives ("my", "our", "the team's", "we", "I"),
- is time-bound ("yesterday", "last week", "what did I"),
- names a person, meeting, or artifact by name.

Otherwise answer from general knowledge without a search. If a search is performed and returns empty: say so, then offer a general answer — *"Nothing in your notes on this. Generally..."*.

### 9. Archiving — flag only, no moving
- `archive project: <slug>` sets `status: archived` in the hub's frontmatter and moves the line in `projects/INDEX.md` from Active to Archived.
- Hub file does not move. No links break.
- Archived projects remain searchable; excluded only from default project suggestions.
- `todos.md` is **not** touched. Open todos for an archived project stay visible via `all todos` / `todos for <slug>`. User completes or reopens manually if they care.
- `rename project: <old> -> <new>` also renames the `## <old>` heading in both `todos.md` and `todos-archive.md`.
- No auto-archive. Archival is an explicit user action.

### 10. Tags — freeform, tracked in `knowledge/TAGS.md`
- `TAGS.md` is a flat list with usage counts, updated automatically when new tags appear.
- Before inventing a tag, Claude scans `TAGS.md` and reuses an existing one if a synonym exists.

### 11. Canonical command phrases
Loose variants accepted; these are the documented contract:
- `what can I say` — print this list
- `weekly summary` / `weekly summary <Www or date>`
- `triage inbox`
- `new project: <name>` / `archive project: <slug>` / `rename project: <old> -> <new>`
- `what did I work on <today|this week|on <project>>`
- `show project: <slug>` — hub file plus that project's open todos (composed at query time)
- `link: <file-a> <-> <file-b>`
- `add todo: <text>` / `add todo for <project>: <text>`
- `todos` (all open, grouped) / `todos for <project>` / `todos for none`
- `complete todo: <text or partial match>` / `reopen todo: <text or partial match>`
- `done todos [this week|<Www>|for <project>]`

### 12. First-run bootstrap
If `TAGS.md`, `projects/INDEX.md`, or `journal/INDEX.md` is missing, Claude creates it with an empty scaffold on first use. This is the only implicit write at session start. `todos.md` and `todos-archive.md` are created lazily on the first `add todo` / `complete todo` respectively — not at session start.

### 13. Todos

Single source of truth for open todos: `knowledge/todos.md`. Completed items are cut and pasted into `knowledge/todos-archive.md` on completion. Neither file has frontmatter.

**`todos.md` format:**

```markdown
# Todos

## none
- [ ] 2026-04-17 — Try new Rust async runtime

## auth-refactor
- [ ] 2026-04-17 — Write migration for session tokens
- [ ] 2026-04-17 !high due:2026-04-25 — Security review before launch
```

Line grammar: `- [ ] <created-date> [!priority] [due:YYYY-MM-DD] — <text>`. Created-date always present. `!priority` (`!low|!med|!high`) and `due:YYYY-MM-DD` optional, included only when the user explicitly states them — Claude never invents either. Heading order: `## none` first, then active projects alphabetical, archived projects after.

**`todos-archive.md` format:** identical grouping; each line is `- [x] <created-date> — <text> (done YYYY-MM-DD)`.

**Adding:** `add todo: <text>` defaults to `project: none`. If content clearly matches an active project (same heuristic as ingestion), Claude infers it; if substantial and ambiguous, it asks once. `add todo for <project>: <text>` is explicit. New headings created on demand. Close match to an existing open item → warn and ask (no silent duplicates).

**Completing:** `complete todo: <text or partial match>` finds the open item. One match → mark `[x]`, append `(done YYYY-MM-DD)`, cut from `todos.md`, paste under the same heading in `todos-archive.md`. Multiple matches → ask. No match → say so.

**Reopening:** `reopen todo: <text or partial match>` reverses completion: cut from archive, strip `(done …)`, restore `[ ]`, re-insert under the same heading in `todos.md`.

**Queries (not journaled):** `todos`, `todos for <project>`, `todos for none`, `done todos [this week|<Www>|for <project>]`.

**Auto-extraction from meetings (conservative):** when distilling a transcript whose text explicitly signals commitment ("action items", "TODO", "will do", "I'll …"), propose extracted items and ask once: *"Transcript lists N action items: [list]. Add as todos for <project>? y / n / partial"*. Never auto-create without confirmation. If no such signals, don't extract.

**`show project:` composition:** render the hub file, then append that project's open todos (read from `todos.md`) as a "Todos" section in the output. The hub file on disk is never mutated to mirror todos.

## Project hub template

Used when `new project: <name>` is invoked:

```markdown
---
title: <Project name>
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: project
project: <slug>
tags: []
status: active
---

# <Project name>

## Summary
<one paragraph, updated as understanding grows>

## Entries
<forward links to meetings, snippets, notes, commands as they are filed>

## Open questions
<running list, items removed when resolved>

## Decisions
<dated one-liners>
```

## Design decisions and rationale

A condensed log of the choices made, for future revisions:

| Decision | Choice | Reason |
|---|---|---|
| Organization axis | Hybrid (type folders + project hubs) | Deterministic ingestion + readable per-project view |
| Project creation | Explicit registry, ask when unclear | Avoid silent misfiling; smooth ingestion for obvious matches |
| Raw content | Verbatim + distilled, linked | Preserve ground truth, keep retrieval useful |
| Journal granularity | One file per day | Matches mental retrieval model, keeps files scannable |
| Bootstrap state | Separate index files | Stable rules in CLAUDE.md, volatile state elsewhere |
| Linking | Relative markdown, forward-only | Works anywhere, no maintenance burden |
| Naming | Mixed — dated for time-bound, slug for evergreen | Matches how each folder is queried |
| Week boundary | Monday–Sunday (ISO 8601) | Matches `YYYY-Www` and `date +%V` |
| Weekly summary | User-requested | User controls when synthesis happens |
| Query scope | Plausibly-personal heuristic | Librarian behavior: check files for personal questions, general answers otherwise |
| Inbox | Yes, for ambiguous or low-value pastes | Avoid interruption; triage on demand |
| Frontmatter | Minimal (title/dates/type/project/tags/source/status) | Extra fields go stale |
| Tags | Freeform, tracked in `TAGS.md` | Flexibility + synonym prevention |
| Updates | New-file for time-bound, update-in-place for evergreen, never silent | Preserve history where it matters, refine where it doesn't |
| Commands | Canonical phrases with loose variants | Predictable contract + natural interaction |
| Archiving | Flag only, no file movement | Avoid breaking links |
| Todo storage | Single `todos.md` grouped by project | Both "all" and "per project" queries stay cheap |
| Todo metadata | Created date always; `!priority` and `due:` optional, user-stated only | Readable file; format grows with demand |
| Todo completion | Cut to `todos-archive.md` + journal | Keep open list focused; preserve history |
| Todo journaling | Writes journal, reads don't | State changes matter; query noise doesn't |
| Meeting → todos | Auto-extract only when text signals commitment, always ask first | Action items shouldn't slip through; aspirational mentions shouldn't auto-become todos |
| Weekly summary + todos | "Completed this week" section only | Retrospective value; open lists would go stale |
| Hub + todos | Composed at query time in `show project:`, never mirrored to hub file | Avoid sync bugs, match backlink-free precedent |
| Archive + todos | No change to `todos.md` | An archived project can still have open work |

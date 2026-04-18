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
    ├── goals.md                      # user-managed goals (freeform)
    ├── todos.md                      # open todos, grouped by project
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
    ├── reading/
    │   └── <slug>.md                 # evergreen external sources (articles, papers, books)
    ├── exports/
    │   └── <slug>-YYYY-MM-DD.md      # single-file project dumps (tracked in git)
    ├── people/
    │   ├── INDEX.md                  # all people, flat list
    │   └── <slug>.md                 # one hub per person
    ├── projects/
    │   ├── INDEX.md                  # Active / Archived project registry
    │   └── <slug>.md                 # one hub per project
    └── journal/
        ├── INDEX.md                  # pointer to current week + open threads
        ├── YYYY-MM-DD.md             # one file per day (append-only within day)
        ├── weekly/YYYY-Www.md        # Monday–Sunday summaries, on request
        ├── monthly/YYYY-MM.md        # calendar-month summaries, on request
        └── quarterly/YYYY-Qn.md      # calendar-quarter summaries, on request
```

`knowledge/.git/` is created the first time `save` is invoked.

Rationale for hybrid: type-first storage keeps ingestion deterministic (a transcript always goes to `meetings/`). Project hub files provide a readable per-project view without duplicating data. The two axes meet through forward links.

## Frontmatter schema (minimal)

Every entry begins with:

```yaml
---
title: <human-readable>
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: meeting | snippet | command | note | raw | project | journal | person | reading
project: <slug> | none | [slug-a, slug-b]    # single preferred; list only when genuine
tags: [freeform, lowercase-hyphenated]
source: raw/<file>.md                        # distilled entries only
status: active | archived                    # project hubs only
attendees: [<slug>, <slug>]                  # meetings only, optional
url: <url>                                   # reading only, optional
author: <name>                               # reading only, optional
role: <short role>                           # person only, optional
relationship: <short context>                # person only, optional
---
```

Extra fields go in the body, not the frontmatter. Frontmatter is for routing and retrieval only.

## Behavioral rules

### 1. Bootstrap on every session
Read order: `CLAUDE.md` → `knowledge/projects/INDEX.md` → `knowledge/journal/INDEX.md` → current-week journal files. Then ready.

### 2. Ingestion — verbatim + distilled
- Every paste is saved verbatim to `knowledge/raw/YYYY-MM-DD-HHMM-<slug>.md`.
- A distilled companion is written to the appropriate type folder (`meetings/`, `snippets/`, `commands/`, `notes/`, `reading/`) with `source:` pointing to the raw file.
- The relevant project hub gets a forward link to the distilled entry (not the raw).
- **Meetings:** if the transcript explicitly names attendees, propose them — *"Attendees detected: Alice, Bob. Add to meeting frontmatter? y / n / edit"* — before writing. Never auto-add.
- **Articles / papers / books:** distilled type is `reading`; `url:` and `author:` populated when present.

### 3. Project assignment — explicit registry + ask when unclear
- `knowledge/projects/INDEX.md` is authoritative. Projects only exist if listed there.
- Obvious match → file directly, state what was done.
- Substantial content, ambiguous project → ask once, then file.
- Low-value or ambiguous small paste → `knowledge/inbox/` with timestamped filename; mention it. User runs `triage inbox` periodically.
- `new project: <name>` is the only way a project gets added to `INDEX.md`.

### 4. Updates — new-file vs update-in-place by folder
- Time-bound folders (`meetings/`, `journal/`, `raw/`, `inbox/`): always new file.
- Evergreen folders (`snippets/`, `commands/`, `notes/`, `reading/`, `projects/`, `people/`): update in place, bump `updated:`, state explicitly which file is being overwritten (never silent).
- If an evergreen update meaningfully changes meaning (not just refinement), ask whether to update or create a sibling.

### 5. Linking — relative paths, forward-only
- Standard markdown: `[text](../path/to/file.md)`. No wikilinks.
- Forward links only. Backlinks are not maintained; Claude greps on demand when asked.
- Project hubs link to their entries; entries link back to their hub.

### 6. Journal — append immediately, one file per day
Every interaction that touches the knowledge base (ingest, answer-from-KB, librarian action, todo write) appends an entry to `knowledge/journal/YYYY-MM-DD.md` *before* replying to the user.

Entry format:
```markdown
## HH:MM — <short verb-phrase title>
- project: <slug> | none
- added/updated: <file path(s)>    # omit if none
- searched: <folders or files>     # only for queries
- note: <one-line summary of what happened>
```

Purely generic questions (no reference to stored content) are not journaled. Read-only commands are not journaled: `todos`, `todos for …`, `done todos`, `deadlines`, `brief me`, `show goals`, `show project:`, `show person:`, `stats`, `health`, `what have I read about`, `who do I meet with most`, `what did I work on`.

### 7. Weekly summary — user-requested only
- Week = Monday to Sunday, ISO 8601 (`YYYY-Www`).
- Triggered by `weekly summary` (current week) or `weekly summary <Www>` / `weekly summary for <date>`.
- Claude reads all `journal/YYYY-MM-DD.md` files in the target week, synthesizes into `journal/weekly/YYYY-Www.md` with sections: *projects touched*, *key decisions*, *artifacts added*, *completed this week*, *progress on goals* (only if `goals.md` exists), *open questions / threads*.
- "Completed this week" is built by filtering `todos-archive.md` entries by `(done YYYY-MM-DD)` within the target week, grouped by project. Omit the section if nothing was completed. Open todos are **not** snapshotted — they would go stale; use live `todos` queries for current state.
- "Progress on goals" reads `goals.md` and lists which goals saw activity in the week (journal mentions + hub updates) and which did not. Omit the section entirely when `goals.md` is missing.
- Never auto-generated. If the target file already exists, ask before overwriting.

### 8. Query heuristic — search KB when plausibly personal
Search `knowledge/` first if the question:
- references a known project name (from `projects/INDEX.md`) or person name (from `people/INDEX.md`),
- uses possessives ("my", "our", "the team's", "we", "I"),
- is time-bound ("yesterday", "last week", "what did I"),
- names a person, meeting, or artifact by name.

Otherwise answer from general knowledge without a search. If a search is performed and returns empty: say so, then offer a general answer — *"Nothing in your notes on this. Generally..."*.

### 9. Archiving — flag only, no moving
- `archive project: <slug>` sets `status: archived` in the hub's frontmatter and moves the line in `projects/INDEX.md` from Active to Archived.
- Hub file does not move. No links break.
- Archived projects remain searchable; excluded only from default project suggestions.
- `todos.md` is **not** touched. Open todos for an archived project stay visible via `todos` / `todos for <slug>`. User completes or reopens manually.
- `rename project: <old> -> <new>` also renames the `## <old>` heading in both `todos.md` and `todos-archive.md`.
- No auto-archive. Archival is an explicit user action.
- People have no archive concept.

### 10. Tags — freeform, tracked in `knowledge/TAGS.md`
- `TAGS.md` is a flat list with usage counts, updated automatically when new tags appear.
- Before inventing a tag, Claude scans `TAGS.md` and reuses an existing one if a synonym exists.

### 11. Canonical command phrases
Loose variants accepted; these are the documented contract.

**Projects & content**
- `what can I say` — print this list
- `new project: <name>` / `archive project: <slug>` / `rename project: <old> -> <new>`
- `show project: <slug>` — hub contents plus open todos
- `link: <file-a> <-> <file-b>`
- `triage inbox`
- `what did I work on <today|this week|on <project>>`

**People & reading**
- `new person: <name>` / `show person: <slug>` / `who do I meet with most`
- `save link: <url>` / `save link: <url> for <project>`
- `what have I read about <topic>`

**Todos**
- `add todo: <text>` / `add todo for <project>: <text>`
- `todos` (all open, grouped) / `todos for <project>` / `todos for none`
- `complete todo: <text or partial match>` / `reopen todo: <text or partial match>`
- `done todos [this week|<Www>|for <project>]`
- `deadlines`

**Goals & planning**
- `show goals` / `edit goals`
- `brief me`

**Summaries**
- `weekly summary [<Www or date>]`
- `monthly summary [<YYYY-MM>]`
- `quarterly summary [<YYYY-Qn>]`

**Maintenance**
- `health`
- `stats [for <project>|this week|this month|<YYYY-MM>]`
- `export project: <slug>`
- `save`

### 12. First-run bootstrap
If `TAGS.md`, `projects/INDEX.md`, or `journal/INDEX.md` is missing, Claude creates it with an empty scaffold on first use. These are the only implicit writes at session start. Everything else is lazy-scaffolded on first need:

- `todos.md` / `todos-archive.md` → on first `add todo` / `complete todo`
- `people/INDEX.md` → on first `new person`
- `goals.md` → on first `show goals` / `edit goals`
- `knowledge/.git/` → on first `save`
- `journal/weekly/`, `journal/monthly/`, `journal/quarterly/`, `exports/` directories → on first write into them

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

**Queries (not journaled):** `todos`, `todos for <project>`, `todos for none`, `done todos [this week|<Www>|for <project>]`, `deadlines`.

**Auto-extraction from meetings (conservative):** when distilling a transcript whose text explicitly signals commitment ("action items", "TODO", "will do", "I'll …"), propose extracted items and ask once: *"Transcript lists N action items: [list]. Add as todos for <project>? y / n / partial"*. Never auto-create without confirmation. If no such signals, don't extract.

**`show project:` composition:** render the hub file, then append that project's open todos (read from `todos.md`) as a "Todos" section in the output. The hub file on disk is never mutated to mirror todos.

### 14. Deadlines

`deadlines`: read `todos.md`, filter to lines with `due:YYYY-MM-DD`, sort ascending. Group by:

- **Overdue** (due before today)
- **This week** (today through the upcoming Sunday)
- **Later**

Each line: `[<project>] <text> (due YYYY-MM-DD)`. Read-only, not journaled.

### 15. People

Each person has a hub at `knowledge/people/<slug>.md` with `type: person` and optional `role:` and `relationship:` frontmatter fields (single-line strings). `people/INDEX.md` lists every person flat — no Active/Archived split.

**Person hub template:**

```markdown
---
title: <Name>
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: person
role: <short role, e.g. "backend lead at Acme">
relationship: <short context, e.g. "collaborator on auth-refactor">
tags: []
---

# <Name>

## About
<short context>

## Entries
<forward links to meetings/entries mentioning them>

## Notes
<running notes>
```

**Meeting attendees:** optional `attendees: [<slug>, <slug>]` frontmatter on meeting files. Only populated when the transcript explicitly lists attendees and the user confirms (see Ingestion rule).

**Commands:**
- `new person: <name>` — create hub, append to `people/INDEX.md`.
- `show person: <slug>` — render hub, then grep `meetings/` for `attendees:` containing the slug and list linked meetings as backlinks. Read-only.
- `who do I meet with most` — aggregate `attendees:` frontmatter across `meetings/`, return slugs ranked by frequency. Read-only.

### 16. Reading log

Evergreen `knowledge/reading/<slug>.md` with `type: reading` and optional `url:` and `author:` frontmatter. Distinct from `notes/` because the source is external — update in place when revising your notes on the same source; do not create siblings.

**Format:**

```markdown
---
title: <Article/paper/book title>
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: reading
url: <url-or-empty>
author: <author-or-empty>
project: <slug> | none
tags: []
---

# <Title>

<notes, quotes, reactions>
```

**Ingesting pasted articles:** normal ingestion path with distilled type `reading`.

**Query:** `what have I read about <topic>` — grep `reading/` bodies + frontmatter for the topic; return ranked matches with file paths. Read-only.

### 17. Bookmark ingest (`save link:`)

`save link: <url>` or `save link: <url> for <project>`:

1. WebFetch the URL with a short summarization prompt.
2. Create `raw/YYYY-MM-DD-HHMM-<slug>.md` with the raw fetched content (or a stub noting fetch failure, preserving the URL).
3. Create `reading/<slug>.md` distilled, `source:` points to raw, `url:` populated.
4. Project assignment follows the standard rule.
5. Journal it.
6. If WebFetch fails, still create the `reading/` stub with `url:` only and ask the user to paste notes/content.

### 18. Goals

`knowledge/goals.md` is user-managed freeform markdown. Example scaffold (created by `show goals` / `edit goals` when the file is missing):

```markdown
# Goals

## 2026
- <year-level goal>

## 2026 Q2
- <quarter-level goal>

## 2026 <Month>
- <month-level goal>
```

**Commands:**
- `show goals` — print `goals.md` (creates the scaffold on first run). Read-only.
- `edit goals` — when the user describes changes, Claude proposes diffs and asks for confirmation. **Never auto-edit `goals.md`.**

Weekly / monthly / quarterly summaries include a "Progress on goals" section only when `goals.md` exists.

### 19. Morning briefing (`brief me`)

Composed read-only command, not journaled. Output structure:

```
## Today — YYYY-MM-DD

### Overdue (N)
- [<project>] <text> (due YYYY-MM-DD)

### Open todos
<grouped by project, `none` first>

### Open threads
<from journal/INDEX.md>

### Yesterday — YYYY-MM-DD
<tail of last 3–5 journal entries>
```

If yesterday's journal file is missing (first day back after a gap), scan the last existing journal file and label it with its actual date.

### 20. Monthly / quarterly summary

Calendar-based boundaries. Month = 1st to last day of the calendar month. Quarter = Jan–Mar (Q1), Apr–Jun (Q2), Jul–Sep (Q3), Oct–Dec (Q4).

- `monthly summary [<YYYY-MM>]` → writes `journal/monthly/YYYY-MM.md`. Synthesizes from weekly digests within the month (plus any daily journals not yet captured in a weekly).
- `quarterly summary [<YYYY-Qn>]` → writes `journal/quarterly/YYYY-Qn.md`. Synthesizes from monthly summaries in the quarter.

Same section set as weekly: *Projects touched*, *Key decisions*, *Artifacts added*, *Completed this month/quarter*, *Progress on goals*, *Open questions / threads*. "Completed" filters `todos-archive.md` by date range.

Never auto-generated. If the target file already exists, ask before overwriting.

### 21. Health check

`health`: read-only scan. Reports but never modifies. Sections:

- **Stale inbox** — `inbox/` items older than 7 days
- **Aging todos** — open todos older than 30 days
- **Quiet projects** — active projects with no journal mentions in the last 60 days
- **Orphan entries** — distilled files in `meetings/`, `snippets/`, `commands/`, `notes/`, `reading/` that no project hub links to *and* are not `project: none`
- **Tag synonym candidates** — tags in `TAGS.md` whose slugs differ only in hyphenation, pluralization, or near-string-distance (e.g. "backend" vs "back-end")
- **Possible duplicate entries** — files in the same folder with high title/content similarity (folded-in dedup)

Thresholds (7 / 30 / 60 days) are documented defaults; adjust by editing `CLAUDE.md`. No config file.

### 22. Export

`export project: <slug>` → produces `knowledge/exports/<slug>-YYYY-MM-DD.md`. Overwritten on re-export the same day. Exports folder is tracked in git like everything else.

Structure:
1. Hub body (frontmatter stripped)
2. `## Entries` — each linked file's content as subsections (distilled only, never raw)
3. `## Open todos` — from `todos.md`
4. `## Completed todos` — from `todos-archive.md`

### 23. Stats

`stats [for <project>|this week|this month|<YYYY-MM>]`: read-only aggregation. Output:

- Entries added, by type (meetings, snippets, commands, notes, reading)
- Todos opened / completed
- Top 5 tags used
- Longest-standing open question (oldest line from project hubs' "Open questions" sections)

Filters: `for <project>` scopes via forward-link graph + `project:` frontmatter; time-scope filters by `created:` / `updated:`.

### 24. Git versioning and `save`

**Setup:** the first time `save` is invoked, if `knowledge/.git/` is missing, run `git init` in `knowledge/`, write an empty `.gitignore` (we track everything), and make an initial commit of the tree.

**`save`:** stage all changes in `knowledge/`, commit with an auto-generated message (one-line summary derived from today's journal tail). No confirmation, no diff prompt. Output: `git diff --stat` for the session + short commit hash.

**Auto-`save` at session end:** when the user signals end-of-session ("bye", "thanks that's all", "ttyl", closing language), run `save` automatically. If ambiguous, ask once: *"Save session before I stop?"*.

**No inline commits mid-session.** All writes accumulate in the working tree; one commit per session via `save`. **Never force-push, amend, or rewrite history.**

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
<forward links to meetings, snippets, notes, commands, reading as they are filed>

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
| Frontmatter | Minimal core + per-type optional fields | Extra fields go stale; optional fields stay off unless needed |
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
| People | Dedicated hubs + optional `attendees:` on meetings | First-class entity; enables "who do I meet with most" without heuristic scraping |
| Attendee extraction | Ask before writing, only when transcript explicitly lists attendees | Avoid false positives from casual name mentions; keep frontmatter trustworthy |
| Reading log | Separate `reading/` folder, not `notes/` | External source + `url:`/`author:` fields warrant their own type and query surface |
| Bookmark ingest | Fetch → raw + distilled reading entry | Preserves source even if the URL rots; fits existing verbatim+distilled pattern |
| Goals | Freeform user-managed file, never auto-edited | Goals are personal; Claude proposes, user commits |
| Goals in summaries | "Progress on goals" section, only when `goals.md` exists | Zero-config when absent, surfaces drift when present |
| Deadlines | Derived from `todos.md` `due:` fields, grouped overdue/this-week/later | No second storage; `todos.md` remains the single source of truth |
| Morning briefing | Composed read: overdue + open todos + threads + yesterday's tail | One command to "get oriented"; cheaper than multiple queries |
| Monthly / quarterly summary | Calendar boundaries, synthesize from the layer below | Matches how people think about time; compounds the weekly effort |
| Health check | Read-only, report stale/aging/quiet/orphan/dup, no auto-fix | Surface drift; user decides what to prune |
| Thresholds (7/30/60 days) | Constants documented in `CLAUDE.md` | No config file needed; editable in one place |
| Export | Concatenated single file per project per day, in `exports/` tracked in git | Easy to share/archive; reproducible; no external tool |
| Stats | Read-only aggregation filterable by project / time | Gives shape to activity without requiring a summary ritual |
| Git integration | One commit per session via explicit `save` (+ end-of-session auto) | Keeps history meaningful; avoids noisy per-write commits |
| Git scope | `knowledge/` only; empty `.gitignore` (track everything) | Versions everything that matters, no exclusion edge cases |
| History policy | No force-push, amend, or rewrite | Memory is append-only; loss would be silent and bad |

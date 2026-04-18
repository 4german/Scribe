# Scribe: A Personal Knowledge Base 

Open Claude in a directory with this `CLAUDE.md` file and you'll have your own personal librarian. Paste notes, meeting transcripts, todos and they will be stored away, linked together, and available for you to fetch. You can even get weekly summaries of all the things you did.

All the data ends up in markdown files (tracked by git) so there's no harm in restarting Claude.

---

## Directory layout

```
knowledge/
├── TAGS.md                    # tag vocabulary with usage counts
├── goals.md                   # user-managed goals (freeform)
├── todos.md                   # open todos, grouped by project
├── todos-archive.md           # completed todos
├── inbox/                     # unclassified pastes
├── raw/                       # verbatim pastes: YYYY-MM-DD-HHMM-<slug>.md
├── meetings/                  # distilled: YYYY-MM-DD-<slug>.md
├── snippets/                  # evergreen code: <slug>.md
├── commands/                  # evergreen CLI recipes: <slug>.md
├── notes/                     # evergreen concepts: <slug>.md
├── reading/                   # evergreen external sources: <slug>.md
├── exports/                   # single-file project exports (tracked in git)
├── people/
│   ├── INDEX.md
│   └── <slug>.md              # per-person hub
├── projects/
│   ├── INDEX.md               # Active / Archived registry
│   └── <slug>.md              # per-project hub
└── journal/
    ├── INDEX.md               # current week, open threads
    ├── YYYY-MM-DD.md          # daily journal, append-only
    ├── weekly/YYYY-Www.md
    ├── monthly/YYYY-MM.md
    └── quarterly/YYYY-Qn.md
```

`knowledge/.git/` is created the first time `save` is invoked.

---

## Canonical commands

There are some commands that you can use to quickly interact with your librarian. 

**`what can I say`** will print out this list

**Projects & content**
- `new project: <name>` / `archive project: <slug>` / `rename project: <old> -> <new>`
- `show project: <slug>` — hub contents plus open todos
- `link: <file-a> <-> <file-b>` — force a cross-link
- `triage inbox`

**People & reading**
- `new person: <name>` / `show person: <slug>` / `who do I meet with most`
- `save link: <url>` / `save link: <url> for <project>` — fetch and file
- `what have I read about <topic>`

**Todos**
- `add todo: <text>` / `add todo for <project>: <text>`
- `todos` / `todos for <project>` / `todos for none`
- `complete todo: <text or partial match>` / `reopen todo: <text or partial match>`
- `done todos [this week|<Www>|for <project>]`
- `deadlines` — open todos with `due:` dates, sorted

**Goals & planning**
- `show goals` / `edit goals`
- `brief me` — morning briefing

**Summaries**
- `weekly summary [<Www or date>]`
- `monthly summary [<YYYY-MM>]`
- `quarterly summary [<YYYY-Qn>]`
- `what did I work on <today|this week|on <project>>`

**Maintenance**
- `health` — stale/orphan report
- `stats [for <project>|this week|this month|<YYYY-MM>]`
- `export project: <slug>` — single-file project dump
- `save` — commit session changes to `knowledge/.git`
- `what can I say` — print this list

---

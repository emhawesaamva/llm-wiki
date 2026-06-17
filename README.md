# LLM WIKI

The **Karpathy LLM Wiki** is a note-taking pattern popularized by Andrej Karpathy for using a local folder of markdown files as persistent memory for an LLM.

The core idea: instead of re-explaining your context every conversation, you maintain a structured set of `.md` files that get read into the LLM's context on demand. The LLM reads, reasons over, and writes back to those files — so knowledge accumulates over time.

**Key principles:**

- Flat or lightly nested folder of markdown files
- An `index.md` as the entry point / table of contents
- Files are human-readable and human-editable
- LLM reads relevant files at the start of a session
- LLM writes updates back after sessions

**Analogy:** Think of it like a shared Doc that both you and the LLM can read and edit — except it lives locally, you own it entirely, and it grows smarter the more you use it.

**Credit:** [Karpathy's GitHub](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) — the original pattern, published as a single markdown file.

**Explainer:** [MindStudio — What Is the Karpathy LLM Wiki Pattern?](https://www.mindstudio.ai/blog/karpathy-llm-wiki-knowledge-base-pattern)

# new-wiki — Bootstrap a Claude-Maintained Wiki

This implementation of the LLM WIKI has been developed for users at *AAMVA*.
These files contain everything needed to stand up a fresh, personal, Claude-maintained 
knowledge base. You source and direct; Claude writes and maintains the pages.

## What's in this folder

| File            | Role                                                                 |
| --------------- | -------------------------------------------------------------------- |
| `add-wiki.md`   | Setup orchestrator — the entry point. Run this first.                |
| `add-claude.md` | The generic `CLAUDE.md` template (schema, conventions, skill table). |
| `add-skills.md` | Build instructions for the four skill files.                         |
| `README.md`     | This file.                                                           |

The three `add-*` files are consumed during setup and then archived into `skills/archive/`.

---

## How to start

### 1. Prerequisites
- **Claude Code or Claude Desktop with Code** (or another agent that can read/write files). This is
  the only hard requirement — querying, ingest, and the main lint checks all run through Claude's
  built-in tools and PowerShell.
- An **empty folder** for the wiki (or a brand-new one). The setup refuses to touch a
  folder that already holds a wiki — see the guardrails in `add-wiki.md`.

### 2. Place the setup files
Copy `add-wiki.md`, `add-claude.md`, and `add-skills.md` into the target wiki root (the folder
that will *become* the wiki). The `README.md` can come along too.

### 3. Run the setup
Open the target folder in Claude Code and tell Claude:

> Read `add-wiki.md` and set up the wiki here.

Claude will:
1. **Safety preflight** — confirm the folder is empty / a new wiki, or abort.
2. **Ask you** for the wiki root path (absolute) and an optional owner label.
3. **Build the structure** from `add-skills.md` — folders, operational stubs, and the four
   skill files, substituting the real path for every `<WIKI_ROOT>` token.
4. **Write `CLAUDE.md`** from `add-claude.md`.
5. **Archive** the three `add-*` files into `skills/archive/`.
6. **Verify** — runs `skills/wiki-lint.ps1` once; a fresh wiki reports `TOTAL PAGES: 0`.

The process is **create-only**: it never overwrites, edits, or deletes existing content.

### 4. Resulting layout
```
<WIKI_ROOT>/
├── CLAUDE.md          ← schema, conventions, skill table
├── index.md           ← catalog of all pages
├── todo.md            ← personal to-do list
├── meta/              ← log.md, done.md, ingest-progress.md
├── projects/          ← one page per workstream
├── entities/
│   ├── people/
│   └── apps/
├── concepts/          ← architecture, processes, recurring topics
├── sources/           ← ingest inbox
│   └── processed/     ← files land here after ingest
└── skills/            ← wiki-read-query, source-ingest, lint, wiki-lint.ps1
    └── archive/       ← the original add-* setup files
```

---

## How to use the wiki

There is **no special command syntax** — you just talk to Claude in plain language. The phrases
below are the triggers each skill listens for (defined in the skill table in `CLAUDE.md`); Claude
reads the matching skill file and does the work **inline** in the session.

### Quick reference

| Say something like…                                       | What Claude does                                                                                               | Skill             |
| --------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- | ----------------- |
| *"What's on my plate?"*, *"What are my tasks?"*           | Reads `todo.md` and lists active tasks                                                                         | `wiki-read-query` |
| *"What do we know about {topic}?"*                        | Reads `index.md`, then the relevant pages, and answers with `[[links]]`                                        | `wiki-read-query` |
| *"Add a to-do: {task}"*                                   | Adds the item to `todo.md` (or edit the file directly)                                                         | —                 |
| Mark a line `[x]` in `todo.md`                            | Moves it to `meta/done.md` and removes it from the active list                                                 | `wiki-read-query` |
| *"ingest"*, *"ingest sources"*, *"resume the doc ingest"* | Files everything in `sources/` onto wiki pages, moves originals to `processed/`, ends with an omissions report | `source-ingest`   |
| *"lint the wiki"*                                         | Health check (dry run) — reports issues, changes nothing                                                       | `lint`            |
| *"lint and fix"*                                          | Health check, then applies the safe fixes                                                                      | `lint`            |
| *"compare X and Y"*, *"summarize the decision on Z"*      | Answers, and offers to file the synthesis as a new page                                                        | `wiki-read-query` |

Each workflow is described in more detail below.

### Ask questions
Anything about your projects, people, apps, tasks, or domain context.
- *"What's on my plate?"* / *"What are my tasks?"* → reads `todo.md`
- *"What do we know about {topic}"* → reads `index.md`, then the relevant pages
- Skill: `skills/wiki-read-query.md`

### Capture tasks
- Add to-dos in `todo.md`. Mark a line `[x]` when done — Claude moves it to `meta/done.md`
  and removes it from the active list.

### Ingest documents
Drop files (`.docx`, `.pdf`, `.xlsx`, `.html`, `.txt`, `.md`) into `sources/`, then say:
- *"ingest"* / *"ingest sources"* / *"resume the doc ingest"*

Claude extracts the content, files it onto the right wiki pages (additively — it never drops
existing content), moves the processed file into `sources/processed/`, logs the run, and ends
with an **omissions report** of what it deliberately left out.
- Skill: `skills/source-ingest.md`

### Health-check (lint)
- *"lint the wiki"* — dry run: reports orphan pages, broken links, stale pages, missing
  frontmatter, index gaps, and encoding corruption. Makes **no** changes.
- *"lint and fix"* — applies only the safe fixes (cross-references, index entries, frontmatter,
  encoding repair).
- Skill: `skills/lint.md`

### Conventions worth knowing
- **Frontmatter** on every page (`type`, `status`, `last_updated`, `aliases`, …) — see `CLAUDE.md`.
- **`[[wiki-links]]`** (Obsidian style) for cross-references between pages.
- **`reviewed: true`** on a page tells Claude to append only — never overwrite your manual edits.
- **Encoding:** all files are **UTF-8 without BOM**. Never edit wiki files with Notepad or
  PowerShell `Set-Content`/`Out-File` — they corrupt em dashes and arrows.

---

## Using Obsidian (optional)

The wiki is plain markdown, so [Obsidian](https://obsidian.md) works as a graphical front end —
useful for browsing, backlinks, and the graph view alongside Claude's editing.

### Setup
1. Install Obsidian.
2. **Open folder as vault** → select your `<WIKI_ROOT>`. No conversion needed; Obsidian reads
   the existing files in place.

### Why it fits
- **`[[wiki-links]]`** are native Obsidian syntax — links are clickable and backlinks appear
  automatically.
- **YAML frontmatter** is read as page properties.
- **Aliases** in frontmatter make pages findable by their shorthand in quick-switch and search.
- The **graph view** visualizes how projects, people, apps, and concepts connect.

### Working alongside Claude
- Editing a page by hand? Add `reviewed: true` to its frontmatter so Claude appends rather than
  overwrites on the next ingest.
- Obsidian saves as UTF-8, so it's a safe editor for wiki files (unlike Notepad).
- Obsidian's per-machine state lives in `.obsidian/` — the lint script already ignores it.

Obsidian is purely optional. The wiki is fully usable through Claude alone; Obsidian just adds a
visual, navigable layer on top of the same files.

# Wiki — Schema & Conventions

## Purpose
This wiki is a persistent knowledge base maintained by Claude. The owner sources and directs; Claude writes and maintains. Replace `<WIKI_ROOT>` anywhere it appears with the absolute path to this wiki's root directory.

## Directory Structure
```
<WIKI_ROOT>/
├── CLAUDE.md          ← this file (schema, conventions, workflows)
├── index.md           ← catalog of all pages with one-line summaries
├── todo.md            ← personal to-do list
├── meta/              ← wiki operational records (log, archive, tracking)
│   ├── log.md         ← append-only chronological record
│   ├── done.md        ← completed items archived from todo.md
│   └── ingest-progress.md ← doc ingestion progress tracker
├── projects/          ← one page per active workstream
├── entities/
│   ├── people/        ← team members, stakeholders
│   └── apps/          ← applications and platforms
├── concepts/          ← architecture patterns, processes, recurring topics
├── sources/           ← ingest inbox: drop docs here (raw files git-ignored; .md/.txt summaries tracked)
│   └── processed/     ← files moved here after successful ingest
└── skills/            ← workflow guides for Claude
    ├── wiki-read-query.md
    ├── source-ingest.md
    ├── lint.md
    └── wiki-lint.ps1
```

## Skills
Before performing these workflows, read the relevant skill file. **Check specific skills first — fall back to `wiki-read-query.md` only if no specific skill matches.**

| Workflow | Skill File | Trigger Phrases |
|----------|-----------|-----------------|
| Ingesting documents from `sources/` (or other doc folders) | `skills/source-ingest.md` | "ingest", "ingest sources", "ingest the docs", "resume the doc ingest", "process sources/", files added to `sources/` |
| Wiki health check / lint | `skills/lint.md` | "lint the wiki", "health check the wiki", "check the wiki for issues", "what's stale", "find orphan pages", "wiki cleanup" |
| Answering a question (generic fallback) | `skills/wiki-read-query.md` | any other question not matched above |

All workflows run **inline** directly from the skill files — read the relevant skill, then perform the work in the main session.

## Page Conventions
- All pages use markdown.
- Each page starts with a `# Title` and a one-line summary.
- Use `[[wiki-links]]` for cross-references (Obsidian format).
- Use YAML frontmatter on every page.
- Keep pages factual and terse. No fluff.

## Frontmatter Schema

### Projects
```yaml
---
type: project
status: active | complete | paused | at-risk
last_updated: YYYY-MM-DD
deadline: YYYY-MM-DD
sources: ["[[sources/filename]]"]
aliases: ["alternate name", "acronym"]
---
```

### Entities & Concepts
```yaml
---
type: entity | concept
status: active | legacy
last_updated: YYYY-MM-DD
sources: ["[[sources/filename]]"]
aliases: ["alternate name", "acronym"]
reviewed: true   # add this flag to protect a page from Claude overwriting on ingest
---
```

### Sources
```yaml
---
type: source
ingested: YYYY-MM-DD
origin: meeting | document | paste | upload
---
```

## Aliases
- Every entity and concept page must have at least one alias.
- Aliases should include: acronyms, alternate names, common shorthand.
- Examples: a product named "Example Service Management" → aliases: ["ESM"]; "Structured Query Language" → aliases: ["SQL"].
- Aliases enable finding pages by shorthand during queries and duplicate detection.

## Reviewed Flag
- Add `reviewed: true` to frontmatter on any page you have manually edited.
- Claude will not overwrite pages with `reviewed: true` during ingest — it will only append new information.

## Sources Tracking
- Every page updated during an ingest gets the source file added to its `sources: []` list.
- This creates a traceable record of what evidence supports each page.

## Filing Good Answers Back
When a query produces a valuable synthesis — a comparison, analysis, decision summary, or connection between topics — file it as a new wiki page rather than letting it disappear into chat history. Good candidates:
- A comparison you asked for (e.g., "compare option A vs B")
- An architectural analysis
- A decision and its rationale
- A connection between topics you hadn't documented

Create the page in the most appropriate directory (`concepts/`, `projects/`, etc.) with proper frontmatter, and add it to `index.md`. Append a `query` entry to `log.md`.

## Editing Rules
- Always read the full file before editing it.
- Always rewrite files from the confirmed read content — never from memory.
- Never write a partial file. Always write the complete file content.

## File Encoding
All wiki files are **UTF-8 without BOM**. Corrupted `â€"` / `â†'` / `â"œâ"€â"€` sequences are a sign that a tool wrote the file in Windows-1252 instead of UTF-8.

**Never use PowerShell `Set-Content` or `Out-File` to write wiki files** — both default to Windows-1252 in PowerShell 5.1 and will corrupt any em dashes, arrows, or box-drawing characters on the next save.

Safe write methods:
- **Preferred:** The `Write` or `Edit` tools (handle encoding correctly)
- **PowerShell (only if necessary):** `[System.IO.File]::WriteAllText($path, $content, [System.Text.UTF8Encoding]::new($false))`

If you ever need to manually edit a wiki file, use **VS Code** or **Obsidian** — not Notepad.

## Log Format
Each log entry starts with: `## [YYYY-MM-DD] TYPE | Title`
Types: `ingest`, `query`, `lint`, `update`, `scaffold`

## Owner
Owner — sources and directs.
Claude — wiki maintainer.

# Recreate Wiki Skills — Build Instructions for an AI Agent

This file instructs an AI agent to recreate a set of generic, reusable wiki **skill** files
in a brand-new wiki. Nothing here is tied to a specific person, organization, or machine.
Follow the steps in order.

---

## 0. Set the wiki root

Decide the absolute path of the wiki's root directory and use it consistently everywhere this
document says `<WIKI_ROOT>`. Examples:
- Windows: `C:\Users\<you>\Documents\my-wiki`
- macOS / Linux: `/home/<you>/my-wiki`

Throughout the files you create, replace every `<WIKI_ROOT>` token with that absolute path.

---

## 1. Create folders if they do not already exist

Before writing any file, make sure the following directory tree exists. Create any that are
missing (do not delete or overwrite anything that already exists):

```
<WIKI_ROOT>/
├── skills/                 ← the skill files created below live here
├── meta/                   ← operational records (log, archive, progress tracker)
├── sources/                ← document ingest inbox
│   └── processed/          ← documents are moved here after a successful ingest
├── projects/               ← one page per workstream
├── entities/
│   ├── people/             ← people pages
│   └── apps/               ← application / system pages
└── concepts/               ← architecture, processes, recurring topics
```

Also ensure these operational files exist (create them as empty or minimal stubs if missing —
the skills below read and append to them):

- `<WIKI_ROOT>/index.md`                  — catalog of all pages with one-line summaries
- `<WIKI_ROOT>/todo.md`                    — active personal task list
- `<WIKI_ROOT>/meta/done.md`               — archive of completed to-do items
- `<WIKI_ROOT>/meta/log.md`                — append-only chronological record
- `<WIKI_ROOT>/meta/ingest-progress.md`    — per-file ingest tracker

**Encoding requirement:** write every file as **UTF-8 without BOM**. Do not use PowerShell
`Set-Content`/`Out-File` (they default to a non-UTF-8 codepage and corrupt em dashes, arrows,
and box-drawing characters). Use a tool/method that writes UTF-8 directly.

---

## 2. File: `skills/wiki-read-query.md`

Write the following content verbatim to `<WIKI_ROOT>/skills/wiki-read-query.md`:

````markdown
# Wiki Read/Query Skill

## Purpose
Read the wiki before answering questions. Ensures answers are grounded in current, maintained
context rather than the model's memory.

## Trigger
Any question about projects, people, tasks, applications, architecture, or domain context the
wiki covers — **but check whether a more specific skill matches first.** Only fall back to this
skill if no specific skill applies.

## Wiki Location
`<WIKI_ROOT>`

## Critical Facts About This Wiki
- **The active task list lives here**: `todo.md` — personal tasks, follow-ups, action items.
- **Completed items live here**: `meta/done.md` — items archived from `todo.md` when checked off with `[x]`.
- **This is the most important source for domain information** — prefer wiki pages over training
  memory for anything specific to this wiki's domain.
- If the user asks "what do I have to do", "what's on my plate", "what are my tasks", or anything
  similar — read `todo.md` first.
- If the user marks something `[x]` in `todo.md`, it is complete — move it to `meta/done.md` and
  remove it from `todo.md`.

## Steps

### 1. Read index.md first
Always start here. One read shows everything in the wiki and where it lives.

### 2. Identify relevant pages
Scan the index summaries and match to the question:
- To-do / personal tasks → `todo.md`
- Completed tasks → `meta/done.md`
- Project questions → `projects/` pages
- People questions → `entities/people/` pages
- App/system questions → `entities/apps/` pages
- Architecture / process questions → `concepts/` pages

### 3. Handle [x] completed items in todo.md
When reading `todo.md`, if any lines are marked `[x]` (checked as done):
1. Read `meta/done.md`
2. Append each completed item to `meta/done.md` with format: `[YYYY-MM-DD] [source: todo] item text`
3. Rewrite `todo.md` without the completed lines
This keeps the active list clean and preserves history.

### 4. Read relevant pages
Fetch only the pages that match. Avoid reading the whole wiki.

### 5. Read meta/log.md if recency matters
If the question involves recent changes, status updates, or what happened lately, read
`meta/log.md`.

### 6. Answer with page references
Use `[[wiki-links]]` to reference pages in the response. If the answer is valuable and not already
captured, offer to file it as a new wiki page.

## Notes
- The wiki is the source of truth — prefer it over memory for anything domain-specific.
- `todo.md` is the active task list — always check it for task/priority questions.
- `meta/done.md` is the archive — items with `[x]` in `todo.md` go here, then are removed from `todo.md`.
- If something seems missing from the wiki, note it and offer to add it.
- Multiple relevant pages may apply — read them all before answering.
````

---

## 3. File: `skills/source-ingest.md`

Write the following content verbatim to `<WIKI_ROOT>/skills/source-ingest.md`:

````markdown
# Source / Document Ingest Skill

## Purpose
Process documents in the `sources/` folder (and any other document drop folders), extract
knowledge into the wiki, and keep `meta/ingest-progress.md` accurate.

## When to use
- "ingest", "ingest sources", "ingest the docs", "resume the doc ingest", "process sources/"
- Any time files have been added to `sources/`.

## Folder — inbox / processed model
`<WIKI_ROOT>/sources`
- **`sources/` root = inbox**: files here are unprocessed and pending ingest.
- **`sources/processed/` = done**: after a successful write-and-verify, each file is moved here.
- An (almost) empty `sources/` root means nothing is pending.

## Step 1 — Identify what's pending
1. List `sources/` **root** (ignore `processed/`) — those files are the work.
2. As a backstop, read `meta/ingest-progress.md` and confirm nothing in root is already recorded
   as done (a file could have been re-dropped). Trust the folder state first, the tracker second.
3. Report the pending files and the batch plan before processing.
   - *Note: `processed/` is the history. Don't re-ingest from it. If asked to "resume the doc
     ingest," check root first; only diff against the tracker if root is empty but you suspect a gap.*

## Step 2 — Extract text
Files in `sources/` are mostly `.docx`, `.pdf`, `.xlsx`, `.html`, `.txt`, `.md`.
- **DOCX**: unzip `word/document.xml`, strip tags (e.g. `unzip -p file.docx word/document.xml`).
- **XLSX**: unzip `xl/sharedStrings.xml` for cell text; sheets in `xl/worksheets/`.
- **PDF**: use the Read tool's `pages` param. Print-to-PDF files are often image-only with no
  extractable text — note that and rely on metadata.
- **HTML / TXT / MD**: read directly.
- **Large files (>10 MB)**: flag to the user before reading.
- Ignore boilerplate (cover pages, Lorem Ipsum, TBD stubs, copyright) — capture only substantive content.

## Step 3 — Batch size
Process a **maximum of 3 files per batch**. Pause after each batch, report, then continue.
Prevents content landing on the wrong page.

## Step 4 — Match to wiki pages
Read `index.md`, identify target page(s). Always read the full target page before writing — never
write from memory. If no good home exists, create a new page in the right directory with proper
frontmatter and add it to `index.md`.

## Step 5 — Write (read-modify-write, one page at a time)
- Read full target page → write/edit → verify. Do not batch-write multiple pages blind.
- **Additive edits only**: never drop existing list items or sections; append/extend. Flag
  ambiguity and ask.
- Respect `reviewed: true` — append only, never overwrite those pages.
- Add the source file to the page's `sources: []` list and bump `last_updated`.
- UTF-8 without BOM only. Use a UTF-8-safe write method — never PowerShell `Set-Content`/`Out-File`,
  which corrupt non-ASCII characters.

## Step 6 — Verify after every write
Read back (or trust the edit confirmation): file not truncated, ends where expected, key facts
correct, no cross-source leakage.

## Step 7 — Move the file to `processed/`
Only after write AND verify both pass, move the file from `sources/` root into `sources/processed/`.
Confirm the move succeeded.
- For a `skip` (duplicate / no value): still move it to `processed/` so the inbox empties — note
  the skip reason in the log.
- If a move fails, leave the file in place and report it.
- If the wiki is under git, raw binaries can stay ignored in `processed/` while `.md`/`.txt`
  summaries are tracked (see `.gitignore`).

## Step 8 — Log
- Append a per-file row to `meta/ingest-progress.md` (Status: `done` / `skip` / `partial`, file,
  wiki pages). Update its "Last updated" line.
- Append a session entry to `meta/log.md`:
  ```
  ## [YYYY-MM-DD] ingest | sources/ (<N> files)
  - Updated: <pages>
  - Created: <pages>
  - Skipped: <files and reasons>
  ```
- Update `index.md` for any new pages.

## Step 9 — Report what you OMITTED (required, every ingest)
End every ingest with an **Omissions Report** so the user has visibility into what was left out,
not just what was filed:
- **Files skipped** and why (duplicate, no wiki value, too large, image-only, etc.).
- **Within-document content not captured** — boilerplate (cover pages, copyright, signatures),
  placeholder/Lorem-Ipsum, redundant glossaries already in the wiki, TBD stubs, and any data
  deliberately compressed.
- **People/entities referenced but not given pages** (one-off contacts).
- **Attachments/links referenced but not ingested**.
- Anything **truncated or sampled** rather than read in full.
Keep it terse and scannable. The goal is that the user can spot anything that should have been
kept but wasn't.

## Edge Cases
- **Personal working / scratch files are never ingested.** If the wiki has daily-note or scratchpad
  files, do not read them for knowledge, do not update them during ingest, and do not flag them as
  pending.
- **Duplicate of an already-ingested source**: mark `skip` (duplicate) in the tracker; don't re-ingest.
- **Spreadsheet with no UI context**: pull cell text where possible; if structure isn't recoverable,
  mark `skip` and note why.
- **Image-only PDF**: capture metadata (author, date) and mark `partial`.
- **Credentials in a doc**: record only where consistent with existing pages (e.g. a credentials or
  runbook page), but call it out in the batch report.
````

---

## 4. File: `skills/lint.md`

Write the following content verbatim to `<WIKI_ROOT>/skills/lint.md`:

````markdown
# Wiki Lint Skill

## Purpose
Health-check the wiki. Find structural problems, stale content, missing links, knowledge gaps, and
source-to-page fidelity issues. Run periodically — after a big ingest batch, or whenever the wiki
feels disorganized.

## Trigger
- **Dry run (default):** "lint the wiki", "health check the wiki", "check the wiki for issues",
  "what's stale", "find orphan pages", "wiki cleanup"
- **Apply fixes:** "lint the wiki and fix", "lint and apply changes", "go ahead and fix the lint issues"

## Default Behavior: Dry Run
Unless the user explicitly asks to apply changes, always run in dry run mode:
- Run all checks (Parts 1–4)
- Produce a high-level summary report — issue counts and categories only, not the full text of
  proposed changes
- Make zero changes to the wiki
- End with a prompt: "Run 'lint and fix' to apply safe fixes."

## Apply Mode
Only when the user explicitly requests fixes:
- Run all checks
- Apply safe fixes (Part 5)
- Report what was changed

---

## Wiki Location
`<WIKI_ROOT>`

---

## Part 0: Encoding Check (always runs first)

Run this **before** any other check. Corrupted encoding is the highest-priority issue — it means
file content is damaged and other checks may produce false results.

Detection is built into `skills/wiki-lint.ps1` (the **ENCODING CORRUPTION** section). It scans every
page for the tell-tale mojibake signatures — a stray `â` followed by `€`, `"`, `†`, etc. — that
appear when a file is saved as Windows-1252 instead of UTF-8.

- If the script lists nothing under **ENCODING CORRUPTION**: encoding is clean — proceed to Part 1.
- If it lists files: report them as 🔴 **Encoding corruption** in the lint report. Do not
  proceed with other checks until this is resolved.

**In apply mode:** repair each flagged file with the **Write/Edit tools** — read the full file,
replace the corrupted sequences with the intended characters (e.g. `â€"` → `—`, `â†'` → `→`, the
`â"œâ"€` box-drawing runs → `├─`), and write the corrected file back as **UTF-8 without BOM**. Never
use PowerShell `Set-Content`/`Out-File` — they re-introduce the corruption. After writing, re-run the
lint script to confirm the file no longer appears under **ENCODING CORRUPTION**.

---

## Part 1: Automated Checks (PowerShell)

The script is saved at `skills/wiki-lint.ps1`. Run it directly:

```powershell
powershell -NoProfile -NonInteractive -File "<WIKI_ROOT>\skills\wiki-lint.ps1"
```

(See `skills/wiki-lint.ps1` for the script source. It reports total pages, encoding corruption,
orphan pages, broken links, missing frontmatter, stale pages, and index gaps.)

---

## Part 2: Source-to-Page Linkage Check

Read `meta/ingest-progress.md` and spot-check fidelity on a sample of pages — key facts match
their source, no cross-contamination. Prioritize pages that aggregate sensitive or high-value
information (credentials/runbook pages, pages assembled from many sources, frequently-updated
pages).

Flag fidelity issues but never auto-fix them — always flag for the user.

---

## Part 3: LLM Analysis Pass

Quick sampling pass:
- Concepts mentioned repeatedly but no page exists → flag as candidates
- Pages that clearly relate but don't link → flag as missing cross-references
- Facts that conflict across pages → flag as contradictions
- Thin topics → flag as data gaps

---

## Part 4: Dry Run Report (default output)

Keep this high-level — counts and categories only. Do not show proposed text changes.

```
## Wiki Lint Report — [YYYY-MM-DD] ⚠️ DRY RUN

| Category | Count |
|----------|-------|
| 🔴 Encoding corruption | N |
| 🔴 Broken links | N |
| 🔴 Fidelity issues | N |
| 🟡 Orphan pages | N |
| 🟡 Stale pages | N |
| 🟡 Missing frontmatter | N |
| 🟡 Index gaps | N |
| 🔵 Concepts without pages | N |
| 🔵 Missing cross-references | N |
| 🔵 Contradictions | N |

**Safe to auto-fix:** N items (encoding, broken links, index gaps, missing cross-references, frontmatter)
**Needs the user's review:** N items (fidelity issues, contradictions, orphans)

Run "lint and fix" to apply safe fixes.
```

---

## Part 5: Apply Fixes
*(Only runs when the user explicitly requests it.)*

Fix automatically where safe:
- Repair encoding corruption (replace the mojibake with the intended characters via the Write/Edit
  tools, then re-run the lint script to confirm)
- Add missing cross-references
- Add pages missing from `index.md`
- Update missing frontmatter fields

Do **not** auto-fix: fidelity issues, contradictions, orphans, stale pages.

### ⚠️ Write Method: read-modify-write the full file
- Read the full target page into context first
- Write the full file back — never append-style operations on existing pages
- Never use an append/partial write on a multi-line wiki page — it can silently overwrite content
- After every write, read the tail to confirm the file ends where it should and the size is reasonable
- If the write damaged the file, restore from the in-context copy immediately

After fixing, report what changed:
```
## Lint Fix Summary — [YYYY-MM-DD]
- Cross-references added: N
- Index entries added: N
- Frontmatter updated: N
- Issues flagged for review (not auto-fixed): N
```

---

## Part 6: Log

Append to `meta/log.md`:

```
## [YYYY-MM-DD] lint | [dry run / fixes applied]
- Total issues found: N
- Auto-fixed: N
- Flagged for review: N
```
````

---

## 5. File: `skills/wiki-lint.ps1`

Write the following content verbatim to `<WIKI_ROOT>/skills/wiki-lint.ps1`, then replace
`<WIKI_ROOT>` on the first line with the actual absolute path. Keep the file **ASCII-only** so
PowerShell 5.1 reads it correctly.

```powershell
$wiki = "<WIKI_ROOT>"
$wikiSlash = $wiki + "\"
$skipDirs = @('.obsidian', '.claude', 'email', 'sources', 'skills')
$skipFiles = @('CLAUDE.md', 'index.md', 'log.md', 'todo.md', 'done.md', 'ingest-progress.md', 'README.md')

$allPages = Get-ChildItem $wiki -Recurse -Filter "*.md" | Where-Object {
    $skip = $false
    foreach ($d in $skipDirs) { if ($_.FullName -like "*\$d\*") { $skip = $true } }
    if ($skipFiles -contains $_.Name) { $skip = $true }
    -not $skip
}

Write-Host "=== TOTAL PAGES: $($allPages.Count) ==="
Write-Host ""

# ENCODING CORRUPTION
# Mojibake from a file saved as Windows-1252 instead of UTF-8. Every corrupted run starts with
# U+00E2 (a-circumflex) followed by a high character. Needles are built by code point so this
# script stays ASCII-only.
Write-Host "=== ENCODING CORRUPTION ==="
$aHat = [char]0x00E2
$badNeedles = @(
    ($aHat + [char]0x0161),   # corrupted warning sign
    ($aHat + [char]0x0152),   # corrupted check mark
    ($aHat + [char]0x201D),   # corrupted box-drawing / curly quotes
    ($aHat + [char]0x20AC),   # corrupted em / en dash
    ($aHat + [char]0x2020)    # corrupted arrows
)
foreach ($page in $allPages) {
    $text = [System.IO.File]::ReadAllText($page.FullName, [System.Text.Encoding]::UTF8)
    foreach ($needle in $badNeedles) {
        if ($text.Contains($needle)) {
            $rel = $page.FullName.Replace($wikiSlash, '')
            Write-Host "  CORRUPTED: $rel"
            break
        }
    }
}
Write-Host ""

# Link map
$linkMap = @{}
$allLinks = @()
foreach ($page in $allPages) {
    $content = Get-Content $page.FullName -Raw -ErrorAction SilentlyContinue
    $links = [regex]::Matches($content, '\[\[([^\]|#]+)') | ForEach-Object { $_.Groups[1].Value.Trim() }
    $relPath = $page.FullName.Replace($wikiSlash, '').Replace('\', '/').Replace('.md', '')
    $linkMap[$relPath] = $links
    $allLinks += $links
}

$linkedTo = $allLinks | Group-Object | Select-Object -ExpandProperty Name

# ORPHANS
Write-Host "=== ORPHAN PAGES ==="
foreach ($page in $linkMap.Keys) {
    $name = Split-Path $page -Leaf
    $linked = $linkedTo | Where-Object { $_ -eq $page -or $_ -eq $name }
    if (-not $linked) { Write-Host "  ORPHAN: $page" }
}

# BROKEN LINKS
# Resolver uses ALL .md files in the wiki (including skipped ones) so links to
# excluded files (index, CLAUDE.md, skills/*) don't appear as broken.
Write-Host ""
Write-Host "=== BROKEN LINKS ==="
$allKnownFiles = Get-ChildItem $wiki -Recurse -Filter "*.md"
$allKnownNames = $allKnownFiles | ForEach-Object { $_.BaseName }
$allKnownPaths = $allKnownFiles | ForEach-Object { $_.FullName.Replace($wikiSlash, '').Replace('\', '/').Replace('.md', '') }
foreach ($page in $linkMap.Keys) {
    foreach ($link in $linkMap[$page]) {
        if ($link -match '^https?://') { continue }
        if ($link -match '\.[a-zA-Z0-9]+$' -and $link -notmatch '\.md$') { continue }
        $linkName = (Split-Path $link -Leaf) -replace '\.md$', ''
        $linkPath = $link -replace '\.md$', ''
        $resolved = ($allKnownPaths -contains $linkPath) -or ($allKnownNames -contains $linkName)
        if (-not $resolved) { Write-Host "  BROKEN: [[$link]] in $page" }
    }
}

# MISSING FRONTMATTER
Write-Host ""
Write-Host "=== MISSING FRONTMATTER ==="
foreach ($page in $allPages) {
    $content = Get-Content $page.FullName -Raw -ErrorAction SilentlyContinue
    if ($content -notmatch '(?s)^---.*?---') {
        $rel = $page.FullName.Replace($wikiSlash, '')
        Write-Host "  NO FRONTMATTER: $rel"
    }
}

# STALE PAGES (active, not updated in 30+ days)
Write-Host ""
Write-Host "=== STALE PAGES ==="
$cutoff = (Get-Date).AddDays(-30).ToString("yyyy-MM-dd")
foreach ($page in $allPages) {
    $content = Get-Content $page.FullName -Raw -ErrorAction SilentlyContinue
    if ($content -match 'status:\s*(active|at-risk)' -and $content -match 'last_updated:\s*(\d{4}-\d{2}-\d{2})') {
        $lastDate = $Matches[1]
        if ($lastDate -lt $cutoff) {
            $rel = $page.FullName.Replace($wikiSlash, '')
            Write-Host "  STALE: $rel (last: $lastDate)"
        }
    }
}

# INDEX GAPS
Write-Host ""
Write-Host "=== INDEX GAPS ==="
$indexContent = Get-Content "$wiki\index.md" -Raw
foreach ($page in $allPages) {
    if ($indexContent -notlike "*$($page.BaseName)*") {
        $rel = $page.FullName.Replace($wikiSlash, '')
        Write-Host "  NOT IN INDEX: $rel"
    }
}

Write-Host ""
Write-Host "=== DONE ==="
```

---

## 6. Final verification

After creating all files:
1. Confirm all folders from Step 1 exist.
2. Confirm these four files exist with the generic content above:
   - `skills/wiki-read-query.md`
   - `skills/source-ingest.md`
   - `skills/lint.md`
   - `skills/wiki-lint.ps1`
3. Confirm every `<WIKI_ROOT>` token has been replaced with the real absolute path.
4. Confirm the `.md` files are UTF-8 without BOM and `wiki-lint.ps1` is ASCII-only.
5. Run `skills/wiki-lint.ps1` once — on an empty wiki it should report `TOTAL PAGES: 0` and `DONE`
   without errors.

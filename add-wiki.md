# Add Wiki — Setup Orchestrator

This is the entry-point runbook for standing up a new Claude-maintained wiki. An AI agent follows
these steps in order: gather a few required inputs, apply the two build files (`add-claude.md` and
`add-skills.md`), then archive all three `add-*` files.

Companion files (must be present in the same directory as this file):
- `add-claude.md` — the generic `CLAUDE.md` template (schema & conventions)
- `add-skills.md` — build instructions for the skill files (`wiki-read-query`, `source-ingest`, `lint`, `wiki-lint.ps1`)

---

## ⛔ GUARDRAILS — read first, non-negotiable

**This process may ONLY create a brand-new wiki. It must NEVER overwrite, modify, or delete any
file in an existing wiki.** These rules override every other step below. If any of them is violated
or in doubt, **ABORT immediately, change nothing, and report to the user.**

1. **Empty-target rule.** The target wiki root must be empty, non-existent, or contain *only* the
   three `add-*` setup files (and at most a `.git` folder / `.gitignore` / `.gitattributes` /
   `.obsidian` / `.claude`). If it contains anything else — abort.
2. **No-overwrite rule.** Every file write in this process is **create-only**. Before writing any
   file, check whether it already exists. If it exists, **do not write it** — abort and report.
   This applies to `CLAUDE.md`, all skill files, `.gitignore`, `.gitattributes`, and every stub.
   The only files that may be touched if already present are the *append-only* operational stubs
   (`index.md`, `todo.md`, `meta/*.md`) **and only if** the empty-target rule already passed and
   they are empty/near-empty.
3. **No-delete / no-edit rule.** This process never deletes a file and never edits existing content.
   It only adds new files and moves the three `add-*` files in Step 4.
4. **No destructive git.** Never run `git init` on a directory that already contains a `.git`.
   Never `git reset`, `git clean`, `git checkout --`, force-push, or rewrite history.
5. **Abort-by-default.** If the safety preflight (Step 0) cannot positively confirm the target is a
   new/empty wiki, stop. Do not "best-effort" partial setup. No silent skips — report exactly what
   was found and why you stopped.
6. **Confirmation gate.** Make zero changes until the user has confirmed the collected answers in
   Step 1.

---

## Step 0 — Safety preflight (MANDATORY, blocking — runs before anything else)

Determine the target wiki root (from the user, see Step 1) and inspect it **read-only**. Do not
write anything during this step.

Treat the target as an **EXISTING WIKI (must abort)** if any of these are true:
- A `CLAUDE.md` already exists at the root.
- An `index.md` already exists with real content (more than a placeholder heading).
- Any of `projects/`, `entities/`, `concepts/`, `skills/` exists and contains `.md` files that are
  not the four skill files this process creates.
- A `.git` directory exists **with commit history** (a repo that already tracks content).
- `meta/log.md` exists and is non-empty (the wiki has an activity history).
- The directory contains files other than the allowed set in Guardrail #1.

If the target is judged an existing wiki: **STOP. Write nothing. Report to the user** what was found
(list the specific files/dirs that tripped the check) and state that the add process is refusing to
overwrite an existing wiki. Offer the user the option to choose a different, empty target path.

Only if the target is empty / non-existent / contains just the `add-*` files may you proceed to
Step 1.

---

## Step 1 — Ask the user the required questions

Ask the user the following. Do not assume defaults for the required items — wait for answers.
Present them together (a single batched prompt is fine) and confirm before proceeding.

**Required**
1. **Wiki root path** — the absolute path to the wiki's root directory. This becomes `<WIKI_ROOT>`
   everywhere the build files reference it.
   - e.g. Windows `C:\Users\<you>\Documents\my-wiki`, macOS/Linux `/home/<you>/my-wiki`

**Optional (ask, but a "no" / "skip" is acceptable)**
2. **Owner label** (optional) — a name/role to put in the `## Owner` line of `CLAUDE.md`. If skipped,
   leave the generic "Owner — sources and directs."

Echo the collected answers back to the user and get a go-ahead before making any changes.

---

## Step 2 — Apply `add-skills.md` (folders + skill files)

Follow `add-skills.md` exactly, using the answers from Step 1. **Create-only — per Guardrail #2,
if any target file below already exists, abort and report; never overwrite it.**
1. Create the folder tree if missing: `skills/`, `meta/`, `sources/processed/`, `projects/`,
   `entities/people/`, `entities/apps/`, `concepts/`.
2. Create the operational file stubs if missing: `index.md`, `todo.md`, `meta/done.md`,
   `meta/log.md`, `meta/ingest-progress.md`.
3. Write the four skill files with the verbatim content from `add-skills.md`:
   - `skills/wiki-read-query.md`
   - `skills/source-ingest.md`
   - `skills/lint.md`
   - `skills/wiki-lint.ps1`
4. Replace every `<WIKI_ROOT>` token with the real absolute path.
5. Write `.md` files as **UTF-8 without BOM**; keep `wiki-lint.ps1` **ASCII-only**.

---

## Step 3 — Apply `add-claude.md` (the CLAUDE.md spec)

**Create-only — per Guardrail #2, if `CLAUDE.md` already exists at the root, abort and report;
never overwrite it.**
1. Take the content of `add-claude.md` and write it to `<WIKI_ROOT>/CLAUDE.md`.
2. Replace every `<WIKI_ROOT>` token with the real absolute path.
3. If an owner label was provided, set the `## Owner` line accordingly; otherwise leave the generic text.
4. Write as **UTF-8 without BOM**.

---

## Step 4 — Archive the three `add-*` files

Once the wiki is created and verified:
1. Create the folder `<WIKI_ROOT>/skills/archive/` if it does not already exist.
2. Move all three setup files into it:
   - `add-wiki.md`  → `skills/archive/add-wiki.md`
   - `add-claude.md` → `skills/archive/add-claude.md`
   - `add-skills.md` → `skills/archive/add-skills.md`
3. Confirm each move succeeded and that none of the three remain in the wiki root.

These files are kept (not deleted) so the wiki retains a record of how it was bootstrapped.

---

## Step 5 — Final verification

1. `CLAUDE.md` exists at `<WIKI_ROOT>` with no remaining `<WIKI_ROOT>` placeholders.
2. All folders from Step 2 exist, and the four skill files are present in `skills/`.
3. The three `add-*` files now live in `skills/archive/` and nowhere else.
4. Run `skills/wiki-lint.ps1` once — on a fresh wiki it should report `TOTAL PAGES: 0` and `DONE`
   with no errors.
5. Report a short summary to the user: what was created and where.

# claude-domain-memory
<img width="1209" height="880" alt="IMG_0557" src="https://github.com/user-attachments/assets/5993a9ea-daa8-4b73-b314-0611c2f97d2a" />


A lightweight knowledge accumulation system for Claude Code. Captures what you learn while working — API quirks, patterns, confirmed rules — and makes it available in every future session automatically.

## The problem it solves

Claude Code has per-project memory, but no built-in way to track patterns that emerge over time. You rediscover the same quirks. Hypotheses never get confirmed or discarded. Knowledge stays in your head instead of the project.

This system adds a `domain` memory type that accumulates structured knowledge across sessions — with a clear lifecycle from observation to hypothesis to confirmed rule.

## How it works

Each project gets one `domain_*.md` file per knowledge area (e.g. `domain_notion_api.md`, `domain_pricing.md`). Each file has three sections:

- **Facts** — confirmed, no ambiguity
- **Hypotheses** — patterns seen but not confirmed; tracked with a confirmation count
- **Rules** — promoted from hypotheses at 5+ confirmations; demoted back if contradicted

At the end of a session, run `/knowledge-update`. Claude reviews what was worked on and updates the relevant domain files — adding facts, incrementing hypothesis counts, and promoting anything that crossed the threshold. Promoted rules are also written into the project's CLAUDE.md so they're treated as binding instructions, not just context.

## How memory location works

Domain files are stored per **working directory** — the folder Claude Code was opened from, not the folder containing the files you edited.

Claude Code maps your working directory to a project slug by replacing `/` with `-`:
```
/Users/you/Documents/my-project → -Users-you-Documents-my-project
```

Memory for that session goes into:
```
~/.claude/projects/-Users-you-Documents-my-project/memory/
```

**What this means in practice:**
- If you open Claude Code from `/Users/you/Documents/my-project`, domain files land in that project's memory folder (in .claude/projects/username-path-to-my-project..) and load automatically next time you open from the same directory
- If you open from a different directory (e.g. your home folder) and edit files in `my-project`, memory goes into the home-level folder (.claude/projects/username) instead — and won't be available when working in `my-project`
- There is no setting to change this — it is determined by working directory at session start

**Memory is never stored in your project folder by Claude design.** Claude Code stores all memory under `~/.claude/projects/<slug>/memory/` — completely separate from your code. Your project folder stays clean. The slug is Claude's internal mapping of your working directory path.

**If your domain files landed in the wrong place**, move them manually:
```bash
mv ~/.claude/projects/<wrong-slug>/memory/domain_*.md \
   ~/.claude/projects/<correct-slug>/memory/
```

## Installation

### 1. Install the skill

Copy `skills/knowledge-update/` to your Claude Code skills folder:

```bash
cp -r skills/knowledge-update ~/.claude/skills/
```

The `/knowledge-update` command is now available in Claude Code.

### 2. Set up domain files for a project

Copy the template into your project's memory folder:

```bash
# Find your project slug (it mirrors the path with slashes replaced by hyphens)
# e.g. /Users/you/Documents/my-project → -Users-you-Documents-my-project

cp templates/domain_template.md \
  ~/.claude/projects/<project-slug>/memory/domain_<name>.md
```

Edit the frontmatter and rename `<name>` to match the domain (e.g. `notion_api`, `pricing`, `auth`).

### 3. Add it to MEMORY.md

In `~/.claude/projects/<project-slug>/memory/MEMORY.md`, add:

```markdown
## Domain Knowledge
- [Your domain](domain_<name>.md) — one-line description
```

Claude Code auto-loads `MEMORY.md` at session start, so the domain file will be available from then on.

### 4. Add the rules block to CLAUDE.md

In your project root's `CLAUDE.md` (create it if it doesn't exist), add:

```markdown
## Confirmed Rules
Rules promoted from domain knowledge — follow by default.
If a user request conflicts with a rule, flag it rather than silently breaking it.

<!-- rules-start -->
<!-- rules-end -->
```

The block between the markers is managed by `/knowledge-update`. Rules are written here automatically on promotion and removed on demotion.

## Usage

At the end of any session where you learned something non-obvious:

```
/knowledge-update
```

Claude will:
1. Identify which domain(s) the session touched
2. Extract facts, new hypotheses, and confirmations
3. Update the relevant `domain_*.md` files
4. Promote any hypothesis that reached 5+ confirmations to a Rule
5. Sync all current rules into a managed block in CLAUDE.md, so they're enforced as instructions

## What to store

**Good candidates:**
- API quirks not obvious from documentation
- Schema details that caused bugs when assumed wrong
- Patterns confirmed across multiple sessions
- Rules that should apply by default going forward

**Skip:**
- Information already in code comments or build docs
- Things easily found by reading the current files
- One-off task details

## File structure

```
<project-root>/
  CLAUDE.md                  ← contains managed rules block (<!-- rules-start/end -->)

~/.claude/projects/<project-slug>/memory/
  MEMORY.md                  ← index, auto-loaded by Claude Code
  domain_<name>.md           ← one per knowledge domain
  domain_<other>.md
  user_*.md                  ← existing memory types
  feedback_*.md
  project_*.md
  reference_*.md
```

## Example domain file

```markdown
---
name: Notion API domain knowledge
description: Facts and patterns from working with the Notion API
type: domain
domain: notion_api
---

## Facts
- Property names are case-sensitive — "Name" and "name" are different
- Pagination must be a while loop, not recursion — recursion doesn't correctly pass start_cursor

## Hypotheses

| Hypothesis | Confirmations | Last seen |
|---|---|---|
| Filtering by last_edited_time is more reliable than filtering by a Done date field | 3 | 2026-04-01 |

## Rules
_(none yet)_
```

## Upgrading from an earlier version

Re-run the install command to get the latest skill:

```bash
cp -r skills/knowledge-update ~/.claude/skills/
```

If you already have promoted rules in your `domain_*.md` files, they won't appear in CLAUDE.md automatically. The sync only runs during `/knowledge-update`. To backfill, just run `/knowledge-update` at the start of any session — even if nothing new happened. Claude will pick up the existing rules and write the CLAUDE.md block.

You'll also need to add the rules block to your project's CLAUDE.md manually (step 4 of installation above) if it doesn't exist yet — the skill creates the markers if missing, but only when it runs.

## Domain files are personal

The template and skill are generic. The domain files themselves contain knowledge specific to your projects — they're not meant to be shared. Each person builds their own through use.

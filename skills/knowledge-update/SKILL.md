---
name: knowledge-update
description: Extract domain knowledge from the current session and store it in the project's domain_*.md memory files. Updates facts, adds hypotheses, increments confirmation counts, promotes hypotheses to rules at 5+ confirmations, and syncs promoted rules into CLAUDE.md for enforcement. Run at the end of a work session.
---

# Knowledge Update

Extract what was learned in this session and persist it to the project's domain memory files.

## When to Use

Run at the end of a session with `/knowledge-update` when:
- You just completed a non-trivial task
- You encountered a surprising bug, API quirk, or pattern
- Something worked differently than expected
- A previous hypothesis was confirmed or contradicted

## How It Works

### Step 1 — Identify the domain
Look at what was worked on. Map it to an existing domain file or decide if a new one is needed.

Existing domain files live in the project memory folder:
`~/.claude/projects/<project-slug>/memory/domain_*.md`

Only create a new domain file if the work clearly belongs to a different domain than existing ones.

### Step 2 — Extract insights from the session

Review the conversation and identify:

**Facts** (certain, repeatable, no ambiguity):
- API behaviors, schema details, confirmed constraints
- "Property names are case-sensitive" — not "I think they might be"

**Hypotheses** (pattern seen, not confirmed enough):
- Things that worked once or twice but need more evidence
- Add to the hypotheses table with confirmation count 1

**Confirmations** (a hypothesis proved true again):
- Find the matching row in the hypotheses table
- Increment the confirmation count

**Contradictions** (a rule proved wrong):
- Move it back to Hypotheses with a note

### Step 3 — Apply promotion logic

- Hypothesis with **5+ confirmations** → move to Rules section
- Rule **contradicted by new data** → move back to Hypotheses with a note explaining what contradicted it

### Step 4 — Update the domain file(s)

Edit the relevant `domain_*.md` file(s). If creating a new domain file, also add it to `MEMORY.md`.

### Step 5 — Sync rules into CLAUDE.md

Rules carry enforcement weight only when they live in CLAUDE.md, not just in domain files. After any promotion or demotion, rebuild the rules block in the project's CLAUDE.md.

**Locate CLAUDE.md:** Look for it in the project root (not `~/.claude/`). If it doesn't exist yet, create it.

**Find the managed block** — the section between these markers:
```
<!-- rules-start -->
<!-- rules-end -->
```

If the markers don't exist yet in CLAUDE.md, append them (and the header from `rules_template.md`) now.

**Rebuild the block** from scratch using the Rules sections from this project's `domain_*.md` files only:

```markdown
<!-- rules-start -->
## <domain_name>
- <rule>
- <rule>

<!-- repeat for each domain_*.md in this project that has rules -->
<!-- rules-end -->
```

Only include domains that have at least one rule. Omit domains with no rules. Replace the entire block content — do not append.

If a rule was **demoted** this session, remove it from the block. If a rule was **promoted**, add it.

New domain file template:
```markdown
---
name: <Domain> domain knowledge
description: Facts, hypotheses, and confirmed rules about <domain> in <project>
type: domain
domain: <domain_name>
---

## Facts

## Hypotheses

| Hypothesis | Confirmations | Last seen |
|---|---|---|

## Rules
_(none yet)_
```

## What NOT to store

- Information already in code comments or build instructions
- Things easily found by reading the current files
- Step-by-step how-to guides (those belong in instruction docs, not domain memory)
- Ephemeral task details from this session only

## Example output

After a session fixing a Notion API pagination bug:

> **Domain:** notion_api  
> **New fact:** `has_more` can be true even when `next_cursor` is null if the last page is exactly 100 items — always check both  
> **Hypothesis confirmed (2→3):** Filtering by `last_edited_time` is more reliable than filtering by a Done date field  
> **No promotions** — no hypothesis reached 5 confirmations

Then edit `domain_notion_api.md` accordingly.

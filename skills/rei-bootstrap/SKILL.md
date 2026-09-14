---
name: rei-bootstrap
description: Interactively bootstrap a Rei intention hierarchy — a root intention with context, category, and optional project scope, 2–5 child intentions, support links, supporting habits, focus areas and a focus cycle, recurring reflections, and agent coaching guidance — creating everything after one confirmation and verifying it by reading back.
allowed-tools: AskUserQuestion, Bash, Read
---

# Rei Bootstrap

Turns a goal the user describes into a structured Rei setup: root intention → children, plus
the habits, focus rhythm, reflections, and coaching guidance that support it. Use
`/rei-bootstrap-habit` when the user only wants a single habit, and follow its guidance for
designing habits in depth.

## Key Concepts

Keep these distinct when proposing structure — they are easy to conflate:

- **Intention** — a goal with a `--horizon` (`5d`, `2w`, `3m`, `1q`, `2y`, `1decade`). Active
  by default; `--future` defers it (hidden from daily views, blocks operational actions,
  activate later with `rei intention activate ID`). Deferring a parent cascades to children.
- **Context** — the life domain label (`work`, `personal`). Root intentions only; children
  inherit it. `rei intention contexts` lists labels in use, displayed with an `@` prefix —
  pass the label **without** the `@`.
- **Category** — the *type* of activity (`meeting`, `learning/reading`). May carry note
  guidance and property bindings that auto-set custom properties on entities assigned to it
  (see `rei category show SLUG`). Can map to a focus area.
- **Focus area** — the *nature* of a day's activity (Study, Create, Play); one intention spans
  several. Selected per day within a **cycle** (only one active at a time).
- **Support** — an informational cross-hierarchy link ("Exercise" supports "Focus at work"),
  not a dependency.
- **Project** — a durable subject modeled as a typed topic. Scoping the root intention to a
  project passes the scope to all descendants and their work. Software projects come from Mori
  via `rei project sync mori://NS/PROJECT`; anything else via `rei project create`.

## Workflow

Always use `rei --actor claude-code <command>` for writes and pass every ID explicitly —
omitted IDs open fzf pickers that hang the run. Most of these commands can print an error and
still exit `0`, so capture each created ID from the output and rely on the Verify phase.

### 1. Discover

Understand the goal, why it matters, the timeframe, and whether it starts now or later. Ask
only what the user hasn't said. Then survey what already exists so you reuse rather than
duplicate:

```bash
rei intention list --roots --all -s "KEYWORD" --json   # an intention for this goal may exist
rei intention contexts
rei category list --flat --descriptions --json
rei focus list
rei cycle status
rei project list
```

### 2. Design the structure

Draft, with the user, everything below. Each section except the root is optional — skip what
the user doesn't want, and skip focus setup if focus areas exist, and cycle setup if a cycle
is active.

- **Root**: title, horizon, active/future, context (reuse an existing label), category, optional
  deadline (`--deadline YYYY-MM-DD`), optional project scope.
- **Children**: 2–5 milestones or areas, each with a horizon no longer than the root's.
- **Categories**: reuse existing ones; create new ones only for activity types this goal
  introduces, optionally mapped to a focus (`-f FOCUS_ID`).
- **Supports**: links from/to existing intentions found in Phase 1.
- **Habits**: name, purpose, schedule, and which intention each supports. Fixed schedules
  (`--daily`, `--weekly DAY`, `--weekly-on mon,wed,fri`, `--monthly N`, `--every N`), flexible
  ones (`--free-weekly N`, ...), cue-based (`--cue`), or break habits (`--break`) — see
  `/rei-bootstrap-habit` for the design detail.
- **Focus areas** (only if none exist): `rei focus templates` / `import anatomy-of-equanimity`,
  or custom areas.
- **Cycle** (only if none active): length (7–10 days suits beginners) and optional name.
- **Reflections**: recurring daily and/or weekly schedules, with prompts tailored to the goal
  and filtered by the root's context.
- **Guidance**: coaching style, success indicators, review questions, action patterns,
  constraints, and motivation — synthesized from the conversation.

### 3. Confirm

Present the complete plan as a tree (root, children, categories, supports, habits with
schedules, focus/cycle, reflection schedules, guidance summary) and ask for approval once
before creating anything. Apply edits and re-show if the user adjusts.

### 4. Create

In dependency order, capturing each `intention_…`, `habit_…`, `focus_…` ID.

```bash
# categories first, so intentions can be assigned at creation
rei --actor claude-code category create NAME [-p PARENT_SLUG] -d "DESCRIPTION" [-f FOCUS_ID]

# root, then children
rei --actor claude-code intention create "TITLE" --horizon 3m [--future] \
  [-c CONTEXT] [--category SLUG] [--deadline YYYY-MM-DD]
rei --actor claude-code intention create "CHILD TITLE" -p ROOT_ID --horizon 1m [--category SLUG]

# project scope (optional)
rei --actor claude-code project create KEY --label "LABEL" --description "..." --json   # or: rei project sync mori://NS/PROJECT
rei --actor claude-code project scope add KEY ROOT_ID --entity-type intention --json

# supports
rei --actor claude-code support add -f SUPPORTING_ID -t SUPPORTED_ID -n "HOW IT HELPS"

# focus areas (only if none exist) and cycle (only if none active)
rei --actor claude-code focus import anatomy-of-equanimity
rei --actor claude-code focus create -n "NAME" -s "SYMBOL" -d "DESCRIPTION"
rei --actor claude-code category link-focus SLUG FOCUS_ID
rei --actor claude-code cycle start -l 10 -n "NAME"

# habits, each linked to the intention it supports
rei --actor claude-code habit create -n "NAME" -p "PURPOSE" --daily -i INTENTION_ID

# recurring reflections
rei --actor claude-code reflect schedule daily -p "PROMPT" -c CONTEXT
rei --actor claude-code reflect schedule weekly -d sunday -p "PROMPT" -c CONTEXT

# guidance
rei --actor claude-code intention guidance ROOT_ID --stdin <<'EOF'
## Coaching Style
## Success Indicators
## Review Focus Areas
## Action Patterns
## Constraints
## Motivation Context
EOF
```

Set context or category after creation with `rei --actor claude-code intention set-context -i ID CONTEXT` and
`rei --actor claude-code intention set-category -i ID SLUG`. If a step fails, report it and continue with
independent steps; don't create children under a root that failed.

### 5. Verify

```bash
rei intention show ROOT_ID --json \
  | jq '{intention: .intention | {intentionId, title, status, horizon, deadline},
         categorySlug, children: [.children[] | {intentionId, title, status}],
         habits: [.habits[]?], supports: [.supports[]?]}'
rei intention list --roots --all -c CONTEXT --json | jq -r '.[].id' | grep ROOT_ID
rei habit list --json | jq -c --arg r ROOT_ID '.[] | select(.intentionId == $r) | {habitId, name, scheduleType}'
rei project scope list KEY            # if scoped
rei cycle status
rei reflect pending
```

Check each planned entity against what came back; retry a missing write once, otherwise report
it. Habits linked to children are found by filtering on that child's ID.

## Output Format

```
## Bootstrap Complete

### Root Intention
- <title> (<intention_id>) — horizon <h>, <active/future>, context <ctx>, category <slug>
- Project: <key or "none">

### Children
1. <title> (<id>) — <horizon>

### Supports / Categories / Focus & Cycle
- <supporting> → <supported> ("<note>")
- Categories: <created or reused slugs>
- Focus: <existing / imported / custom / skipped>; Cycle: <existing "<name>" / started <name> (<n> days) / skipped>

### Habits
- <name> (<schedule>) → <intention title>

### Reflections & Guidance
- Scheduled: <daily / weekly on <day>>
- Guidance: <style summary or "not set">

### Failed / Skipped
- <item — reason>

### Next Steps
- <if future> Activate when ready: `rei intention activate <id>`
- <if cycle> Pick today's focus: `rei cycle select-focus FOCUS_ID`
- Record a first action: `rei action record -i <id> "DESCRIPTION"`
- See it in context: `rei today` / `rei tomorrow`
```

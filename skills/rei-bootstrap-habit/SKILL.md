---
name: rei-bootstrap-habit
description: Interactively create a new Rei habit — pin down polarity (build or break), name, purpose, and cadence (fixed, flexible, or cue-based), link it to a supporting intention, optionally classify it with a category (which carries its focus area), set an action template, context, custom properties, and a first reminder, then verify it by reading it back. Use when the user wants to start a practice or stop a behavior.
allowed-tools: AskUserQuestion, Bash, Read
---

# Rei Bootstrap Habit

Turns "I want to start/stop doing X" into a well-formed Rei habit. `rei-bootstrap` delegates
habit creation to this skill, so keep the phases self-contained.

## When to Use

- "Help me create a new habit" / "Add a habit for…"
- "I want to start a daily/weekly practice"
- "I want to stop doing X" / "Help me avoid X" (break habit)
- "/rei-bootstrap-habit"

## Key Concepts

- **Polarity** — *build* (cultivate; record actions) or *break* (avoid; `--break`, then
  `start-abstinence`, and `log-occurrence` on relapse, which resets the streak).
- **Cadence** — exactly one of:
  - fixed: `--daily`, `--weekly DAY`, `--weekly-on mon,wed,fri`, `--monthly D` (1–31),
    `--quarterly`, `--yearly-month M --yearly-day D`, `--every N`
  - flexible: `--free-weekly N`, `--free-monthly N`, `--free-quarterly N`, `--free-yearly N`
  - cue-based: `--cue "After meals"` with optional `--target "3/week"`
- **Linked intention** — the goal the habit serves. Recorded habit actions count toward it,
  and the habit inherits its context. Context (`set-context`) is only allowed on unlinked habits.
- **Category** — habits have one (`rei habit set-category`). It drives classification, scoped
  custom properties, and category property bindings (auto-set values). A category may map to a
  **focus area** (`focusId`); that is how a habit aligns with a focus — there is no separate
  habit→focus command.
- **Blockers** — `rei blocker declare --habit ID` pauses the habit automatically; resolving
  the last one resumes it. Mention this if the user anticipates obstacles; don't create one
  during setup.

## Workflow

Use `rei --actor claude-code …` for every write and always pass IDs explicitly — several habit
subcommands open an fzf picker when the ID is omitted. Ask only for what the user hasn't
already said; if the request is complete, go straight to the plan.

### 1. Understand the practice

Establish what the user wants to do or avoid, why, and when. Derive:
- **Polarity** — ask only if ambiguous.
- **Name** — short and action-oriented ("Morning meditation"; for break: "No social media
  before noon").
- **Purpose** — the user's own reason, in their words.
- **Cadence** — map their description to one cadence flag. For weekly, prefer `--weekly-on`
  when more than one day. For cue-based, ask whether they want a target frequency.
- **Action template** (build habits, optional) — a default description for recorded actions
  when the action is the same each time.

### 2. Gather Rei context

Run in parallel:

```bash
rei intention list --json
rei category list --flat --descriptions --json   # slug, focusId, propertyBindings
rei focus list
rei custom-property list -e habit --json
rei intention contexts
rei habit list --json | jq -r '.[] | "\(.habitId)\t\(.status)\t\(.name)"'
```

- **Duplicate check** — if an active or paused habit already covers this practice, offer to
  resume/rename/reschedule it instead (`rei habit resume`, `rename`, `update-schedule`).
- **Intention** — propose the best-matching intention (search with
  `rei intention list --all -s KEYWORD --json`); standalone is fine if nothing fits.
- **Category** — propose one only if it clearly fits. Prefer a category whose `focusId`
  matches the focus area the habit belongs to. If the right category doesn't exist, offer to
  create it (optionally mapped to a focus with `-f FOCUS_ID`). Note its `propertyBindings` —
  those values will be set automatically; don't set them again.
- **Properties** — only habit properties in scope for the chosen category whose value is
  obvious; values must come from the definition (`rei custom-property show KEY --json`).
- **Context** — only when unlinked, and reuse an existing context.
- **Reminder** (optional) — useful for flexible or low-frequency habits (e.g. a nudge before a
  monthly one).

### 3. Confirm the plan

Show the full plan and get one approval (apply / adjust / cancel):

```
Habit plan
  Name:       Morning meditation            Polarity: build
  Purpose:    Start the day calm and focused
  Cadence:    --daily
  Intention:  intention_01… — Improve mental health   (or: standalone)
  Category:   health/mindfulness  → focus: Play         (or: none)
  Template:   "10 minutes of meditation"               (or: none)
  Context:    —  (inherited from intention)
  Properties: none
  Reminder:   none
```

### 4. Create

```bash
rei --actor claude-code habit create -n "NAME" -p "PURPOSE" CADENCE_FLAGS \
  [-i INTENTION_ID] [--action-template "TEMPLATE"] [--break]
```

Capture `HABIT_ID` from the output (`grep -o 'habit_[0-9a-z]*' | head -1`). If none is printed,
find it: `rei habit list --json | jq -r --arg n "NAME" '.[] | select(.name == $n) | .habitId'`.
Without a `HABIT_ID`, stop and report — don't run the follow-up commands.

Then, as planned:

```bash
rei --actor claude-code category create "NAME" [-p PARENT_SLUG] [-d "DESC"] [-f FOCUS_ID]   # only if creating
rei --actor claude-code habit set-category HABIT_ID CATEGORY_SLUG
rei --actor claude-code habit set-context HABIT_ID "CONTEXT"        # unlinked habits only
rei --actor claude-code habit set-property --habit HABIT_ID KEY VALUE
rei --actor claude-code habit start-abstinence HABIT_ID             # break habits
rei --actor claude-code habit remind HABIT_ID -d "DESCRIPTION" --at "TIME"
```

`--at` accepts natural and ISO times (`rei help time`). Check each subcommand's `--help`
before using an argument form not shown here.

### 5. Verify

Habit writes may print an error yet exit `0`, so read back:

```bash
rei habit show HABIT_ID --json \
  | jq '.habit | {name, polarity, scheduleType, scheduleData, intentionId, context, actionTemplate, abstinenceStartedAt, status}'
```

Check name, polarity, schedule, linked intention, template, context, and (for break habits)
`abstinenceStartedAt`. The category isn't in this JSON — rely on the `set-category` output and
report it as such. Retry a missing write once, then report it as failed.

## Output Format

```
## Habit Created

- **Name**: <name>  (<habit_id>)
- **Type**: Build / Break
- **Purpose**: <purpose>
- **Cadence**: <human-readable schedule or cue + target>
- **Linked Intention**: <title> (<intention_id>) or "standalone"
- **Category**: <slug> (focus: <focus>) or "none"
- **Action Template / Context / Properties / Reminder**: <values or "none">
- **Failed / Skipped**: <item — reason> or "nothing"

### Next Steps
- Build: `rei habit record-action HABIT_ID` (uses the template; `--at`, `--duration` available)
- Break: `rei habit log-occurrence HABIT_ID -n "trigger"` if a relapse happens
- Progress: `rei habit status`, `rei habit tracker HABIT_ID`
- Obstacle: `rei blocker declare --habit HABIT_ID "DESCRIPTION"` (pauses the habit)
```

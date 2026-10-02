---
name: rei-capture-session
description: Capture a planning or design session in Rei at its end — identify the goal the user was pursuing (not the worked example that came up), reuse or create that goal intention, link examples and side work to it with support or dependency edges, record one action summarizing what the session produced, write a structured Question / Decided / Leaning / Produced / Open note, relate the note to each project topic the session involved by the role that project played (`about` for subject matter, `produced-in`, `consulted`, or `worked-example` edges otherwise; resolved by mori:// reference), and verify by reading back. Use at the end of an agent session that planned or designed something so the day's thinking is reviewable; use rei-note-from-tmp to file an arbitrary scratch note instead.
allowed-tools: AskUserQuestion, Bash, Read
---

# Rei Capture Session

Turns a finished planning or design conversation into Rei records the user can review later.
The day's activity view gets one **action**, and one **note** holds the reasoning. Both are
anchored to the **goal** the user was working toward, and the note is linked to every project
it involved, with an edge that says what role the project played.

Use `rei-note-from-tmp` to file an arbitrary scratch note. Use `rei-bootstrap` to plan a new
intention hierarchy from scratch. This skill records a session that has already happened.

## Key Concepts

- **Anchor to the goal, not the example.** Users often plan a capability by working through
  one concrete case. If the session used project X to work out how to do Y, the anchor is Y,
  and X is linked to it. Example from a real session: the conversation spent hours on
  keiro-runtime-kenshou's verification MasterPlan, but the user's actual intention was "manage
  multi-project, multi-repo initiatives". Filing the session under kenshou would have hidden the
  capability being built.
- **No matching intention is a signal.** It usually means the user has been pursuing an
  unnamed goal. Creating that intention is part of capturing the session, but confirm it with
  the user first.
- **Decided vs Leaning.** Users evaluate out loud. Record as *Decided* only what the user
  explicitly chose. Options the agent recommended, or the user said they "lean" toward, go
  under *Leaning*. Never ratify a suggestion as a decision.
- **Why an action too.** `rei today` and `rei yesterday` list actions, not notes. Planning
  sessions mostly produce `docs:` and `chore:` commits, which mori-rei-app skips when it records
  actions from commit trailers. Without an explicit action, a planning day looks empty.
- **Projects are topics with a mori:// reference.** `rei project show mori://NS/NAME` resolves
  a repository to its project topic.
- **One edge per project, chosen by the role it played.** A session involves projects in
  different ways, and each way has its own predicate. Don't use `about` as a catch-all:

  | Role in the session | Edge | Example |
  |---|---|---|
  | Its design or decisions are what the note is about | `about` association | the runtime whose capability was planned |
  | Artifacts were created or changed there | `produced-in` edge | the repo that received a plan, IR, or use case |
  | Read as reference or evidence, not changed | `consulted` edge | a library whose API was checked |
  | The concrete case used to reason about the goal | `worked-example` edge | the service whose runbook motivated the design |

  A project can hold more than one role. For example, a subject project that also received
  commits gets both `about` and `produced-in`.
- **Don't add `scoped-to`.** It means operational membership and is inherited. The note already
  inherits its scope from the anchor intention's hierarchy (check with
  `rei topic associations NOTE_ID --effective`). A direct `scoped-to` edge to every touched
  project would make the note a member of projects it merely visited.
- **Note tag vs `tags` property.** Unlike `rei-note-from-tmp`, this skill uses the note tag
  `rei note new -t planning-session`. That is the tag `rei note list -t planning-session` filters
  on, which is how sessions are reviewed. If you ever filter on the `tags` tag-set property
  instead, use membership syntax, `--where 'tags any planning-session'`. `tags=…` compares the
  whole set and silently matches nothing.
- **AGENT** below means the actor name of the harness running the skill: `claude-code` or
  `codex`. Pass IDs explicitly on every command, because an omitted ID opens an fzf picker that
  hangs the run.

## Workflow

### 1. Reconstruct the Session

From the conversation, extract:

- **GOAL**: what the user was trying to achieve, in one line. Ask "what capability or outcome
  was this for?", not "which repository did we edit?".
- **EXAMPLES / SIDE WORK**: intentions for concrete cases used along the way, such as a plan's
  `intention:` frontmatter or an `Intention:` commit trailer.
- **DECIDED**: the user's explicit choices.
- **LEANING**: favoured but uncommitted options, each with the alternative it was weighed against.
- **PRODUCED**: commit SHAs per repository, and canonical `mori://` URIs for created or changed
  artifacts such as plans, OKF concepts and improvement requests.
- **OPEN**: questions left for the user and next steps.
- **PROJECTS**: every repository the session read or changed meaningfully, as `mori://NS/NAME`,
  each with its role or roles: subject (`about`), `produced-in`, `consulted`, or
  `worked-example`. Only projects whose design or decisions the note records are subjects,
  usually one to three.

### 2. Find or Propose the Goal Intention

Search by the GOAL's key words, using several short keywords:

```bash
rei intention list --all -s "KEYWORD" --json | jq -r '.[] | "\(.id)\t\(.title)"'
```

Shortlist up to three candidates. When none fits, propose a new title and a home:

- If the goal belongs to a repository with a `mina.kdl` that declares
  `rei { create { default-parent … } }`, create it there with `mina ci`. That applies the
  repository's parent and title prefix (for example `機関`), so pass the title **without** the
  prefix. Check with `grep -A4 'create' mina.kdl`.
- Otherwise use `rei intention create` with an explicit `--parent`.

### 3. Confirm

Make one AskUserQuestion call with two questions:

1. **Goal intention.** List the shortlist with the best one marked Recommended, plus "Create
   new: <title> under <parent>".
2. **Plan.** Show the action text, the note outline (Decided and Leaning visibly separate), the
   support or dependency links, and each project topic with the edge or edges it will get.
   Options: apply (Recommended), let me adjust, cancel.

On cancel, write nothing.

### 4. Create the Goal Intention (only if new)

```bash
cd REPO_WITH_MINA_KDL && mina ci --json "TITLE" | jq -r .intentionId
# or
rei --actor AGENT intention create "TITLE" --parent PARENT_ID
rei intention show INTENTION_ID      # confirm title, parent, category
```

### 5. Link Examples and Side Work

Link the example or side-work intention to the goal. Don't move or re-file it.

```bash
# the example proves or advances the goal
rei --actor AGENT support add --from EXAMPLE_ID --to GOAL_ID --note "ONE-LINE WHY"
# the goal cannot finish until the other intention does
rei --actor AGENT dependency add --from GOAL_ID --on OTHER_ID --note "ONE-LINE WHY"
```

### 6. Record the Action

Write one action for the whole session, summarizing what it produced:

```bash
rei --actor AGENT action record -i GOAL_ID --no-habit "Planning session on …: <outcomes>"
```

Capture `ACTION_ID` with `grep -o 'action_[0-9a-z]*' | head -1`. Add `-d DURATION` only when the
user states how long the session took.

### 7. Write the Note

Write the note to a scratch file first, then store it verbatim. The first heading becomes the
title.

```markdown
# Session YYYY-MM-DD: <goal in a few words>

## Question
<what the session set out to answer; name the worked example as an example>

## Decided
- <explicit user choice> — <artifact mori:// URI when one records it>

## Leaning (not decided)
- <favoured option> over <alternative> — <why>

## Produced
- <repo> `<sha>` (<what>) · `mori://…`

## Open
- <question or next step>
```

```bash
rei --actor AGENT note new --intention GOAL_ID --stdin -t planning-session < NOTE_FILE
```

Capture `NOTE_ID`. A non-zero exit means nothing was stored, so stop and report.

### 8. Relate the Note and Action to Each Project

The three role predicates must exist. Check once per run; if any is missing, define it (these
are the definitions already in use):

```bash
for p in produced-in consulted worked-example; do rei predicate show "$p" >/dev/null 2>&1 || echo "missing: $p"; done
rei --actor AGENT predicate define produced-in -l "Produced in" --source-types note,action --target-types topic \
  -d "Source records artifacts created or changed in the target project, such as commits, plans, improvement requests, or use cases (PROV-O wasGeneratedBy, loosely)"
rei --actor AGENT predicate define consulted -l "Consulted" --source-types note,action --target-types topic \
  -d "Source's reasoning drew on the target as reference or evidence without changing it (PROV-O used, loosely)"
rei --actor AGENT predicate define worked-example -l "Worked example" --source-types note,action --target-types topic \
  -d "Source used the target as a concrete case while reasoning about its subject; unlike exemplifies, the source is not itself an example of the target"
```

For each `mori://NS/NAME` in PROJECTS, resolve its topic:

```bash
rei project show mori://NS/NAME --no-mori --json | jq -r .identity.topicId
```

Exit code `2` with "No project matches" means Rei has no topic for that project. Offer to sync
it (`rei --actor AGENT project sync mori://NS/NAME`, which needs `MORI_API_URL`) rather than
guessing a topic key.

Then write one edge per role. `about` is a first-class topic association; the other three are
graph edges. Give each graph edge a short `-d` saying what was produced, consulted, or used:

```bash
# subject matter
rei --actor AGENT topic associate TOPIC_ID NOTE_ID --relation about
# artifacts created or changed there: on the note and on the action
rei --actor AGENT edge add -f NOTE_ID -t TOPIC_ID -p produced-in -d "Plan 77 (commit abc1234)"
rei --actor AGENT edge add -f ACTION_ID -t TOPIC_ID -p produced-in -d "Plan 77"
# read as reference, not changed
rei --actor AGENT edge add -f NOTE_ID -t TOPIC_ID -p consulted -d "Keiro 0.18 timer APIs"
# the concrete case the session reasoned through
rei --actor AGENT edge add -f NOTE_ID -t TOPIC_ID -p worked-example -d "MP-10 repair window"
```

### 9. Verify

```bash
rei note show NOTE_ID --json | jq '{title, anchor, tags}'
rei topic associations NOTE_ID --relation about --json | jq -r '.[].topic.key'
rei edge show NOTE_ID | grep -E 'produced-in|consulted|worked-example'
rei edge show ACTION_ID | grep produced-in
rei support supported-by GOAL_ID
rei action list -i GOAL_ID --today
```

Redo any write that did not land, once. Otherwise report it.

## Output Format

```
## Session Captured

- **Goal**: <intention_id> — <title> (reused | created under <parent>)
- **Action**: <action_id> — <summary>
- **Note**: <note_id> — <title> (tag planning-session)
- **Linked**: <example_id> supports goal | <goal> depends on <id> | none
- **About**: <subject topic keys>
- **Produced in**: <topic keys> (note and action)
- **Consulted**: <topic keys> | none
- **Worked example**: <topic keys> | none
- **Skipped / Failed**: <item — reason> | nothing

Review later: `rei yesterday` · `rei note list -t planning-session` · `rei note show <note_id>`
```

## Worked Examples

A session on 2026-10-01 worked through keiro-runtime-kenshou's MasterPlan 2. The goal was
running multi-project, multi-repo initiatives in general.

- **Goal created** with `mina ci` in kikan: `intention_01m3w5tqcyevdbxsq0qan012rt`, "機関 Manage
  multi-project, multi-repo initiatives".
- **Example linked**: kenshou MP-2's intention `support add` → the goal.
- **Action**: `action_01m3w5tz0be4xb6nzs9y03nckz`.
- **Note**: `note_01m3w5vfwjej9s0hgfq5kmgf5m`, `about` eight project topics (kikan, rei, mori,
  mina, mori-rei-app, shikigami, keiro-runtime-kenshou, okf-profiles). This capture predates the
  role predicates. Today only the subject projects would be `about`, keiro-runtime-kenshou would
  be a `worked-example`, and the others would be `produced-in` or `consulted`.

A session on 2026-10-02 planned how Shikigami could monitor production services and conduct
multi-day deployment runbooks, using mls-service-v2's MP-10 repair window as the example.

- **Goal created** with `mina ci` in shikigami: `intention_01m3z3em6ceekt6e2rgrf10xd1`.
- **Note**: `note_01m3z3fm5rey39pezxytx27f3m`:
  - `about` shikigami, shiki, and kikan-en (the note records decisions about their contracts);
  - `produced-in` shikigami, kikan, shiki, and kikan-en;
  - `worked-example` mls-service-v2;
  - `consulted` keiro, okf-profiles, and dotfiles.nix.
- **Action**: `action_01m3z3etrweek9b5sbmckztz15`, `produced-in` the same four repositories.

## Important Notes

- **One confirmation gate**, and nothing is written before it.
- **Never ratify a leaning as a decision**; when unsure, file it under Leaning.
- **Never re-file an example**: link it to the goal and leave it where it is.
- **`about` means subject matter only.** A project the session merely read, edited, or used as an
  example gets the matching role edge, not `about`.
- **Use `mori://` URIs for artifacts in other repositories**, never bare paths or plan numbers.
- **Not yet available:** once Rei supports a `mori-ref` / `mori-ref-list` custom-property value
  type (an improvement request is being filed in rei), Produced artifacts should also be
  recorded as typed references on the action and note.

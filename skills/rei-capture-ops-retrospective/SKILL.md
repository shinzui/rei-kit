---
name: rei-capture-ops-retrospective
description: Capture the retrospective of a production or deployment session in Rei at its end — resolve the anchor intention from the session's `Intention:` commit trailers and MasterPlan/ExecPlan `intention:` frontmatter (asking only when they are missing or disagree), link other intentions with support or dependency edges, record one action summarizing the operational outcome, write a scrubbed retrospective note (Goal / UTC timeline / incidents / measurements / went well and badly / learnings / Decided vs Leaning / Open / evidence index) tagged ops-retrospective, relate it to each project by role (`about` for the service whose production behavior is the subject, `produced-in`, or `consulted` edges; resolved by mori:// reference), and verify by reading back. Use at the end of an agent session that released, rolled out, ran prod jobs, or fixed an incident; use rei-capture-session for a planning or design session instead.
allowed-tools: AskUserQuestion, Bash, Read
---

# Rei Capture Ops Retrospective

Turns a finished production or deployment session into Rei records the user can review later.
The day's activity view gets one **action** for the operational outcome, and one **note** holds
the retrospective: what was done in production and when, what broke and why, what was measured,
and what was learned. Both are anchored to the **intention** the operational work served, and
the note is linked to every project it involved, with an edge that says what role the project
played.

Use `rei-capture-session` for a planning or design session; this skill shares its discipline
(explicit IDs, Decided vs Leaning, one explicit action, one project edge per role, no
`scoped-to`, read back to verify) and differs in how the anchor is found and what the note
records. Use `rei-note-from-tmp` to file an arbitrary scratch note.

## Key Concepts

- **The anchor is usually already recorded.** Production work normally follows a plan, so the
  intention is explicit: an `intention:` field in the MasterPlan or ExecPlan frontmatter and an
  `Intention:` trailer on the session's commits. Resolve it from those first, and ask only when
  they are missing or disagree. This is the opposite of `rei-capture-session`, where the goal
  often has to be named.
- **Anchor to what the operational work served.** When several intentions appear (a hotfix for
  one initiative shipped alongside another's change), anchor to the one the session's production
  actions were for, and link the others with support or dependency edges. Never re-file them.
  Confirm with the user before creating a new intention.
- **A retrospective is evidence, not narrative.** Every production action gets a UTC time and an
  outcome. Every incident gets symptom, root cause, fix, and the evidence that the fix worked.
  Every measurement names the query or command that produced it, so it can be rerun.
- **Learnings are tracked to where they live.** Mark each generalizable lesson as promoted (to
  an ADR, plan, runbook or IR, with its `mori://` URI) or `note only`. A lesson that exists only
  in this note is a follow-up candidate, so it also goes under Open.
- **Decided vs Leaning.** Record as *Decided* only what the user explicitly chose (a repair
  window, fix-in-code over backfill). Options the agent recommended, or the user "leans"
  toward, go under *Leaning*. Never ratify a suggestion as a decision.
- **Sensitive data stays out.** Production sessions see credentials, connection strings, and
  customer data. The note holds only identifiers and aggregates: commit SHAs, tags, run ids,
  k8s job names, counts, latencies, memory figures. No secrets, passwords, tokens, connection
  strings with credentials, hostnames of private databases, or PII (names, emails, phone
  numbers, street addresses of customers or listings). Record shiki run ids rather than job log
  contents; quote an error class or message, not the payload around it.
- **Why an action too.** `rei today` and `rei yesterday` list actions, not notes. mori-rei-app
  records actions from `feat:`/`fix:` commit trailers, but releases, rollouts, holds, manual syncs
  and one-off jobs leave no commits, so without an explicit action the operational day is
  invisible.
- **Projects are topics with a mori:// reference.** `rei project show mori://NS/NAME` resolves
  a repository to its project topic.
- **One edge per project, chosen by the role it played:**

  | Role in the session | Edge | Example |
  |---|---|---|
  | Its production behavior is what the retrospective is about | `about` association | the service that was released and repaired |
  | Received commits, releases, or tags | `produced-in` edge | the service repo; a library patched upstream |
  | Read or used for root-causing or operating, not changed | `consulted` edge | a dependency's changelog and source; the job runner |

  A project can hold more than one role; the subject service usually gets both `about` and
  `produced-in`. `worked-example` rarely applies here, since the production work is the subject.
- **Don't add `scoped-to`.** The note inherits its scope from the anchor intention's hierarchy
  (check with `rei topic associations NOTE_ID --effective`).
- **Note tag.** Use `rei note new -t ops-retrospective`, so `rei note list -t ops-retrospective`
  lists retrospectives apart from planning sessions. If you ever filter on the `tags` tag-set
  property instead, use `--where 'tags any ops-retrospective'`.
- **AGENT** below means the actor name of the harness running the skill: `claude-code` or
  `codex`. Pass IDs explicitly on every command, because an omitted ID opens an fzf picker that
  hangs the run.

## Workflow

### 1. Reconstruct the Session

From the conversation, extract:

- **REPOS**: every repository that received commits, tags, or releases, with its local path.
- **GOAL**: what the production work was for, in one line.
- **TIMELINE**: each production action with its UTC time and outcome: release and tag,
  rollout, one-off job (shiki run id, k8s job name), hold or resume, manual sync, CronJob change,
  config change, rollback. Convert local times to UTC; mark a time `~` when it is reconstructed
  rather than observed.
- **INCIDENTS / FINDINGS**: symptom, root cause, fix (commit and release), verification
  evidence. Include issues caught before they reached production; they are near misses.
- **MEASUREMENTS**: before and after values, each with the query or command that produced it.
- **WENT WELL / WENT BADLY / NEAR MISSES**.
- **LEARNINGS**: each with where it was promoted (`mori://` URI) or `note only`.
- **DECIDED** and **LEANING** (with the alternative each was weighed against).
- **OPEN**: follow-ups, scheduled windows, distillation still owed.
- **EVIDENCE**: commit SHAs per repository, release tags, shiki run ids, plan and ADR `mori://`
  URIs.
- **PROJECTS**: each repository as `mori://NS/NAME`, with its role or roles.

### 2. Resolve the Anchor Intention

In each repository in REPOS, collect the `Intention:` trailers of the session's commits and
the plans they cite. Bound the range to the session (`--since` the session's start, or
`FIRST_SHA^..HEAD`):

```bash
cd REPO
git log --since="SESSION_START" --format='%h%x09%(trailers:key=Intention,valueonly,separator=%x2C)%x09%s'
git log --since="SESSION_START" --format='%(trailers:key=MasterPlan,valueonly)%n%(trailers:key=ExecPlan,valueonly)' | sort -u | grep .
```

For each cited plan, read its frontmatter `intention:`:

```bash
sed -n '1,/^---$/{/^intention:/p}' PLAN_PATH   # or: awk '/^---$/{n++} n==1 && /^intention:/' PLAN_PATH
```

Tally the intention IDs across trailers and frontmatter, then check each one:

```bash
rei intention show INTENTION_ID --json | jq -r '.intention | "\(.intentionId)\t\(.status)\t\(.title)"'
```

Decide:

- **One ID, consistent, active** → that is the anchor.
- **Several IDs** → the anchor is the one the production actions served; the others become
  support (it advanced the anchor) or dependency (the anchor could not finish without it) links.
- **Missing, unknown to Rei, completed, or in conflict** (trailers say A, the plan says B) →
  ask. Offer a keyword search as the fallback:

  ```bash
  rei intention list --all -s "KEYWORD" --json | jq -r '.[] | "\(.id)\t\(.title)"'
  ```

  If nothing fits, propose a new intention. In a repository whose `mina.kdl` declares
  `rei { create { default-parent … } }`, create it with `mina ci` (title **without** the
  prefix); otherwise `rei intention create` with an explicit `--parent`.

### 3. Draft and Scrub the Note

Write the note to a scratch file first. The first heading becomes the title.

```markdown
# Ops retrospective YYYY-MM-DD: <outcome in a few words>

## Goal
<what the production work was for; the plan it served as a mori:// URI>

## Timeline (UTC)
| Time | Action | Outcome |
|---|---|---|
| HH:MM | Released vX.Y.Z (tag) | rolled out, healthy |
| HH:MM | shiki run `<run-id>` / job `<k8s-job>`: <what> | <result> |

## Incidents and findings
### <short name>
- **Symptom:** …
- **Root cause:** …
- **Fix:** <commit / release>
- **Verified by:** <run id, query result, metric>

## Measurements
| Metric | Before | After | Source |
|---|---|---|---|
| … | … | … | `<query or command>` |

## What went well
## What went badly
## Near misses

## Learnings
- <lesson> — promoted: `mori://…` | note only

## Decided
- <explicit user choice> — <artifact mori:// URI when one records it>

## Leaning (not decided)
- <favoured option> over <alternative> — <why>

## Open
- <follow-up or next step>

## Evidence
- <repo> `mori://NS/NAME`: commits `<sha>`, `<sha>`; tags `vX.Y.Z`
- shiki runs: `<run-id>` (<what>)
- plans / ADRs: `mori://…`
```

Omit an empty section rather than writing "none", except Decided and Open, which are always
shown so their absence is visible.

Then scrub. Read the draft once looking for secrets and PII, and run a pattern check; review
every hit and remove or replace it with an identifier or aggregate:

```bash
grep -nEi 'password|passwd|secret|token|api[_-]?key|bearer |authorization:|BEGIN [A-Z ]*PRIVATE KEY|[a-z]+://[^/ ]*:[^@ ]*@|[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[a-z]{2,}' NOTE_FILE
```

A hit that is only a word (for example "rotated the token") may stay; a value never does.

### 4. Confirm

Make one AskUserQuestion call with two questions:

1. **Anchor intention.** The resolved intention marked Recommended, with where it came from
   ("trailers on 12 commits and MasterPlan 10 frontmatter"), plus any alternatives or "Create
   new: <title> under <parent>". When the sources agreed, this is a confirmation, not a search.
2. **Plan.** Show the action text, the note outline (Decided and Leaning visibly separate), the
   support or dependency links, and each project topic with the edge or edges it will get.
   Options: apply (Recommended), let me adjust, cancel.

On cancel, write nothing.

### 5. Create the Intention (only if new)

```bash
cd REPO_WITH_MINA_KDL && mina ci --json "TITLE" | jq -r .intentionId
# or
rei --actor AGENT intention create "TITLE" --parent PARENT_ID
rei intention show INTENTION_ID      # confirm title, parent, category
```

### 6. Link the Other Intentions

```bash
# the other intention's work advanced the anchor
rei --actor AGENT support add --from OTHER_ID --to ANCHOR_ID --note "ONE-LINE WHY"
# the anchor cannot finish until the other intention does
rei --actor AGENT dependency add --from ANCHOR_ID --on OTHER_ID --note "ONE-LINE WHY"
```

### 7. Record the Action

One action for the operational outcome, naming the releases and what they fixed:

```bash
rei --actor AGENT action record -i ANCHOR_ID --no-habit "Released v9.14.3–v9.14.5; fixed …"
```

Capture `ACTION_ID` with `grep -o 'action_[0-9a-z]*' | head -1`. Add `-d DURATION` only when the
user states how long the session took, and `--at TIME` when capturing after the day the work
happened.

### 8. Store the Note

```bash
rei --actor AGENT note new --intention ANCHOR_ID --stdin -t ops-retrospective < NOTE_FILE
```

Capture `NOTE_ID`. A non-zero exit means nothing was stored, so stop and report.

### 9. Relate the Note and Action to Each Project

The role predicates must exist. Check once per run:

```bash
for p in produced-in consulted; do rei predicate show "$p" >/dev/null 2>&1 || echo "missing: $p"; done
```

If one is missing, define it with the definitions in `rei-capture-session` step 8.

For each `mori://NS/NAME` in PROJECTS, resolve its topic:

```bash
rei project show mori://NS/NAME --no-mori --json | jq -r .identity.topicId
```

Exit code `2` with "No project matches" means Rei has no topic for that project. Offer to sync
it (`rei --actor AGENT project sync mori://NS/NAME`, which needs `MORI_API_URL`) rather than
guessing a topic key.

Then write one edge per role, each graph edge with a short `-d`:

```bash
# the service whose production behavior is the subject
rei --actor AGENT topic associate TOPIC_ID NOTE_ID --relation about
# received commits or releases: on the note and on the action
rei --actor AGENT edge add -f NOTE_ID -t TOPIC_ID -p produced-in -d "v9.14.3–v9.14.5"
rei --actor AGENT edge add -f ACTION_ID -t TOPIC_ID -p produced-in -d "v9.14.3–v9.14.5"
# read or used for root-causing or operating, not changed
rei --actor AGENT edge add -f NOTE_ID -t TOPIC_ID -p consulted -d "0.11.1 changelog: parBuffered deadlock fix"
```

### 10. Verify

```bash
rei note show NOTE_ID --json | jq '{title, anchor, tags}'
rei topic associations NOTE_ID --relation about --json | jq -r '.[].topic.key'
rei edge show NOTE_ID | grep -E 'produced-in|consulted'
rei edge show ACTION_ID | grep produced-in
rei action list -i ANCHOR_ID --today
```

Add `rei support supported-by ANCHOR_ID` / `rei dependency …` checks when step 6 wrote links.
Redo any write that did not land, once. Otherwise report it.

## Output Format

```
## Ops Retrospective Captured

- **Anchor**: <intention_id> — <title> (from trailers + plan frontmatter | chosen | created under <parent>)
- **Action**: <action_id> — <summary>
- **Note**: <note_id> — <title> (tag ops-retrospective)
- **Linked**: <id> supports anchor | anchor depends on <id> | none
- **About**: <subject topic keys>
- **Produced in**: <topic keys> (note and action)
- **Consulted**: <topic keys> | none
- **Learnings not yet promoted**: <count> (listed under Open)
- **Skipped / Failed**: <item — reason> | nothing

Review later: `rei yesterday` · `rei note list -t ops-retrospective` · `rei note show <note_id>`
```

## Worked Examples

A session on 2026-10-02 in mls-service-v2 released v9.14.3–v9.14.5 and fixed two production
problems in the service's property synchronization.

- **Anchor**: `intention_01m28xzencejcb1f53emz9hj1v`, read from the `intention:` frontmatter of
  `mori://tan/mls-service-v2/masterplans/10-isolate-mls-identity-from-constellation1-key-drift-and-repair-the-corrupted-bareis-streams`
  and the `Intention:` trailer on every session commit. The sources agreed, so the question
  was a confirmation.
- **Projects**: `mori://tan/mls-service-v2` gets `about` and `produced-in`;
  `mori://composewell/streamly` gets `consulted` (the fix was found in its changelog and
  source); `mori://shinzui/shiki` gets `consulted` (used to run and track the prod one-off jobs).
- **Incident 1**: the MLSPIN property sync crashed daily with `BlockedIndefinitelyOnMVar`. Root
  cause: the new mint budget tripped inside streamly 0.11.0's `parBuffered` worker, deadlocking
  the driver and hiding the original error. Fix: streamly 0.11.1 (v9.14.3). MLSPIN was then
  caught up with an audited `--mint-budget 8000` shiki run, recorded by run id.
- **Incident 2**: synchronization ran 2.5× slower since v9.14.0, about 28 DB round trips per
  listing. Fix: per-page identity resolution
  (`mori://tan/mls-service-v2/plans/113-resolve-c1-references-once-per-page-so-synchronization-regains-its-pre-resolver-throughput`,
  v9.14.4). **Near miss**: v9.14.4 OOMed in a bounded manual run because the pages-ahead buffer
  now held pages rather than listings; it was caught before the scheduled unbounded run and
  fixed in v9.14.5.
- **Measurements**: 60–75 ms per listing before, 18–34 ms after; peak memory 353 MiB after the
  fix, against an OOM at the 4 GiB limit before it. Each row names the query or command.
- **Learnings**, all `note only` at capture time: check a dependency's newer patch releases when
  an error comes from library internals; a concurrent stage's buffer counts elements, so
  batching changes its memory bound; verify with a bounded manual run before a scheduled
  unbounded one; C1 bulk-touching MLSes is normal load, not an incident.
- **Decided**: the BAREIS repair window is Saturday 15:00 UTC; the speed fix is done in code,
  not by the backfill.
- **Open**: the EP-6 BAREIS repair window; ExecPlan 113 ADR distillation; promoting the
  learnings above.
- **Action**: "Released v9.14.3–v9.14.5; fixed MLSPIN sync deadlock and post-v9.14 sync
  slowdown".

## Important Notes

- **One confirmation gate**, and nothing is written before it.
- **Never store secrets or PII.** Scrub before storing; identifiers and aggregates only. A note
  cannot be un-seen once synced, so when in doubt, leave it out.
- **Every timeline entry is UTC with an outcome**; every measurement names its source.
- **Never ratify a leaning as a decision**; when unsure, file it under Leaning.
- **Never re-file another intention**: link it to the anchor and leave it where it is.
- **`about` means subject matter only.** A library read for root-causing or a tool used to run
  jobs gets `consulted`, not `about`.
- **Use `mori://` URIs for artifacts in other repositories**, never bare paths or plan numbers.

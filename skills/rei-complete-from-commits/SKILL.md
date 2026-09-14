---
name: rei-complete-from-commits
description: Backlog cleanup — for a parent Rei intention and one or more git repositories, match open child intentions to commits by their `Intention:` trailers and bulk-complete each with `--at` set to the latest matching commit's author timestamp; optionally infer dates for children with no trailer matches from pre-trailer commit subjects, then verify every completion by reading it back.
allowed-tools: AskUserQuestion, Bash, Read
---

# Rei Complete From Commits

Closes out shipped-but-still-open child intentions by mining git history. Commit trailers are
the authoritative work-to-intention mapping; this skill turns that mapping back into Rei state
with backdated completions.

## When to Use

- "Close out the children of <intention> based on git commits"
- "Backdate completions from commit trailers"
- "Clean up the backlog under <intention> — some of its commits live in `mori` and some in `rei`"
- "/rei-complete-from-commits"

## Key Concepts

- **Trailer convention** — commits carry `Intention: intention_...` trailers. A commit may
  carry several (multiple trailer lines, or comma-separated values).
- **Authoritative timestamp** — the **author date** of the latest trailer-matching commit.
  `git log --format=%aI` gives ISO 8601 with offset (`2026-04-25T13:03:21-07:00`); pass it to
  `--at` verbatim. Rei parses offsets exactly. Don't use date-only forms: `rei help time`
  resolves `2026-04-25` to that date *at the current time of day*, not the moment of the work.
- **Adoption date** — trailers were adopted at some point; earlier commits have none. The
  earliest trailer-bearing commit approximates that boundary, and pre-boundary work can only be
  matched by subject keywords (medium confidence, confirmed per child).
- **TypeID prefix ≠ work date** — an intention's ID encodes when it was *created* in Rei, often
  retroactively. Trust commit timestamps, never the ID.
- **`rei intention complete` is not covered by the automation exit contract** — it may print an
  error and still exit `0`. Success is established only by reading `completedAt` back.

## Workflow

### 1. Resolve inputs

Needed: parent intention ID, repo path(s), and whether to include `future` children (default:
active only). Use what the user gave; default the repo to the current working directory and ask
only for what is missing. Search for the parent if given a title:

```bash
rei intention list --all -s "KEYWORD" --json | jq -r '.[] | "\(.id)\t\(.title)"'
```

If the parent is scoped to a software project, its checkouts are a good repo suggestion:

```bash
rei topic associations PARENT_ID --relation scoped-to --effective --json
rei project show PROJECT --with-mori --json   # then `mori path <mori-uri>` for the checkout
```

Confirm each repo: `git -C REPO rev-parse --git-dir`.

### 2. Enumerate children

Always pass the ID explicitly (an omitted ID opens an fzf picker):

```bash
rei intention show PARENT_ID --json \
  | jq -r '.children[] | select(.status == "active") | "\(.intentionId)|\(.status)|\(.title)"'
```

`.children` holds only **direct, not-yet-completed** children (`active` / `future`). Use
`select(.status == "active" or .status == "future")` if the user opted into future ones. For
grandchildren, repeat on each child only if the user asks.

**Never include the parent in the completion set** — strip `PARENT_ID` defensively, even if the
user asks to complete it too; that is a separate command for them to run after reviewing. If
there are no candidate children, stop.

### 3. Index commits (once per repo)

```bash
git -C REPO log --all \
  --format='%H|%aI|%(trailers:key=Intention,valueonly,separator=%x2C)' \
  > "$SCRATCH/commit-intentions-REPO_NAME.txt"
```

Use the session scratchpad (or a `mktemp -d`) rather than a fixed `/tmp` path. For multiple
repos, build one index each and match across all of them.

### 4. Match children via trailers

For each child, the latest matching timestamp and match count:

```bash
awk -F'|' -v id="CHILD_ID" '
  { n = split($3, ts, ","); for (i = 1; i <= n; i++) { gsub(/ /, "", ts[i]); if (ts[i] == id) { print $2; break } } }
' "$SCRATCH"/commit-intentions-*.txt | sort -r | awk 'NR==1{latest=$0} END{print NR "|" latest}'
```

`sort -r` on `%aI` is only reliable within one timezone offset. If the repos or authors mix
offsets, compare as instants (e.g. convert with `date -j -f` / `gdate -d` to epoch) before picking
the latest.

### 5. Unmatched children (optional)

Ask once how to handle children with zero trailer matches: infer from pre-trailer subjects
(recommended), skip, or complete now. Never default to "now" silently.

To infer, find the adoption boundary and search before it with salient title keywords (drop
emoji/kanji prefixes like `森`, `怜`, `工程` and generic verbs like "Add", "Support"):

```bash
awk -F'|' '$3 != "" { print $2 }' "$SCRATCH"/commit-intentions-*.txt | sort | head -1
git -C REPO log --all --before='ADOPTION_ISO' --format='%aI|%s' -i --grep='KEYWORD' | head -20
```

Show the candidate commits per child and let the user pick a timestamp, give another, or skip.
A correct skip beats a wrong completion.

### 6. Confirm the plan

Show every child with its decision, then ask for approval before any write. This gate is
mandatory for every batch, even if the user approved an earlier one.

| Intention | Source | Commits | Completion timestamp |
|---|---|---|---|
| <title> | trailer | 8 | 2026-04-25T13:03:21-07:00 |
| <title> | inferred ("<subject>") | 0 | 2026-02-12T05:52:11-08:00 |
| <title> | skipped | 0 | — |

Options: run all / let me edit / cancel.

### 7. Execute

Write the approved rows to `"$SCRATCH/completions.txt"` as `id|iso_timestamp|title`, then run
sequentially:

```bash
while IFS='|' read -r id at title; do
  echo ">>> $title ($id) @ $at"
  rei --actor claude-code intention complete "$id" --at "$at" 2>&1 | tail -3
done < "$SCRATCH/completions.txt"
```

Don't retry failures silently; report them.

### 8. Verify

Read each completion back — this, not the exit status, is the success signal:

```bash
while IFS='|' read -r id at title; do
  rei intention show "$id" --json | jq -r --arg id "$id" '"\($id)|\(.intention.status)|\(.intention.completedAt)"'
done < "$SCRATCH/completions.txt"
```

A row passes when `status` is `completed` and `completedAt` equals the planned instant
(`completedAt` is UTC, e.g. `2026-04-25T20:03:21Z` for `13:03:21-07:00`). Then list what remains
open under the parent (`rei intention show PARENT_ID --json | jq '.children'`).

## Output Format

```
## Completed N intentions under <Parent Title>

### Trailer-matched (M)
- <title> — completed @ <iso> (latest of <count> commits)

### Inferred from pre-trailer commits (K)
- <title> — completed @ <iso>, source: "<commit subject>"

### Skipped / Failed (S)
- <title> — <reason, incl. any verification mismatch>

### Still open under <Parent Title>
- <title> (<intention_id>) — <status>
```

## Important Notes

- **Never complete the parent intention.**
- **Confirm before any write**; completions are bulk and not trivially undone.
- **Index each repo once**, not once per child.
- **Latest timestamp wins** across all repos and trailers for the same intention.

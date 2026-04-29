---
name: rei-complete-from-commits
description: Backlog cleanup — for a given Rei intention and git repository, find commits whose `Intention:` trailer matches the intention's children (or the intention itself), then bulk-complete those intentions with `--at` set to the latest matching commit timestamp. Optionally infer completion dates for children with no trailer matches by searching pre-trailer commit messages.
allowed-tools: AskUserQuestion, Bash, Read
---

# Rei Complete From Commits

This skill closes out shipped-but-still-open Rei intentions by mining git history. For a parent intention and a git repository, it walks the children, scans commits for `Intention: <id>` trailers, and bulk-completes each child with `rei intention complete --at <latest-commit-iso-timestamp>`.

It's the right tool when an intention's children represent shipped work but the intention list has accumulated as a backlog rather than an active working set. The user adopts commit trailers as the authoritative work-to-intention mapping; this skill turns that mapping back into Rei state.

## When to Use

Activate when the user says things like:
- "A lot of children of intention X have been completed — close them out"
- "Complete the children of <intention> based on git commits"
- "Mark intentions as done based on their commit trailers"
- "Backdate completions from git history"
- "Clean up the backlog under <intention> using commit trailers"
- "/rei-complete-from-commits"

## Key Concepts

### Intention trailer convention

Commits include a trailer line like:

```
Intention: intention_01kq30m0j5efta9z5entwe7cz9
```

The trailer is the authoritative link between a commit and the intention it advances. A single commit can carry multiple `Intention:` trailers (comma-separated when multiple).

### Adoption date

The user usually adopts trailers at a specific point in time. Commits before that date have no trailer — but the work they represent may still belong to a child intention. For those, the skill falls back to keyword-matching commit subjects (with the user's confirmation per intention).

### Authoritative timestamp

The completion time is the **author date** of the latest commit referencing the intention via trailer. Use ISO 8601 with timezone (e.g., `2026-04-25T13:03:21-07:00`) so `rei intention complete --at` records it precisely, not midnight UTC.

## Workflow Overview

1. **Resolve inputs** — Parent intention ID, git repo path, optional child filter
2. **Enumerate children** — Pull child intentions from `rei intention show --full`
3. **Map commits to intentions** — Build an index of every commit's intention trailers
4. **Match each child** — Find latest trailer-matching commit per child
5. **Inferred dates for unmatched** — Optionally guess from pre-trailer commit subjects
6. **Confirm the plan** — Show full table of intention → date → reasoning
7. **Execute** — Loop `rei intention complete --at` for each
8. **Verify** — Show parent's remaining open children

## Instructions for Claude

### Phase 1: Resolve Inputs

If the user already named a parent intention and is invoking the skill from inside a git repo, default to those and confirm. Otherwise ask:

```
Question: "Which parent intention's children should I close out?"
Header: "Parent"
Options:
- Let me paste the intention ID
- I'll list intentions first (run `rei intention list`)
```

```
Question: "Which git repository should I scan for Intention: trailers?"
Header: "Repo"
Options:
- The current working directory (Recommended)
- Let me paste a path
```

Confirm the repo is a git repo:

```bash
git -C <repo> rev-parse --git-dir 2>/dev/null
```

If the user wants to scan **multiple repos**, repeat the trailer-mapping step (Phase 3) for each and merge by intention ID, keeping the latest timestamp wins.

### Phase 2: Enumerate Children

Pull the parent's structure:

```bash
rei intention show <PARENT_ID> --full
```

Parse the **Child Intentions** block. Each line looks like:

```
  ● Title here (intention_xxxxxxxxxxxxxxxx)
  ◌ Future-status title (intention_xxxxxxxxxxxxxxxx)
```

Collect `(id, title, status)` tuples. By default, only consider **active** children (`●`); skip already-completed ones. Ask the user if they want to include `future` (`◌`) status too.

**Never include the parent intention itself in the completion set.** The parent is a scope marker for this run, not a target. Strip the parent ID from the candidate list defensively — even if the user pastes it again, even if it appears as its own ancestor through some data anomaly, even if the user explicitly asks "and complete the parent too." If they want to close the parent, that's a separate `rei intention complete <PARENT_ID>` they should run themselves after the children are done.

If the parent has no active children, stop and tell the user there's nothing to close — do not fall through to "well, complete the parent then."

### Phase 3: Map Commits to Intentions

Run **once** for the repo and stash the result — this is the expensive step:

```bash
git -C <repo> log --all \
  --format='%H|%aI|%(trailers:key=Intention,valueonly,separator=,)' \
  > /tmp/rei-commit-intentions.txt
```

Each line is `commit_sha|iso_author_date|intention_id_1,intention_id_2,...` (trailers field is empty for commits without one).

### Phase 4: Match Each Child via Trailers

For each child, find the **latest** commit whose trailer list contains its ID:

```bash
awk -F'|' -v id="<CHILD_ID>" '
  {
    n = split($3, ts, ",")
    for (i = 1; i <= n; i++) {
      if (ts[i] == id) { print $2; break }
    }
  }
' /tmp/rei-commit-intentions.txt | sort -r | head -1
```

Record `(child_id, title, commit_count, latest_iso_timestamp)`. A commit count of 0 means no trailer matches — handle in Phase 5.

### Phase 5: Inferred Dates for Unmatched Children

For children with zero trailer matches, ask the user what to do:

```
Question: "Some children have no Intention: trailer matches — likely shipped before trailers were adopted. How should I handle them?"
Header: "Unmatched"
Options:
- Try to infer dates from commit subjects before the trailer-adoption date (Recommended)
- Skip them (leave open)
- Complete them at today's date
```

If inferring, find the **earliest** trailer-bearing commit (the rough adoption boundary):

```bash
awk -F'|' '$3 != "" { print $2 }' /tmp/rei-commit-intentions.txt | sort | head -1
```

Then for each unmatched child, search commit subjects **before that date** for keywords drawn from the child's title:

```bash
git -C <repo> log --all --before='<ADOPTION_ISO>' \
  --format='%aI|%s' --grep='<keyword>' -i | head -20
```

Pick keywords aggressively (split title into salient words, drop emoji prefixes like `森`, `怜`, `渋谷`, drop generic words like "Add", "Support", "Build"). Show the candidate commits to the user and ask:

```
Question: "For child '<title>', what completion date should I use?"
Header: "<title-truncated>"
Options:
- <best-guess ISO date> from "<best matching commit subject>" (Recommended)
- A different date — let me paste it
- Skip this one
```

Treat inferred dates as **medium confidence** and call them out in the final summary so the user can override.

### Phase 6: Confirm the Plan

Build a table covering every child to be completed and show it to the user:

| Intention | Source | Commits | Completion timestamp |
|---|---|---|---|
| <title> | trailer | 8 | 2026-04-25T13:03:21-07:00 |
| <title> | inferred | 0 | 2026-02-12T05:52:11-08:00 |
| <title> | skipped | 0 | — |

End with an explicit confirmation prompt — never run `rei intention complete` until the user has approved the full list:

```
Question: "Run the completion loop for the N intentions above?"
Header: "Run"
Options:
- Yes, complete them all
- Let me edit the list first
- Cancel
```

### Phase 7: Execute the Loop

Write the approved plan to a file (so the loop is auditable and re-runnable on partial failure):

```bash
cat > /tmp/rei-completions.txt <<'EOF'
<intention_id>|<iso_timestamp>|<title>
...
EOF
```

Run the completions sequentially and capture per-row output:

```bash
while IFS='|' read -r id at title; do
  echo ">>> Completing: $title"
  echo "    ID: $id  AT: $at"
  rei -a claude-code intention complete "$id" --at "$at" 2>&1 | tail -3
  echo ""
done < /tmp/rei-completions.txt
```

If any row fails, surface it clearly — do not retry silently.

### Phase 8: Verify

Re-show the parent and confirm only the intended remainders are still open:

```bash
rei intention show <PARENT_ID> --full
```

Highlight any children that are still active and were not in the completion list — those are the genuinely-not-yet-done remainders.

## Output Format

After the loop, summarize:

```
## Completed N intentions under <Parent Title>

### Trailer-matched (M)
- <title> — completed @ <iso> (latest of <count> commits)
- ...

### Inferred from pre-trailer commits (K)
- <title> — completed @ <iso>, source: "<commit subject>"
- ...

### Skipped (S)
- <title> — <reason>

### Still open under <Parent Title>
- <title> (intention_id) — <status>
```

## Important Notes

- **Never complete the parent intention.** The parent is the scope marker for this run, never a target. Strip the parent ID from the candidate list before Phase 6, regardless of what the user says. This is a hard rule — if the user wants to close the parent, they can run `rei intention complete <PARENT_ID>` themselves once they've reviewed the closed children.
- **Always include `--actor claude-code`** on `rei intention complete` (use `rei -a claude-code …`).
- **Confirm before any mutation.** This skill performs bulk irreversible writes; never skip the Phase 6 confirmation, even if the user said "yes" to a previous batch.
- **One commit-trailer scan per repo.** Building the index is the expensive step — do it once per Phase 3, not once per child.
- **TypeID prefix ≠ work date.** A child intention's TypeID prefix tells you when the *intention* was created in Rei, not when its work was done. The intention may have been created retroactively to attribute already-shipped work. Trust the commit timestamp, not the ID.
- **Multiple trailers per commit are real.** A commit can carry several `Intention:` trailers — split by comma when matching.
- **Use ISO 8601 with timezone.** `git log --format=%aI` already emits this. Pass it through verbatim to `--at`. Avoid date-only forms; they collapse to midnight UTC and can land on the wrong calendar day.
- **Status filter.** Default to active children. Include `future` only if the user opts in.
- **Multiple repos.** When multiple repos contribute commits for the same intention, keep the latest timestamp across all of them — that's still the most recent shipping signal.
- **Skipping is fine.** If trailer matches are zero and the user can't confidently infer a date, leave the intention open. A correct skip is better than a wrong completion that the user has to undo.

## Example Invocations

**Parent + current repo (typical):**
> "Close out the children of `intention_01kf57t4hje3et4awqv5kggqkz` based on commits in this repo."

**Parent + explicit repo:**
> "Complete the children of <intention> based on commits in `~/code/myproj`."

**Parent + multiple repos:**
> "Some of these children's commits live in `mori` and some in `rei` — scan both."

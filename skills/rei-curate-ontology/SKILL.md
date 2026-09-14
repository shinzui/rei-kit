---
name: rei-curate-ontology
description: Build and maintain a Rei knowledge ontology — topics, predicates, associations, and edges — scoped tightly to the user's actual use cases. Surveys what exists, reuses it, creates only what's missing (classifying instances with instance-of, placing concepts with broader-than, naming new predicates from Schema.org), connects existing links and notes, proposes pruning and is-a → instance-of cleanup, and deliberately refuses to model the whole world.
allowed-tools: AskUserQuestion, Bash, Read
---

# Rei Curate Ontology

Builds and maintains the ontology that organizes the user's links, notes, and docs in Rei:
**topics** (subject nodes), **predicates** (relationship types), **associations** (validated
entity → topic relations), and **edges** (open typed graph facts). It surveys what exists and
extends it for the user's concrete use case — and **only** that use case.

**The single most important rule: do not overengineer.** The ontology models the user's real
interests, not the world. If the user tracks two git forges (`github`, `gitdot`), the ontology
gets two forge topics — not GitLab, Bitbucket, SourceHut, and every other forge. Usefulness to
*this user* is the only goal; coverage of a domain is a non-goal.

Use `rei-bookmark-url` to file a single URL (it grows the ontology just enough for that link and
follows the same modeling rules). Use this skill to design, extend, restructure, or prune.

## Key Concepts

- **Topic** — a subject node with a stable lowercase `KEY`, a `LABEL`, an optional description,
  a `topic_...` ID, and optional external references and custom properties.
- **Attachment** (filing) — a note, doc, or link stored under a topic (`--topic` at creation,
  or `set-topic` later). Listed by `rei topic attachments TOPIC`.
- **Association** — a validated relation written by `rei topic associate TOPIC ENTITY`:
  `about` (subject matter), `scoped-to` (operational membership — **the default**), and
  `instance-of` (topic → topic classification). Argument order is **target-first**:
  `associate ui-libraries topic_react --relation instance-of` reads *react instance-of
  ui-libraries*. Associations verify both endpoints and follow the automation exit contract
  (`2` refused/invalid, `70` store failure).
- **Attachment ≠ association.** Filing creates no `about` row; an `about` row files nothing.
- **Edge** — `rei edge add -f ID -t ID -p KEY`, the open graph for everything the association
  commands don't write: `broader-than`, `related-to`, `is-a`, borrowed predicates. Takes full
  IDs, never keys.
- **Seeded by `rei ontology seed-system`** (idempotent; `seed-predicates` is an alias): `about`,
  `scoped-to`, `instance-of`, `broader-than` (transitive, inverse `narrower-than`),
  `narrower-than`, `related-to` (symmetric), plus the system `project` topic and project review
  properties.
- **Classification vs. subsumption — the #1 modeling mistake.**
  - **`instance-of`**: a concrete, named thing (tool, product, library, service, forge, model,
    project) → its type. `github` instance-of `git-forge`.
  - **`broader-than`**: subject subsumption between two **abstract** subjects only.
    `version-control` broader-than `git-forge`.
  - Litmus test: could you put a homepage on it? → instance → `instance-of`. Is it another
    field you'd browse? → concept → `broader-than`. Never hang a named thing under a category
    with `broader-than`. Classify distinct instances into their distinct correct types (React →
    `ui-libraries`, StyleX → `css-in-js`), not one generic bucket.
- **`is-a` legacy** — before `instance-of` was seeded, classification used a workspace-defined
  `is-a` edge predicate. Existing workspaces may hold many `is-a` edges. New classification uses
  `instance-of`; see Phase 2 and Phase 7 for handling the divergence.
- **External reference** — `rei topic add-ref TOPIC SCHEME://KEY` records a topic's canonical
  identity (homepage, `mori://` URI, DOI, Schema.org IRI). One active reference belongs to at
  most one topic, so `rei topic ref-show URI` (exits nonzero when unowned) is the duplicate-topic
  detector.
- **Projects** — a project is just a topic `instance-of project`. `rei project create KEY
  --label LABEL` for any durable effort; `rei project sync mori://NS/PROJECT` for a Mori
  software project (creates the topic with a verified reference). Don't hand-build these. The
  old `local-repo` property is deprecated in favor of project scope.
- **Inference** is query-time only (`--include-inferred` on `topic related` / `topic tree`);
  never write both directions of an inverse pair.
- **Validation** — `rei ontology validate [--json]` checks active edges against predicate
  source/target types (JSON: `[]` when clean).

## Scoping Discipline

1. **Model use cases, not domains.** Start from "what does the user want to find again?".
2. **Reuse before create** — search topics by key, label, and `ref-show`; reuse predicates
   verbatim.
3. **Borrow vocabulary before inventing it** — new predicates are named from Schema.org
   (Phase 4), taking only the one term needed.
4. **Keep hierarchies shallow** — one or two `broader-than` levels is usually enough.
5. **Prefer attachments + `about` over new predicates.** Define a predicate only for a recurring
   relationship the seeded set can't express; push back on one used by a single edge.
6. **Defer, don't pre-build** — a topic with no content is premature.
7. **Confirm before writing**, always showing an explicit out-of-scope list.

## Workflow

Every write uses the global `rei --actor claude-code <command>`, and every ID/key is passed
explicitly (omitted arguments open fzf pickers). Topic, edge, and predicate writes predate the
exit contract and can print an error yet exit `0` — confirm them by reading back (Phase 9).

### Phase 1: Understand the Use Case

Ask what the user wants to organize and find again. Push until you can state the subject
area(s), the specific things inside it they care about, and what is explicitly **out of scope**.
Restate the boundary in one or two sentences before proceeding, e.g.:

> "In scope: the two forges you use (GitHub, gitdot) and the workflow concepts you compare them
> on. Out of scope: every other forge, and Git internals you didn't mention."

If the request is vague ("organize everything"), pick the one or two areas with the most
existing content (counted in Phase 3).

### Phase 2: Investigate the Existing Ontology

```bash
rei topic list --json | jq -r '.[] | "\(.topicKey)\t\(.topicLabel)\t\(.topicId)"'
rei predicate list --json | jq -r '.[] | "\(.predicateKey)\t\(.sourceTypes)\t\(.targetTypes)"'
rei ontology validate --json
for p in instance-of is-a broader-than related-to about; do
  printf '%s\t' "$p"; rei edge list -p "$p" --json | jq length
done
```

For overlapping topics and intended types/parents:

```bash
rei topic show TOPIC --json
rei topic refs TOPIC
rei topic attachments TOPIC --json          # .attachments
rei topic edges TOPIC --json                # .graphEdges.incoming / .outgoing[].predicateKey
rei topic entities TOPIC --relation instance-of --json   # rows: .entity.id, .topic.key
rei topic tree TOPIC --direction narrower --include-inferred
```

Record: topics to reuse, predicates present, pre-existing validation errors, and the
**classification convention** (how many `is-a` vs `instance-of` edges, and which one each
relevant type's instances use).

Building fresh → focus on Phases 5–8. Maintaining → also collect cleanup candidates:
near-duplicate topics (same subject, different keys; `ref-show` collisions), orphans (no
classification, no parent, no attachments or associations), topics outside the user's current
use, instances wrongly placed under `broader-than`, and `is-a` edges that could become
`instance-of`. Propose these; never apply silently.

### Phase 3: Survey the Knowledge to Be Organized

```bash
rei link list --all -q "TERM" --json
rei note list --title "TERM" --json
rei custom-property entities tags --json \
  | jq -r '.entities[].value' | tr ',' '\n' | sed 's/^ *//; s/ *$//' | sort | uniq -c | sort -rn
```

A tag shared by many artifacts is a good topic candidate. A proposed topic with zero matching
content should usually be deferred.

### Phase 4: Ensure the Predicates

```bash
rei ontology seed-system
```

**Classification relation.** Use `instance-of` (via `rei topic associate`). Only if a type's
existing instances are wired with `is-a` should new instances of that same type follow `is-a`
for consistency — and mention the divergence; migration is a Phase 7 proposal. Never define
`is-a` in a workspace that doesn't already have it.

**A new predicate** is warranted only for a recurring relationship the seeded set can't express
("built on", "supersedes", "part of"). Check `rei predicate list` first (note: `rei predicate
show KEY` exits `0` even when the key is missing — test existence via `predicate list --json |
jq -e`). Then name it from Schema.org, vendored at `mori://schemaorg/schemaorg`:

```bash
SCHEMAORG=$(mori path mori://schemaorg/schemaorg)
(cd "$SCHEMAORG" && just find-prop "built on")      # properties by name OR description — search in plain words
(cd "$SCHEMAORG" && just find "design system")      # types
(cd "$SCHEMAORG" && just show SoftwareApplication)  # one type in full
```

`find-prop` flags `[pending]` but not supersession, so dump the candidate's row:

```bash
(cd "$SCHEMAORG" && python3 - runtimePlatform <<'EOF'
import csv, sys
csv.field_size_limit(10_000_000)
for r in csv.DictReader(open('schemas/schemaorg-current-https-properties.csv')):
    if r['label'] == sys.argv[1]:
        print(f"{r['label']}  <{r['id']}>\n{r['comment']}")
        for k in ('inverseOf', 'supersededBy', 'isPartOf', 'domainIncludes', 'rangeIncludes'):
            print(f"{k}: {r[k] or '(empty)'}")
EOF
)
```

Vet it:

| Signal | Action |
|---|---|
| `supersededBy` non-empty | adopt the successor (`runtime` → `runtimePlatform`) |
| `isPartOf` = `https://pending.schema.org` | usable, but say it is pending; prefer a core term if one fits |
| absent from the current CSV | retired — treat as no match |

Translate only that one term:

| Schema.org | `rei predicate define` |
|---|---|
| camelCase `label` | kebab-case `KEY` (`runtimePlatform` → `runtime-platform`) |
| `comment` | `-d`, one line, with the term IRI appended |
| `inverseOf` | `--inverse-of`, only if that direction is actually needed |
| `domainIncludes` / `rangeIncludes` | **don't copy** — set `--source-types`/`--target-types` (rei entity types: `topic`, `note`, `link`, `doc`, …) from how the edges actually run; `validate` enforces them |

```bash
rei --actor claude-code predicate define runtime-platform -l "Runtime platform" \
  -d "Runtime platform or script interpreter dependency (schema.org/runtimePlatform)" \
  --source-types topic --target-types topic
```

Never import a type's property list or subtype tree. Schema.org has no subsumption term —
`broader-than`/`narrower-than` stay rei-native. If nothing fits without distorting the meaning,
define a rei-native key and say in the description that no standard term applied.

### Phase 5: Design the Minimal Change Set and Confirm

For each candidate topic decide **reuse / create / defer**, label it **concept** or
**instance**, and give instances their type (creating the type topic if needed) and concepts
their parent. New keys: lowercase, hyphenated, specific, matching the neighbourhood's style,
never colliding; a type topic may borrow a Schema.org type name (`software-application`) as a
naming aid only. Give named things their canonical reference. Use `rei project create/sync` for
projects.

Show the plan and get approval before any write:

```
Proposed ontology (scoped to: <boundary>)

REUSE:     git-forge — Git Forge (concept/type), github — GitHub (instance)
CREATE:    gitdot — gitdot (instance) — ref https://gitdot.io
HIERARCHY: version-control broader-than git-forge
CLASSIFY:  gitdot instance-of git-forge
PEERS:     github related-to gitdot
PREDICATES: runtime-platform ← schema.org/runtimePlatform (core)
ATTACH/ABOUT: 12 links → github (about), 3 notes filed under gitdot
CLEANUP:   archive stale-topic (no content); migrate 4 is-a edges under git-forge → instance-of
OUT OF SCOPE: gitlab, bitbucket, sourcehut, gitea, forgejo, Git internals
```

Options: apply as proposed / adjust / topics only (skip wiring and cleanup) / cancel. If the
user wants to go beyond the stated use case, restate the upkeep trade-off first.

### Phase 6: Create Topics

Types and parents first. `topic create` prints no JSON — read each ID back:

```bash
rei --actor claude-code topic create KEY "LABEL" -d "DESCRIPTION"
rei topic show KEY --json | jq -r .topicId
rei --actor claude-code topic add-ref KEY https://CANONICAL-HOMEPAGE -l "LABEL"
```

If `add-ref` is refused because another topic owns the reference, that topic is a duplicate
candidate — reuse it and drop the new one from the plan (archive it if already created).

Projects:

```bash
rei --actor claude-code project create KEY --label "LABEL" --description "TEXT"
rei --actor claude-code project sync mori://NS/PROJECT
```

### Phase 7: Wire Classification, Hierarchy, Relations, and Cleanup

```bash
# classification: instance → type (target-first)
rei --actor claude-code topic associate TYPE_KEY INSTANCE_TOPIC_ID --relation instance-of
# (only where the type's instances already use is-a)
rei --actor claude-code edge add -f INSTANCE_TOPIC_ID -t TYPE_TOPIC_ID -p is-a
# concept hierarchy: broader → narrower, both abstract, one direction only
rei --actor claude-code edge add -f BROADER_TOPIC_ID -t NARROWER_TOPIC_ID -p broader-than
# peers (symmetric)
rei --actor claude-code edge add -f TOPIC_A_ID -t TOPIC_B_ID -p related-to
```

Approved cleanup:

```bash
# is-a → instance-of migration, per edge: add the association, then remove the old edge
rei edge list -p is-a --json | jq -r --arg t TYPE_TOPIC_ID '.[] | select(.targetId == $t and .status == "active") | "\(.edgeId)\t\(.sourceId)"'
rei --actor claude-code topic associate TYPE_KEY SOURCE_TOPIC_ID --relation instance-of
rei --actor claude-code edge remove EDGE_ID

# instance wrongly placed with broader-than: classify it, then remove the broader-than edge
rei --actor claude-code edge remove EDGE_ID            # soft-delete; `rei edge restore` undoes

# metadata fixes and pruning
rei --actor claude-code topic update KEY -l "LABEL" -d "DESCRIPTION"
rei --actor claude-code topic archive KEY               # `rei topic restore` undoes
rei --actor claude-code topic dissociate TOPIC ENTITY --relation RELATION
```

Merging duplicates has no single command: re-file attachments and re-point associations/edges
to the surviving topic, move the external reference (`remove-ref` on the loser, `add-ref` on the
survivor), then archive the loser. Never loosen a predicate's types to make an edge fit.

### Phase 8: Connect Existing Knowledge

**File** artifacts (moves the artifact's topic anchor — confirm when it is currently filed
elsewhere):

```bash
rei --actor claude-code link add --topic TOPIC_ID "URL" -t "TITLE"      # new link
rei --actor claude-code link set-topic LINK_ID TOPIC [--attachment ATTACHMENT_ID]
rei --actor claude-code note set-topic NOTE_ID TOPIC
rei --actor claude-code doc set-topic DOC_ID TOPIC                      # standalone docs only; a note's doc moves with the note
```

**Associate** what artifacts are about (always pass `--relation about` — the default is
`scoped-to`):

```bash
rei --actor claude-code topic associate TOPIC LINK_ID --relation about
rei --actor claude-code topic associate TOPIC NOTE_ID --relation about
```

Work through the Phase 3 content; don't force-fit. A cluster of unfiled content around a subject
the user cares about is a reason to loop back to Phase 5 for one more topic.

### Phase 9: Verify

```bash
rei ontology validate --json                                   # [] when clean
rei topic tree ROOT --direction narrower --include-inferred --explain
rei topic entities TYPE --relation instance-of --json | jq -r '.[].entity.id'
rei topic entities TOPIC --relation about --json | jq length
rei topic attachments TOPIC --json | jq '.attachments'
rei topic edges TOPIC --json
```

- Every planned topic, reference, association, edge, attachment, removal, and archive is
  actually present — redo any write that silently failed.
- Every new topic is reachable from its type's `instance-of` list or its parent's tree.
- A validation error caused by this session's edges: fix the edge. Pre-existing errors: report
  them; fix only with approval.

## Output Format

```
## Ontology Curated

### Scope
- In scope: <boundary>
- Deliberately out of scope: <list>

### Topics
- Reused: <key>, ...
- Created: <key> instance-of <type> (ref: <uri>) | <key> broader-than ← <parent>
- Deferred: <key>, ...
- Archived / merged: <key → survivor>, ...

### Predicates
- Seeded: about, scoped-to, instance-of, broader-than, narrower-than, related-to
- Defined: <key> ← <schema.org/term | rei-native: no standard term>

### Relations
- <broader> broader-than <narrower>
- <instance> instance-of <type>   (N is-a edges migrated)
- <a> related-to <b>

### Knowledge connected
- <N> links, <M> notes, <K> docs filed; <A> `about` associations

### Verification
- rei ontology validate: <passed | N pre-existing, unchanged | N fixed>
- Unresolved / failed: <item — reason> (or: none)
```

## Important Notes

- **Restraint is the feature.** When in doubt, model less, and always say what you chose not to
  model.
- **`instance-of` for named things, `broader-than` between concepts** — never the reverse.
- **Never restructure silently.** Archiving, merging, migrating `is-a`, re-filing artifacts, and
  fixing pre-existing validation errors all require explicit approval in Phase 5.
- **Read back every write**; topic, edge, and predicate commands can fail with exit `0`.

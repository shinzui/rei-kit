---
name: rei-curate-ontology
description: Build and maintain a Rei knowledge ontology — topics, predicates, and edges — scoped tightly to the user's actual use cases. Investigates the existing ontology, reuses what's there, creates only what's missing, and deliberately refuses to model the whole world.
allowed-tools: AskUserQuestion, Bash, Read
---

# Rei Curate Ontology

This skill helps a user build and maintain the ontology that organizes their links and
notes in Rei. Rei provides first-class **topics** (durable subject nodes), **predicates**
(relationship types with lightweight semantics), and **edges** (typed relationships between
entities). This skill surveys what already exists, then expands the ontology to fit the
user's concrete use case — and **only** that use case.

The single most important rule: **do not overengineer.** A Rei ontology models the user's
real interests, not the real world. If the user tracks two git forges they care about
(`github`, `gitdot`), the ontology gets two topics — not GitLab, Bitbucket, SourceHut, and
the hundred other forges that exist. Coverage of a domain is a non-goal. Usefulness to *this
user* is the only goal.

## When to Use

Activate when the user says things like:
- "Help me build an ontology in Rei"
- "Set up topics so I can organize my links and notes"
- "Create topics for <subject area>"
- "Maintain / clean up / extend my Rei ontology"
- "Organize my saved links by topic"
- "/rei-curate-ontology"

## Key Concepts

- **Topic** — a durable, first-class subject node with a stable lowercase `KEY`, a
  human-readable `LABEL`, and an optional description (e.g., `github` / "GitHub"). Topics own
  notes, docs, and links directly. Created with `rei topic create KEY LABEL`.
- **Attachment** — a *storage* relationship: a note, doc, or link directly anchored to a
  topic via `--topic` at creation time or `rei <kind> set-topic` afterward. Lists with
  `rei topic attachments TOPIC`. This is how links, notes, and docs get "filed under" a topic.
- **Predicate** — a reusable relationship type with optional ontology semantics:
  `--transitive`, `--symmetric`, `--inverse-of KEY`, and `--domain-types` / `--range-types`
  constraints. Defined with `rei predicate define`.
- **Edge** — a *semantic* assertion: `from -[predicate]-> to`. Created with `rei edge add`.
  Distinct from an attachment — Rei never auto-creates `about` edges from attachments.
- **Default ontology predicates** (seeded by `rei ontology seed-predicates`):
  - `about` — knowledge artifact (note/link/doc) → topic ("this is about that")
  - `broader-than` — topic → topic, transitive, inverse of `narrower-than`
  - `narrower-than` — topic → topic, transitive, inverse of `broader-than`
  - `related-to` — topic → topic, symmetric (peers / cross-links)
- **Inference** — query-time only. `rei topic related --include-inferred` and
  `rei topic tree --include-inferred` expand inverse/transitive facts without writing them
  back as edges. There is no OWL/SPARQL reasoning — semantics are intentionally minimal.
- **Validation** — `rei ontology validate` checks every active edge against its predicate's
  declared domain/range types.

## Scoping Discipline (read this before creating anything)

The hard part of this skill is restraint. Apply these rules at every step:

1. **Model use cases, not domains.** Start from "what does the user want to find again?",
   never "what is the complete taxonomy of X?". The git-forge example: model `github` and
   `gitdot` because the user uses them; do **not** add `gitlab`, `bitbucket`, `sourcehut`,
   `gitea`, `forgejo` just because they exist and would "complete" the picture.
2. **Reuse before create.** Always check the existing topic and predicate lists first. Reuse
   an existing topic or predicate verbatim rather than minting a near-duplicate.
3. **Keep hierarchies shallow.** One or two levels of `broader-than`/`narrower-than` is
   usually enough. Don't build a deep tree for elegance — build the minimum that makes
   browsing useful.
4. **Prefer attachments + `about` over elaborate predicates.** Most "organize my links"
   needs are met by filing links/notes/docs under topics and asserting `about` edges when
   there is a genuine semantic claim. Only define
   a *new* predicate when the user needs a relationship the defaults genuinely can't express
   (and even then, confirm it earns its place).
5. **Defer, don't pre-build.** It's fine to leave a topic out now and add it when the user
   actually has content for it. An empty topic created "for completeness" is overengineering.
6. **Confirm before creating.** Always show the proposed minimal set and the explicit
   out-of-scope list, and get the user's sign-off before running create commands.

## Workflow Overview

1. **Understand the use case** — what to organize, and the in/out-of-scope boundary
2. **Investigate the existing ontology** — topics, predicates, edges, validation
3. **Survey the knowledge to be organized** — existing links, notes, docs, tags
4. **Seed default predicates** — `rei ontology seed-predicates` (idempotent)
5. **Design the minimal topic set** — reuse + the smallest set of new topics; confirm
6. **Create missing topics** — `rei topic create`
7. **Wire hierarchy & relations** — `broader-than` / `narrower-than` / `related-to` edges
8. **Connect existing knowledge** — file existing artifacts under topics and/or assert `about` edges
9. **Validate & review** — `rei ontology validate`, `rei topic tree`, `rei topic related`
10. **Summary & maintenance guidance**

## Instructions for Claude

### Phase 1: Understand the Use Case

The goal is a sharp boundary, not a broad survey. Ask:

```
Question: "What are you trying to organize, and what do you want to be able to find again?"
Header: "Use case"
Options:
- Let me describe what I want to track
```

Follow up until you can state, concretely:
- The **subject area(s)** in play (e.g., "git forges I use", "papers on retrieval", "recipes").
- The **specific things inside** that area the user cares about (e.g., "github and gitdot",
  not "all forges").
- What's **explicitly out of scope** — name the adjacent things you will *not* model, and
  say so back to the user. This list is as important as the in-scope list.

Restate the boundary back to the user in one or two sentences before proceeding, e.g.:
> "In scope: the two forges you use (GitHub, gitdot) and the workflow concepts you compare
> them on. Out of scope: every other forge, and Git internals you didn't mention. Sound
> right?"

If the user is vague ("organize everything"), narrow them down — pick the one or two areas
with the most existing links/notes (you'll confirm counts in Phase 3) and start there.

### Phase 2: Investigate the Existing Ontology

Survey what's already modeled so you reuse instead of duplicate:

```bash
rei topic list --json
rei predicate list --json
rei edge list --json
rei ontology validate
```

Note, from the output:
- Which **topics already exist** that overlap the user's use case (reuse these).
- Which **predicates** are defined (and whether the defaults are present).
- Any **validation errors** already present — surface these to the user; they may indicate a
  prior modeling mistake worth fixing during this session.

This phase also tells you whether you are **building fresh** (few/no relevant topics) or
**maintaining/extending** (relevant topics exist). The rest of the workflow adapts:
- *Building fresh* → emphasize Phases 5–8 (design and create).
- *Maintaining* → also look for cleanup: over-broad topics, orphaned topics with no
  attachments, or topics that drifted outside the user's actual use (candidates for
  `rei topic archive`). Propose pruning, don't do it silently.

### Phase 3: Survey the Knowledge to Be Organized

Find the links and notes this ontology is meant to make findable, so topics map to real
content (not hypothetical content):

```bash
# Links — search/scan for ones relevant to the use case
rei link list --all --json
rei link list --all --query "TERM" --json    # narrow by a use-case keyword

# Notes relevant to the use case
rei note list --json

# Existing free-form tag vocabulary — often the seed of good topic candidates
rei custom-property entities tags --json
```

Existing `tags` are a strong signal: a tag many links already share is a good topic
candidate. Conversely, if a proposed topic would have **zero** matching links or notes,
question whether it's needed yet (rule 5 — defer, don't pre-build).

### Phase 4: Seed Default Predicates

Ensure the standard ontology predicates exist. This is idempotent — already-present
predicates are reported, not overwritten:

```bash
rei ontology seed-predicates
```

This guarantees `about`, `broader-than`, `narrower-than`, and `related-to` are available for
Phases 7–8. Only consider defining a **custom** predicate if the use case needs a
relationship the defaults can't express (e.g., `forked-from` for forge lineage). If so,
define it minimally and with type constraints:

```bash
rei predicate define forked-from --label "Forked from" \
  --description "Source project/forge is a fork of the target" \
  --source-types topic --target-types topic --actor claude-code
```

Push back on inventing predicates that won't be reused. A predicate used by one edge is
usually a sign the relationship should have been an `about` edge or an attachment.

### Phase 5: Design the Minimal Topic Set

Produce the smallest topic set that covers the in-scope use case. For each candidate topic,
decide: **reuse** an existing topic, **create** a new one, or **defer** it.

Apply the Scoping Discipline rules. For new topics, choose:
- `KEY` — stable, lowercase, hyphen-separated, specific (`github`, `gitdot`, `forge-features`).
  Reuse an existing key's style; never collide with an existing key.
- `LABEL` — human-readable (`GitHub`, `gitdot`).
- `--description` — optional one-liner; add it when the key isn't self-explanatory.

Present the design as a table and **confirm before creating**:

```
Proposed ontology (scoped to: <boundary restated>)

REUSE (already exist):
- github — GitHub
- forge-features — Git Forge Features

CREATE (new):
- gitdot — gitdot — "Neovim-native git forge the user tracks"

HIERARCHY:
- forge-features  broader-than  github
- forge-features  broader-than  gitdot
- github  related-to  gitdot

OUT OF SCOPE (deliberately NOT modeling): gitlab, bitbucket, sourcehut, gitea, forgejo,
and Git internals.
```

```
Question: "Here's the minimal ontology I propose. Create it?"
Header: "Approve"
Options:
- Create as proposed (Recommended)
- Let me adjust the topics or hierarchy
- Just create topics, skip the hierarchy for now
```

If the user wants to add topics beyond the stated use case, gently restate the scoping
trade-off (more topics = more upkeep, diluted browsing) before agreeing.

### Phase 6: Create Missing Topics

For each approved new topic:

```bash
rei topic create KEY "LABEL" --description "DESCRIPTION" --actor claude-code
```

Capture each new `topic_...` ID from the output. Skip any topic the user chose to defer.

### Phase 7: Wire Hierarchy & Relations

Add the approved edges. Use topic IDs (or keys) on both ends:

```bash
# Hierarchy: broader topic -[broader-than]-> narrower topic
rei edge add --from BROADER_TOPIC --to NARROWER_TOPIC --predicate broader-than --actor claude-code

# Peers / cross-links (symmetric)
rei edge add --from TOPIC_A --to TOPIC_B --predicate related-to --actor claude-code
```

Keep it shallow (rule 3). Add only the inverse direction you mean — `broader-than` already
implies `narrower-than` at query time via inference, so don't create both directions
manually. After wiring, you can preview the shape:

```bash
rei topic tree ROOT_KEY --direction narrower --include-inferred
```

### Phase 8: Connect Existing Knowledge

This is the payoff — make the user's existing links and notes findable under topics. Two
complementary mechanisms:

**Attachments (filing).** Anchor an artifact's storage to a topic. For new artifacts,
use `--topic` at creation time:

```bash
rei link add --topic TOPIC_ID "URL" --title "TITLE" --actor claude-code
rei note new --topic TOPIC_ID --stdin --actor claude-code
rei doc add ./PATH --topic TOPIC_ID --title "TITLE" --actor claude-code
```

For existing artifacts, use the topic re-anchor commands:

```bash
rei link set-topic LINK_ID TOPIC_ID --actor claude-code
rei note set-topic NOTE_ID TOPIC_ID
rei doc set-topic DOC_ID TOPIC_ID --actor claude-code
```

**`about` edges (semantic).** Assert that an artifact is about a topic when the user needs
that semantic fact in the graph. This is separate from filing: an `about` edge does not move
the artifact into `rei topic attachments TOPIC`.

```bash
rei edge add --from LINK_ID --to TOPIC_ID --predicate about --actor claude-code
rei edge add --from NOTE_ID --to TOPIC_ID --predicate about --actor claude-code
rei edge add --from DOC_ID --to TOPIC_ID --predicate about --actor claude-code
```

Work through the relevant links/notes found in Phase 3 and wire each to its best-fitting
topic. Don't force-fit: a link that doesn't clearly belong to an in-scope topic stays
unattached rather than spawning a new off-topic topic. If several unattached links cluster
around a subject the user *does* care about, that's a signal to propose one more topic — loop
back to Phase 5 for it.

Verify what landed where:

```bash
rei topic attachments TOPIC_KEY
rei topic edges TOPIC_KEY --predicate about
rei link list --topic TOPIC_ID
rei note list --topic TOPIC_ID
rei doc list --topic TOPIC_ID
```

### Phase 9: Validate & Review

Confirm the graph is consistent and show the user the result:

```bash
rei ontology validate
rei topic tree ROOT_KEY --direction narrower --include-inferred --explain
rei topic related TOPIC_KEY --include-inferred --explain
```

If `validate` reports errors (e.g., an `about` edge whose source type is outside the
predicate's domain), explain each and fix by removing/redoing the offending edge — never by
loosening a predicate's constraints just to silence the validator unless the user explicitly
wants that.

### Phase 10: Summary & Maintenance Guidance

Show what was created and how to keep it lean.

## Output Format

After completing the workflow, provide a summary:

```
## Ontology Curated

### Scope
- **In scope**: <restated boundary>
- **Deliberately out of scope**: <the things we chose not to model>

### Topics
- Reused: <key — label>, ...
- Created: <key — label>, ...
- Deferred (add later when there's content): <key>, ...

### Relations
- <broader> broader-than <narrower>
- <a> related-to <b>

### Knowledge connected
- <N> links, <M> notes, and <K> docs wired to topics (attachments + `about` edges)

### Validation
- rei ontology validate: <passed / N errors fixed>

### Maintenance tips
- Add a topic only when you have real links/notes for it — defer otherwise.
- Re-run `rei ontology validate` after bulk edge changes.
- Browse with `rei topic tree KEY --include-inferred` and `rei topic attachments KEY`.
- Archive topics that stop matching your use: `rei topic archive KEY`.
- Re-run `/rei-curate-ontology` to extend the ontology as your use case grows.
```

## Important Notes

- **Restraint is the feature.** The hardest and most valuable thing this skill does is *not*
  create topics. When in doubt, model less. Always state what you are choosing *not* to model.
- **Always include `--actor claude-code`** when creating topics, predicates, and edges.
- **Reuse before create**: check `rei topic list` / `rei predicate list` first and reuse
  existing keys verbatim rather than minting near-duplicates (`github` vs `git-hub`).
- **Topic keys are stable identifiers**: lowercase, hyphen-separated, specific. They're hard
  to change cleanly, so pick well the first time.
- **Attachment ≠ `about` edge**: an attachment (`--topic` or `rei <kind> set-topic`) files
  an artifact's storage under a topic; an `about` edge is a separate semantic assertion.
  Rei does not auto-create one from the other. Use `set-topic` to file already-stored links,
  notes, and docs; use `about` edges for additional semantic claims.
- **Inference is query-time only**: never create both `broader-than` and `narrower-than`
  for the same pair — one implies the other via `--include-inferred`.
- **Seeding is idempotent**: `rei ontology seed-predicates` is safe to run anytime; it
  reports already-present predicates and never overwrites incompatible ones.
- **Don't define single-use predicates**: if a relationship is used by one edge, it probably
  should have been an `about` edge or an attachment. Defaults (`about`, `broader-than`,
  `narrower-than`, `related-to`) cover most needs.
- **Empty topics are a smell**: a topic with no attachments or `about` edges is usually
  premature. Defer it until the user has content for it.
- **Maintenance is part of the job**: extending an existing ontology means pruning too —
  surface over-broad or orphaned topics as archive candidates, but never archive silently.

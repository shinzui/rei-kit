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
- **Classification vs. subsumption (read this twice — it's the #1 modeling mistake).** Two
  *different* relationships that are constantly conflated:
  - `broader-than`/`narrower-than` express **subject subsumption** — one *concept/subject* is
    a broader field of study than another (`version-control` broader-than `git-forge`;
    `frontend` broader-than `design-systems`). **Both ends are abstract subjects**, and the
    narrower one is a *kind of subject*, never a concrete thing.
  - **Classification** expresses that a *concrete thing* — a specific tool, product, library,
    system, service, forge, model — **is an instance of** a type (`github` *is a* git forge;
    `react` *is a* UI library; `astryx` *is a* design system). This is **not** subsumption.
    Model it with an **`is-a`** predicate. `is-a` is **not** seeded by default — define it
    once (Phase 4). **Never** attach a concrete instance to a category with `broader-than`.
  - Litmus test: if the child is a *thing you could put a link/homepage on* (a named product,
    tool, library), it's an **instance** → `is-a`. If the child is *another field/subject you'd
    browse*, it's a **concept** → `broader-than`. Also classify distinct instances into their
    *distinct correct types* — don't lump (React → `ui-libraries`, StyleX → `css-in-js`, not
    both under one generic bucket).
- **Schema.org alignment** — a vendored copy of the Schema.org vocabulary lives at
  `mori://schemaorg/schemaorg` (resolve with `mori path mori://schemaorg/schemaorg`, currently
  `/Users/shinzui/Keikaku/hub/schemaorg-project`). It is the **first place to look when a new
  predicate is needed**: Schema.org has already named most relationships between things
  (`about`, `isBasedOn`, `citation`, `runtimePlatform`, `memberOf`, `sameAs`, …), with a
  definition, an inverse, and expected domain/range. Borrow the name and the meaning instead of
  inventing a synonym. Borrow only the **relevant slice** — the one property you actually need —
  never the type's full property list or its subtype tree. See Phase 4.
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
3. **Borrow vocabulary before inventing it.** If the ontology genuinely needs a new predicate,
   look it up in the vendored Schema.org corpus (`mori://schemaorg/schemaorg`) before naming
   one yourself. Take the standard term's name and definition; take only the slice you need,
   not the surrounding type. Invent a key only when nothing in Schema.org fits (Phase 4).
4. **Keep hierarchies shallow.** One or two levels of `broader-than`/`narrower-than` is
   usually enough. Don't build a deep tree for elegance — build the minimum that makes
   browsing useful.
5. **Prefer attachments + `about` over elaborate predicates.** Most "organize my links"
   needs are met by filing links/notes/docs under topics and asserting `about` edges when
   there is a genuine semantic claim. Only define
   a *new* predicate when the user needs a relationship the defaults genuinely can't express
   (and even then, confirm it earns its place).
6. **Defer, don't pre-build.** It's fine to leave a topic out now and add it when the user
   actually has content for it. An empty topic created "for completeness" is overengineering.
7. **Confirm before creating.** Always show the proposed minimal set and the explicit
   out-of-scope list, and get the user's sign-off before running create commands.

## Workflow Overview

1. **Understand the use case** — what to organize, and the in/out-of-scope boundary
2. **Investigate the existing ontology** — topics, predicates, edges, validation
3. **Survey the knowledge to be organized** — existing links, notes, docs, tags
4. **Seed default predicates** — `rei ontology seed-predicates` (idempotent); define `is-a`
   when the use case has concrete instances; name any further predicate from Schema.org
5. **Design the minimal topic set** — reuse + the smallest set of new topics; mark each as a
   *concept* or an *instance*; confirm
6. **Create missing topics** — `rei topic create`
7. **Wire hierarchy, classification & relations** — `broader-than` between concepts, `is-a`
   for instances, `related-to` for peers
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
question whether it's needed yet (rule 6 — defer, don't pre-build).

### Phase 4: Seed Default Predicates

Ensure the standard ontology predicates exist. This is idempotent — already-present
predicates are reported, not overwritten:

```bash
rei ontology seed-predicates
```

This guarantees `about`, `broader-than`, `narrower-than`, and `related-to` are available for
Phases 7–8.

**Define `is-a` whenever the use case has concrete instances (it almost always does).**
Nearly every real ontology mixes *subjects* (fields you browse) with *instances* (the actual
tools, products, libraries, systems you track). Instances must be classified with `is-a`,
which is **not** seeded. Define it once, up front:

```bash
rei --actor claude-code predicate define is-a --label "Is a" \
  --description "Source is a concrete instance/member of the target type or class" \
  --source-types topic --target-types topic
```

This is a *reusable backbone* predicate — every instance→type edge uses it — so it is
emphatically **not** a single-use predicate. Reach for it before ever attaching a named thing
to a category with `broader-than`.

Beyond `is-a`, define a **custom** predicate when the use case needs a recurring relationship
the defaults can't express — e.g. "this system is built on that technology" for every
design-system→framework edge, or forge lineage. **Do not name it yourself first.**

#### Look the relationship up in Schema.org before inventing a predicate

Schema.org has already named, defined, and typed most relationships between things. Reusing
its term gives the predicate a definition the user didn't have to write, a documented inverse,
and interop with anything that speaks structured data. Inventing `built-with` when
`runtimePlatform` exists is exactly the "invent a private synonym" mistake this step prevents.

The vocabulary is vendored locally — no network needed:

```bash
SCHEMAORG=$(mori path mori://schemaorg/schemaorg)

(cd "$SCHEMAORG" && just find-prop "runtime platform")   # properties by name or description
(cd "$SCHEMAORG" && just find "design system")           # types by name or description
(cd "$SCHEMAORG" && just show SoftwareApplication)       # one type in full
```

`just find-prop` matches the description too, so search the *relationship in plain words*
("built on", "derived from", "part of", "member of"), not just a guessed camelCase name. The
corpus README is `mori://schemaorg/schemaorg/docs/consuming-schemas` — read it if a lookup
needs more than these recipes.

**Vet the candidate before adopting it** — three signals, all three must pass:

| Signal | Meaning | Action |
|---|---|---|
| `supersededBy` non-empty | obsolete, kept for compatibility | adopt the **successor** instead (`runtime` → `runtimePlatform`) |
| `isPartOf` = `https://pending.schema.org` | proposed, may change or be dropped | usable, but say so when proposing it; prefer a core term if one fits |
| term absent from `current` | retired to the attic (not vendored) | don't resurrect it — treat as "no match" |

`just find-prop` flags `[pending]` but **not** supersession, and `just show` reads the *types*
CSV only — so dump the candidate property's full row before committing to it:

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

**Translate the term into a `rei predicate define`, borrowing only the relevant slice:**

| Schema.org | Becomes |
|---|---|
| `label` (camelCase) | the predicate `KEY`, kebab-cased — `runtimePlatform` → `runtime-platform` |
| `comment` | `--description`, trimmed to one line, with the term IRI appended for provenance |
| `inverseOf` | `--inverse-of` — only if the user needs that direction; inference already covers reads |
| `domainIncludes` / `rangeIncludes` | **do not copy.** They are non-binding hints naming Schema.org *types*, while rei's `--source-types` / `--target-types` take rei entity types (`topic`, `note`, `link`, `doc`) and *are* enforced by `rei ontology validate`. Set those from how the user's edges actually run. |

Worked example — "this design system is built on that framework":

```bash
# 1. Search the relationship in plain words
(cd "$SCHEMAORG" && just find-prop "runtime platform")
#    runtime           <https://schema.org/runtime>
#    runtimePlatform   <https://schema.org/runtimePlatform>

# 2. Dump both rows (snippet above): runtime is supersededBy runtimePlatform;
#    runtimePlatform has empty supersededBy and empty isPartOf → current and core

# 3. Adopt the successor, named and defined from the standard term
rei --actor claude-code predicate define runtime-platform --label "Runtime platform" \
  --description "Runtime platform or script interpreter dependency (schema.org/runtimePlatform)" \
  --source-types topic --target-types topic
```

**Only the relevant slice.** Adopting `runtimePlatform` does *not* mean modeling
`SoftwareApplication`. Never import a type's inherited property list (`Recipe` alone carries
148), never mirror its subtype tree as topics, and never add sibling properties "for
completeness" — that is Schema.org-flavored overengineering, and rule 1 still applies. One
predicate per relationship the user actually asserts.

**When nothing fits, say so and invent deliberately.** Schema.org has no subject-subsumption
term — `broader-than` / `narrower-than` are rei-native (SKOS-shaped) and stay that way, as does
`is-a`. A forced-fit standard term is worse than an honest local one: if the closest match
means something materially different, name your own key and note in the description that no
standard term applied.

Push back only on predicates that will be used by exactly *one* edge with no prospect of
reuse — those usually should have been an `about` edge or an attachment. A predicate reused
across many edges (like `is-a`) is exactly what you want.

### Phase 5: Design the Minimal Topic Set

Produce the smallest topic set that covers the in-scope use case. For each candidate topic,
decide: **reuse** an existing topic, **create** a new one, or **defer** it — and label it a
**concept** (a browsable subject) or an **instance** (a named thing). This label determines
its edges: concepts hang off `broader-than`; instances attach to their type via `is-a` (and
may carry a borrowed predicate like `runtime-platform`, plus `about`/attachments). If an
instance's type doesn't exist yet as a concept topic, add that concept too — and classify distinct instances into distinct correct
types rather than one generic bucket.

Apply the Scoping Discipline rules. For new topics, choose:
- `KEY` — stable, lowercase, hyphen-separated, specific (`github`, `gitdot`, `forge-features`).
  Reuse an existing key's style; never collide with an existing key. When a **type/category**
  topic corresponds to a Schema.org type, borrowing that type's name is a good default
  (`SoftwareApplication` → `software-application`) — it makes the category unambiguous. This is
  a naming aid only: never pull in the type's subtypes or properties, and never rename a topic
  the user already knows by another word just to match the standard.
- `LABEL` — human-readable (`GitHub`, `gitdot`).
- `--description` — optional one-liner; add it when the key isn't self-explanatory.

Present the design as a table and **confirm before creating**:

```
Proposed ontology (scoped to: <boundary restated>)

REUSE (already exist):
- git-forge — Git Forge / Hosting Provider   (a CONCEPT / type)
- github — GitHub                            (a concrete forge — an INSTANCE)

CREATE (new):
- gitdot — gitdot — "Neovim-native git forge the user tracks"  (an INSTANCE)

CONCEPT HIERARCHY (broader-than — subjects/concepts only):
- version-control  broader-than  git-forge

CLASSIFICATION (is-a — a concrete forge → its type; NOT broader-than):
- github  is-a  git-forge
- gitdot  is-a  git-forge

PEERS (related-to):
- github  related-to  gitdot

NEW PREDICATES (with the standard term each is taken from):
- runtime-platform — "Runtime platform"  ← schema.org/runtimePlatform (core)
  (nothing standard exists for broader-than/is-a — those stay rei-native)

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
rei --actor claude-code topic create KEY "LABEL" --description "DESCRIPTION"
```

Note the flag position: `topic create`, `predicate define`, and `edge add` have **no local
`--actor`** — it is a global option and must come *before* the subcommand, as above. (`link
add`, `note new`, `doc add`, `link set-topic`, and `doc set-topic` do accept a trailing
`--actor`; `note set-topic` accepts none — see Phase 8.)

Capture each new `topic_...` ID from the output. Skip any topic the user chose to defer.

### Phase 7: Wire Hierarchy, Classification & Relations

Add the approved edges. **`rei edge add` requires topic `topic_...` IDs, not keys** — capture
the IDs from Phase 6's `rei topic create` output and use those (keys are rejected with an
"is not an allowed source type" error).

```bash
# CONCEPT hierarchy: broader concept -[broader-than]-> narrower concept (BOTH ends are subjects)
rei --actor claude-code edge add --from BROADER_TOPIC --to NARROWER_TOPIC --predicate broader-than

# CLASSIFICATION: a concrete instance -[is-a]-> its type/category (NOT broader-than)
rei --actor claude-code edge add --from INSTANCE_TOPIC --to TYPE_TOPIC --predicate is-a

# Peers / cross-links (symmetric)
rei --actor claude-code edge add --from TOPIC_A --to TOPIC_B --predicate related-to
```

**Never use `broader-than` to attach a concrete instance** (a named tool, product, library,
system, service, model) to a category — that is classification, so use `is-a`. Reserve
`broader-than` for concept→narrower-concept only. Getting this wrong now is the single most
expensive thing to unwind later, because it pollutes the browsable subject tree with
instances. Keep it shallow (rule 4). Add only the inverse direction you mean — `broader-than` already
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

For existing artifacts (created earlier, or filed under a different anchor), re-anchor them
onto the topic with the `set-topic` commands. These **move** the artifact's single anchor: it
leaves its previous anchor and now appears under the topic. `TOPIC` may be a topic **key or a
`topic_...` ID**:

```bash
rei link set-topic LINK_ID TOPIC --actor claude-code
rei note set-topic NOTE_ID TOPIC
rei doc set-topic DOC_ID TOPIC --actor claude-code
```

Three things to get right when running these:

- **Actor flag differs by command.** `link set-topic` and `doc set-topic` accept
  `--actor claude-code`; `note set-topic` does **not** (the note re-anchor records no actor) —
  passing `--actor` to it errors. The lines above reflect this.
- **fzf fallback.** Omit the artifact ID and/or `TOPIC` to pick interactively from an fzf
  list. When running this skill non-interactively, always pass both explicitly so nothing
  blocks on a picker.
- **Docs must be standalone.** `doc set-topic` moves only a *standalone* doc. A doc attached
  to a note is moved by re-anchoring its parent note (`rei note set-topic`), not on its own.

**`about` edges (semantic).** Assert that an artifact is about a topic when the user needs
that semantic fact in the graph. This is separate from filing: an `about` edge does not move
the artifact into `rei topic attachments TOPIC`.

```bash
rei --actor claude-code edge add --from LINK_ID --to TOPIC_ID --predicate about
rei --actor claude-code edge add --from NOTE_ID --to TOPIC_ID --predicate about
rei --actor claude-code edge add --from DOC_ID --to TOPIC_ID --predicate about
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

### Predicates
- Seeded/reused: about, broader-than, narrower-than, related-to, is-a
- Defined: <key — label>  ← <schema.org/term, or "no standard term — rei-native">

### Relations
- <broader-concept> broader-than <narrower-concept>   (subjects)
- <instance> is-a <type>                               (classification)
- <a> related-to <b>                                   (peers)

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
- **Always attribute writes to `claude-code`** when creating topics, predicates, and edges —
  but mind the flag position: `topic create`, `predicate define`, and `edge add` take it only
  as the **global** `rei --actor claude-code <command>`; `link add`, `note new`, `doc add`,
  `link set-topic`, and `doc set-topic` accept a trailing `--actor claude-code`;
  `note set-topic` accepts none.
- **Name new predicates from Schema.org, not from imagination.** Before `rei predicate define`,
  search the vendored vocabulary at `mori://schemaorg/schemaorg`
  (`just find-prop "built on"`), check `supersededBy` and `isPartOf`, and adopt the standard
  term's name and definition. Copy only that one property — never its type's inherited property
  list or subtype tree — and set rei's `--source-types`/`--target-types` from real usage, not
  from `domainIncludes`. If nothing genuinely fits (as for `broader-than` and `is-a`), define a
  rei-native key and say that no standard term applied.
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
- **Classify instances with `is-a`, subsume concepts with `broader-than`**: a named thing
  (tool/product/library/system) relates to its category by classification (`is-a`), never by
  subject subsumption (`broader-than`). This is the most important rule in the skill — see
  "Classification vs. subsumption" in Key Concepts.
- **Define reusable predicates; avoid single-use ones**: `is-a` (instance → type) is a
  *standard* predicate to define whenever the use case has concrete instances — it's reused by
  every classification edge, so it is not single-use. A genuine recurring relationship the
  defaults can't express also earns its place — named from the standard vocabulary where one
  exists (e.g. `runtime-platform` ← `schema.org/runtimePlatform`). What to avoid is a
  predicate used by exactly one edge with no prospect of reuse — that should have been an
  `about` edge or an attachment.
- **Empty topics are a smell**: a topic with no attachments or `about` edges is usually
  premature. Defer it until the user has content for it.
- **Maintenance is part of the job**: extending an existing ontology means pruning too —
  surface over-broad or orphaned topics as archive candidates, but never archive silently.

---
name: rei-bookmark-url
description: Bookmark a URL into Rei by filing it under the right topic — survey the existing ontology, reuse an existing topic wherever one fits, create only the topics the bookmark genuinely needs (classified with instance-of / broader-than, named from Schema.org where a standard term exists), anchor the link to the topic, and assert `about` associations for secondary subjects.
allowed-tools: AskUserQuestion, Bash, Read, WebFetch
---

# Rei Bookmark URL

This skill bookmarks a URL into Rei as a **topic-anchored link**. Unlike `rei-ingest-url`
(which anchors a link to an *intention* and writes a summary note), this skill's whole job is
**ontology placement**: work out what the page is about, find the topic that already covers it,
and only when nothing fits, extend the ontology by the smallest amount that makes the bookmark
findable.

The bookmark is the cheap part — `rei link add --topic` is one command. The valuable part is
that the ontology stays small, consistent, and reused. So the same restraint that governs
`/rei-curate-ontology` governs this skill: **one bookmark almost never justifies a new topic.**
Reuse is the default outcome; creating a topic is the exception you argue for and the user
approves.

## When to Use

Activate when the user says things like:
- "Bookmark this link in Rei"
- "Save <url> under the right topic"
- "Add this URL and file it in my ontology"
- "Bookmark this — figure out which topic it belongs to"
- "/rei-bookmark-url <url>"

Use `/rei-ingest-url` instead when the user wants the page **summarized into a note** and
anchored to an intention. Use `/rei-curate-ontology` instead when the user wants to build or
restructure the ontology itself, not to file one link.

## Key Concepts

- **Link** — a first-class Rei entity for a URL, with **canonical-URL deduplication**. Rei
  normalizes URLs (strips tracking params, applies per-domain rules, resolves redirects), so
  adding a URL that already exists **reuses the existing link** and just adds a new attachment.
  You will not create a duplicate entity by re-adding a URL.
- **Attachment (the anchor / where it is filed)** — a link's storage location: an intention,
  action, outcome, reflection, or **topic**. A bookmark is a link whose anchor is a topic. A
  link may hold **several** attachments (the same URL filed under an intention *and* a topic);
  `rei link show` lists them all. `rei topic attachments TOPIC` lists what is filed under a
  topic.
- **Association (what it is about)** — a *validated* first-class relation between an entity and
  a topic, written with `rei topic associate`. Three relations exist: `about` (subject matter),
  `scoped-to` (operational membership), `instance-of` (topic → topic classification). For
  bookmarks you use `about`, plus `instance-of` when you create a topic for a named thing.
- **Attachment ≠ association.** Filing a link under a topic does **not** create an `about`
  association, and an `about` association does **not** move the link into
  `rei topic attachments`. Use the anchor for the *primary* subject and `about` associations for
  the secondary ones.
- **`rei topic associate` vs `rei edge add`** — the association commands are the everyday API:
  they verify both endpoints exist and are active, and they only ever write the three validated
  relations. `rei edge add` is the open graph, for predicates the association commands never
  create (`summarizes`, `broader-than`, `related-to`, a borrowed Schema.org predicate). **Prefer
  `rei topic associate` for `about` and `instance-of`; use `rei edge add` for everything else.**
- **Seeded ontology predicates** (`rei ontology seed-system`, idempotent):
  `about`, `scoped-to`, `instance-of`, `broader-than` (transitive, inverse `narrower-than`),
  `narrower-than`, `related-to` (symmetric). It also seeds the system `project` topic.
- **Classification vs. subsumption** — the #1 modeling mistake, and it bites hardest when
  bookmarking, because most bookmarks are about a *named thing*:
  - **`instance-of`** — a concrete thing is an instance of a type: `react` instance-of
    `ui-libraries`, `pkl` instance-of `configuration-language`. Both ends are topics; the source
    is a thing you could put a homepage on.
  - **`broader-than` / `narrower-than`** — *subject subsumption* between two abstract subjects:
    `version-control` broader-than `git-forge`. Both ends are fields you would browse.
  - Litmus test: if the child is a named product/tool/library/service, it is an **instance** →
    `instance-of`. If it is another subject area, it is a **concept** → `broader-than`. Never
    attach a named thing to a category with `broader-than`.
- **Topics vs `tags`** — a topic is a durable node the link is *filed under* and browsable via
  `rei topic tree`; `tags` is a free-form `tag-set` custom property. Prefer topics for anything
  you would want to browse; use tags only for the long tail that does not deserve a topic.
- **External reference** — `rei topic add-ref TOPIC SCHEME://KEY` records a topic's canonical
  identity (a homepage, a `mori://` project URI, a Schema.org IRI). One reference belongs to at
  most one topic, so it is also a **duplicate-topic detector**: `rei topic ref-show URL` answers
  "does a topic for this thing already exist under some other key?".
- **Schema.org alignment** — the vendored vocabulary at `mori://schemaorg/schemaorg` is the
  first place to look when a *new predicate* is needed. See Phase 6.

## Scoping Discipline (read before creating anything)

1. **Reuse is the expected outcome.** Run the survey in Phase 4 and genuinely try to place the
   bookmark in an existing topic before proposing a new one. "Close enough to browse together"
   is close enough.
2. **One bookmark rarely justifies a topic.** A topic that would hold exactly this one link, with
   no prospect of more, is premature — file the link under the nearest existing topic and put the
   specificity in `tags`. Create the topic when the user says they will keep collecting on it, or
   when Phase 4 shows existing links that would move under it too.
3. **Never create a topic for the publisher.** A blog post on a personal site is about its
   *subject*, not about the site. `platform` and `content-type` already record the shape and
   source — do not mint `some-persons-blog` as a topic.
4. **One anchor, few associations.** Pick the single best-fitting topic as the anchor, then add
   at most one or two `about` associations. A link that is "about" six topics is about none of
   them.
5. **Borrow vocabulary before inventing it.** If a genuinely new *predicate* is needed, look it
   up in the vendored Schema.org corpus before naming one (Phase 6). Most bookmarks need no new
   predicate at all.
6. **Confirm before creating.** Show the proposed placement — reuse vs create, with the
   deliberately-not-created list — and get sign-off before running any create command.

## Workflow Overview

1. **Get the URL** — from args or the user, plus any subject hint they offer
2. **Check what Rei already knows about this URL** — dedup, existing anchors, existing properties
3. **Work out what the page is about** — fetch lightly; title + subject + a few candidate topics
4. **Survey the ontology** — topics, refs, predicates, existing tags, related links
5. **Decide the placement** — reuse / create / defer; anchor topic + `about` topics; confirm
6. **Ensure predicates exist** — `rei ontology seed-system`; reconcile `is-a` vs `instance-of`;
   name any new predicate from Schema.org
7. **Create missing topics** — `rei topic create`, classify and place them, optional `add-ref`
8. **Bookmark the link** — `rei link add --topic` (or `rei link set-topic` to re-anchor)
9. **Assert secondary subjects** — `rei topic associate LINK --relation about`
10. **Classify and tag the link** — `content-type`, `platform`, and the long-tail `tags`
11. **Validate and review** — `rei ontology validate`, `rei topic attachments`
12. **Summary**

## Instructions for Claude

### Phase 1: Get the URL

The URL may be supplied as an argument. If not:

```
Question: "What URL should I bookmark into Rei?"
Header: "URL"
Options:
- Let me paste the URL
```

Validate it is a well-formed absolute `http(s)` URL; if not, stop and say so.

If the user volunteered *why* they are saving it ("for the Pkl work", "another jj article"),
keep that — it is the strongest signal for topic placement and often lets you skip Phase 3's
fetch entirely.

### Phase 2: Check What Rei Already Knows About This URL

Rei deduplicates by canonical URL, so this phase is **not** about avoiding a duplicate entity —
it is about learning whether the link already exists and where it is already filed.

```bash
rei link list --all --domain DOMAIN --json | jq -r --arg url "URL" '.[] | select(.url == $url)'
```

Fallback without `jq`:

```bash
rei link list --all --query "URL" --json
```

If a match exists, capture its ID and inspect every attachment and property:

```bash
rei link show LINK_ID --json
```

Then branch:

- **Already anchored to the intended topic** → nothing to file. Tell the user, and offer to
  only refine (title, tags, an extra `about` association). Skip Phase 8.
- **Anchored elsewhere (an intention, action, or another topic)** → decide *with the user* in
  Phase 5 between **adding** a topic attachment (keeps the existing anchor; `rei link add
  --topic`) and **moving** the existing one (`rei link set-topic`). Adding is the safer default —
  it never removes an existing filing.
- **No match** → a fresh bookmark; continue.

Note any properties already set (`content-type`, `platform`, `tags`) so Phase 10 does not
overwrite them.

### Phase 3: Work Out What the Page Is About

Skip this phase if the user already stated the subject clearly and you recognize it.

Otherwise fetch the URL with WebFetch, asking only for what placement needs — this is a
bookmark, not an ingest, so do **not** produce a full summary:

> "Return: the page title; a 1–2 sentence statement of what this page is about; the 3–6 main
> subjects/technologies/concepts it covers, most central first; whether each of those is a named
> product/tool/library or an abstract subject area; and the publishing platform (blog, docs site,
> GitHub, YouTube, …)."

Extract `TITLE`, a one-line subject statement, and a ranked list of candidate subjects each
labelled **instance** (named thing) or **concept** (subject area).

If the fetch fails, do not stop — ask the user for the title and subject in one question and
continue. Nothing has been written yet at this point.

### Phase 4: Survey the Ontology

Find what already exists so you reuse rather than duplicate:

```bash
rei topic list --json
rei predicate list --json
```

For each candidate subject from Phase 3, look for an existing home. Search by key fragment and
by label, and check whether a topic already claims the thing's canonical identity:

```bash
rei topic list --json | jq -r '.[] | "\(.key) — \(.label)"' | grep -i "CANDIDATE"

# Does a topic already own this thing's canonical reference (under any key)?
rei topic ref-show https://CANDIDATE-HOMEPAGE

# What already lives under a promising topic, and how is it wired?
rei topic show TOPIC_KEY
rei topic attachments TOPIC_KEY
rei topic edges TOPIC_KEY
rei topic tree TOPIC_KEY --direction narrower --include-inferred
```

Also pull existing links on the same subject and the current tag vocabulary — both tell you
whether a proposed topic would have company or would sit empty:

```bash
rei link list --all --query "SUBJECT_TERM" --json
rei custom-property entities tags --json
```

Record, per candidate: **exact topic match**, **near match worth reusing**, or **nothing**. A
near match is usually the right answer (rule 1).

### Phase 5: Decide the Placement and Confirm

Produce the smallest placement that makes this bookmark findable:

- **Anchor topic** — exactly one. The topic the user would browse to look for this page. Prefer
  an existing topic; prefer the more general one when torn between two.
- **`about` topics** — zero to two additional subjects worth asserting, only when the anchor
  alone would not surface it.
- **New topics** — only those that survive rules 2 and 3. For each, state whether it is an
  **instance** or a **concept**, and where it attaches (`instance-of` a type, or `broader-than`
  from a parent concept). If an instance's type topic does not exist yet, add that concept too —
  and classify distinct instances into their *distinct correct* types rather than one generic
  bucket.
- **Deferred** — subjects you are deliberately not modeling yet. Name them.

Present it and confirm:

```
Bookmark placement for: <TITLE>
<one-line statement of what the page is about>

ANCHOR (link is filed here):
- pkl — Pkl                              REUSE (existing topic, 4 links already)

ABOUT (secondary subject associations):
- configuration-language — Configuration Language   REUSE

NEW TOPICS: (none — everything this page covers already has a home)

DEFERRED (not modeling for one bookmark): schema-validation, cue-lang
  → captured as `tags` instead

PROPERTIES: content-type=documentation, platform=docs_site, tags=pkl,schema-validation
```

```
Question: "Here's where I'd file this bookmark. Go ahead?"
Header: "Placement"
Options:
- File it as proposed (Recommended)
- Let me pick a different topic
- Create a new topic for it instead
- Just bookmark it, skip the associations and tags
```

When the link already exists (Phase 2) and is anchored elsewhere, ask this too:

```
Question: "This URL is already filed under <existing anchor>. Add a topic attachment or move it?"
Header: "Anchor"
Options:
- Add a topic attachment, keep the existing one (Recommended)
- Move it to the topic
```

If the user asks for a new topic that fails rules 2 or 3, state the trade-off in one sentence
(an empty topic dilutes browsing and has to be maintained), then follow their call.

### Phase 6: Ensure the Predicates Exist

Seed the standard ontology. This is idempotent — existing entries are reported, never
overwritten:

```bash
rei ontology seed-system
```

This guarantees `about`, `scoped-to`, `instance-of`, `broader-than`, `narrower-than`, and
`related-to`, plus the system `project` topic.

**Reconcile `is-a` with `instance-of` before classifying anything.** Some workspaces were curated
before `instance-of` was seeded and classify instances with a locally-defined `is-a` predicate
instead. Check how the *neighbours* under the intended type topic are already wired:

```bash
rei topic edges TYPE_TOPIC_KEY
```

- Siblings wired with `instance-of`, or no siblings at all → use `instance-of` via
  `rei topic associate` (the validated, first-class relation — prefer it).
- Siblings wired with `is-a` → **stay consistent with the neighbours** and use `is-a` via
  `rei edge add`, so browsing that type does not return half the instances. Mention the
  divergence once and suggest `/rei-curate-ontology` for a proper migration — do not migrate
  edges as a side effect of a bookmark.

**A new predicate is almost never needed for a bookmark.** Filing plus `about` covers it. Only
if the user names a recurring relationship the defaults cannot express, look it up in the
vendored Schema.org vocabulary before inventing a key:

```bash
SCHEMAORG=$(mori path mori://schemaorg/schemaorg)

(cd "$SCHEMAORG" && just find-prop "built on")     # properties by name or description
(cd "$SCHEMAORG" && just find "software")          # types by name or description
(cd "$SCHEMAORG" && just show SoftwareApplication) # one type in full
```

Search the relationship *in plain words* — `find-prop` matches descriptions too. Vet the
candidate: a non-empty `supersededBy` means adopt the successor instead; `isPartOf =
https://pending.schema.org` means proposed (usable, but say so). Then translate: the camelCase
`label` becomes a kebab-case `KEY`, the `comment` becomes a one-line `--description` with the
term IRI appended for provenance. Do **not** copy `domainIncludes` / `rangeIncludes` — set
`--source-types` / `--target-types` from how the edges actually run.

```bash
rei --actor claude-code predicate define runtime-platform --label "Runtime platform" \
  --description "Runtime platform or script interpreter dependency (schema.org/runtimePlatform)" \
  --source-types topic --target-types topic
```

Borrow only the one property you need — never a type's inherited property list or its subtype
tree. If nothing standard genuinely fits, define a rei-native key and say that no standard term
applied. For the full lookup-and-vetting procedure, see `/rei-curate-ontology` Phase 4.

### Phase 7: Create the Approved Topics

For each approved new topic:

```bash
rei --actor claude-code topic create KEY "LABEL" --description "DESCRIPTION"
```

Keys are stable, lowercase, hyphen-separated, and specific (`pkl`, `configuration-language`).
Match the style of neighbouring keys and never collide with an existing one. Capture each new
`topic_...` ID — the next commands need IDs.

Note the flag position: `topic create`, `predicate define`, and `edge add` take **no local
`--actor`** — it is the global `rei --actor claude-code <command>`. (`link add`, `link
set-topic`, `note new`, and `doc add` do accept a trailing `--actor`.)

Wire each new topic into the ontology:

```bash
# Classification — a named thing is an instance of its type.
# Argument order: associate TARGET_TYPE_TOPIC SOURCE_INSTANCE_TOPIC.
rei topic associate TYPE_TOPIC_KEY INSTANCE_TOPIC_ID --relation instance-of

# ...or, in an is-a workspace (Phase 6), stay consistent with the neighbours:
rei --actor claude-code edge add --from INSTANCE_TOPIC_ID --to TYPE_TOPIC_ID --predicate is-a

# Concept placement — broader subject -[broader-than]-> narrower subject. BOTH ends abstract.
rei --actor claude-code edge add --from BROADER_TOPIC_ID --to NARROWER_TOPIC_ID --predicate broader-than

# Peers (symmetric) — only when the user would want the cross-link
rei --actor claude-code edge add --from TOPIC_A_ID --to TOPIC_B_ID --predicate related-to
```

`rei edge add` requires `topic_...` **IDs**, not keys (keys are rejected as "not an allowed
source type"). `rei topic associate` accepts a key or an ID for the topic argument.

Never create both `broader-than` and `narrower-than` for a pair — inference derives the inverse
at query time.

Optionally record the topic's canonical identity, which makes future duplicate detection work
(`rei topic ref-show`):

```bash
rei topic add-ref TOPIC_KEY https://CANONICAL-HOMEPAGE --label "LABEL"
```

Use the thing's own canonical home (project homepage, `mori://` project URI, or the Schema.org
IRI for a borrowed type name) — not the URL being bookmarked.

### Phase 8: Bookmark the Link

Anchor the link to the chosen topic. Pass the `topic_...` **ID** to `--topic`:

```bash
rei link add --topic TOPIC_ID "URL" -t "TITLE" --actor claude-code
```

Because Rei deduplicates by canonical URL, this reuses an existing link entity when the URL is
already known and simply files it under the topic as an additional attachment. Capture the link
ID from the output.

To **move** an existing attachment instead of adding one (only when the user chose that in
Phase 5) — `TOPIC` here may be a key or an ID:

```bash
rei link set-topic LINK_ID TOPIC --actor claude-code

# When the link has several attachments, name the one to move:
rei link set-topic LINK_ID TOPIC --attachment LINK_ATT_ID --actor claude-code
```

Always pass both arguments explicitly; omitting them opens an fzf picker that will block a
non-interactive run. If the title was unknown at add time, set it after:

```bash
rei link title LINK_ID "TITLE"
```

### Phase 9: Assert Secondary Subjects

For each approved `about` topic (not the anchor — filing already covers that):

```bash
rei topic associate TOPIC_KEY LINK_ID --relation about
```

This is idempotent: re-associating reports "Already associated" and changes nothing. Both
endpoints must be active — an archived topic is refused.

Read it back from either end:

```bash
rei topic associations LINK_ID
rei topic entities TOPIC_KEY --relation about --entity-type link
```

Use `rei edge add` here only for a genuinely different predicate (e.g. a borrowed
`runtime-platform`), never to hand-write an `about` row.

### Phase 10: Classify and Tag the Link

Set the two properties that are cheap and reliable for a bookmark, skipping any already set on a
reused link (Phase 2) and skipping rather than guessing:

```bash
rei link set-property -l LINK_ID content-type VALUE
rei link set-property -l LINK_ID platform VALUE
```

Allowed values (verify with `rei custom-property show KEY` if a set fails):

- `content-type`: `homepage`, `page`, `article`, `blog_post`, `essay`, `documentation`,
  `api_reference`, `tutorial`, `guide`, `research_paper`, `whitepaper`, `case_study`,
  `announcement`, `changelog`, `repository`, `repository_issue`, `repository_pr`,
  `repository_release`, `package`, `social_post`, `thread`, `discussion`, `qa_question`,
  `qa_answer`, `video`, `podcast`, `image`, `presentation`, `pdf`, `dataset`, `course`
- `platform`: `website`, `blog`, `x`, `linkedin`, `reddit`, `hackernews`, `github`, `gitlab`,
  `bitbucket`, `youtube`, `vimeo`, `substack`, `medium`, `notion`, `wikipedia`, `arxiv`,
  `stack_overflow`, `lobsters`, `newsletter`, `docs_site`, `package_registry`,
  `podcast_platform`, `chatgpt`, `claude`

`author-type` and `media` exist too — set them only if the page made them obvious.

**Tags carry the long tail only.** Topics already record what the bookmark is about; tags are for
the deferred subjects from Phase 5 that did not earn a topic. Do not re-encode the anchor topic,
the `about` topics, or the enum classifiers as tags. Harvest the existing vocabulary first and
reuse verbatim rather than minting near-duplicates (`llm` vs `llms` vs `large-language-models` is
the failure mode):

```bash
rei custom-property entities tags --json \
  | jq -r '.entities[].properties.tags' \
  | tr ',' '\n' | sed 's/^ *//; s/ *$//' | sort -u
```

New tags are lowercase, hyphen-separated, singular where natural. Aim for 0–4 — a well-placed
bookmark needs few. `tags` is a `tag-set`: pass every desired tag in **one** call, since setting
replaces the full set. On a reused link, read the current tags from Phase 2 and merge first:

```bash
rei link set-property -l LINK_ID tags "tag-one,tag-two"
```

### Phase 11: Validate and Review

```bash
rei ontology validate
rei link show LINK_ID
rei topic attachments ANCHOR_TOPIC_KEY
rei topic associations LINK_ID
rei topic tree ROOT_KEY --direction narrower --include-inferred
```

If `validate` reports an error caused by an edge you just wrote, fix the edge — never loosen a
predicate's constraints to silence the validator. Pre-existing errors: surface them and suggest
`/rei-curate-ontology`; do not fix them as a side effect of a bookmark.

### Phase 12: Summary

Report what was reused, what was created, and what was deliberately skipped.

## Output Format

```
## Bookmarked

- **URL**: <url>
- **Title**: <title>
- **Link**: <link_id>  (new link / reused existing canonical link)
- **Anchor topic**: <key> — <label>  (reused / created)  [attachment added / moved from <old anchor>]

### Ontology
- Topics reused: <key>, ...
- Topics created: <key — label> (instance-of <type> | broader-than from <parent>), ...
- Topics deferred (not worth one bookmark): <name>, ... → recorded as tags
- Predicates: seeded/reused <about, instance-of, broader-than, ...>; defined <key ← schema.org/term, or none>
- External refs added: <topic → scheme://key>, or none

### Associations
- about: <topic-key>, ...  (or none — the anchor covers it)

### Properties
- content-type: <value or "skipped">
- platform: <value or "skipped">
- tags: <a, b>  (N reused, M new)  (or none)

### Validation
- rei ontology validate: <passed / N pre-existing errors, unchanged>

### Next Steps
- Browse the topic: `rei topic attachments <anchor-key>`
- See the link's subjects: `rei topic associations <link_id>`
- Extend the ontology properly: `/rei-curate-ontology`
```

## Important Notes

- **Reuse is the expected outcome.** One bookmark rarely justifies a new topic. File it under
  the nearest existing topic and put the specificity in `tags`; create a topic when the user will
  keep collecting on it, or when existing links would move under it too.
- **Links deduplicate by canonical URL.** `rei link add` on a known URL reuses the link entity
  and adds an attachment — it never creates a second link. So Phase 2 is about discovering the
  existing anchors, not about preventing a duplicate.
- **Adding an attachment ≠ moving one.** `rei link add --topic` keeps every existing filing;
  `rei link set-topic` moves one. Default to adding, and confirm before moving.
- **Attachment ≠ `about` association.** Filing does not create an `about` row and an `about` row
  does not file. Anchor = primary subject; `about` = the one or two secondary ones.
- **Prefer `rei topic associate` over `rei edge add`** for `about` and `instance-of` — it
  validates both endpoints and is the supported everyday API. Reserve `rei edge add` for
  `broader-than`, `related-to`, and borrowed predicates.
- **Argument order for `associate` is target-first**: `rei topic associate TOPIC ENTITY` reads
  "ENTITY <relation> TOPIC". So `associate ui-libraries topic_react --relation instance-of`
  means *react instance-of ui-libraries*.
- **Classify with `instance-of`, subsume with `broader-than`.** A named product/tool/library is
  an *instance* of its type; only two abstract subjects stand in a broader/narrower relation.
  Check `rei topic edges TYPE_TOPIC` first: in a workspace that still classifies with `is-a`,
  stay consistent with the neighbours rather than splitting the type in half.
- **Never create a topic for the publisher** — `platform` and `content-type` already record the
  source and shape.
- **Actor flag position differs by command**: `topic create`, `predicate define`, and `edge add`
  take only the global `rei --actor claude-code <command>`; `link add` and `link set-topic` take
  a trailing `--actor claude-code`; `rei topic associate` takes no actor flag.
- **Always pass IDs explicitly** to `link set-topic` and `link set-property` — omitting them
  opens an fzf picker that blocks non-interactive runs. `rei edge add` needs `topic_...` IDs;
  `rei topic associate`, `link set-topic`, and `topic attachments` accept keys.
- **`tags` is a `tag-set`**: one `set-property` call with every tag, since setting replaces the
  full set. Merge with existing tags on a reused link.
- **Seeding is idempotent**: `rei ontology seed-system` is safe anytime; it reports
  already-present entries and never overwrites incompatible ones.
- **Don't fix the ontology as a side effect.** Surface pre-existing validation errors,
  over-broad topics, or `is-a`/`instance-of` divergence, and point at `/rei-curate-ontology`.

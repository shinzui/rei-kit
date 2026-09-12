---
name: rei-file-temp-note
description: File a temporary markdown note into Rei — analyze its content, propose the intention (or topic) to anchor it to, a category, topical tags, and applicable note custom properties, place it in the ontology (reuse or create topics, `about` associations, edges to the links and notes it references), create the note, verify the stored content, and then delete the temporary file.
allowed-tools: AskUserQuestion, Bash, Read
---

# Rei File Temp Note

This skill takes a **temporary markdown note** — a scratch file written in an editor, dumped by
another tool, or drafted by an agent — and files it into Rei properly. It reads the note, works
out where it belongs and what it is about, and proposes a complete filing plan: the intention to
anchor it to, a category, `tags`, any note custom properties whose definitions fit, the topics it
should be associated with (reusing or growing the ontology), and edges to the links and notes it
references. Once the user approves the plan, it creates the note, applies everything, verifies
that Rei holds the full content, and **only then** deletes the temporary file.

Contrast with sibling skills: `rei-note-from-url-markdown` captures a *web page snapshot* tied
to a source URL; `rei-bookmark-url` files a *link* into the ontology. This skill files a note the
user wrote themselves, whose content has no single source URL.

## When to Use

Activate when the user says things like:
- "File /tmp/scratch.md into Rei"
- "Add this temp note to Rei and delete the file"
- "Figure out which intention this note belongs to and save it"
- "Tag and file this markdown note, then clean up the draft"
- "/rei-file-temp-note /tmp/note-2026-09-12.md"

## Key Concepts

- **Temporary note file** — a markdown file on disk that is *not* the source of truth. Once its
  content is stored and verified in Rei, the file is deleted. Nothing is deleted before that.
- **Anchor** — where a note is stored: an intention (most common), an action, an outcome, a
  habit, or a **topic**. Exactly one. Use a topic anchor only for pure knowledge notes that
  serve no intention.
- **Category** — optional hierarchical classification (`work/project`, `system/journal`).
  Categories also gate which scoped custom properties apply.
- **`tags`** — the global `tag-set` custom property, scoped to notes. Lightweight facets for
  filtering (`rei note list -w 'tags=...'`). Reuse the existing vocabulary aggressively.
- **Note custom properties** — definitions whose `allowedEntityTypes` include `note_entity`
  (e.g. `language`, `reading-status`, `srs-deck-path`). A property with non-empty
  `scopedToCategories` only applies when the note's category is in that scope.
- **Topics and associations** — first-class ontology subjects. `rei topic associate TOPIC NOTE
  --relation about` records what the note is about. Topics are browsable and relate to each
  other; tags are not. **Subjects go in topics; tags are facets on top.**
- **Edges** — typed graph relations via `rei edge add`. Seeded/defined predicates relevant
  here: `references` (note → link), `summarizes` (note → link/note/topic). Topic ↔ topic wiring
  (`instance-of`, `broader-than`, `related-to`) follows `rei-bookmark-url`'s modeling discipline.

## Workflow Overview

1. **Get the file path** — from args or the user; validate it
2. **Read and analyze the note** — frontmatter, title, subjects, references, language, shape
3. **Survey Rei** — candidate intentions, categories, note properties, tag vocabulary, topics,
   predicates, and entities the note references
4. **Build the filing plan** — anchor, category, tags, properties, topics, edges
5. **Confirm with the user** — one approval gate (the file will be deleted afterwards)
6. **Create the note** — pipe the content into `rei note new`, set the title
7. **Apply tags and custom properties**
8. **Place the note in the ontology** — create/wire topics, `about` associations, edges
9. **Verify** — stored content matches the file, ontology validates
10. **Delete the temporary file** — only if verification passed
11. **Summary**

## Instructions for Claude

### Phase 1: Get the File Path

The path is normally supplied as an argument. If not, ask:

```
Question: "Which temporary markdown note should I file into Rei? (e.g., /tmp/scratch.md)"
Header: "File"
Options:
- Let me paste the path
```

Validate before doing anything else:

```bash
test -f "PATH" && test -r "PATH" && echo ok
```

- If the file doesn't exist or isn't readable, stop and say so.
- If it isn't markdown (`.md` / `.markdown`, or clearly markdown content), confirm with the user
  before continuing.
- If it is empty, stop — there is nothing to file, and do **not** delete it.
- If the path is inside a git repository or a notes vault (not a scratch location like `/tmp`,
  the scratchpad, or `~/Downloads`), mention that before the deletion step so the user isn't
  surprised — it may not actually be temporary.

### Phase 2: Read and Analyze the Note

Read the whole file with the Read tool. Extract:

- **Frontmatter** (YAML between leading `---` fences), if any. Honor explicit hints the author
  left: `title`, `tags`, `intention` / `intention_id`, `topic` / `topics`, `category`, or any key
  that matches an existing note custom property key. Explicit hints outrank inference.
- **TITLE** — frontmatter `title`, else the first `# Heading`, else a short title you derive.
- **SUMMARY** — one or two sentences on what the note is about and what it is *for*
  (planning, meeting notes, research, a decision record, a journal entry, a how-to…).
- **SUBJECTS** — the 1–5 distinct subjects it substantively covers, most central first, each
  labelled **instance** (a named product/tool/library/project/person) or **concept** (a subject
  area). Drop things only mentioned in passing.
- **INTENTION SIGNALS** — project names, repo names, goals, `Intention:` trailers, intention IDs
  (`intention_...`), or wording that matches an ongoing effort.
- **REFERENCES** — URLs, Rei entity IDs (`note_...`, `link_...`, `topic_...`), and `[[wikilinks]]`.
- **FACETS** — language, whether it contains tasks (`- [ ]`), dates, and anything that maps to
  an existing note property (checked in Phase 3).

### Phase 3: Survey Rei

Gather the context needed to make grounded suggestions. Run these in parallel where possible.

**3a — Candidate intentions.** Search by each intention signal and subject keyword:

```bash
rei intention list --json
rei intention list --all -s "KEYWORD" --json
```

Shortlist up to 3 candidates, ranked by how well their titles match the note's purpose. If a
frontmatter hint or an `intention_...` ID is present, verify it exists and put it first.

**3b — Categories.**

```bash
rei category list --flat --descriptions --json
```

Pick a category only when one clearly fits; no category is better than a wrong one. If a
candidate category has note guidance, read it — it may dictate structure or properties:

```bash
rei category print-note-guidance CATEGORY_SLUG
```

**3c — Note custom properties.**

```bash
rei custom-property list -e note --json
```

For each definition, note `key`, `valueType` (enum values, state machine states, tag-set,
path, url, bool, text), and `scopedToCategories`. Ignore archived properties and obvious smoke-
test properties (keys starting with `ep`/`smoke` or similar noise). A property scoped to
categories is only a candidate if the chosen category is in scope — resolve category IDs with
`rei category show SLUG`/`--json` if needed.

**3d — Existing tag vocabulary.**

```bash
rei custom-property entities tags --json \
  | jq -r '.entities[].properties.tags' \
  | tr ',' '\n' \
  | sed 's/^ *//; s/ *$//' \
  | sort -u
```

Tags are stored as one comma-separated string per entity, hence the split.

**3e — Ontology.**

```bash
rei topic list --json | jq -r '.[] | "\(.key) — \(.label)"'
rei predicate list --json | jq -r '.[] | "\(.predicateKey)\t\(.sourceTypes)\t\(.targetTypes)"'
```

For each subject, check for an existing topic by key *and* label, and by canonical identity for
named things:

```bash
rei topic list --json | jq -r '.[] | "\(.key) — \(.label)"' | grep -i "SUBJECT"
rei topic ref-show https://SUBJECT-HOMEPAGE
rei topic show TOPIC_KEY
rei topic edges TOPIC_KEY
```

**3f — Referenced entities.** For each URL in the note, check whether Rei already has a link:

```bash
rei link list --all --domain DOMAIN --json | jq -r --arg url "URL" '.[] | select(.url == $url) | .id'
```

For each `note_...` / `link_...` / `topic_...` ID, confirm it exists (`rei note show ID`,
`rei link show ID`, `rei topic show ID`). For `[[wikilinks]]`, search note titles:

```bash
rei note list --title "WIKILINK TEXT" --json
```

### Phase 4: Build the Filing Plan

Decide each element. Skip rather than guess — every item is optional except the anchor.

- **Anchor** — the top intention candidate. Use a topic anchor only when the note is pure
  knowledge and no intention fits (then the anchor topic is the most specific subject topic).
- **Category** — at most one, only when clearly fitting.
- **Tags** — 3–7 facets. Rules, in order:
  1. **Prefer reuse** — use existing tags verbatim; watch singular/plural, abbreviation,
     synonym, and case variants and reuse whichever already exists.
  2. **Normalize new tags** — lowercase, hyphen-separated, singular-where-natural.
  3. **Don't duplicate structure** — don't re-encode the category, the intention title, or the
     note's format (`note`, `draft`, `markdown`) as tags.
  4. **Complement topics** — a subject that gets a topic can still carry a tag for quick
     filtering, but never use a tag *instead of* a topic for a real subject.
- **Custom properties** — only properties that apply to notes, are in scope for the chosen
  category, and whose value the content makes obvious. Enum and state-machine values must come
  from the definition; never invent values. For state machines, only set the initial or an
  obviously correct state. Use `append-property` for multi-value types when the author hinted
  additions.
- **Topics** — every subject from Phase 2 gets an `about` association. Mark each **REUSE** or
  **CREATE**. Every topic to create needs a classification (`instance-of` its type) or a
  placement (`broader-than` from a parent concept), creating the type/parent too if missing,
  plus a canonical reference when it has one. Follow the Modeling Discipline in
  `/rei-bookmark-url` (no near-duplicate topics, instance-of vs broader-than, no orphans,
  shallow hierarchies, no topic for a publisher).
- **Edges** —
  - `note -[references]-> link` for each URL that already has a link. For URLs without a link,
    propose creating one anchored to the same intention (`rei link add`) only when the URL is
    substantive to the note, not incidental.
  - `note -[summarizes]-> link|note|topic` only when the note genuinely *is* a summary of that
    entity.
  - Note → note relationships (wikilinks to existing notes) have no seeded predicate. Do **not**
    invent a key; if one is truly needed, name it from Schema.org as in `/rei-bookmark-url`
    Phase 6, and call it out in the plan.

Present the plan:

```
Filing plan for: /tmp/scratch.md
Title: <TITLE>
<one-line SUMMARY>

ANCHOR:      intention_01... — <intention title>
             (alternatives: intention_01... — <title>, intention_01... — <title>)
CATEGORY:    work/project            (or: none)
TAGS:        event-sourcing (reuse), postgres (reuse), fifo-ordering (new)
PROPERTIES:  language = English
             reading-status = requires-synthesis   (or: none)

TOPICS (about):
- keiro — Keiro                      REUSE
- pgmq — PGMQ                        CREATE (instance)
    instance-of → message-queue         REUSE
    ref → https://github.com/pgmq/pgmq

EDGES:
- note -[references]-> link_01...  (https://...)
- note -[references]-> NEW LINK     (https://...)  anchored to the same intention

AFTER VERIFYING: delete /tmp/scratch.md
```

### Phase 5: Confirm with the User

This skill deletes a file, so the plan is an approval gate. Ask both questions in one
AskUserQuestion call (the intention options list the Phase 3a shortlist):

```
Question: "Which intention should this note be anchored to?"
Header: "Intention"
Options:
- <top candidate title> (Recommended)
- <second candidate title>
- <third candidate title>
- Anchor to a topic instead
```

```
Question: "Apply this filing plan and delete the temp file afterwards?"
Header: "Plan"
Options:
- Apply everything and delete the file (Recommended)
- Apply everything but keep the file
- Let me adjust the plan
- Cancel
```

If the user adjusts, apply their edits, re-validate tags against the vocabulary and property
values against their definitions, show the revised plan, and ask again. On **Cancel**, stop
without writing anything to Rei and without touching the file.

If the user picks "Other" on the intention question with an ID or a search term, resolve it with
`rei intention show ID` or `rei intention list --all -s TERM --json`.

### Phase 6: Create the Note

Store the file content **verbatim**, frontmatter included — Rei becomes the only copy once the
file is deleted.

Intention anchor:

```bash
rei note new -i INTENTION_ID -c CATEGORY_SLUG --stdin --actor claude-code < "PATH"
```

Topic anchor (use the `topic_...` ID):

```bash
rei note new --topic TOPIC_ID -c CATEGORY_SLUG --stdin --actor claude-code < "PATH"
```

Omit `-c` when no category was chosen. Capture the new `note_...` ID from the output as
`NOTE_ID` (e.g. `| grep -o 'note_[0-9a-z]*' | head -1` if the output is verbose).

If a topic anchor is being created in Phase 8, create that topic first, then the note.

Set the title when it came from frontmatter or was derived (so listings don't show a first-line
fallback):

```bash
rei note set-title -n NOTE_ID "TITLE"
```

Skip `set-title` when the note's first line is already the `# TITLE` heading.

### Phase 7: Apply Tags and Custom Properties

Tags are a `tag-set`; pass the full set as one comma-separated value in a single call —
`set-property` replaces the set:

```bash
rei note set-property -n NOTE_ID tags "tag-one,tag-two,tag-three"
```

Other properties, one call each:

```bash
rei note set-property -n NOTE_ID KEY VALUE
rei note append-property -n NOTE_ID KEY "value-a" "value-b"   # multi-value types, if needed
```

`append-property` takes **one shell argument per element** — commas inside a value are kept
literally, so don't pass a comma-joined string to it.

Always pass `-n NOTE_ID`; omitting it opens an fzf picker that blocks a non-interactive run. If a
set fails validation, run `rei custom-property show KEY`, fix the value, and retry once; if it
still fails, skip that property and report it.

If the note contains tasks and a boolean property is meant to track their completion (per the
category's note guidance), link them:

```bash
rei note link-tasks --help   # check syntax first
```

### Phase 8: Place the Note in the Ontology

**8a — Seed predicates** (idempotent):

```bash
rei ontology seed-system
```

**8b — Create and wire missing topics**, types and parents first. Capture each `topic_...` ID:

```bash
rei --actor claude-code topic create KEY "LABEL" --description "DESCRIPTION"

# instance → type (target-first argument order: reads "INSTANCE instance-of TYPE")
rei topic associate TYPE_TOPIC_KEY INSTANCE_TOPIC_ID --relation instance-of

# concept placement: broader -[broader-than]-> narrower (IDs, not keys)
rei --actor claude-code edge add --from PARENT_TOPIC_ID --to NEW_TOPIC_ID --predicate broader-than

# canonical identity, so the next filing finds this topic
rei topic add-ref TOPIC_KEY https://CANONICAL-HOMEPAGE --label "LABEL"
```

Before classifying, check `rei topic edges TYPE_TOPIC_KEY`: if siblings use `is-a` rather than
`instance-of`, stay consistent with them (`rei --actor claude-code edge add ... --predicate is-a`)
and mention the divergence — see `/rei-bookmark-url` Phase 6.

**8c — Associate the note with every subject topic** (skip the anchor topic if the note is
topic-anchored):

```bash
rei topic associate TOPIC_KEY NOTE_ID --relation about
```

**8d — Create links and edges** from the plan:

```bash
# new link for a substantive URL, anchored like the note
rei link add "URL" -i INTENTION_ID -t "LINK TITLE" --actor claude-code

rei --actor claude-code edge add --from NOTE_ID --to LINK_ID --predicate references
rei --actor claude-code edge add --from NOTE_ID --to TARGET_ID --predicate summarizes
```

If an edge fails because a predicate is missing or doesn't allow these types, report it; do not
loosen an existing predicate's source/target types as a side effect.

### Phase 9: Verify

Deletion depends on this phase passing.

**9a — Content check.** The stored note must contain the full file content:

```bash
diff <(rei note print NOTE_ID) "PATH" && echo IDENTICAL
```

If `diff` shows only trailing-newline/whitespace differences, treat it as a pass. Any other
difference (truncation, missing sections) is a **failure**: do not delete the file; report the
note ID and the difference.

**9b — Entity check.**

```bash
rei note show NOTE_ID
rei topic associations NOTE_ID
rei edge show NOTE_ID
```

Confirm the anchor, category, tags, properties, associations, and edges match what was applied.

**9c — Ontology check.**

```bash
rei ontology validate
```

An error caused by something just written must be fixed (fix the edge, never the predicate).
Pre-existing errors are surfaced and left alone — point at `/rei-curate-ontology`. Ontology
validation problems do not block deletion once 9a passed, but report them.

### Phase 10: Delete the Temporary File

Only when **all** of these hold:
- the user chose "Apply everything and delete the file" in Phase 5,
- the note was created and 9a passed.

```bash
rm -- "PATH"
test ! -e "PATH" && echo deleted
```

Delete only the one file that was filed — never its directory, never globs, never sibling files
(even if they look like related temp notes). If partial failures happened earlier (a property or
edge failed) the content is still safe in Rei, so deletion may proceed; list those failures in
the summary.

## Output Format

```
## Note Filed

- **File**: <path>  (deleted / kept — <reason>)
- **Note**: <note_id> — <title>
- **Anchor**: <intention_id — title | topic key — label>
- **Category**: <slug or "none">

### Tags
- <tag-one, tag-two, ...>  (N reused, M new)

### Custom Properties
- <key>: <value>
- (or: none applied)

### Ontology
- Topics reused: <key>, <key>
- Topics created: <key> instance-of <type> | <key> broader-than ← <parent>
- about: <topic-key>, <topic-key>
- Edges: note -[references]-> <link_id>, ...
- rei ontology validate: <passed / N pre-existing errors, unchanged>

### Skipped / Failed
- <item — reason>   (or: nothing)

### Next Steps
- Review: `rei note show <note_id>`
- Edit: `rei note open <note_id>`
- Browse subjects: `rei topic associations <note_id>`
```

## Important Notes

- **Never delete the file before the note is verified.** Cancelled, failed, or mismatched runs
  leave the file untouched. Delete only the single file passed in, with `rm --`.
- **Store content verbatim.** Don't rewrite, summarize, or strip the note — the file is about to
  disappear and Rei must hold exactly what the user wrote.
- Always use `--actor claude-code` on entity creation. Flag position differs: `note new` and
  `link add` take a trailing `--actor`; `topic create`, `predicate define`, and `edge add` take
  the global `rei --actor claude-code <command>`; `rei topic associate` and `set-property` take
  none.
- **Explicit hints win.** Frontmatter keys (`intention`, `tags`, `topics`, `category`, property
  keys) outrank inference — but still validate them against Rei.
- **Subjects are topics, facets are tags.** Every real subject gets a topic and an `about`
  association; tags add quick filters on top and never replace a topic.
- **Tags are a `tag-set`**: one `set-property … tags "a,b,c"` call with the whole set. Reuse the
  existing vocabulary verbatim.
- **Property values must come from the definition.** Respect `scopedToCategories`; skip a
  property rather than guess its value.
- **Follow `/rei-bookmark-url`'s modeling discipline** for topics: no near-duplicates (check
  `rei topic ref-show`), `instance-of` for named things, `broader-than` only between concepts, no
  orphaned topics, no invented predicates when Schema.org has a term.
- **Always pass IDs explicitly** (`-n NOTE_ID`, `-i INTENTION_ID`) — omitted IDs open fzf pickers
  that block non-interactive runs. `rei edge add` needs full IDs, not topic keys.
- **Don't restructure the ontology as a side effect** — surface pre-existing issues and point at
  `/rei-curate-ontology`.

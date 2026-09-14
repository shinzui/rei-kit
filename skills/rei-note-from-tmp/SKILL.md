---
name: rei-note-from-tmp
description: File a temporary markdown note into Rei — analyze its content, propose the intention (or topic) to anchor it to, a category, facet tags, and applicable note custom properties, associate it `about` reused or newly created topics, rely on Rei's automatic extraction for the URLs and wikilinks it contains, create the note verbatim, verify the stored content and every write by reading back, and only then delete the temporary file.
allowed-tools: AskUserQuestion, Bash, Read
---

# Rei Note From Tmp

Files a **temporary markdown note** — a scratch file from an editor, another tool, or an agent —
into Rei. It works out where the note belongs and what it is about, proposes a filing plan
(anchor, category, tags, properties, topics, edges), applies it after one approval, verifies Rei
holds the full content, and **only then** deletes the file.

Use `rei-note-from-url-markdown` instead for a *web page snapshot* tied to a source URL, and
`rei-bookmark-url` to file a *link*. This skill is for notes the user wrote, with no single
source URL.

## Key Concepts

- **The file is not the source of truth once filed.** Nothing is deleted until the stored
  content is verified.
- **Anchor** — exactly one: an intention (usual), action, outcome, habit, or **topic** (only for
  pure knowledge notes that serve no intention).
- **Category** — optional (`work/project`). It gates scoped custom properties, may carry note
  guidance (`rei category print-note-guidance SLUG`), and may have **property bindings** that
  auto-set properties when assigned (listed by `rei category show SLUG`) — don't set those by
  hand.
- **`tags`** — the global `tag-set` custom property. Facets for filtering, not subjects. (This
  is distinct from `rei note new -t`, a separate note tag; don't use that.)
- **Topics** — subjects go in topics via `rei topic associate TOPIC NOTE_ID --relation about`.
  Topic creation and wiring follow `rei-bookmark-url`'s Modeling Discipline.
- **Automatic extraction** — Rei's worker processes note content after creation: URLs become
  Link entities attached to the note (actor `note-extractor`), `[[wikilinks]]` become
  references resolved by title (`rei note outgoing-links`, `rei note broken-links`), `- [ ]`
  tasks are discovered, and an H1 becomes the title. **Don't recreate any of this by hand** —
  no `link add` for URLs in the note, no invented note → note predicate for wikilinks.
- **Exit status** — `note new` follows the automation exit contract (`2` refused/invalid, `70`
  store failure); property, edge, and topic-create writes may print an error and exit `0`.
  Verify by reading back.

## Instructions

Use `rei --actor claude-code <command>` for every write and always pass IDs explicitly
(`-n NOTE_ID`, `-i INTENTION_ID`, full entity IDs) — omitted IDs open fzf pickers that hang.

### Phase 1: Validate the File

Take the path from the argument or ask. Stop if it doesn't exist, isn't readable, or is empty
(and don't delete it). Confirm before continuing if it isn't clearly markdown. If it lives in a
git repository or notes vault rather than a scratch location (`/tmp`, a scratchpad,
`~/Downloads`), say so before the deletion step — it may not really be temporary.

### Phase 2: Analyze the Note

Read the whole file and extract:

- **Frontmatter hints** — `title`, `tags`, `intention`/`intention_id`, `topic`/`topics`,
  `category`, or keys matching note property keys. Explicit hints outrank inference, but are
  still validated.
- **TITLE** — frontmatter `title`, else the first `# Heading`, else derived.
- **SUMMARY** — one or two sentences on what it is about and what it is *for*.
- **SUBJECTS** — 1–5 central subjects, each **instance** (named product/tool/project/person) or
  **concept** (subject area). Ignore passing mentions.
- **INTENTION SIGNALS** — project/repo names, goals, `Intention:` trailers, `intention_…` IDs.
- **REFERENCES** — URLs, `[[wikilinks]]`, and Rei entity IDs (`note_…`, `link_…`, `topic_…`).
- **FACETS** — language, tasks, dates, anything matching a note property.

### Phase 3: Survey Rei

Run these in parallel where possible.

```bash
# intentions — per signal/keyword; shortlist up to 3 (verify any hinted ID first)
rei intention list --all -s "KEYWORD" --json | jq -r '.[] | "\(.id)\t\(.title)"'

# categories, with bindings and guidance
rei category list --flat --descriptions --json \
  | jq -r '.[] | select(.status == "active") | "\(.slug)\t\(.description // "")\t\(.propertyBindings | length) bindings"'
rei category show SLUG
rei category print-note-guidance SLUG

# note custom properties (skip archived and smoke-test noise)
rei custom-property list -e note --json \
  | jq -r '.[] | select(.isArchived | not) | "\(.key)\t\(.valueType.type)\t\(.scopedToCategories)"'
rei custom-property show KEY --json        # enum values, state machine states

# tag vocabulary (one comma-separated string per entity)
rei custom-property entities tags --json \
  | jq -r '.entities[].value' | tr ',' '\n' | sed 's/^ *//; s/ *$//' | sort -u

# ontology
rei topic list --json | jq -r '.[] | "\(.topicKey)\t\(.topicLabel)\t\(.topicId)"' | grep -i "SUBJECT"
rei topic ref-show https://SUBJECT-HOMEPAGE --json     # exit 2 = no topic owns it
rei topic show TOPIC --json
rei topic edges TOPIC --json
rei predicate list --json | jq -r '.[] | "\(.predicateKey)\t\(.sourceTypes)\t\(.targetTypes)"'
```

For referenced entity IDs, confirm they exist (`rei note show ID --json`, `rei link show ID`,
`rei topic show ID --json`). For wikilinks, check the target exists so a broken reference can be
flagged in the plan: `rei note list --title "WIKILINK TEXT" --json | jq -r '.[] | "\(.noteId)\t\(.title)"'`.
For URLs the note is *about* (not incidental), look for an existing curated link:

```bash
rei link list --all -d DOMAIN --json \
  | jq -r --arg u "URL" '.[] | select(.original_url == $u or .canonical_url == $u) | .id'
```

### Phase 4: Build the Filing Plan

Everything except the anchor is optional; skip rather than guess.

- **Anchor** — the top intention; topic anchor only for pure knowledge notes (the most specific
  subject topic).
- **Category** — at most one, only when it clearly fits.
- **Tags** — 3–7 facets. Reuse existing tags verbatim (watch plural/abbreviation/synonym/case
  variants); new tags lowercase and hyphenated; don't re-encode the category, intention title,
  or note format; never use a tag instead of a topic.
- **Properties** — only note properties in scope for the chosen category, not already set by a
  category binding, and whose value the content makes obvious. Enum/state-machine values come
  from the definition; for state machines set only the initial or an obviously correct state.
- **Topics** — every subject gets an `about` association, marked REUSE or CREATE. Each new topic
  needs `instance-of` its type or `broader-than` from a parent (created too if missing) and a
  canonical reference when it has one.
- **Edges** — only relationships extraction doesn't already capture:
  - `note -[summarizes]-> link|note|topic` only when the note genuinely is a summary of it.
    Target the existing curated link if there is one.
  - Any other predicate only if it already exists and allows the types; if a new one is truly
    needed, name it from Schema.org as in `rei-bookmark-url` Phase 6 and call it out.

Present the plan:

```
Filing plan for: /tmp/scratch.md
Title: <TITLE> — <SUMMARY>

ANCHOR:      intention_… — <title>   (alternatives: …)
CATEGORY:    work/project (binds: language=English)   | none
TAGS:        event-sourcing (reuse), fifo-ordering (new)
PROPERTIES:  reading-status = requires-synthesis   | none
TOPICS (about):
- keiro — Keiro                 REUSE
- pgmq — PGMQ                   CREATE instance-of message-queue (REUSE), ref https://github.com/pgmq/pgmq
EDGES:       note -[summarizes]-> link_… | none
EXTRACTED BY REI: 3 URLs → links on the note; wikilinks [[Foo]] (resolves), [[Bar]] (broken)

AFTER VERIFYING: delete /tmp/scratch.md
```

### Phase 5: Confirm

One AskUserQuestion call with two questions: the intention (shortlist, top one Recommended, plus
"Anchor to a topic instead"), and the plan — *apply and delete the file* (Recommended) / *apply
but keep the file* / *let me adjust* / *cancel*. Re-validate and re-show after adjustments. On
cancel, write nothing and leave the file alone. Resolve an "Other" intention answer with
`rei intention show ID --json` or a search.

### Phase 6: Create the Note

If the anchor topic is to be created, do Phase 8b for it first. Store the content **verbatim**,
frontmatter included:

```bash
rei --actor claude-code note new -i INTENTION_ID [-c CATEGORY_SLUG] --stdin < "PATH"
rei --actor claude-code note new --topic TOPIC_ID [-c CATEGORY_SLUG] --stdin < "PATH"
```

Capture `NOTE_ID` (`grep -o 'note_[0-9a-z]*' | head -1`). A non-zero exit means nothing was
filed — stop and keep the file.

If the title came from frontmatter or was derived (the first line isn't `# TITLE`), set it:

```bash
rei --actor claude-code note set-title -n NOTE_ID "TITLE"
```

### Phase 7: Tags and Properties

```bash
# tag-set: one call with the whole set (set-property replaces it)
rei --actor claude-code note set-property -n NOTE_ID tags "tag-one,tag-two,tag-three"

rei --actor claude-code note set-property -n NOTE_ID KEY VALUE
# multi-value: one shell argument per element
rei --actor claude-code note append-property -n NOTE_ID KEY "value-a" "value-b"
```

On a validation error, check `rei custom-property show KEY`, fix, retry once, else skip and
report. If the category's note guidance calls for linking the note's tasks to a boolean
property: `rei --actor claude-code note link-tasks -n NOTE_ID --source-note NOTE_ID -p KEY`.

### Phase 8: Ontology

**8a** — `rei ontology seed-system` (idempotent).

**8b — Create and wire missing topics**, types and parents first. `topic create` prints no JSON:

```bash
rei --actor claude-code topic create KEY "LABEL" -d "DESCRIPTION"
rei topic show KEY --json | jq -r .topicId

rei --actor claude-code topic associate TYPE_TOPIC INSTANCE_TOPIC_ID --relation instance-of   # target-first
rei --actor claude-code edge add -f PARENT_TOPIC_ID -t NEW_TOPIC_ID -p broader-than
rei --actor claude-code topic add-ref TOPIC https://CANONICAL-HOMEPAGE -l "LABEL"
```

If siblings under the type use `is-a` (`rei topic edges TYPE_TOPIC --json`), stay consistent
with `edge add … -p is-a` and mention it. If `add-ref` is refused because another topic owns
the reference, reuse that topic.

**8c — About associations** (skip the anchor topic for a topic-anchored note). Always pass
`--relation about` — the default is `scoped-to`:

```bash
rei --actor claude-code topic associate TOPIC NOTE_ID --relation about
```

**8d — Edges** from the plan. Check the predicate exists (`predicate show` exits 0 even when
missing):

```bash
rei predicate list --json | jq -e '.[] | select(.predicateKey == "summarizes")' >/dev/null
rei --actor claude-code edge add -f NOTE_ID -t TARGET_ID -p summarizes
```

If a predicate is missing or disallows the types, report it; never loosen an existing predicate.

### Phase 9: Verify

Deletion depends on **9a**.

**9a — Content.**

```bash
diff <(rei note print NOTE_ID) "PATH" && echo IDENTICAL
```

Trailing-newline/whitespace-only differences pass. Anything else (truncation, missing sections)
fails: keep the file and report the note ID and the difference.

**9b — Writes.**

```bash
rei note show NOTE_ID --json | jq '{title, anchor, categoryId}'
rei custom-property entities KEY -e note --json \
  | jq -r --arg n NOTE_ID '.entities[] | select(.entity_id == $n) | .value'   # per property, incl. tags
rei topic associations NOTE_ID --relation about --json | jq -r '.[].topic.key'
rei edge show NOTE_ID --json | jq -r '.[] | "\(.predicateKey) -> \(.targetId) \(.status)"'
```

Redo any write that silently didn't land (once); otherwise report it.

**9c — Extraction.** Once the worker has processed the note, `rei note show NOTE_ID --json`
lists `extractedLinks` and `outgoingReferences`, and `rei note broken-links NOTE_ID` lists
unresolved wikilinks. If they are still empty, the worker may not be running — report it; don't
recreate links by hand. This does not block deletion.

**9d — Ontology.** `rei ontology validate`. Fix errors caused by your edges (the edge, never the
predicate); surface pre-existing ones and point at `rei-curate-ontology`. Doesn't block deletion.

### Phase 10: Delete the File

Only if the user chose "apply and delete" **and** 9a passed:

```bash
rm -- "PATH" && test ! -e "PATH" && echo deleted
```

Delete only that one file — never its directory, globs, or siblings. Failed properties or edges
don't block deletion (the content is safe in Rei); list them in the summary.

## Output Format

```
## Note Filed

- **File**: <path> (deleted | kept — <reason>)
- **Note**: <note_id> — <title>
- **Anchor**: <intention_id — title | topic key — label>
- **Category**: <slug (bindings applied: …) | none>
- **Tags**: <tags> (N reused, M new)
- **Properties**: <key=value, … | none>

### Ontology
- Topics reused: <keys>; created: <key> instance-of <type> | <key> broader-than ← <parent>
- about: <keys>
- Edges: <note -[summarizes]-> id | none>
- Extracted by Rei: <N links, M wikilinks (K broken) | pending — worker not running?>
- ontology validate: <passed | N pre-existing errors>

### Skipped / Failed
- <item — reason> | nothing

### Next Steps
- `rei note show <note_id>` · `rei note open <note_id>` · `rei topic associations <note_id>`
```

## Important Notes

- **Never delete before 9a passes**; cancelled, failed, or mismatched runs leave the file alone.
- **Store content verbatim** — Rei becomes the only copy.
- **Subjects are topics, facets are tags**; follow `rei-bookmark-url` for topic modeling and
  don't restructure the ontology as a side effect.

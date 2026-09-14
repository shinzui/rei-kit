---
name: rei-ingest-url
description: Ingest a single URL into Rei — reuse or create a link anchored to an intention, summarize the content into a note, connect them with a `summarizes` edge, classify the link with author-type, content-type, media, and platform, tag it with reused/new facet tags, and associate the link and note `about` the existing topics they cover.
allowed-tools: AskUserQuestion, Bash, Read, WebFetch
---

# Rei Ingest URL

Ingests one URL into Rei: a link anchored to an intention, a summary note on the same
intention, a `note -[summarizes]-> link` edge, enum classification of the link, facet `tags`,
and `about` associations to the topics that already exist for the page's subjects.

Use `/rei-bookmark-url` instead when the link should be filed under a topic and the ontology
grown to hold it (it creates topics; this skill only reuses them). Use
`/rei-note-from-url-markdown` or `/rei-ingest-markdown` when a markdown snapshot of the page
already exists. `/rei-ingest-url-collection` runs this skill per URL and refers to its phase
numbers — keep Phases 1–9 and steps 9a–9e stable.

## Key Concepts

- **Link dedup** — Rei canonicalizes URLs, and `rei link add` on a known URL reuses the link
  and just adds an attachment. Still check first, so a re-ingest doesn't create a second
  summary note.
- **Enum classifiers** — `author-type`, `content-type`, `media`, `platform` are link-only enum
  properties; only defined values are valid (`rei custom-property show KEY --json` →
  `.valueType.data.enumValues`).
- **Tags vs topics** — `tags` is a `tag-set` of lightweight facets for filtering. Real subjects
  belong in topics, recorded with `rei topic associate TOPIC ENTITY --relation about`. This
  skill associates with existing topics only; subjects with no topic are reported, with a
  pointer to `/rei-bookmark-url` (which follows the full modeling discipline).
- **Exit status** — `link`, `edge`, and `custom-property` writes may print an error and still
  exit `0`. Capture IDs from output and verify by reading back (step 9e).

## Instructions

Use the global actor form `rei --actor claude-code …` for every write, and always pass IDs
explicitly (`-i`, `-l`, `-n`, full entity IDs) — omitted IDs open fzf pickers that hang.

### Phase 1: Get the URL

From the argument, or ask. It must be an absolute `http(s)://` URL; otherwise stop.

### Phase 2: Check for an Existing Link

```bash
rei link list --all --domain DOMAIN --json \
  | jq -r --arg url "URL" '.[] | select(.original_url == $url or .canonical_url == $url) | .id'
```

`DOMAIN` is the registrable domain (`github.com`). If nothing matches, also try
`rei link list --all --query "URL" --json` (the stored URL may be canonicalized differently).

If a link exists: capture `LINK_ID` and inspect it with `rei link show LINK_ID` (the text view
lists attachments and custom properties; `--json` omits properties). Reuse an intention it is
attached to, skip Phases 3–4, don't overwrite enum properties it already has, and check
`rei edge show LINK_ID --json` for an existing incoming `summarizes` edge — if one exists,
don't create another summary note unless the user asks.

### Phase 3: Get the Intention ID (new links only)

Ask for the intention to anchor the link and note to. If the user wants to browse, search with
`rei intention list --all -s "KEYWORD" --json` and offer the top matches.

### Phase 4: Create the Link

```bash
rei --actor claude-code link add "URL" -i INTENTION_ID -t "TITLE"
```

Omit `-t` if the title isn't known yet and set it after Phase 5 with
`rei --actor claude-code link title LINK_ID -t "TITLE"`. Capture `LINK_ID` (`link_…`).

### Phase 5: Fetch and Summarize

Fetch with WebFetch, asking for the title, author, a thorough markdown summary (key points,
arguments, notable quotes/data), and hints for author type, content type, media, and platform.
Extract `TITLE`, `SUMMARY_MD`, the classification hints, and the 1–5 central `SUBJECTS`.

If the fetch fails, stop and report it; a link created in Phase 4 stays in place for a retry.

### Phase 6: Create the Summary Note

```bash
printf '# Summary of "%s"\n\nSource: %s\n\n%s\n' "TITLE" "URL" "$SUMMARY_MD" \
  | rei --actor claude-code note new -i INTENTION_ID --stdin
```

Capture `NOTE_ID` (`note_…`). `note new` follows the exit contract: `2` = refused/invalid,
`70` = store failure.

### Phase 7: Create the Edge

```bash
rei predicate list --json | jq -e '.[] | select(.predicateKey == "summarizes")' >/dev/null || \
  rei --actor claude-code predicate define summarizes --label "Summarizes" \
    --source-types note --target-types link,note,topic
rei --actor claude-code edge add --from NOTE_ID --to LINK_ID --predicate summarizes
```

`summarizes` is not part of `rei ontology seed-system`. Never redefine or loosen an existing
predicate.

### Phase 8: Classify the Link

Set each of `author-type`, `content-type`, `media`, `platform` only when a defined value clearly
fits; skip rather than guess. Read the allowed values at runtime:

```bash
rei custom-property show KEY --json | jq -r '.valueType.data.enumValues[]'
rei --actor claude-code link set-property -l LINK_ID KEY VALUE
```

### Phase 9: Tag and Associate Topics

**9a — Harvest vocabulary.**

```bash
rei custom-property entities tags --json \
  | jq -r '.entities[].value' | tr ',' '\n' | sed 's/^ *//; s/ *$//' | sort -u
rei topic list --json | jq -r '.[] | "\(.topicKey)\t\(.topicLabel)"'
```

Tag values are one comma-separated string per entity. For named subjects with a homepage,
`rei topic ref-show https://HOMEPAGE --json` finds a topic filed under another key.

**9b — Propose.**
- **Topics**: for each subject, the existing topic that genuinely covers it (match key, label,
  or reference). Subjects with no topic go on an "uncovered" list — do not create topics here.
- **Tags**: 3–7 facets. Reuse existing tags verbatim (watch plural, abbreviation, synonym, and
  case variants); new tags are lowercase and hyphenated; never re-encode the enum classifiers
  (`article`, `github`) as tags; keep them specific but not sentence-like.

**9c — Confirm.** Show tags (marked reused/new) and topic associations, and ask to apply as
proposed, edit, or skip. Re-check edits against the vocabulary for near-duplicates.

**9d — Apply.**

```bash
# new link: one call with the full set (set-property replaces the set)
rei --actor claude-code link set-property -l LINK_ID tags "tag-one,tag-two,tag-three"
# reused link that already has tags: append, one shell argument per tag
rei --actor claude-code link append-property -l LINK_ID tags tag-one tag-two

rei --actor claude-code topic associate TOPIC_KEY LINK_ID --relation about
rei --actor claude-code topic associate TOPIC_KEY NOTE_ID --relation about
```

**9e — Verify.**

```bash
rei link show LINK_ID                                   # attachments, properties, tags
rei edge show NOTE_ID --predicate summarizes --json     # targetId == LINK_ID
rei topic associations LINK_ID --relation about --json
```

Retry a missing write once; otherwise report it as failed.

### Phase 10: Summary

```
## URL Ingested

- **URL**: <url>
- **Link**: <link_id>  (reused / new)
- **Intention**: <intention_id>
- **Note**: <note_id>
- **Edge**: <edge_id>  (note -[summarizes]-> link)

### Classification
- author-type / content-type / media / platform: <values or "skipped">
- tags: <tag-one, tag-two, ...>  (N reused, M new)
- about: <topic-key, ...>
- uncovered subjects: <subject, ...> — file with `/rei-bookmark-url` to grow the ontology

### Failed / Skipped
- <item — reason>  (or: nothing)
```

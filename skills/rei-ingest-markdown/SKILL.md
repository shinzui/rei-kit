---
name: rei-ingest-markdown
description: Ingest a pre-converted markdown file (with frontmatter) into Rei — reuse or create a link from the source URL, store the full markdown verbatim as an archive note, summarize it into a separate summary note, wire them with `archives` and `summarizes` edges, classify the link with author-type, content-type, media, and platform, tag it with reused/new facet tags, associate the link and summary note `about` the existing topics they cover, and verify everything by reading it back.
allowed-tools: AskUserQuestion, Bash, Read
---

# Rei Ingest Markdown

Ingests a markdown file produced by a URL-to-markdown converter (Defuddle, Jina Reader, …)
into Rei. The frontmatter supplies the source URL. The skill reuses or creates a link for that
URL, stores the **whole file verbatim** as an archive note, writes a separate **summary note**,
connects the three with edges, classifies and tags the link, associates it with the topics that
already exist for its subjects, and verifies the result.

Siblings: `rei-ingest-url` is the same flow from a live URL, without an archive copy — keep the
two consistent. `rei-note-from-url-markdown` stores a snapshot as a single note with no summary.
Use `rei-bookmark-url` when the ontology should grow to hold the page: this skill **reuses
topics only** and reports subjects that have none.

## When to Use

- "Ingest this markdown file into Rei"
- "Archive /tmp/article.md and summarize it"
- "/rei-ingest-markdown /tmp/article.md"

## Key Concepts

- **Archive note** — the file stored byte-for-byte (frontmatter included), so the page is
  recoverable from Rei alone. Unrelated to `rei note new --archive`, which hides the note —
  never pass that flag.
- **Summary note** — a separate, concise note for quick recall.
- **Edges** — `archive_note -[archives]-> link`, `summary_note -[summarizes]-> link`, and
  `summary_note -[summarizes]-> archive_note`. Both predicates normally exist already.
- **Classifiers** — `author-type`, `content-type`, `media`, `platform`: link-only enums
  describing shape and source. Values must come from the definition.
- **Tags vs topics** — `tags` is a `tag-set` of lightweight filter facets. Real subjects belong
  in topics, recorded with `rei topic associate TOPIC ENTITY --relation about`.
- **Canonical-URL dedup** — `rei link add` normalizes the URL and reuses an existing link,
  adding an attachment rather than a duplicate entity.

## Instructions

Use the global actor form, `rei --actor claude-code <command>`, on every write. Always pass IDs
explicitly (`-l LINK_ID`, `-n NOTE_ID`, `-i INTENTION_ID`); an omitted ID opens an fzf picker
that hangs a non-interactive run. Many rei commands still exit `0` on failure, so trust the
read-back in Phase 9, not exit codes.

### Phase 1: Parse the File

Take the path from the arguments or ask for it. Stop if it is missing, unreadable, or empty.
Read the whole file and parse the leading YAML frontmatter:

- **URL** (required) — first of `url`, `source`, `source_url`, `canonical_url`, `link`. Must be
  an absolute `http(s)` URL. If there is no frontmatter or no URL key, ask the user for the URL
  (offering to abort if it isn't from a URL) — never guess.
- **TITLE** — `title`, `name`, `headline`. **AUTHOR** — `author`, `byline`, `creator`.
  **PUBLISHED** — `published`, `date`, `published_at`, `pubdate`. **SITE** — `site_name`,
  `site`, `source_site` (a `platform` hint).

### Phase 2: Check Existing State

`link list --json` exposes `canonical_url` and `original_url` — there is no `url` field:

```bash
rei link list --all --domain DOMAIN --json \
  | jq -r --arg url "URL" '.[] | select(.canonical_url == $url or .original_url == $url) | .id'
```

If nothing matches, retry with `--query "URL"` in place of `--domain` (a tracking-param variant
may have been canonicalized). If a link exists:

```bash
rei link show LINK_ID      # attachments, custom properties, tags (text; --json omits them)
rei edge show LINK_ID --json \
  | jq -r '.[] | select(.status == "active") | "\(.predicateKey)\t\(.sourceId)"'
rei topic associations LINK_ID --relation about --json
```

- Active `archives` / `summarizes` edges from notes mean the page was ingested before. Ask
  whether to **add a fresh archive** (the page changed) or **stop**; never duplicate silently.
- Record properties, tags, and associations already present so later phases don't overwrite
  or repeat them.

### Phase 3: Choose the Anchor

The link and both notes share one intention. Reuse a reused link's intention attachment when it
has one; otherwise take the intention from the request, or ask — offering candidates from
`rei intention list --all -s "KEYWORD" --json`.

### Phase 4: Analyze and Propose

From the body and frontmatter produce **SUMMARY_MD** (thesis plus 3–7 key points, preserving
notable claims, data, and quotes), a **TITLE** if the frontmatter lacked one, and the
**subjects** the content substantively covers. Then gather vocabulary:

```bash
rei custom-property show KEY --json | jq -r '.valueType.data.enumValues | join(", ")'
rei custom-property entities tags --json | jq -r '.entities[].value' \
  | tr ',' '\n' | sed 's/^ *//; s/ *$//' | sort -u
rei topic list --json | jq -r '.[] | "\(.topicKey)\t\(.topicLabel)"'
rei topic ref-show https://SUBJECT-HOMEPAGE --json     # topic filed under another key?
```

Propose:

- **Classifiers** — only confident values from each enum; skip keys already set.
- **Topics** — for each subject, the existing topic that genuinely covers it (match key, label,
  or reference). Subjects with no topic go on an **uncovered** list; do not create topics here.
- **Tags** — 3–7 facets. Reuse existing tags verbatim (watch plural, abbreviation, synonym, and
  case variants); new tags are lowercase and hyphenated; never re-encode classifiers
  (`article`, `github`) as tags; specific but not sentence-like.

Show the classifiers, tags (marked reused/new), and topic associations, and ask once whether to
apply as proposed, edit, or skip tagging/associations. Nothing has been written yet.

### Phase 5: Create the Link and Notes

```bash
rei --actor claude-code link add "URL" -i INTENTION_ID -t "TITLE"
```

Skip this when the link is already attached to the intention. Capture `LINK_ID`; fix a missing
title with `rei --actor claude-code link title LINK_ID -t "TITLE"`.

Archive note — the file verbatim, no wrapper, then an explicit title:

```bash
rei --actor claude-code note new -i INTENTION_ID --stdin < "PATH"
rei --actor claude-code note set-title -n ARCHIVE_NOTE_ID "Archive: TITLE"
```

Summary note:

```bash
printf '# Summary of "%s"\n\nSource: %s\n\n%s\n' "TITLE" "URL" "SUMMARY_MD" \
  | rei --actor claude-code note new -i INTENTION_ID --stdin
```

Capture each `note_...` ID from the output. `note new` exits `2` when the write is refused and
`70` on a store failure — stop and report on either.

### Phase 6: Edges

```bash
rei --actor claude-code edge add --from ARCHIVE_NOTE_ID --to LINK_ID --predicate archives
rei --actor claude-code edge add --from SUMMARY_NOTE_ID --to LINK_ID --predicate summarizes
rei --actor claude-code edge add --from SUMMARY_NOTE_ID --to ARCHIVE_NOTE_ID --predicate summarizes
```

Only if `rei predicate show KEY` reports a predicate missing, define it (never widen an existing
one):

```bash
rei --actor claude-code predicate define archives --label "Archives" --source-types note --target-types link
rei --actor claude-code predicate define summarizes --label "Summarizes" --source-types note --target-types link,note,topic
```

### Phase 7: Classify and Tag

```bash
rei --actor claude-code link set-property -l LINK_ID KEY VALUE        # once per classifier
rei --actor claude-code link set-property -l LINK_ID tags "tag-one,tag-two,tag-three"
# reused link that already has tags: append, one shell argument per tag
rei --actor claude-code link append-property -l LINK_ID tags tag-one tag-two
```

`set-property` on `tags` replaces the whole set. If a value is rejected, re-read the enum and
retry once; otherwise skip that key.

### Phase 8: Associate Topics

For each approved topic, associate the link and the summary note (skip existing associations;
the archive note is reachable through its edges):

```bash
rei --actor claude-code topic associate TOPIC_KEY LINK_ID --relation about
rei --actor claude-code topic associate TOPIC_KEY SUMMARY_NOTE_ID --relation about
```

### Phase 9: Verify

```bash
diff <(rei note print ARCHIVE_NOTE_ID) "PATH" && echo IDENTICAL   # trailing newline only = pass
rei edge show LINK_ID --json \
  | jq -r '.[] | select(.status == "active") | "\(.predicateKey) \(.sourceId)"'
rei edge show ARCHIVE_NOTE_ID --predicate summarizes --json
rei link show LINK_ID                                   # attachments, properties, tags
rei topic associations LINK_ID --relation about --json
rei topic associations SUMMARY_NOTE_ID --relation about --json
```

A content difference beyond trailing whitespace is a failure — report it with the note ID.
Re-run once any step whose result is missing from the read-back, then report what still is.

### Phase 10: Summary

```
## Markdown Ingested

- **File**: <path>
- **URL**: <url>
- **Link**: <link_id> (reused / new)
- **Intention**: <intention_id>
- **Archive note**: <note_id> (content verified)
- **Summary note**: <note_id>

### Edges
- archive -[archives]-> link; summary -[summarizes]-> link, archive

### Classification
- author-type / content-type / media / platform: <value or skipped>
- tags: <tag-one, tag-two, ...>  (N reused, M new)
- about: <topic-key, ...>
- uncovered subjects: <subject, ...> — file with `/rei-bookmark-url` to grow the ontology

### Skipped / Failed
- <item — reason> (or: nothing)
```

## Important Notes

- The URL comes from the frontmatter or the user, never inference.
- The archive note is the file exactly; the summary lives only in the summary note.
- If a later phase fails after the link and notes exist, leave them in place and report which
  phases remain.

---
name: rei-ingest-markdown
description: Ingest a pre-converted markdown file (with frontmatter) into Rei — create (or reuse) a link from the source URL in the frontmatter, save the full markdown as an archive note, summarize it into a separate summary note, connect them with `archives` and `summarizes` edges, and classify the link with author-type, content-type, media, platform, and tags properties.
allowed-tools: AskUserQuestion, Bash, Read
---

# Rei Ingest Markdown

This skill ingests a markdown file (typically produced by a URL-to-markdown converter such
as Defuddle, Jina Reader, or similar) into Rei. The file usually has YAML frontmatter
containing the source URL and metadata (title, author, date). The skill extracts the URL,
reuses or creates a link for it, saves the **full markdown content** as an archive note for
future reference, summarizes the content into a separate **summary note**, and wires the
three entities together with `archives` and `summarizes` edges. It then classifies the link
with the same enum properties (`author-type`, `content-type`, `media`, `platform`) and
topical `tags` as `rei-ingest-url`, reusing existing tag vocabulary wherever possible.

## When to Use

Activate when the user says things like:
- "Ingest this markdown file into Rei"
- "Archive /tmp/article.md and summarize it"
- "Add this saved markdown as a link with full archive and summary"
- "/rei-ingest-markdown /tmp/article.md"

## Key Concepts

- **Archive note** — a Rei note containing the **full markdown content** of the source page,
  preserved verbatim for future offline reference. Anchored to the same intention as the link.
- **Summary note** — a separate Rei note with a concise summary of the content. Anchored to
  the same intention as the link and the archive note.
- **Link** — a Rei entity for the source URL (extracted from the markdown frontmatter).
- **Edge** — a typed relationship between two entities, identified by a predicate.
- **`archives` predicate** — `archive_note -[archives]-> link`: this archive note holds a
  saved copy of the linked URL's content.
- **`summarizes` predicate** — already used by `rei-ingest-url`; here we use it twice:
  `summary_note -[summarizes]-> link` and `summary_note -[summarizes]-> archive_note`.
- **Frontmatter** — YAML block at the top of the markdown file, between `---` fences. Keys
  vary by converter but typically include some of: `url`, `source`, `source_url`, `title`,
  `author`, `published` / `date`, `site_name`. The URL is the only **required** field.
- **Custom properties on links** — `author-type`, `content-type`, `media`, `platform` (enum)
  and `tags` (tag-set). Same semantics as `rei-ingest-url`.

## Workflow Overview

1. **Get the markdown file path** — from args or from the user
2. **Read the file and parse frontmatter** — extract URL, title, author, etc.
3. **Check for existing link** — search by URL; if found, reuse it
4. **Get intention ID** — only when creating a new link
5. **Create the link** — `rei link add` with title from frontmatter
6. **Create the archive note** — pipe the full markdown into `rei note new`
7. **Create the `archives` edge** — `archive_note -[archives]-> link`
8. **Summarize the markdown content** — produce a concise summary
9. **Create the summary note** — short summary, attached to the same intention
10. **Create the `summarizes` edges** — `summary_note -[summarizes]-> link` and
    `summary_note -[summarizes]-> archive_note`
11. **Classify the link** — set `author-type`, `content-type`, `media`, `platform`
12. **Tag the link** — harvest existing tags, pick + propose new ones, set `tags`
13. **Summary** — show everything that was created

## Instructions for Claude

### Phase 1: Get the Markdown File Path

The path may be supplied as an argument. If not, ask:

```
Question: "What markdown file should I ingest into Rei? (e.g., /tmp/article.md)"
Header: "File"
Options:
- Let me paste the path
```

Validate that the path exists and is readable. If not, stop and tell the user.

### Phase 2: Read the File and Parse Frontmatter

Read the file with the Read tool. Then parse the YAML frontmatter (the block between the
first two `---` fences at the top of the file).

Extract these fields, trying common alternative keys for each:

- **URL** — try `url`, `source`, `source_url`, `canonical_url`, `link` (in that order).
  This field is **required**. If missing, stop and ask the user to supply it manually.
- **TITLE** — try `title`, `name`, `headline`. Optional but strongly preferred.
- **AUTHOR** — try `author`, `byline`, `creator`. Optional.
- **PUBLISHED** — try `published`, `date`, `published_at`, `pubdate`. Optional.
- **SITE** — try `site_name`, `site`, `source_site`. Optional, used as a hint for `platform`.

Validate that **URL** is a well-formed absolute URL (`http://` or `https://`). If not, stop
and report the problem.

Also capture the **full markdown body** (everything after the closing `---` fence, or the
whole file if there is no frontmatter) into `BODY_MD`. If there is no frontmatter, ask the
user for the URL explicitly:

```
Question: "This file has no frontmatter. What URL is this markdown a copy of?"
Header: "URL"
Options:
- Let me paste the URL
- This isn't from a URL — abort
```

### Phase 3: Check Whether a Link Already Exists

Extract the registrable domain from the URL (e.g., `example.com`) and search for an
existing link:

```bash
rei link list --all --domain DOMAIN --json | jq -r --arg url "URL" '.[] | select(.url == $url)'
```

If `jq` is unavailable, fall back to:

```bash
rei link list --all --query "URL" --json
```

and scan the output for an exact URL match.

- **If a matching link exists**: capture its ID and any anchor intention ID from the output,
  inform the user, and **skip Phase 4 and Phase 5**. Also skip property-setting (Phase 11)
  for enum keys that are already set on the existing link — check with
  `rei link show LINK_ID --json` first.
- **If no match**: proceed to Phase 4.

### Phase 4: Get Intention ID (only for new links)

Use AskUserQuestion:

```
Question: "Which intention should the link, archive note, and summary note be attached to? Provide the intention ID (e.g., intention_01h455vb4pex)."
Header: "Intention"
Options:
- Let me type the intention ID
- Let me browse intentions first
```

If the user wants to browse, run:

```bash
rei intention list
```

Then ask again for the ID.

### Phase 5: Create the Link

If `TITLE` was extracted from the frontmatter, pass it with `-t`:

```bash
rei link add "URL" -i INTENTION_ID -t "TITLE" --actor claude-code
```

Otherwise:

```bash
rei link add "URL" -i INTENTION_ID --actor claude-code
```

Capture the new link ID from the command output. If no title was set at creation time, you
can set it later with `rei link title LINK_ID "TITLE"` once a title is determined during
summarization.

### Phase 6: Create the Archive Note

The archive note holds the **full, verbatim markdown content** of the file — frontmatter
included — so that the saved page is preserved even if the source URL goes away.

Pipe the entire file contents into `rei note new`. Wrap with a small header for context:

```bash
{
  echo '# Archive: TITLE'
  echo ''
  echo 'Source: URL'
  [ -n "AUTHOR" ] && echo 'Author: AUTHOR'
  [ -n "PUBLISHED" ] && echo 'Published: PUBLISHED'
  echo ''
  echo '---'
  echo ''
  cat "/path/to/file.md"
} | rei note new -i INTENTION_ID --stdin --actor claude-code
```

(Substitute the bracketed placeholders inline rather than using shell variables — the
example above shows the structure.)

Capture the **archive note ID** from the output as `ARCHIVE_NOTE_ID`.

### Phase 7: Create the `archives` Edge

```bash
rei edge add --from ARCHIVE_NOTE_ID --to LINK_ID --predicate archives
```

If this fails because the `archives` predicate doesn't exist, define it once and retry:

```bash
rei predicate show archives >/dev/null 2>&1 || \
  rei predicate define archives --label "Archives" --source-types note --target-types link
```

### Phase 8: Summarize the Markdown Content

You already have the full markdown in `BODY_MD` from Phase 2 — no need to fetch anything.
Read it and produce:

- `SUMMARY_MD` — a concise but thorough markdown summary preserving the author's key points,
  arguments, and any notable quotes or data.
- Classification hints for `author-type`, `content-type`, `media`, `platform` (used in
  Phase 11). Use frontmatter signals (`SITE`, `AUTHOR`, URL domain) plus the body content.
- A definitive `TITLE` if it wasn't already in the frontmatter.

If the body is very long, focus the summary on the main argument or thesis plus 3–7 key
supporting points.

### Phase 9: Create the Summary Note

```bash
{
  echo '# Summary of "TITLE"'
  echo ''
  echo 'Source: URL'
  echo ''
  echo "$SUMMARY_MD"
} | rei note new -i INTENTION_ID --stdin --actor claude-code
```

Capture the **summary note ID** from the output as `SUMMARY_NOTE_ID`.

### Phase 10: Create the `summarizes` Edges

Two edges, both using the `summarizes` predicate:

```bash
rei edge add --from SUMMARY_NOTE_ID --to LINK_ID --predicate summarizes
rei edge add --from SUMMARY_NOTE_ID --to ARCHIVE_NOTE_ID --predicate summarizes
```

If the first call fails because the `summarizes` predicate doesn't exist, define it and
retry both:

```bash
rei predicate show summarizes >/dev/null 2>&1 || \
  rei predicate define summarizes --label "Summarizes" --source-types note --target-types link,note
```

### Phase 11: Classify the Link

For each of the four enum properties, pick the best-fitting value **from the allowed list**
based on the markdown content and frontmatter. Only set a property if you are reasonably
confident — skip it otherwise rather than guessing. If the link was reused from Phase 3 and
already has a value for a key, skip that key.

**Allowed values (authoritative reference — verify at runtime with `rei custom-property show KEY` if unsure):**

- `author-type`: `individual`, `company`, `organization`, `research_group`, `community`,
  `anonymous`, `government`, `media_outlet`, `mixed`
- `content-type`: `homepage`, `page`, `article`, `blog_post`, `essay`, `documentation`,
  `api_reference`, `tutorial`, `guide`, `research_paper`, `whitepaper`, `case_study`,
  `announcement`, `changelog`, `repository`, `repository_issue`, `repository_pr`,
  `repository_release`, `package`, `social_post`, `thread`, `discussion`, `qa_question`,
  `qa_answer`, `video`, `podcast`, `image`, `presentation`, `pdf`, `dataset`, `course`
- `media`: `text`, `video`, `audio`, `image`, `slides`, `interactive`, `dataset`, `mixed`
- `platform`: `website`, `blog`, `x`, `linkedin`, `reddit`, `hackernews`, `github`,
  `gitlab`, `bitbucket`, `youtube`, `vimeo`, `substack`, `medium`, `notion`, `wikipedia`,
  `arxiv`, `stack_overflow`, `lobsters`, `newsletter`, `docs_site`, `package_registry`,
  `podcast_platform`, `chatgpt`, `claude`

Set each property:

```bash
rei link set-property -l LINK_ID author-type VALUE
rei link set-property -l LINK_ID content-type VALUE
rei link set-property -l LINK_ID media VALUE
rei link set-property -l LINK_ID platform VALUE
```

If any `set-property` call fails because the value is not in the enum, re-read the allowed
values with `rei custom-property show KEY` and retry with a valid one. If no valid value
fits, skip that property.

### Phase 12: Tag the Link

Goal: assign 3–7 topical `tags` that categorize what the content is *about* (subject
matter, technologies, concepts, domains). Tags are orthogonal to the enum classifiers from
Phase 11.

**Step 12a — Harvest existing tag vocabulary.**

```bash
rei custom-property entities tags --json
```

The output is an object of the shape:

```json
{
  "count": <int>,
  "entities": [ { ..., "properties": { "tags": "tag-one,tag-two,..." } }, ... ],
  "property": "tags",
  "value_type": "tag-set"
}
```

If `count` is `0` (or `entities` is empty), the vocabulary is empty — proceed with
freshly-minted tags. Otherwise extract every tag value across all entities into a single
deduplicated list (the `EXISTING_TAGS` vocabulary). One-liner:

```bash
rei custom-property entities tags --json \
  | jq -r '.entities[].properties.tags' \
  | tr ',' '\n' \
  | sed 's/^ *//; s/ *$//' \
  | sort -u
```

Note: tags are stored as a single comma-separated string per entity (not a JSON array),
which is why the `tr ','` split is needed.

**Step 12b — Propose tags.** Based on the markdown content and the existing vocabulary,
choose 3–7 tags that best describe the subject matter. Apply these rules in order:

1. **Prefer reuse.** If an existing tag covers the concept, use it verbatim — do not create
   a variant. Watch for near-duplicates:
   - singular vs plural (`graph` vs `graphs`) — reuse whichever exists
   - abbreviation vs expansion (`llm` vs `large-language-models`) — reuse whichever exists
   - synonyms (`event-sourcing` vs `event-sourced`) — reuse whichever exists
   - case / punctuation (`EventSourcing` vs `event-sourcing`) — reuse whichever exists
2. **Normalize new tags.** When no existing tag fits, mint a new one in lowercase,
   hyphen-separated, singular-where-natural (e.g., `event-sourcing`, `haskell`,
   `distributed-systems`, `claude-code`). No spaces, no underscores, no punctuation beyond
   hyphens.
3. **Stay topical.** Tags should describe the subject matter, not the format. Don't
   re-encode `content-type` / `media` / `platform` as tags (no `article`, `video`,
   `github`).
4. **Be specific but not narrow.** Prefer `event-sourcing` over both `programming` (too
   broad) and `event-sourcing-in-haskell-with-postgres` (too narrow — that's a sentence).
5. **Cap at ~7.** More than 7 tags dilutes the signal; aim for 3–5 when in doubt.

**Step 12c — Confirm with the user.** Show the proposed tags, clearly marking which are
reused from the existing vocabulary and which would be newly minted, then ask:

```
Question: "I propose these tags for the link. Apply them?"
Header: "Tags"
Options:
- Apply as proposed (Recommended)
- Let me edit the list
- Skip tagging
```

If the user picks "Let me edit the list", accept their revised comma-separated list and
re-validate against the normalization rules (lowercase, hyphenated, no duplicates). Also
re-check the edited list against `EXISTING_TAGS` for near-duplicates and surface any you
notice before applying.

**Step 12d — Apply the tags.** Tags are a `tag-set`, so pass them as a single comma-separated
value in one call:

```bash
rei link set-property -l LINK_ID tags "tag-one,tag-two,tag-three"
```

Setting replaces the full set. If the link already had tags (reused-link case from Phase 3),
merge the new tags with the existing ones before calling `set-property`, deduplicating.

### Phase 13: Summary

Display a final summary:

```
## Markdown Ingested

- **File**: <path>
- **URL**: <url>
- **Link**: <link_id>  (reused existing / newly created)
- **Intention**: <intention_id>
- **Archive Note**: <archive_note_id>
- **Summary Note**: <summary_note_id>

### Edges
- archive_note -[archives]-> link
- summary_note -[summarizes]-> link
- summary_note -[summarizes]-> archive_note

### Classification
- author-type: <value or "skipped">
- content-type: <value or "skipped">
- media: <value or "skipped">
- platform: <value or "skipped">
- tags: <tag-one, tag-two, ...>  (N reused, M new)

### Next Steps
- Review the summary note: `rei note show <summary_note_id>`
- Review the archive note: `rei note show <archive_note_id>`
- Inspect the link: `rei link show <link_id>`
```

## Important Notes

- Always use `--actor claude-code` when creating entities (link, note).
- **The URL comes from frontmatter, not from the user.** If frontmatter is missing or has no
  recognizable URL key, ask the user explicitly — don't guess.
- **Archive note is verbatim.** Preserve the full markdown body in the archive note,
  including frontmatter, so the captured page is fully recoverable from Rei alone. The
  small wrapper header (Source/Author/Published) is for human convenience above a `---`
  separator; the original body follows unchanged.
- **Summary note is separate.** Don't conflate the archive and the summary — they serve
  different purposes (long-term reference vs quick recall).
- **Three edges, two predicates.** `archives` is new to this skill; `summarizes` is shared
  with `rei-ingest-url`. Define `archives` once with `--source-types note --target-types link`.
- **Check before creating**: `rei link list --all --domain DOMAIN --json` is the cheapest
  way to detect an existing link; fall back to `--query URL` if needed.
- **Don't re-ingest**: if a link already exists, do not create a second link. If archive
  and summary notes already exist for it (check via edges), do not duplicate them unless
  the user explicitly asks for a fresh archive (e.g., the page changed since last archive).
- **Property values must come from the enum** — never invent new values. If unsure, run
  `rei custom-property show KEY` to list them.
- **Skip classification rather than guess**: a skipped property is better than a wrong one.
- **Tags are a `tag-set`**: pass all desired tags as one comma-separated value in a single
  `rei link set-property … tags "a,b,c"` call — `set-property` replaces the full set, so
  splitting it across calls would lose earlier tags. When re-tagging an existing link,
  read current tags first (`rei link show LINK_ID --json`) and merge before setting.
- **Reuse tags aggressively**: always run `rei custom-property entities tags --json` first
  and prefer existing tag strings verbatim over near-duplicates.
- The link is created *before* the archive/summary notes; if note creation or summarization
  fails after the link is in place, leave the link, report the error, and let the user
  retry the remaining phases manually.

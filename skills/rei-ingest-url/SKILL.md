---
name: rei-ingest-url
description: Ingest a single URL into Rei — create (or reuse) a link, summarize the content as a note, connect them with a "summarizes" edge, and classify the link with author-type, content-type, media, platform, and tags properties.
allowed-tools: AskUserQuestion, Bash, Read, WebFetch
---

# Rei Ingest URL

This skill ingests a single URL into Rei. It first checks whether a link already exists for
that URL; if not, it asks for an intention ID and creates the link. It then fetches the
URL's content, summarizes it into a new note attached to the same intention, and connects
the note to the link with a `summarizes` edge. Finally, it classifies the link with the
`author-type`, `content-type`, `media`, and `platform` enum properties and applies
free-form topical `tags` — reusing existing tags from the workspace wherever possible to
avoid near-duplicate vocabulary.

## When to Use

Activate when the user says things like:
- "Ingest this URL into Rei"
- "Summarize this link and add it to Rei"
- "Add <url> as a link with a summary"
- "/rei-ingest-url <url>"

## Key Concepts

- **Link** — a Rei entity representing a URL, anchored to an intention (or action/outcome/reflection).
- **Note** — a Rei entity holding markdown content, anchored to an intention.
- **Edge** — a typed relationship between two entities, identified by a predicate.
- **`summarizes` predicate** — used here to point from the summary note to the link it summarizes.
- **Custom properties on links** — `author-type`, `content-type`, `media`, `platform` are
  enum properties defined globally for links. Only values from the enum may be used.
- **`tags` property** — a `tag-set` property scoped to links (and other entities). Free-form,
  no predeclared values; assignments are a comma-separated list passed in a single
  `set-property` call. Setting replaces the full set, so include every desired tag in one
  call. Tags should be short, lowercase, hyphen-separated (e.g., `event-sourcing`, not
  `Event Sourcing` or `events`).

## Workflow Overview

1. **Get the URL** — from args or from the user
2. **Check for existing link** — search by URL; if found, reuse it; otherwise proceed
3. **Get intention ID** — ask the user (only when creating a new link)
4. **Create the link** — `rei link add`
5. **Fetch and summarize** — pull URL content, summarize it
6. **Create the summary note** — attached to the same intention
7. **Create the edge** — `note -[summarizes]-> link`
8. **Classify the link** — set `author-type`, `content-type`, `media`, `platform`
9. **Tag the link** — harvest existing tags, pick + propose new ones, set `tags`
10. **Summary** — show everything that was created

## Instructions for Claude

### Phase 1: Get the URL

The URL may be supplied as an argument. If not, ask:

```
Question: "What URL should I ingest into Rei?"
Header: "URL"
Options:
- Let me paste the URL
```

Validate that it is a well-formed absolute URL (`http://` or `https://`). If not, stop and
tell the user.

### Phase 2: Check Whether a Link Already Exists

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
  inform the user, and **skip Phase 3 and Phase 4**. Also skip property-setting (Phase 8)
  for enum keys that are already set on the existing link — check with
  `rei link show LINK_ID --json` first.
- **If no match**: proceed to Phase 3.

### Phase 3: Get Intention ID (only for new links)

Use AskUserQuestion:

```
Question: "Which intention should the link and summary note be attached to? Provide the intention ID (e.g., intention_01h455vb4pex)."
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

### Phase 4: Create the Link

```bash
rei link add "URL" -i INTENTION_ID --actor claude-code
```

If the page's `<title>` is known (e.g., from a prior fetch), pass it with `-t "TITLE"`.
Otherwise, set the title after Phase 5 once you have it:

```bash
rei link title LINK_ID "TITLE"
```

Capture the new link ID from the command output.

### Phase 5: Fetch and Summarize the URL

Fetch the URL's content using the WebFetch tool with a prompt like:

> "Extract the page title, author (if any), and the main content. Return a concise but
> thorough summary of the article/page in markdown, preserving the author's key points,
> arguments, and any notable quotes or data. Also report: what type of author this is
> (individual/company/organization/etc.), what type of content this is
> (article/blog_post/documentation/etc.), the primary media format, and the platform/site."

From the response, extract:
- `TITLE` — the page's title
- `SUMMARY_MD` — the markdown summary body
- Classification hints for `author-type`, `content-type`, `media`, `platform`

If the fetch fails, stop and report the error. The link has already been created at this
point; leave it in place and let the user retry.

### Phase 6: Create the Summary Note

Write the summary to a temp file with a title header, then pipe it into `rei note new`:

```bash
(echo '# Summary of "TITLE"'; echo ''; echo 'Source: URL'; echo ''; echo "$SUMMARY_MD") \
  | rei note new -i INTENTION_ID --stdin --actor claude-code
```

Capture the note ID from the output.

### Phase 7: Create the Edge (note summarizes link)

```bash
rei edge add --from NOTE_ID --to LINK_ID --predicate summarizes
```

If this fails because the `summarizes` predicate doesn't exist, define it and retry:

```bash
rei predicate show summarizes >/dev/null 2>&1 || \
  rei predicate define summarizes --label "Summarizes" --source-types note --target-types link,note
```

### Phase 8: Classify the Link

For each of the four enum properties, pick the best-fitting value **from the allowed list**
based on the fetched content. Only set a property if you are reasonably confident — skip it
otherwise rather than guessing.

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

### Phase 9: Tag the Link

Goal: assign 3–7 topical `tags` that categorize what the content is *about* (subject
matter, technologies, concepts, domains). Tags are orthogonal to the enum classifiers from
Phase 8 — `content-type: article` says *what shape* it is; `tags: event-sourcing,haskell`
says *what it's about*.

**Step 9a — Harvest existing tag vocabulary.** Before proposing tags, pull the current set
of tags already in use across the workspace so you can reuse them instead of minting
near-duplicates:

```bash
rei custom-property entities tags --json
```

From the JSON, collect every tag value present on any entity into a single deduplicated
list (the `EXISTING_TAGS` vocabulary). If the command returns zero entities, the vocabulary
is empty — proceed with freshly-minted tags.

**Step 9b — Propose tags.** Based on the fetched content and the existing vocabulary,
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

**Step 9c — Confirm with the user.** Show the proposed tags, clearly marking which are
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

**Step 9d — Apply the tags.** Tags are a `tag-set`, so pass them as a single comma-separated
value in one call:

```bash
rei link set-property -l LINK_ID tags "tag-one,tag-two,tag-three"
```

Setting replaces the full set. If the link already had tags (reused-link case from Phase 2),
merge the new tags with the existing ones before calling `set-property`, deduplicating.

### Phase 10: Summary

Display a final summary:

```
## URL Ingested

- **URL**: <url>
- **Link**: <link_id>  (reused existing / newly created)
- **Intention**: <intention_id>
- **Note**: <note_id>
- **Edge**: <edge_id>  (note -[summarizes]-> link)

### Classification
- author-type: <value or "skipped">
- content-type: <value or "skipped">
- media: <value or "skipped">
- platform: <value or "skipped">
- tags: <tag-one, tag-two, ...>  (N reused, M new)

### Next Steps
- Review the summary note: `rei note show <note_id>`
- Inspect the link: `rei link show <link_id>`
```

## Important Notes

- Always use `--actor claude-code` when creating entities (link, note).
- **Check before creating**: `rei link list --all --domain DOMAIN --json` is the cheapest
  way to detect an existing link; fall back to `--query URL` if needed.
- **Don't re-ingest**: if a link already exists, do not create a second link, and do not
  create a duplicate summary note unless the user explicitly asks for one.
- **Property values must come from the enum** — never invent new values. If unsure, run
  `rei custom-property show KEY` to list them.
- **Skip classification rather than guess**: a skipped property is better than a wrong one.
- **Tags are a `tag-set`**: pass all desired tags as one comma-separated value in a single
  `rei link set-property … tags "a,b,c"` call — `set-property` replaces the full set, so
  splitting it across calls would lose earlier tags. When re-tagging an existing link,
  read current tags first (`rei link show LINK_ID --json`) and merge before setting.
- **Reuse tags aggressively**: always run `rei custom-property entities tags --json` first
  and prefer existing tag strings verbatim over near-duplicates. A fragmented vocabulary
  (`llm` and `llms` and `large-language-models` all coexisting) is the main failure mode
  to avoid.
- If the `summarizes` predicate is missing, define it once with
  `--source-types note --target-types link,note`.
- The link is created *before* the fetch; if fetching/summarizing fails, the link remains —
  report the error and let the user retry Phases 5–8 manually.

---
name: rei-summarize-links
description: Lightweight bulk pass over a markdown document's links — summarize each URL with the `summarize` CLI, store the summary as a note on one intention, reuse or create the link, and connect them with a `summarizes` edge. No classification, tags, topics, or collection
allowed-tools: AskUserQuestion, Bash, Read, Grep
---

# Rei Summarize Links

Takes a markdown document, and for every external link in it: summarizes the URL with the
`summarize` CLI, stores the summary as a note anchored to one intention, reuses or creates the
Rei link, and connects them with `note -[summarizes]-> link`.

This is the **light** pass. Use a sibling instead when the user wants more:

- `rei-ingest-url` — one URL, plus link classification (`author-type`, `content-type`, `media`,
  `platform`) and tags.
- `rei-ingest-url-collection` — the full `rei-ingest-url` workflow for every link in a file,
  gathered into a collection.
- `rei-bookmark-url` — file a link under topics in the ontology rather than summarizing it.

## When to Use

- "Summarize the links in this document"
- "Create summary notes for every link in <file>"
- "/rei-summarize-links <file>"

## Workflow

### 1. Extract links

Read the file and collect `[title](url)` pairs with `http(s)` URLs. Skip image links
(`![alt](url)`), anchors (`#heading`), and relative paths. Deduplicate by URL (keep the first
title). If none remain, say so and stop. Show the numbered list.

### 2. Choose the intention (and optional category)

Take an intention ID from the arguments or the user; if they only give a name, resolve it:

```bash
rei intention list -s "KEYWORD" --json
```

Confirm the intention and the link list in one question before writing anything. Optionally
accept a note category; if one is given, read its guidance first
(`rei category print-note-guidance SLUG`) — it may dictate note structure, and the category's
property bindings are applied automatically when the note is created.

### 3. Preflight

The `summarizes` predicate is workspace-defined (not created by `rei ontology seed-system`):

```bash
rei predicate show summarizes --json
```

If it is missing, define it:

```bash
rei --actor claude-code predicate define summarizes --label "Summarizes" \
  --description "Source (note or link) is a summary of the target note, link, or topic" \
  --source-types link,note --target-types link,note,topic
```

If it exists but doesn't allow `note → link`, stop and report it — don't change the predicate.

Create a working directory with `mktemp -d` for summaries. Warn that `summarize` can take a
while per URL when there are many links.

### 4. Process each link (sequentially)

**a. Find an existing link.** Rei deduplicates links by canonical URL, so match on both forms:

```bash
rei link list --all --domain DOMAIN --json \
  | jq -r --arg u "URL" '.[] | select(.original_url == $u or .canonical_url == $u) | .id'
```

If a link exists, check whether it is already summarized:

```bash
rei edge show LINK_ID -p summarizes --json \
  | jq -r --arg l LINK_ID '.[] | select(.targetId == $l and .sourceType == "note") | .sourceId'
```

If a summary note already exists, skip this URL (report it) unless the user asked to
re-summarize.

**b. Summarize.**

```bash
summarize --cli claude --length xxl "URL" > "$WORKDIR/SLUG.md"
```

Use the CLI provider matching the session (`--cli codex` under Codex). If it fails or produces
an empty file, record the failure and move to the next link — create nothing for it.

**c. Create the note.**

```bash
(echo "# Summary of \"TITLE\""; echo; echo "Source: URL"; echo; cat "$WORKDIR/SLUG.md") \
  | rei --actor claude-code note new -i INTENTION_ID [-c CATEGORY_SLUG] --stdin
```

Capture `NOTE_ID` (`grep -o 'note_[0-9a-z]*' | head -1`).

**d. Reuse or create the link.** Always pass `-i` — without it `link add` opens an fzf picker.
Adding a URL that already exists reuses that link and adds an attachment to the intention:

```bash
rei --actor claude-code link add "URL" -i INTENTION_ID -t "TITLE"
```

Capture `LINK_ID` (`grep -o 'link_[0-9a-z]*' | head -1`), or keep the one found in step a.

**e. Connect them.**

```bash
rei --actor claude-code edge add --from NOTE_ID --to LINK_ID --predicate summarizes
```

**f. Verify.** Many rei commands print an error yet exit `0`, so read back instead of trusting
the exit status:

```bash
rei note show NOTE_ID --json
rei edge show NOTE_ID -p summarizes --json \
  | jq -r --arg l LINK_ID '.[] | select(.targetId == $l and .status == "active") | .edgeId'
```

A missing note or edge counts as a failure for that link. Print a one-line progress status
(`[N/TOTAL] TITLE — note / link / edge IDs`).

### 5. Finish

Remove the working directory. Report:

```
## Summarize Links Complete

- **Source document**: PATH
- **Intention**: INTENTION_ID — title
- **Summarized**: N / TOTAL   **Skipped (already summarized)**: K   **Failed**: M

| # | Title | Note | Link (new/reused) | Edge |
|---|-------|------|-------------------|------|

### Failed / Skipped
- TITLE (URL) — reason
```

Suggest `rei-ingest-url-collection` if the user now wants these classified and grouped.

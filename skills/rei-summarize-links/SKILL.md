---
name: rei-summarize-links
description: Extract markdown links from a document, summarize each URL, create Rei notes with summaries, create links, and connect them with "summarizes" edges.
allowed-tools: AskUserQuestion, Bash, Read, Grep
---

# Rei Summarize Links

This skill takes a markdown document containing markdown links, summarizes each linked URL using the `summarize` CLI, and creates Rei notes, links, and edges for each.

## When to Use

Activate when the user says things like:
- "Summarize links from this document"
- "Process links in my markdown file"
- "Create summaries for all links in ..."
- "Summarize the links in ..."

## Workflow Overview

1. **Read the markdown file** — extract all `[title](url)` links
2. **Ask for intention** — get the intentionId to anchor notes and links
3. **For each link** — summarize, create note, create link, create edge

## Instructions for Claude

### Phase 1: Read and Parse the Markdown File

The user provides a path to a markdown file (as an argument or in conversation).

Read the file, then extract all markdown links matching the pattern `[title](url)`. Ignore:
- Anchor-only links (`[text](#heading)`)
- Relative file links (`[text](./file.md)`)
- Image links (`![alt](url)`)

Collect a list of `(title, url)` pairs.

If no links are found, inform the user and stop.

Display the discovered links to the user:

```
Found N links:
1. [title1](url1)
2. [title2](url2)
...
```

### Phase 2: Get Intention ID

Use AskUserQuestion to get the intentionId:

```
Question: "Which intention should these summary notes and links be attached to? Provide the intention ID (e.g., intention_01h455vb4pex)."
Header: "Intention"
Options:
- Let me type the intention ID
- Let me browse intentions first
```

If the user wants to browse first, run:
```bash
rei intention list
```
Then ask again for the ID.

### Phase 3: Process Each Link

For each `(title, url)` pair, perform these steps sequentially:

#### Step 3a: Summarize the URL

Generate a slug from the title (lowercase, replace spaces/special chars with hyphens, truncate to 60 chars). Run:

```bash
summarize --cli claude --length xxl "URL" > "/tmp/rei-summary-SLUG.md"
```

If the summarize command fails for a link, log the error and continue to the next link.

#### Step 3b: Create the Rei Note

Read the generated summary file. Prepend an H1 title line and a blank line, then pipe the full content to `rei note new`:

```bash
(echo '# Summary of "TITLE"'; echo ''; cat /tmp/rei-summary-SLUG.md) | rei note new -i INTENTION_ID --stdin --actor claude-code
```

Capture the note ID from the output.

#### Step 3c: Create the Rei Link

```bash
rei link add "URL" -i INTENTION_ID -t "TITLE" --actor claude-code
```

Capture the link ID from the output.

#### Step 3d: Create the Edge

```bash
rei edge add --from NOTE_ID --to LINK_ID --predicate summarizes
```

#### Step 3e: Log Progress

After each link is processed, print a status line:

```
[N/TOTAL] Done: "TITLE"
  Note: NOTE_ID
  Link: LINK_ID
  Edge: EDGE_ID
```

### Phase 4: Summary

After all links are processed, display a final summary:

```
## Summarize Links Complete

- **Source document**: PATH
- **Intention**: INTENTION_ID
- **Links processed**: N / TOTAL
- **Failed**: M (if any)

| # | Title | Note | Link | Edge |
|---|-------|------|------|------|
| 1 | title | note_id | link_id | edge_id |
| ... | ... | ... | ... | ... |
```

If any links failed, list them separately with the error.

## Important Notes

- Always use `--actor claude-code` when creating notes and links
- The `summarize` command can take a while per URL — inform the user that this may take time if there are many links
- If the `summarizes` predicate doesn't exist yet, create it first:
  ```bash
  rei predicate define summarizes --label "Summarizes" --source-types note --target-types link,note
  ```
  Check with `rei predicate show summarizes` before attempting to define it — skip if it already exists.
- Clean up temp files in `/tmp/rei-summary-*.md` after all links are processed
- If a link URL appears multiple times in the document, process it only once (deduplicate by URL)
- Capture entity IDs from command output — rei commands print the created entity's ID to stdout

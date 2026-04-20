---
name: rei-ingest-url-collection
description: Read a markdown file of links, run the full rei-ingest-url workflow for each URL (link + summary note + summarizes edge + classification + tags), and gather every resulting summary note into a single Rei collection.
allowed-tools: AskUserQuestion, Bash, Read, WebFetch, Grep
---

# Rei Ingest URL Collection

This skill takes a markdown document containing links, applies the full `rei-ingest-url`
workflow per URL (creating or reusing a link, summarizing, classifying, tagging), and
gathers every summary note into a single Rei collection. Use it to turn a reading list or
bookmark dump into a structured, classified, searchable collection in one pass.

## When to Use

Activate when the user says things like:
- "Ingest every link in this markdown file as a collection"
- "Run rei-ingest-url on all the links in <file> and group them"
- "Create a collection from the links in <file>"
- "/rei-ingest-url-collection <file>"

Prefer `rei-ingest-url` (singular) for one-off URLs. Prefer `rei-summarize-links` if you
want a lighter pass that skips classification and collections.

## Relationship to Other Skills

This skill orchestrates `rei-ingest-url` in bulk. **Before starting Phase 5, read
`skills/rei-ingest-url/SKILL.md`** — the per-link procedure (its Phases 1–9) is applied
once per URL. Don't reconstruct it from memory; if that skill evolves, this one should
inherit the improvements.

This skill adds three things on top:
- **Link extraction** from a markdown file (Phase 2)
- **Collection bookkeeping** — create or reuse a manual collection, add each note as a member
- **Iteration and progress reporting** — per-link status, final summary table

## Key Concepts

- **Rei collection** — a group of knowledge artifacts (notes, links, docs). Manual
  collections have curated membership. Members are added via `rei collection add`. Only
  manual collections accept explicit adds — virtual collections are query-driven.
- **Per-link workflow** — see `rei-ingest-url/SKILL.md`. Each URL produces a link, a summary
  note, a `summarizes` edge, and (where confident) values for `author-type`, `content-type`,
  `media`, `platform`, plus a tag set.
- **Shared intention** — one intention ID anchors every link and note created by a run.
  The user picks it once at the start.

## Workflow Overview

1. **Get the markdown file** — from args or the user
2. **Extract links** — parse `[title](url)` pairs, dedupe by URL
3. **Get the intention ID** — once, applied to every link and note
4. **Get or create the collection** — name + optional description
5. **Ensure the `summarizes` predicate exists** — once up-front, not per link
6. **Process each link** — apply the `rei-ingest-url` workflow, add the summary note to the collection
7. **Summary** — results table and failure list

## Instructions for Claude

### Phase 1: Get the Markdown File

The path may be supplied as an argument. If not, ask:

```
Question: "Which markdown file contains the links to ingest?"
Header: "File"
Options:
- Let me paste the file path
```

Read the file with the Read tool. If it does not exist, stop and tell the user.

### Phase 2: Extract Links

From the file content, extract all markdown links matching `[title](url)`. Ignore:
- Anchor-only links (`[text](#heading)`)
- Relative file links (`[text](./file.md)`, `[text](../x)`)
- Image links (`![alt](url)`)

**Dedupe by URL** — if the same URL appears multiple times, process it only once. Keep the
first title encountered for display purposes.

Display the discovered links:

```
Found N unique links in PATH:
1. [title1](url1)
2. [title2](url2)
...
```

If no links are found, stop and tell the user.

### Phase 3: Get the Intention ID

One intention anchors every link and note this run creates.

```
Question: "Which intention should these links and summary notes be attached to? (e.g., intention_01h455vb4pex)"
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

### Phase 4: Get or Create the Collection

Create the collection **now**, not at the end. That way partial progress is preserved if
the run is interrupted.

```
Question: "Add the summary notes to an existing collection or create a new one?"
Header: "Collection"
Options:
- Create a new collection (Recommended)
- Add to an existing collection
```

**If new**: ask for a name and optional description.

```
Question: "What should the collection be named?"
Header: "Name"
Options:
- Let me type the name
```

```
Question: "Optional description for the collection?"
Header: "Description"
Options:
- Skip (no description) (Recommended)
- Let me type a description
```

Then create it:

```bash
rei collection create "NAME"                    # no description
rei collection create "NAME" -d "DESCRIPTION"   # with description
```

**If existing**: list collections and confirm the one the user picks is **manual** (not
virtual — `rei collection show` indicates the type). If it is virtual, stop and explain:
virtual collections cannot have members added directly.

```bash
rei collection list
rei collection show "NAME"
```

Store `COLLECTION_NAME` for use in Phase 6.

### Phase 5: Ensure the `summarizes` Predicate Exists

Check once, up-front — don't let the first per-link edge creation discover this:

```bash
rei predicate show summarizes >/dev/null 2>&1 || \
  rei predicate define summarizes --label "Summarizes" --source-types note --target-types link,note
```

### Phase 6: Process Each Link

**Read `skills/rei-ingest-url/SKILL.md` now** if you haven't already. The per-link
procedure below is a thin wrapper around its Phases 1–9.

For each `(title, url)` pair, in order:

**6a. Apply the `rei-ingest-url` workflow** to this URL, with these adjustments to avoid
redundant prompts:

| `rei-ingest-url` phase | In bulk mode |
|---|---|
| Phase 1 (ask for URL) | **Skip** — already known |
| Phase 2 (check for existing link) | Keep — reuse any existing link at this URL |
| Phase 3 (ask for intention) | **Skip** — use the shared `INTENTION_ID` from Phase 3 above |
| Phase 4 (create link) | Keep |
| Phase 5 (fetch + summarize) | Keep |
| Phase 6 (create summary note) | Keep — attach to `INTENTION_ID` |
| Phase 7 (create edge) | Keep |
| Phase 8 (classify link) | Keep |
| Phase 9 (tag link) | Keep, but **auto-apply proposed tags** — do not call the per-link AskUserQuestion confirmation (step 9c). Still harvest existing tags (9a) and propose thoughtfully (9b); just skip the interactive confirm and apply directly in 9d. |

Capture `LINK_ID`, `NOTE_ID`, and the reused-vs-new flag from the per-link output.

**6b. If the per-link workflow fails** (unreachable URL, fetch error, etc.), log the
failure with the URL and error message and **continue** to the next link. Do not abort the
batch. A failed link that had already created a bare `link` entity (Phase 4 succeeded but
Phase 5 failed) stays in place — report it in the failure list so the user can retry
manually.

**6c. Add the summary note to the collection:**

```bash
rei collection add "COLLECTION_NAME" --note NOTE_ID
```

(Only the note. Links are reachable from each note via the `summarizes` edge, so adding
them to the collection too would duplicate signal.)

**6d. Log progress:**

```
[N/TOTAL] Ingested: "TITLE"
  URL:  <url>
  Link: <link_id> (reused / new)
  Note: <note_id>
  Tags: tag-one, tag-two, tag-three  (R reused, M new)
  Added to collection: COLLECTION_NAME
```

### Phase 7: Summary

Display a final summary:

```
## URL Collection Ingested

- **Source document**: PATH
- **Intention**: INTENTION_ID
- **Collection**: COLLECTION_NAME
- **Links processed**: M / TOTAL
- **Failed**: F

| # | Title | Link | Note | Reused? | Tags |
|---|-------|------|------|---------|------|
| 1 | ... | link_id | note_id | yes/no | a, b, c |
| ... |

### Failed
- [url]: <error>
  (If the link was created before the fetch failed, its ID is <link_id> — retry with `/rei-ingest-url <url>`.)

### Next Steps
- Review the collection: `rei collection show COLLECTION_NAME`
- Export as EPUB: `/rei-collection-epub COLLECTION_NAME`
- Retry any failed links individually with `/rei-ingest-url <url>`
```

## Important Notes

- Always use `--actor claude-code` when creating links and notes.
  `rei collection create` and `rei collection add` do **not** accept `--actor` — don't
  pass it.
- **Create the collection before iterating.** Partial progress survives interruption, and
  each note is added as it is created rather than gathered into an in-memory list.
- **Dedupe by URL, not title.** The same URL with different link text should still be
  processed only once.
- **Fetch failures are per-link, not fatal.** Log and continue; surface failures in the
  final summary so the user can retry individually.
- **Suppress only the per-link prompts that would be redundant** — URL, intention, and tag
  confirmation. Keep the rest of the `rei-ingest-url` behavior intact (existing-link
  reuse, enum validation, tag harvesting, `--actor` on entity creation).
- **Only notes become collection members** in this skill. If the user asks for links to be
  collection members too, add `rei collection add "COLLECTION_NAME" --link LINK_ID` after
  step 6c.
- The per-link workflow may take a while (one WebFetch + one Claude summarization per URL).
  Warn the user up-front if there are many links.

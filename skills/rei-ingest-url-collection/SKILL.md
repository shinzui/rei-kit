---
name: rei-ingest-url-collection
description: Read a markdown file of links, run the full rei-ingest-url workflow for each URL (link, summary note, summarizes edge, classification, facet tags, `about` associations to existing topics, read-back verification), and gather every summary note into a single manual Rei collection, reporting uncovered subjects across the batch.
allowed-tools: AskUserQuestion, Bash, Read, WebFetch, Grep
---

# Rei Ingest URL Collection

Turns a markdown reading list into a classified Rei collection: runs `rei-ingest-url` once per
URL against a shared intention, and adds each resulting summary note to one manual collection.

Use `/rei-ingest-url` for a single URL, `/rei-summarize-links` for a lighter pass without
classification, tags, topics, or a collection, and `/rei-bookmark-url` to grow the ontology for
subjects this run reports as uncovered.

## Relationship to rei-ingest-url

**Read `skills/rei-ingest-url/SKILL.md` before processing links** and apply its Phases 1–9 per
URL — don't reconstruct them from memory. This skill adds link extraction, collection
bookkeeping, bulk-mode adjustments, and batch reporting.

## Workflow

Use the global actor form `rei --actor claude-code …` for every write and always pass names/IDs
explicitly (omitted arguments open fzf pickers). Many writes (links, edges, collection adds)
can print an error and still exit `0`, so confirm by reading back.

### 1. Extract links

Take the file path from the argument or the user and read it. Collect `[title](url)` pairs with
absolute `http(s)` URLs; ignore anchors, relative paths, and image links. Dedupe by URL (keep
the first title). Show the numbered list; stop if empty. Warn that each URL costs a fetch and a
summary when the list is long.

### 2. Shared intention

One intention anchors every new link and note. Take an ID from the user, or search:

```bash
rei intention list --all -s "KEYWORD" --json | jq -r '.[] | "\(.id)\t\(.title)"'
```

### 3. Collection (before iterating, so partial progress survives)

New (default) — ask for a name and optional description:

```bash
rei --actor claude-code collection create "NAME" [-d "DESCRIPTION"]
```

Existing — it must be a manual collection (virtual collections are query-driven and reject
adds):

```bash
rei collection list --json \
  | jq -r '.collections[].collection | "\(.collectionId)\t\(.kind.type)\t\(.name)"'
```

Either way, resolve and keep `COLLECTION_ID`, and confirm `kind.type == "manual_collection"`.

### 4. Preflight the `summarizes` predicate (once)

`predicate show` exits `0` even when the key is missing, so check the list:

```bash
rei predicate list --json | jq -e '.[] | select(.predicateKey == "summarizes")' >/dev/null || \
  rei --actor claude-code predicate define summarizes --label "Summarizes" \
    --source-types note --target-types link,note,topic
```

Never redefine or loosen an existing predicate.

### 5. Process each link (sequentially)

**5a. Run `rei-ingest-url`** with these bulk-mode adjustments:

| rei-ingest-url | Bulk mode |
|---|---|
| Phase 1 (URL) | Skip — known |
| Phase 2 (existing link) | Keep. If the link already has a `summarizes` note, don't create another; reuse that note for the collection |
| Phase 3 (intention) | Skip — shared intention (a reused link keeps its own attachments) |
| Phases 4–8 | Keep |
| 9a (harvest tags + topics) | Harvest once before the loop; add tags created during the run to the in-memory vocabulary so later links reuse them |
| 9b (propose) | Keep |
| 9c (confirm) | Skip — apply proposals directly |
| 9d (apply) | Keep |
| 9e (verify) | Keep — record any write still missing after one retry |

Capture `LINK_ID`, `NOTE_ID`, reused/new, tags, `about` topics, uncovered subjects, and verify
failures.

**5b. Failures are per link.** Log the URL and error and continue. A link created before a
failed fetch stays in place — list it for retry.

**5c. Add the summary note** (notes only; links are reachable via `summarizes`):

```bash
rei --actor claude-code collection add COLLECTION_ID --note NOTE_ID
```

Verify membership — skip the add if the note is already a member (reused note):

```bash
rei collection show COLLECTION_ID --json \
  | jq -e --arg n NOTE_ID '.members[] | select(.memberRef.data == $n)' >/dev/null
```

Add links as members too (`--link LINK_ID`) only if the user asks.

**5d. Progress line:** `[N/TOTAL] TITLE — link <id> (reused/new), note <id>, tags …, about …`.

### 6. Summary

```
## URL Collection Ingested

- **Source document**: PATH
- **Intention**: INTENTION_ID
- **Collection**: NAME (COLLECTION_ID)
- **Ingested**: M / TOTAL   **Failed**: F

| # | Title | Link (new/reused) | Note | Tags | About |
|---|-------|-------------------|------|------|-------|

### Uncovered subjects (no existing topic)
- <subject> — seen in N links   → file with `/rei-bookmark-url`

### Failed / Unverified
- <url>: <error or missing write>  (link <link_id> exists — retry with `/rei-ingest-url <url>`)

### Next Steps
- `rei collection show COLLECTION_ID`
- `/rei-collection-epub` to export
```

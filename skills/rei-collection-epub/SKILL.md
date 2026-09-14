---
name: rei-collection-epub
description: Export a Rei collection to EPUB with pandoc — resolve its note, link, and doc members into one chapter per work (preferring a link's full archive note over its summary), infer chapter order from title sequence markers, ordering edges, content cross-references, and addedAt timestamps, confirm the order, and build the book with a table of contents.
allowed-tools: AskUserQuestion, Bash, Read, Write
---

# Rei Collection to EPUB

Turns a Rei collection into an EPUB: fetch its members, resolve each into chapter content,
infer a reading order, confirm it with the user, and run pandoc.

## When to Use

- "Export this collection as an epub"
- "Turn my collection into an ebook"
- "/rei-collection-epub <collection>"

## Key Concepts

- **Collections** are **manual** (curated members) or **virtual** (a saved query that matches
  notes at read time). `rei help collections` has the details.
- **Member shape** (`rei collection show ID --json` → `.members[]`): `memberId`, `addedAt`,
  `displayName`, `memberRef.type` (`member_note` | `member_link` | `member_doc`),
  `memberRef.data` (entity ID), `relativePath` (absolute file path for notes/docs; `null` for
  links). **Link members have `displayName: null`** — get the title and URL from
  `rei link show LINK_ID --json` (`.title`, `.url`).
- **Ingested collections pair links with notes.** Collections built by `rei-ingest-url-collection`
  / `rei-ingest-markdown` often contain a link plus its `summarizes` note, and the link may also
  have an `archives` note holding the full page. Treat the link and its notes as **one work**,
  not two or three chapters.

## Workflow

### 1. Select the collection

Use the name or ID the user gave; otherwise list and ask:

```bash
rei collection list --json \
  | jq -r '.collections[] | "\(.collection.collectionId)\t\(.memberCount)\t\(.collection.name)"'
```

Always pass the collection ID or exact name — `rei collection show` with no argument opens an
fzf picker. Check `pandoc --version` now; if missing, stop and say so.

### 2. Fetch and resolve members

```bash
rei collection show COLLECTION_ID --json
```

Keep `.collection.name` and `.collection.description`. For a virtual collection
(`.collection.kind.type`), if `.members` is empty run `rei collection exec COLLECTION_ID` (text
only, no `--json`) and take the `note_...` IDs from its output. If there are no members, stop.

Resolve members into **works**:

1. For each link member, read its incoming note edges:

   ```bash
   rei link show LINK_ID --json | jq -r '.title, .url'
   rei edge show LINK_ID --json \
     | jq -r --arg l LINK_ID '.[] | select(.status == "active" and .targetId == $l and .sourceType == "note")
              | "\(.predicateKey)\t\(.sourceId)"'
   ```

   The work's body is the `archives` note if one exists (full text), else the `summarizes` note,
   else just the title and URL. Note members that are an `archives` or `summarizes` source of a
   link already in the collection are folded into that work, not emitted separately.
2. A collection holding both `Archive: X` and `Summary of "X"` notes with no link member is the
   same pattern — pair them by the shared title (or by the link both point at via
   `rei edge show NOTE_ID --json`) and keep the archive.
3. Ask once how to handle pairs if it isn't obvious: **full archive only** (recommended),
   archive followed by its summary, or summaries only.
4. Remaining notes and docs are one work each. A link with no note is a title+URL stub — warn
   if many works are stubs, since the book will have little content.

Chapter title: the link title for link-based works; otherwise `displayName` (falling back to
`rei note show NOTE_ID --json | jq -r .title`), with `Archive: ` / `Summary of "…"` wrappers
stripped.

### 3. Infer order

Apply the strongest signal available; use weaker ones only to break ties.

**a. Title sequence markers** — `Part I`/`Part 2`, `Chapter 3`, leading `01 -` / `1.` / `#1`,
`(3/5)`, ordinal words. If most works carry one, sort by it. Positional titles: `Preface`,
`Foreword`, `Introduction`, `Prologue` first; `Conclusion`, `Epilogue`, `Afterword`, `Appendix`
last.

**b. Ordering edges** — check which sequence-like predicates exist
(`rei predicate list --json | jq -r '.[].predicateKey'`, e.g. `follows`, `precedes`, `next`),
then look for them between members with `rei edge show ENTITY_ID --json`. A chain gives the
order directly.

**c. Content** — read each body (`rei note print NOTE_ID`; docs via their `relativePath`) and
build a precedence graph:
- a work that mentions another work's title, or links to it (`rei note outgoing-links NOTE_ID
  --json`, `[[wikilinks]]`), comes after it;
- a work that introduces a term another work uses without introduction comes first;
- backward references ("as we saw", "recall that", "building on") suggest later, forward
  references ("in the next part", "we will see") suggest earlier;
- foundational/motivating content precedes synthesis and advanced application.

Topologically sort; cycles and ties fall through to the next signal.

**d. `addedAt`** — earliest first.

Present the numbered order with the method used, and ask: proceed (recommended), let me
reorder (user gives e.g. `3, 1, 2`), or reverse.

### 4. Assemble chapters

Work in a temporary directory (`WORK=$(mktemp -d)`). For each work in order, write
`$WORK/ch-NNN-<slug>.md` (`printf '%03d'`, slug lowercase-hyphenated, ≤40 chars) — reusing
bodies already fetched in step 3:

- Note: `rei note print NOTE_ID`. Doc: read `relativePath`. Link stub: `# TITLE` then the URL.
- Ensure the chapter starts with exactly one H1 carrying the chapter title: replace a leading
  `# Archive: …` / `# Summary of "…"` heading, or prepend `# TITLE` if there is none. Demote any
  further H1s in the body to H2 so the TOC stays one entry per work.
- For link-based works, add a `Source: <url>` line under the heading if the body lacks one.
- Skip empty bodies and warn.

Chapter bodies often contain their own YAML frontmatter (archive notes store the source page's
verbatim); pandoc would let it **override the book's title and author**. The pandoc command
below disables `yaml_metadata_block` for that reason — keep it.

### 5. Metadata

Title defaults to the collection name; ask for the author if the user hasn't said (for a
single-author series, suggest the author from the archive notes' `Author:`/frontmatter).
Description is the collection description.

```yaml
# $WORK/metadata.yaml
title: "EPUB_TITLE"
author: "AUTHOR"
description: "COLLECTION_DESCRIPTION"
lang: en
```

Set `lang` from the content if it isn't English.

### 6. Build

Default output: `~/Downloads/<collection-slug>.epub` unless the user gives a path.

```bash
pandoc --metadata-file="$WORK/metadata.yaml" \
  -f markdown-yaml_metadata_block -t epub \
  --toc --toc-depth=2 --split-level=1 \
  --resource-path="$WORK:NOTES_DIR" \
  -o "OUTPUT_PATH" "$WORK"/ch-*.md
```

`NOTES_DIR` is the directory of the note members' `relativePath`, so relative image embeds
resolve. The zero-padded names keep the glob in chapter order.

Verify before reporting: the file exists and is non-empty, and
`unzip -p "OUTPUT_PATH" EPUB/content.opf | grep -o '<dc:title[^<]*'` shows the intended title.
On pandoc errors, identify the failing chapter (build chapters individually if needed). Remove
`$WORK` on success and failure.

## Output Format

```
## EPUB Generated

- **Collection**: <name> (<collection_id>)
- **Title / Author**: <title> / <author>
- **Chapters**: <N> (<K> link stubs, <S> skipped)
- **Ordering**: <title markers | edges | content analysis | addedAt | user-specified>
- **Output**: <path> (<size>)

### Chapters
1. <title>  — <archive | summary | note | doc | link stub>
...

Open with: `open "<path>"`
```

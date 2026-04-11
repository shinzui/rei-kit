---
name: rei-collection-epub
description: Export a Rei collection to EPUB format using pandoc, with automatic chapter ordering inferred from titles, edges, content analysis, and timestamps.
allowed-tools: AskUserQuestion, Bash, Read, Write, Glob, Grep
---

# Rei Collection to EPUB

This skill takes a Rei collection (manual or virtual), reads its members, infers a logical
chapter order, and produces an EPUB file using pandoc.

## When to Use

Activate when the user says things like:
- "Export this collection as an epub"
- "Turn my collection into an ebook"
- "Create an epub from collection ..."
- "Generate an epub for ..."
- "Make a book from my collection"

## Key Concepts

### Collections

A Rei collection groups knowledge artifacts (notes, links, docs). Collections can be
**manual** (curated membership) or **virtual** (query-driven). Members have a `displayName`,
`addedAt` timestamp, a `memberRef` (type + ID), and a `relativePath` to the file on disk.

### Ordering Signals

The skill uses multiple signals to infer chapter order, listed by priority:

1. **Explicit sequence markers in titles** — "Part I", "Chapter 3", "01 -", "#2:", ordinals
   like "First", "Second"
2. **Edges between members** — predicates like `follows`, `precedes`, or `continues` that
   encode explicit ordering relationships
3. **Content analysis** — cross-references between notes, concept introduction vs. usage,
   wikilinks, and narrative continuity markers inside the body text
4. **`addedAt` timestamps** — the order members were added to the collection, used as a
   fallback when no stronger signal exists

### Member Types

- `member_note` — a Rei note; content retrieved via `rei note print <ID>`
- `member_link` — a Rei link; metadata only (URL + title), no body content
- `member_doc` — a Rei document; content read from its file path

## Workflow Overview

1. **Select collection** — identify which collection to export
2. **Fetch members** — retrieve collection metadata and member list
3. **Infer order** — analyze titles, edges, content, and timestamps to propose chapter sequence
4. **Confirm order** — present the proposed order to the user for approval or reordering
5. **Extract content** — read the body of each member in order
6. **Configure metadata** — set EPUB title, author, and other pandoc metadata
7. **Generate EPUB** — assemble markdown and run pandoc
8. **Summary** — report the result

## Instructions for Claude

### Phase 1: Select Collection

If the user provided a collection name in their request, use it directly. Otherwise, list
available collections and ask:

```bash
rei collection list
```

```
Question: "Which collection would you like to export as an EPUB?"
Header: "Collection"
Options:
- <list collection names from the output above>
```

Store the chosen collection name as `COLLECTION_NAME`.

### Phase 2: Fetch Members

Retrieve the collection and its members as JSON:

```bash
rei collection show "COLLECTION_NAME" --json
```

Parse the JSON output. Extract:
- `collection.name` — the collection title
- `collection.description` — used as the EPUB description
- `members[]` — array of member objects

Each member has:
- `memberId` — unique member ID
- `displayName` — human-readable title
- `memberRef.type` — one of `member_note`, `member_link`, `member_doc`
- `memberRef.data` — the entity ID (e.g., `note_01k...`)
- `relativePath` — file path on disk (for notes and docs)
- `addedAt` — ISO timestamp of when the member was added

If the collection has zero members, inform the user and stop.

If the collection has link-only members (`member_link`), warn the user that links have no
body content and ask whether to proceed (the epub would contain only titles and URLs) or stop.

### Phase 3: Infer Order

Apply the following ordering strategy, in priority order:

#### 3a: Check for Explicit Sequence Markers in Titles

Scan each `displayName` for patterns that indicate ordering:

- **Numbered parts**: "Part I", "Part II", "Part III" (Roman numerals); "Part 1", "Part 2"
- **Numbered chapters**: "Chapter 1", "Chapter 2"; "Ch. 1", "Ch. 2"
- **Leading numbers**: "01 -", "02 -", "1.", "2.", "#1", "#2"
- **Ordinal words**: "First", "Second", "Third", "Introduction", "Conclusion", "Epilogue",
  "Preface", "Foreword", "Appendix"
- **Series indicators**: "Part I: ...", "— Part II", "(3/5)"

If sequence markers are found in a majority of members, sort by the extracted sequence number.
Place special positional titles first/last:
- **First**: "Preface", "Foreword", "Introduction", "Prologue"
- **Last**: "Conclusion", "Epilogue", "Afterword", "Appendix"

#### 3b: Check for Ordering Edges

If title-based ordering is insufficient, check for edges between the collection's members:

```bash
rei edge show <MEMBER_ENTITY_ID> --json
```

Run this for each member. Look for edges with predicates that imply sequence (e.g.,
`follows`, `precedes`, `continues`, `next`, `previous`). If such edges form a chain, use them
to derive the order.

#### 3c: Analyze Content for Ordering Clues

If titles and edges are insufficient, read the body of every member to look for ordering
signals in the text itself. For each note member, run:

```bash
rei note print <NOTE_ID>
```

For doc members, read the file at `relativePath`. Skip link-only members (no body).

Scan the collected content for these signal types, strongest first:

**Cross-references by title.** Look for mentions of other members' display names or
recognizable fragments of them. If chapter B mentions chapter A's title (e.g., "as we saw in
*Processes and State Machines*" or "building on Part I"), that implies A precedes B. Build a
directed graph of "refers-to" relationships and topologically sort it. Cycles mean the signal
is ambiguous for those nodes — fall through to weaker signals for them.

**Wikilinks between notes.** Rei notes may contain `[[note_ID]]` wikilinks. If note B links
to note A, A likely comes first (B depends on knowledge introduced in A). Use `rei note
outgoing-links <NOTE_ID>` to discover these. Build the same directed dependency graph as with
title cross-references and topologically sort.

**Concept introduction vs. usage.** Identify key domain terms, acronyms, or definitions that
are *introduced* in one chapter (look for phrases like "we define", "let us introduce",
"referred to as", ": a ...", bold or italic first-use markers) and *used without introduction*
in another. A chapter that introduces a term should precede chapters that assume it. Focus on
terms that appear in at least two members — single-member terms are not useful for ordering.

**Narrative continuity markers.** Detect language that implies sequencing:

- **Backward references** (the chapter using these comes later): "as we discussed",
  "recall that", "as shown above", "in the previous section", "building on",
  "we established that", "earlier we saw"
- **Forward references** (the chapter using these comes earlier): "in the next section",
  "we will see", "later we will", "the following chapter", "this sets up",
  "we will return to this"

Count backward-reference and forward-reference markers per chapter. Chapters heavy on
backward references tend to come later; chapters heavy on forward references tend to come
earlier. Use this as a relative ranking signal.

**Structural progression.** Look at the complexity arc of the content:

- Chapters that start with foundational definitions, motivation, or "why" framing tend to
  come first
- Chapters that synthesize, compare alternatives, or present advanced applications tend to
  come last
- Chapters that conclude with "next steps", "future work", or a transition sentence pointing
  to the next topic provide a natural chain

**Combining content signals.** Merge the evidence from all content signals into a single
ordering:

1. Start with the topological sort from cross-references and wikilinks (strongest signal)
2. Break ties using concept-introduction order (introducers before consumers)
3. Break remaining ties using narrative marker counts (more backward refs = later)
4. Break remaining ties using structural progression (foundational before applied)

If the content analysis produces a clear total order, use it. If it produces a partial order
(some members are unambiguous, others are tied), combine with `addedAt` timestamps to break
the remaining ties.

#### 3d: Fall Back to `addedAt` Timestamps

If titles, edges, and content analysis do not provide clear ordering, sort members by their
`addedAt` timestamp (earliest first). This assumes the user added them in reading order.

### Phase 4: Confirm Order

Present the proposed chapter order to the user:

```
## Proposed Chapter Order

1. <displayName 1>
2. <displayName 2>
3. <displayName 3>
...

Ordering method: <"title sequence markers" | "edge relationships" | "content analysis" | "chronological (addedAt)">
```

```
Question: "Does this chapter order look correct?"
Header: "Chapter Order"
Options:
- Yes, proceed (Recommended)
- Let me reorder — I'll provide the correct sequence
- Reverse the order
```

If the user wants to reorder, ask them to provide the numbers in their preferred order (e.g.,
"3, 1, 2") and reorder accordingly.

### Phase 5: Extract Content

For each member in the confirmed order, extract its content. If content was already fetched
during Phase 3c (content analysis), reuse it rather than fetching again.

#### For `member_note` members:

```bash
rei note print <NOTE_ID>
```

Capture the markdown output. If the note content starts with an H1 heading, use it as-is. If
it does not start with an H1, prepend `# <displayName>` as the chapter heading.

#### For `member_doc` members:

Read the file at the member's `relativePath` directly using the Read tool. If the content
does not start with an H1, prepend `# <displayName>`.

#### For `member_link` members:

Generate a minimal chapter:

```markdown
# <displayName>

<URL from link metadata>
```

If the link has an associated note (check edges for notes that `summarizes` this link), fetch
that note's content and include it after the URL.

#### Content Assembly

Write each chapter to a temporary file in sequence:

```bash
mkdir -p /tmp/rei-epub-work
```

For each chapter (in order), write to `/tmp/rei-epub-work/ch-NN-<slug>.md` where `NN` is the
zero-padded chapter number and `<slug>` is a slugified version of the display name (lowercase,
hyphens, max 40 chars).

### Phase 6: Configure Metadata

Ask the user for EPUB metadata:

```
Question: "What should the EPUB title be?"
Header: "Title"
Options:
- Use collection name: "<COLLECTION_NAME>" (Recommended)
- Let me type a custom title
```

```
Question: "Who is the author?"
Header: "Author"
Options:
- Let me type the author name
```

Create a pandoc metadata file at `/tmp/rei-epub-work/metadata.yaml`:

```yaml
---
title: "<EPUB_TITLE>"
author: "<AUTHOR>"
description: "<COLLECTION_DESCRIPTION>"
lang: en
---
```

### Phase 7: Generate EPUB

Ask where to write the output:

```
Question: "Where should the EPUB be saved?"
Header: "Output Path"
Options:
- Default: ~/Downloads/<slugified-collection-name>.epub (Recommended)
- Let me specify a path
```

Run pandoc to produce the EPUB:

```bash
pandoc --metadata-file=/tmp/rei-epub-work/metadata.yaml \
  -f markdown \
  -t epub \
  --toc \
  --toc-depth=2 \
  -o "<OUTPUT_PATH>" \
  /tmp/rei-epub-work/ch-*.md
```

The `--toc` flag generates a table of contents from the H1/H2 headings. The glob
`ch-*.md` expands in alphabetical order, which matches the zero-padded numbering scheme.

If pandoc fails, show the error and attempt to diagnose:
- Missing pandoc: suggest `nix-env -iA nixpkgs.pandoc` or `brew install pandoc`
- Invalid markdown: identify the problematic chapter and show a preview

After success, clean up temporary files:

```bash
rm -rf /tmp/rei-epub-work
```

### Phase 8: Summary

```
## EPUB Generated

- **Collection**: <COLLECTION_NAME>
- **Title**: <EPUB_TITLE>
- **Author**: <AUTHOR>
- **Chapters**: <N>
- **Ordering**: <method used>
- **Output**: <OUTPUT_PATH>
- **File size**: <size>

### Chapter List

1. <chapter 1 displayName>
2. <chapter 2 displayName>
...

### Next Steps

1. Open the EPUB: `open "<OUTPUT_PATH>"`
2. Verify chapter order and formatting in your reader
3. Re-run with a different order if needed
```

## Important Notes

- Always use `rei collection show --json` for structured data — do not parse table output
- Always use `rei note print <ID>` to get note content, not `cat` on the file (print handles
  any preprocessing the CLI applies)
- The glob `/tmp/rei-epub-work/ch-*.md` relies on zero-padded chapter numbers — use
  `printf "%02d"` for collections up to 99 chapters, `"%03d"` for larger ones
- If a note's content is empty or only whitespace, skip it and warn the user
- Pandoc must be installed — check with `which pandoc` before proceeding and report clearly
  if it's missing
- Clean up `/tmp/rei-epub-work` both on success and on failure
- For virtual collections, run `rei collection exec "NAME" --json` if `show` does not
  include members, then use the exec output to discover member IDs

---
name: rei-attach-link-to-intention
description: Attach a URL to an intention and record why it was attached — reuse or create the canonical link, connect link → intention with a purpose predicate (inspiration-for, reference-for, comparable-for, …) whose edge description holds the specific reason ("their pricing table"), wire the aspects that made it relevant to topics (`exemplifies` or `about`), classify the link, add a why-note that `references` the link when the reason is too long for one line, and show how to recall it later. Use when the user attaches a link to work in progress with a reason; use rei-ingest-url to summarize a page and rei-bookmark-url to file it under a topic.
allowed-tools: AskUserQuestion, Bash, Read, WebFetch
---

# Rei Attach Link to Intention

Attaches a URL to an intention **with its reason**, so that months later the user can look at
the link, or at the intention's links, and see why it is there: "UI inspiration for the landing
page — I like their pricing table."

Inputs: a URL, the user's note about it, and the intention. The note is the main input. The
page is fetched only for a title and classification. It is not summarized.

Use `rei-ingest-url` instead when the page should be **summarized** into a note. Use
`rei-bookmark-url` to file a link under a **topic** as general reference and grow the ontology
around it. This skill borrows that skill's topic-modeling rules for aspect topics and does not
repeat them.

## Key Concepts

- **Link dedup**: Rei canonicalizes URLs, and `rei link add` on a known URL reuses the link
  and only adds an attachment. The same link can therefore sit on several intentions for
  different reasons.
- **The reason belongs to the (link, intention) pair, not the link.** Custom properties are
  per link and shared by every intention the link is attached to, so never store the reason in
  a link property (for example `link-note`). Store it on an edge:
  - **Purpose**: the edge *predicate*, `link -[inspiration-for]-> intention`.
  - **Specific reason**: the edge `--description`, one line saying what exactly is useful.
    `rei edge show LINK_ID` prints it as `link_… -> intention_… [inspiration-for] — reason`.
- **Edges are immutable and unique**: there is no `edge update`, and a second edge with the
  same source, target, and predicate is rejected. To change a reason, remove the edge
  (a soft delete that keeps history) and add a new one. To add more detail, write the why-note.
- **Aspects are topics.** The specific thing the user likes (pricing table, onboarding flow,
  typography, rate-limit design) is a topic, so "every link I saved for its pricing table" can
  be queried later. Choose the relation carefully:
  - `about`, via `rei topic associate`: the page's **subject matter** is the aspect, as in an
    article about pricing-table design.
  - `exemplifies`, via `rei edge add` from link to topic: the page **showcases** the aspect
    without being about it, as in a SaaS homepage that has a good pricing table. Don't
    write `about` for this case, because it would say the homepage is about pricing tables.
- **Why-note**: a note on the intention with a `references` edge to the link (`references`
  already exists with source `link,note` and target `link`). Write one only when the reason
  doesn't fit one line.
- **Exit status**: `link`, `edge`, and `predicate` writes can print an error and still exit
  `0`. Topic associations and `note new` follow the exit contract (`2` refused or invalid,
  `70` store failure). Verify by reading back (Phase 8).

## Purpose Predicates

All purpose predicates run **link → intention** (`--source-types link --target-types intention`).
Reuse an existing one when it fits. Otherwise define one from this starter set, the first time
it is needed:

| Key | Label | Use when the link is… |
|-----|-------|-----------------------|
| `inspiration-for` | Inspiration for | design, UX, copy, or ideas to emulate |
| `reference-for` | Reference for | docs, specs, or how-tos to consult while doing the work |
| `comparable-for` | Comparable for | a competitor, alternative, or prior art to benchmark against |
| `evidence-for` | Evidence for | research or data that supports a decision in the work |
| `tool-for` | Tool for | a tool, library, or service to use in the work |
| `counterexample-for` | Counterexample for | an anti-pattern or cautionary example to avoid |

```bash
rei predicate list --json | jq -r '.[] | select(.targetTypes | index("intention")) | "\(.predicateKey)\t\(.sourceTypes)\t\(.predicateDescription)"'

rei --actor claude-code predicate define inspiration-for -l "Inspiration for" \
  -d "Source link is design, UX, copy, or idea inspiration for the target intention" \
  --source-types link --target-types intention
```

If no starter purpose fits, propose a new `<noun>-for` key and ask the user before defining it.
Never redefine or loosen an existing predicate.

## Workflow

Use `rei --actor claude-code …` on every write and always pass IDs explicitly. An omitted
`-i`, `-l`, or `LINK_ID` opens an fzf picker, which hangs the run.

### 1. Gather the Inputs

- **URL**: must be an absolute `http(s)://` URL. Otherwise stop.
- **Note**: the user's words about why they are attaching it. If none was given, ask. Without a
  reason this skill has nothing to record, so point the user at `rei link add` instead.
- **Intention**: an `intention_…` ID, or search by keyword and offer the top matches:

  ```bash
  rei intention list --all -s "KEYWORD" --json | jq -r '.[] | "\(.id)\t\(.title)"'
  ```

### 2. Interpret the Note

From the note, work out:

- **PURPOSE**: one purpose predicate from the table above.
- **REASON**: one line, at most about 160 characters, naming the specific thing and what is
  good about it. Keep the user's own words where possible: `Pricing table — three tiers side by
  side, annual/monthly toggle, highlighted middle plan`. Leave out the purpose itself; the
  predicate already carries it.
- **ASPECTS**: zero to three concrete things the user singled out (`pricing-table`). Skip
  generic ones ("design", "nice site"). Mark each as `exemplifies` (showcased) or `about`
  (subject matter).
- **NEEDS_NOTE**: true when the note has more than the one-line reason can hold, such as several
  distinct things liked, caveats ("but not their colors"), or how to adapt it to the intention.

### 3. Check the Link and Existing Reasons

```bash
rei link list --all -d DOMAIN --json \
  | jq -r --arg url "URL" '.[] | select(.canonical_url == $url or .original_url == $url) | .id'
rei link list --all -q "DISTINCTIVE-URL-FRAGMENT" --json   # if canonicalization changed the URL
```

For an existing `LINK_ID`:

```bash
rei link show LINK_ID                   # attachments + custom properties
rei edge show LINK_ID --json \
  | jq -r --arg i "INTENTION_ID" '.[] | select(.status == "active" and .targetId == $i) | "\(.edgeId)\t\(.predicateKey)\t\(.description)"'
```

- Already attached to the intention: skip the attachment in Phase 5.
- An edge with the **same purpose** already exists: show its reason. If the new note adds
  detail, put that in the why-note. If it corrects the old reason, replace the edge in Phase 6
  after the user confirms.
- An edge with a **different purpose**: add the new one too. A link can serve an intention in
  more than one way.

### 4. Fetch Title and Classification

Skip this phase if the link already has a title and `content-type`. Otherwise use WebFetch and ask
for: the title, whether the aspects the user named are actually on the page, and hints for
`content-type` and `platform`. If the fetch fails, continue without a title. The user's note
is enough to record the reason.

If an aspect the user named isn't visible, for example because the pricing table sits behind
a login, keep it anyway and don't argue. The user saw it.

### 5. Plan, Then Attach and Classify

Show the plan once. It is an announcement, not an approval gate. Stop only when the purpose is
ambiguous, a new non-starter predicate is needed, or an existing reason would be replaced.

```
Attach: <TITLE> (<url>)          link: new / reused <link_id>
To:     <intention title> (<intention_id>)
Why:    inspiration-for — "Pricing table — three tiers side by side, annual toggle"
Aspects: pricing-table  exemplifies   REUSE / CREATE (instance-of ui-pattern)
Note:   none / why-note (details below)
Props:  content-type=homepage, platform=website
```

Attach. `link add` reuses the canonical link, so run it for new and reused links alike, unless
the link is already attached to this intention:

```bash
rei --actor claude-code link add "URL" -i INTENTION_ID -t "TITLE" \
  -p content-type=VALUE -p platform=VALUE
```

Capture `LINK_ID` from the output, or re-run the Phase 3 lookup. Confirm the intention appears
under Attachments in `rei link show LINK_ID`.

On a reused link, don't pass `-p` or `-t`. Instead set only what is missing:

```bash
rei custom-property show content-type --json | jq -r '.valueType.data.enumValues[]'
rei --actor claude-code link set-property -l LINK_ID content-type VALUE
rei --actor claude-code link title LINK_ID -t "TITLE"
```

Set `content-type` and `platform` only when a defined value clearly fits. Don't set `tags` or
`link-note`.

### 6. Record the Reason

```bash
rei ontology seed-system    # idempotent; ensures about / instance-of / broader-than
# define PURPOSE if missing (see Purpose Predicates)
rei --actor claude-code edge add -f LINK_ID -t INTENTION_ID -p PURPOSE -d "REASON"
```

To **replace** a reason, only after the user confirms:

```bash
rei --actor claude-code edge remove OLD_EDGE_ID
rei --actor claude-code edge add -f LINK_ID -t INTENTION_ID -p PURPOSE -d "NEW REASON"
```

### 7. Wire Aspects and the Why-Note

**Aspect topics.** Find each one before creating it:

```bash
rei topic list --json | jq -r '.[] | "\(.topicKey)\t\(.topicLabel)\t\(.topicId)"' | grep -i "ASPECT"
```

Reuse a topic that genuinely covers the aspect. Otherwise create it following
`rei-bookmark-url`'s Modeling Discipline: search by key, label, and reference first; no orphans;
`instance-of` a type for named patterns and things, `broader-than` from a parent between
abstract areas; create a missing type such as `ui-pattern` too.

```bash
rei --actor claude-code topic create pricing-table "Pricing table" -d "UI pattern presenting plans and prices for comparison"
rei topic show pricing-table --json | jq -r .topicId
rei --actor claude-code topic associate ui-pattern TOPIC_ID --relation instance-of
```

Then wire the link to each aspect:

```bash
# page showcases the aspect
rei predicate list --json | jq -e '.[] | select(.predicateKey == "exemplifies")' >/dev/null || \
  rei --actor claude-code predicate define exemplifies -l "Exemplifies" \
    -d "Source is a concrete example of the target pattern or concept (schema.org/exampleOfWork, loosely)" \
    --source-types link,note --target-types topic
rei --actor claude-code edge add -f LINK_ID -t TOPIC_ID -p exemplifies

# page is about the aspect (always pass --relation; the default is scoped-to)
rei --actor claude-code topic associate TOPIC_KEY LINK_ID --relation about
```

**Why-note**, only when NEEDS_NOTE is true:

```bash
printf '# Why "%s" is on this intention\n\nSource: %s\nPurpose: %s\n\n%s\n' \
  "TITLE" "URL" "PURPOSE" "$NOTE_MD" \
  | rei --actor claude-code note new -i INTENTION_ID --stdin
rei --actor claude-code edge add -f NOTE_ID -t LINK_ID -p references
rei --actor claude-code topic associate TOPIC_KEY NOTE_ID --relation about   # per aspect
```

`NOTE_MD` is the user's note, lightly structured into what they like, caveats, and how it applies.
Keep their wording. Don't add content of your own from the page.

### 8. Verify

```bash
rei link show LINK_ID                                  # intention attachment, title, properties
rei edge show LINK_ID                                  # purpose edge with "— REASON", exemplifies edges
rei topic associations LINK_ID --relation about --json
rei edge show NOTE_ID --predicate references --json    # if a why-note was written
rei ontology validate
```

Redo any planned write that is missing, once. If an edge you wrote causes a `validate` error,
fix the edge and never loosen the predicate. Report pre-existing errors and leave them alone.

## Recalling Later

Include the relevant commands in the output so the user learns them:

```bash
# why is this link here?
rei edge show LINK_ID
# every link on an intention, with purpose and reason
rei edge show INTENTION_ID --json \
  | jq -r '.[] | select(.status == "active" and .sourceType == "link") | "\(.predicateKey)\t\(.sourceId)\t\(.description)"'
# all inspiration across intentions
rei edge list --predicate inspiration-for --json
# every link saved for a given aspect
rei topic edges pricing-table
```

## Output Format

```
## Link Attached

- **URL**: <url>
- **Link**: <link_id> — <title>  (new / reused; attachment added / already attached)
- **Intention**: <intention_id> — <title>
- **Why**: <purpose> — "<reason>"  (edge <edge_id>; new / replaced <old_edge_id>)

### Context
- Aspects: <topic> exemplifies (reused / created instance-of <type>), <topic> about
- Why-note: <note_id> -[references]-> link  (or: not needed)
- Properties: content-type=<v>, platform=<v>  (or: already set / skipped)
- Predicates defined: <key>  (or: none)

### Recall
rei edge show <link_id>

### Failed / Skipped
- <item — reason>  (or: nothing)
```

## Important Notes

- The reason goes on the edge (per link and intention), never in a link property.
- Record the user's reason, not your opinion of the page. Don't fetch-and-summarize.
- Don't create topics for generic aspects or for the site's owner. Don't restructure the
  ontology; point at `rei-curate-ontology` for existing problems.

---
name: rei-scaffold-kit-skill
description: Scaffold a new skill or agent for rei-kit — research the rei commands it will drive, write it to match the current kit conventions, and register it in kit.json. Use when creating a new skill or agent definition for rei-kit.
allowed-tools: AskUserQuestion, Bash, Read, Write, Edit, Glob, Grep
---

# Rei Scaffold Kit Skill

Creates a new **skill** (`skills/<name>/SKILL.md`) or **agent** (`agents/<name>.md`) in this
repo and registers it in `kit.json`. The hard part is not the file format — it is getting the
rei commands, flags, and domain modeling right, so most of the effort goes into research.

## How kit content is consumed

`rei kit install` copies items listed in `kit.json` into **both** providers:

- **Claude Code** — `.claude/skills/<name>/` and `.claude/agents/<name>.md`.
- **Codex** — `.agents/skills/<name>/`, and agents wrapped into `.codex/agents/<name>.toml`
  (`name`, `description`, and the Markdown body as `developer_instructions`).

Both scopes are also loaded into `rei agent` sessions, which run under whichever provider
`REI_LLM_PROVIDER` / `llm.commands.*` selects. Consequences for what you write:

- The **body** must stand on its own. Agent frontmatter (`tools`, `model`) and skill
  `allowed-tools` are Claude-only; Codex sees only name, description, and body.
- The **description** is what both providers use to decide when to activate the item and what
  `rei kit list` / the fzf picker shows. Make it specific: what it does, the key steps, and
  when to use it over a sibling.
- Each entry's `version` drives `rei kit status` outdated detection, so new items start at
  `1.0.0` and any edit to an existing item bumps it (patch: wording/fixes, minor: new
  behavior, major: changed workflow or inputs).

## Workflow

### 1. Clarify

From the request, determine: skill or agent, name, purpose, inputs, and what rei entities it
reads or writes. Ask (one AskUserQuestion call, only for what is actually missing) rather than
walking through a fixed questionnaire. If the request is complete, skip straight to research.

Naming: `rei-<verb>-<object>` for skills (`rei-ingest-url`, `rei-note-from-tmp`); agents end in
`-guide` (conversational) or `-expert` (reference).

### 2. Research

**Siblings.** Check `kit.json` and `skills/` for overlapping items. Read the closest one or two
in full — recent skills (`rei-note-from-tmp`, `rei-bookmark-url`, `rei-curate-ontology`) reflect
current conventions better than older ones. If an existing skill already defines a discipline
the new one needs (e.g. topic modeling in `rei-bookmark-url`), reference it instead of
restating it, and state when to use the sibling instead.

**Rei itself.** Never write a command or flag from memory — verify each against the CLI:

```bash
rei --help
rei <command> [<subcommand>] --help
rei help --list          # conceptual topics: edges, topics, projects, custom-properties, ...
rei help <topic>
```

Deeper references live in the rei repo (`mori registry show shinzui/rei --full` for its path):
`docs/user/concepts.md`, `docs/user/cli/<command>.md`, and `docs/user/CHANGELOG.md` — skim the
recent changelog entries for the areas the item touches, since features land faster than
existing skills are updated. Where it is safe, run read-only commands (`rei <x> list --json`)
to see real output shapes before writing `jq` against them.

**Newer surfaces worth checking** before designing around older patterns:

| Need | Use | Instead of |
|------|-----|------------|
| What an entity is about | `rei topic associate/associations/entities` (validated `about`, `scoped-to`, `instance-of`) | ad-hoc edges or tags |
| Canonical identity / dedup of a subject | `rei topic add-ref` / `ref-show` | label matching alone |
| Which software project something belongs to | `rei project` (typed topics, `scope`, `sync` from Mori) | the deprecated `local-repo` property |
| Category-driven defaults | `rei category show` (property bindings, auto-set if missing) and `print-note-guidance` | setting those properties by hand |
| Multi-value / historical properties | `append-property` / `remove-property-value`, `set-property --at` | re-setting whole values, backdating by hand |
| Portfolio reads | `rei view exec --json`, `view exec-batch`, `rei dependency graph --json` | looping `show` calls |

**Is a kit item the right vehicle?** A fixed, judgment-free procedure is better as a
`rei playbook`; recurring or event-triggered work is an agent schedule (`rei help
agent-schedules`), which can invoke a skill; delegated work that needs review should record
checkpoints (`rei help review-checkpoints`). A kit skill is for workflows that need judgment.
Say so if the request fits another vehicle better.

### 3. Write

Match the structure of the sibling you read. Skills typically have: frontmatter, a short
intro (incl. contrast with siblings), **When to Use**, **Key Concepts** (only domain facts the
model can't infer), numbered **phases** with exact commands, an **Output Format** summary, and
**Important Notes** for the invariants that matter most.

Write instructions, not a tutorial: concrete commands, decision rules, and failure handling.
Leave out anything a capable model does by default (how to ask a question, generic advice,
restating the tool list). Mark genuinely unknown domain details `<TODO: ...>` rather than
inventing them.

Rei conventions to apply when relevant:

- **Actor**: use the global form `rei --actor claude-code <command> ...` on anything that
  creates or changes entities — it works on every command.
- **Non-interactive**: always pass IDs explicitly (`-i INTENTION_ID`, `-n NOTE_ID`, ...);
  omitted IDs open fzf pickers that hang an agent run. Use `--json` + `jq` for reading state.
- **Reuse before create**: search for existing intentions, links (canonical-URL dedup), topics
  (`rei topic ref-show URL`), tags, and predicates before creating new ones.
- **Knowledge modeling**: subjects are topics (`rei topic associate ... --relation about`),
  tags are facets; `instance-of` for named things vs `broader-than` between concepts; seed with
  `rei ontology seed-system` and finish with `rei ontology validate`. Defer to
  `rei-bookmark-url` / `rei-curate-ontology` for the full discipline.
- **Custom properties**: values must come from the definition (`rei custom-property show KEY`);
  respect `scopedToCategories`; don't set what a category binding already sets.
- **Don't trust exit status alone**: only some commands follow the automation exit contract
  (`docs/user/cli/automation-exit-contract.md` — e.g. topic associations, note writes: `2`
  refused/invalid, `70` store failure). Many legacy handlers print an error and exit `0`, so
  verify writes by reading back (`show --json`) before anything irreversible.
- **Safety**: one confirmation gate before bulk writes or anything destructive; verify what
  was written before deleting sources; never loosen existing predicates or restructure the
  ontology as a side effect.
- **Cross-repo references** use `mori://` URIs.

Agent specifics: `tools:` should be read-oriented (no Write/Edit); `model: sonnet` is the
current default (see `agents/rei-custom-property-guide.md`). Point to docs and `rei help`
topics instead of duplicating them, and keep quick-reference tables short.

### 4. Register

Add an entry to the matching array in `kit.json`:

```json
{
  "name": "<name>",
  "description": "<same as frontmatter description>",
  "version": "1.0.0",
  "path": "skills/<name>",
  "files": ["SKILL.md"]
}
```

Agents use `"path": "agents"` and `"files": ["<name>.md"]`. Validate:

```bash
jq -e '.skills, .agents' kit.json > /dev/null && echo valid
```

### 5. Report

Summarize the files created/modified, the rei commands the item relies on (and that each was
checked against `--help`), any `<TODO>`s left, and how to try it — for local testing, symlink
it into `.claude/skills/` as this skill is; installed users get it via `rei kit update` after
it is pushed. Suggest a Conventional Commit such as `feat: add <name> skill`; don't commit
unless asked.

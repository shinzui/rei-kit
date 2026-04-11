# Create a Rei Scaffolding Skill for Skills and Agents

This ExecPlan is a living document. The sections Progress, Surprises & Discoveries,
Decision Log, and Outcomes & Retrospective must be kept up to date as work proceeds.

This document is maintained in accordance with `.claude/skills/exec-plan/PLANS.md`.


## Purpose / Big Picture

After this change, a user working inside the rei-kit repo can invoke a `/rei-scaffold` skill
to interactively generate new skill or agent files that conform to rei's established
conventions. The skill will:

1. Ask whether the user wants to create a **skill** or an **agent**.
2. Gather the necessary metadata (name, description, tools, model, domain focus).
3. Generate the file(s) in the correct location inside rei-kit (`skills/<name>/SKILL.md` or `agents/<name>.md`).
4. Update `kit.json` to register the new skill or agent.
5. Produce a summary of what was created and next steps.

The skill uses **this repo (rei-kit)** as the reference for skill and agent conventions —
examining existing skills and agents here to understand structural patterns. The rei repo
at `/Users/shinzui/Keikaku/bokuno/rei-project/rei` and the `rei` CLI help system are used
only for **domain context** — understanding rei concepts, available CLI commands, and entity
types that skills/agents might need to reference.


## Progress

- [x] Create skill directory `skills/rei-scaffold/` (2026-04-08)
- [x] Write `skills/rei-scaffold/SKILL.md` with full scaffolding workflow (2026-04-08)
- [x] Update `kit.json` to register the new skill (2026-04-08)
- [x] Validate: read the generated SKILL.md and confirm it follows conventions (2026-04-08)
- [x] Validate: kit.json is valid JSON with the new entry (2026-04-08)
- [x] Revision: update SKILL.md reference strategy — use rei-kit skills/agents as structural reference, rei repo + `rei --help` for domain context only (2026-04-08)


## Surprises & Discoveries

(None yet.)


## Decision Log

- Decision: Name the skill `rei-scaffold` rather than `rei-create-skill` or `rei-create-agent`.
  Rationale: The skill handles both skills and agents, so a neutral verb ("scaffold") is more
  accurate. It also avoids confusion with `rei-create-module-agent` which lives in the rei
  repo and is development-focused (creates agents from Haskell module docs).
  Date: 2026-04-08

- Decision: The skill lives in `skills/rei-scaffold/SKILL.md` (the kit's user-facing skills
  directory), not in `claude/skills/` (which is reserved for seihou-managed dev tools like
  exec-plan).
  Rationale: This is an end-user skill for kit users, consistent with where `rei-bootstrap`,
  `rei-bootstrap-habit`, and `rei-summarize-links` live.
  Date: 2026-04-08

- Decision: The skill uses rei-kit's own skills/agents as structural reference, not the
  rei repo's skills/agents.
  Rationale: rei-kit is the distribution channel with its own conventions. The rei repo
  contains development-focused artifacts with different scope and complexity. Referencing
  rei-kit's own examples keeps the skill self-contained and ensures generated output matches
  the kit's conventions specifically.
  Date: 2026-04-08 (revised)

- Decision: The rei repo and `rei --help` CLI system are used for domain context only —
  understanding available commands, entity types, and concepts that skills/agents might
  need to reference.
  Rationale: Skills often orchestrate `rei` CLI commands and need to know the correct syntax,
  flags, and entity relationships. The rei repo source and CLI help provide authoritative
  information for this without coupling structural conventions.
  Date: 2026-04-08 (revised)

- Decision: Use the same YAML frontmatter + Markdown body format established by existing
  skills and agents in rei-kit.
  Rationale: Consistency. The skills in rei-kit use this format and Claude Code discovers
  and parses it natively.
  Date: 2026-04-08


## Outcomes & Retrospective

**Completed 2026-04-08.**

- `skills/rei-scaffold/SKILL.md` created (552 lines) with a 7-phase interactive workflow
  covering both skill and agent scaffolding.
- `kit.json` updated with the `rei-scaffold` entry; JSON validates clean.
- The skill follows all established conventions: YAML frontmatter, `AskUserQuestion` prompt
  blocks, `--actor claude-code` guidance, reference corpus reading before generation,
  `kit.json` registration step, and structured output format.
- Both tracks (skill and agent) produce output matching the format of existing rei-kit
  skills (e.g., `skills/rei-bootstrap-habit/SKILL.md`).

**Revision 2026-04-08:** Reference strategy changed — see revision note at bottom.


## Context and Orientation

### Repository: rei-kit

Located at `/Users/shinzui/Keikaku/bokuno/rei-project/rei-kit`. This is a Claude Code skills
and agents distribution repo for Rei end-users.

**Key files and directories:**

- `kit.json` — Manifest listing all available skills and agents. Version 1 format with
  `skills` and `agents` arrays. Each entry has `name`, `description`, `path`, and `files`.
- `skills/` — User-facing skill directories. Each contains a `SKILL.md` file.
  - `skills/rei-bootstrap/SKILL.md` — Large interactive workflow skill (bootstrap intentions).
  - `skills/rei-bootstrap-habit/SKILL.md` — Interactive habit creation skill.
  - `skills/rei-summarize-links/SKILL.md` — Link extraction and summarization.
- `agents/` — Empty directory, ready for agent definitions.
- `claude/skills/exec-plan/` — Seihou-managed dev skill (not user-facing).
- `.claude/skills/exec-plan` — Symlink to `claude/skills/exec-plan`.
- `README.md` — Installation instructions for end-users.

### Repository: rei (domain context only)

Located at `/Users/shinzui/Keikaku/bokuno/rei-project/rei`. This is the main Rei application.
The skill reads from here **only for domain context** — understanding rei concepts, CLI
commands, and entity types. It never writes to it and does not use it as a structural
reference for skill/agent format.

**Useful for domain context:**
- `docs/user/cli/` — CLI command reference (commands, flags, entity types)
- `docs/user/concepts.md` — Key concepts and primitives
- `rei-cli/src/Rei/Cli/Commands/` — Source of truth for CLI command structure

**Also available:** The `rei` CLI help system provides authoritative command reference:
- `rei --help` — Top-level commands
- `rei <command> --help` — Command-specific flags and options
- `rei <command> <subcommand> --help` — Subcommand details

### kit.json format

```json
{
  "version": 1,
  "skills": [
    {
      "name": "skill-name",
      "description": "What it does",
      "path": "skills/skill-name",
      "files": ["SKILL.md"]
    }
  ],
  "agents": [
    {
      "name": "agent-name",
      "description": "What it does",
      "path": "agents",
      "files": ["agent-name.md"]
    }
  ]
}
```

### Conventions (derived from rei-kit's existing skills)

- Skill names are kebab-case, prefixed with `rei-` for Rei-specific skills.
- Agent names are kebab-case, prefixed with `rei-` and suffixed with `-expert` or `-guide`.
- All entity-creation CLI commands include `--actor claude-code`.
- Skills use `AskUserQuestion` for interactive flows with structured options.
- Agents use `model: sonnet` for cost efficiency.
- Agents are read-only (tools: `Read, Grep, Glob` or `Read, Bash`).
- Skills that create files use `Write` in their allowed-tools.
- The `rei` CLI help system (`rei <command> --help`) is the authoritative reference for
  command syntax, flags, and entity types when building skills that orchestrate rei commands.


## Plan of Work

### Milestone 1: Create the rei-scaffold skill

This is the only milestone. At the end, `skills/rei-scaffold/SKILL.md` exists with a complete
scaffolding workflow, and `kit.json` includes the new skill entry.

**Scope:** Create a single SKILL.md that implements an interactive workflow with two tracks
(skill creation and agent creation). The skill should:

1. Ask the user whether they want to scaffold a skill or an agent.
2. For skills: gather name, description, allowed-tools, workflow phases, and generate a
   SKILL.md skeleton with all standard sections populated.
3. For agents: gather name, description, tools, model, domain focus, and generate an agent
   `.md` file with all standard sections populated.
4. In both cases, update `kit.json` to register the new artifact.
5. Show a summary of what was created.

**Files to create:**
- `skills/rei-scaffold/SKILL.md` — The skill definition.

**Files to modify:**
- `kit.json` — Add the `rei-scaffold` entry to the `skills` array.

**Acceptance criteria:**
- The SKILL.md follows the established frontmatter + markdown format.
- The workflow covers both skill and agent creation tracks.
- Generated artifacts follow the conventions documented in Context and Orientation.
- The skill references existing examples from rei-kit for structural patterns, and the rei
  repo + `rei --help` for domain context.
- `kit.json` is valid JSON with the new entry.


## Concrete Steps

All commands run from working directory `/Users/shinzui/Keikaku/bokuno/rei-project/rei-kit`.

**Step 1: Create the skill directory**

```bash
mkdir -p skills/rei-scaffold
```

Expected: directory created, no output.

**Step 2: Write `skills/rei-scaffold/SKILL.md`**

Create the file with the full skill definition (see Plan of Work for content requirements).

**Step 3: Update `kit.json`**

Add this entry to the `skills` array:

```json
{
  "name": "rei-scaffold",
  "description": "Scaffold new Claude Code skills and agents for Rei, following established conventions from the rei codebase",
  "path": "skills/rei-scaffold",
  "files": ["SKILL.md"]
}
```

**Step 4: Validate**

```bash
# Confirm file exists and has content
wc -l skills/rei-scaffold/SKILL.md

# Confirm kit.json is valid JSON
python3 -c "import json; json.load(open('kit.json'))"
```

Expected: SKILL.md has ~300-500 lines. kit.json parses without error.


## Validation and Acceptance

1. **File structure**: `skills/rei-scaffold/SKILL.md` exists with YAML frontmatter containing
   `name`, `description`, and `allowed-tools`.

2. **Skill workflow completeness**: The SKILL.md contains:
   - A "When to Use" section with activation phrases.
   - An interactive workflow that asks skill-vs-agent.
   - A skill creation track that gathers name, description, tools, workflow, and generates
     a SKILL.md with all standard sections.
   - An agent creation track that gathers name, description, tools, model, domain, and
     generates an agent file with all standard sections.
   - A kit.json update step.
   - An output format section showing the summary.

3. **Convention adherence**: Generated templates within the skill match the format of existing
   rei-kit skills (`skills/rei-bootstrap-habit/SKILL.md`). Reference examples come from
   this repo, not the rei repo.

4. **kit.json validity**: `python3 -c "import json; json.load(open('kit.json'))"` succeeds.

5. **Manifest registration**: `kit.json` contains an entry with `"name": "rei-scaffold"`.


## Idempotence and Recovery

All steps are safe to repeat:
- `mkdir -p` is idempotent.
- Writing SKILL.md overwrites any previous version.
- Editing kit.json is idempotent if the entry already exists (check before adding).

If the skill needs to be removed: delete `skills/rei-scaffold/` and remove the entry from
`kit.json`.


## Interfaces and Dependencies

**Tools used by the generated skill (allowed-tools):**
- `AskUserQuestion` — Interactive prompts for metadata gathering.
- `Read` — Reading existing examples from the rei repo for reference.
- `Write` — Creating new SKILL.md or agent .md files.
- `Edit` — Updating kit.json.
- `Glob` — Finding existing skill/agent files for reference.
- `Bash` — Running validation commands.

**External dependencies (for domain context only):**
- The rei repo at `/Users/shinzui/Keikaku/bokuno/rei-project/rei` provides domain context
  (CLI docs, concepts, entity types) but is not required for structural reference.
- The `rei` CLI must be available on PATH for `rei <command> --help` lookups.

**No new types or interfaces are introduced.** The skill produces Markdown files following
existing rei-kit conventions.


---

## Revision Notes

### 2026-04-08: Reference strategy change

**What changed:** The skill's reference strategy was updated. Previously, the skill read
existing skills and agents from the root rei repo (`/Users/shinzui/Keikaku/bokuno/rei-project/rei`)
as structural reference examples. Now:

- **Structural reference** comes from **this repo (rei-kit)** — the existing skills in
  `skills/` are the source of truth for format, conventions, and patterns.
- **Domain context** comes from the **rei repo** and the **`rei` CLI help system** — used
  only to understand available commands, entity types, and concepts that generated
  skills/agents might need to reference.

**Why:** rei-kit has its own conventions and scope. Using the rei repo's development-focused
skills as structural templates led to output that was more complex than needed for end-user
kit artifacts. The rei CLI help system (`rei <command> --help`) provides a lightweight,
always-current way to look up command syntax without reading rei source code.

**Sections affected:** Purpose/Big Picture, Context and Orientation, Decision Log, Plan of
Work, Validation and Acceptance, Interfaces and Dependencies, Outcomes & Retrospective.

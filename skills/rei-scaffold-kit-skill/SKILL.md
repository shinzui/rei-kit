---
name: rei-scaffold-kit-skill
description: Scaffold new Claude Code skills and agents for Rei, following established conventions. Use when creating a new skill or agent definition for rei-kit.
allowed-tools: AskUserQuestion, Read, Write, Edit, Glob, Bash
---

# Rei Scaffold

This skill scaffolds new Claude Code skills and agents for the rei-kit repository. It
examines existing skills and agents in this repo as structural reference, uses the rei CLI
help system and rei repo for domain context, gathers requirements interactively, generates
files following established conventions, and registers them in `kit.json`.

## When to Use

Activate when the user says things like:
- "Create a new skill"
- "Scaffold an agent"
- "Add a skill for..."
- "I need a new agent that..."
- "Generate a skill definition"
- "Create a guide agent for..."
- "/rei-scaffold-kit-skill"

## Reference Sources

### Structural reference (this repo)

Existing skills and agents in rei-kit are the source of truth for format and conventions:

- **Skills** in `skills/*/SKILL.md` — e.g., `rei-bootstrap`, `rei-bootstrap-habit`,
  `rei-summarize-links`
- **Agents** in `agents/*.md` — (as they are added to the kit)

Always read 1-2 existing examples from this repo before generating, so the output matches
established conventions.

### Domain context (rei CLI and rei repo)

When the generated skill or agent needs to reference rei commands, entity types, or concepts:

- **`rei --help`** and **`rei <command> --help`** — authoritative, always-current command
  reference for syntax, flags, and subcommands
- **Rei repo** at `/Users/shinzui/Keikaku/bokuno/rei-project/rei`:
  - `docs/user/cli/` — CLI command documentation
  - `docs/user/concepts.md` — Key concepts and primitives
  - Use for understanding domain concepts, not for structural skill/agent format

## Workflow Overview

1. **Choose artifact type** — Skill or Agent
2. **Study references** — Read existing examples from this repo; check rei CLI for domain context
3. **Gather metadata** — Name, description, tools, domain focus
4. **Gather content** — Workflow phases (skills) or domain knowledge (agents)
5. **Generate the file** — Write to the correct location in rei-kit
6. **Register in kit.json** — Add the manifest entry
7. **Summary** — Show what was created and next steps

## Instructions for Claude

### Phase 1: Choose Artifact Type

If the user hasn't already specified, ask:

```
Question: "What would you like to scaffold?"
Header: "Type"
Options:
- Skill: A workflow that guides users through a task (e.g., bootstrapping a habit, summarizing links)
- Agent: A domain expert or guide that answers questions and provides advice (e.g., habit guide, intention expert)
```

If the user provided enough context in their request (e.g., "create a skill for..."), skip
this question and proceed directly.

### Phase 2: Study References

Before gathering details, read existing examples from **this repo** to calibrate your output.
Then check the rei CLI for domain context if the artifact will reference rei commands.

**Step 2a: Read structural references from rei-kit**

```bash
# List existing skills in this repo
ls skills/
```

Read 1-2 existing skill files to understand the format:
- `skills/rei-bootstrap-habit/SKILL.md` — good example of an interactive workflow skill
- `skills/rei-bootstrap/SKILL.md` — example of a larger multi-phase skill
- `skills/rei-summarize-links/SKILL.md` — example of a processing-oriented skill

For agents, check what exists:
```bash
ls agents/
```

Read any existing agent files. If none exist yet, use the agent template in Phase 5 as the
canonical format.

**Step 2b: Check rei CLI for domain context (if needed)**

If the skill or agent will reference rei commands or concepts, explore the CLI help:

```bash
# Top-level commands
rei --help

# Specific command help
rei <command> --help
rei <command> <subcommand> --help
```

For deeper domain context, consult the rei repo docs:
```bash
# CLI reference docs
ls /Users/shinzui/Keikaku/bokuno/rei-project/rei/docs/user/cli/

# Concepts
cat /Users/shinzui/Keikaku/bokuno/rei-project/rei/docs/user/concepts.md
```

Use the rei repo **only for domain understanding** — do not copy its skill/agent format.

### Phase 3: Gather Metadata

#### For Skills

Ask about the skill's identity:

```
Question: "What should this skill be named? (Use kebab-case, e.g., rei-review-weekly)"
Header: "Name"
Options:
- Let me type the name
```

**Naming conventions:**
- Prefix with `rei-` for Rei-specific skills
- Use action verbs: `rei-bootstrap-X`, `rei-review-X`, `rei-summarize-X`, `rei-update-X`
- Keep it short and descriptive

Then ask about purpose:

```
Question: "In one sentence, what does this skill do?"
Header: "Purpose"
Options:
- Let me describe it
```

Then ask about the tools needed:

```
Question: "What kind of work does this skill do?"
Header: "Tools"
Options:
- Interactive workflow (asks questions, runs CLI commands) — AskUserQuestion, Bash, Read
- File generation (creates or modifies files) — AskUserQuestion, Read, Write, Edit, Glob, Bash
- Read-only analysis (searches and reads code) — Read, Grep, Glob
- CLI orchestration (runs rei commands based on user input) — AskUserQuestion, Bash, Read
```

#### For Agents

Ask about the agent's identity:

```
Question: "What should this agent be named? (Use kebab-case, e.g., rei-cycle-expert)"
Header: "Name"
Options:
- Let me type the name
```

**Naming conventions:**
- Prefix with `rei-`
- Suffix with `-expert` for domain knowledge agents
- Suffix with `-guide` for conversational coaching agents
- Examples: `rei-cycle-expert`, `rei-habit-guide`, `rei-fzf-expert`

Then ask about purpose:

```
Question: "In one sentence, what is this agent an expert on?"
Header: "Expertise"
Options:
- Let me describe it
```

Then determine the agent type:

```
Question: "What kind of agent is this?"
Header: "Style"
Options:
- Domain expert: Answers technical questions about a specific area, includes quick-reference tables and file locations (Recommended)
- Guide: Helps users accomplish tasks conversationally, includes examples and troubleshooting
- Pattern expert: Specializes in a cross-cutting concern (e.g., testing, code style, projections)
```

Then ask about tools:

```
Question: "What tools should this agent have access to?"
Header: "Tools"
Options:
- Read-only (Read, Grep, Glob) — for research and analysis (Recommended)
- Read + Bash (Read, Bash) — can also run commands to inspect state
- Read + Bash + Grep + Glob — full read-only exploration
```

### Phase 4: Gather Content

#### For Skills

Ask about the workflow:

```
Question: "Describe the main steps of this skill's workflow. What should happen from start to finish?"
Header: "Workflow"
Options:
- Let me describe the workflow
```

Follow up to understand:
- How many phases are there?
- What questions should be asked at each phase?
- What commands or file operations happen?
- What does success look like?

If the skill involves `rei` CLI commands, ask:

```
Question: "Which rei commands does this skill use? (e.g., rei habit create, rei intention list)"
Header: "Commands"
Options:
- Let me list them
- I'm not sure — help me figure it out
```

If the user is unsure, explore the rei CLI help:
```bash
# Browse available top-level commands
rei --help

# Get details on a specific command
rei <command> --help
```

For more detailed CLI documentation:
```bash
ls /Users/shinzui/Keikaku/bokuno/rei-project/rei/docs/user/cli/
```

#### For Agents

Ask about the domain knowledge:

```
Question: "What key concepts, rules, or patterns should this agent know about?"
Header: "Knowledge"
Options:
- Let me describe the domain
- Point me to relevant files/docs to read
```

If the user points to files, read them and extract:
- Core concepts and terminology
- Key business rules or constraints
- Important file locations
- Common patterns and idioms
- Cross-module interactions

If the agent relates to a rei module, check for existing documentation and CLI help:
```bash
# Module documentation in the rei repo (for domain context)
ls /Users/shinzui/Keikaku/bokuno/rei-project/rei/docs/dev/modules/

# CLI help for the relevant command
rei <command> --help
```

### Phase 5: Generate the File

#### Generating a Skill

Create `skills/<name>/SKILL.md` following this structure:

```markdown
---
name: <name>
description: <description from Phase 3>
allowed-tools: <tools from Phase 3>
---

# <Title (human-readable form of the name)>

<One paragraph describing what this skill does and when to use it.>

## When to Use

Activate when the user says things like:
- "<phrase 1>"
- "<phrase 2>"
- "<phrase 3>"
- "<phrase 4>"

## Key Concepts

<If the skill involves domain concepts, define them here. Otherwise omit this section.>

### <Concept 1>

<Brief explanation.>

## Workflow Overview

1. **<Phase 1 name>** - <what happens>
2. **<Phase 2 name>** - <what happens>
...

## Instructions for Claude

### Phase 1: <Name>

<Detailed instructions including AskUserQuestion prompts, CLI commands, and decision logic.>

Use AskUserQuestion for interactive steps:

\```
Question: "<question text>"
Header: "<short label>"
Options:
- <option 1>
- <option 2>
\```

<CLI commands to run:>

\```bash
rei <command> --actor claude-code
\```

### Phase 2: <Name>

<Continue for each phase.>

## Output Format

After completing the workflow, provide a summary:

\```
## <Result Title>

- **<Field 1>**: [value]
- **<Field 2>**: [value]

### Next Steps

1. <step 1>
2. <step 2>
\```

## Important Notes

- Always include `--actor claude-code` when creating entities via rei CLI
- <Other important notes specific to this skill>
```

**Important conventions to follow:**
- Use `AskUserQuestion` blocks exactly as shown (Question, Header, Options format)
- Include `--actor claude-code` on all `rei` entity-creation commands
- Show CLI commands in fenced bash code blocks
- Keep the workflow actionable — tell Claude exactly what to do, don't just describe
- Add "(Recommended)" suffix to the best default option in AskUserQuestion prompts
- If the user provided enough info upfront, note that questions can be skipped

Write the file:
```bash
# Working directory: rei-kit repo root
```

Create `skills/<name>/SKILL.md` using the Write tool.

#### Generating an Agent

Create `agents/<name>.md` following this structure:

```markdown
---
name: <name>
description: <description from Phase 3, include "Use for..." guidance>
tools: <tools from Phase 3>
model: sonnet
---

# <Title (human-readable form of the name)>

You are an expert on <domain>. <Brief description of role and scope.>

<If there are reference docs:>
**Full documentation:** `<path>` - Always read this first for complete domain details.

## Quick Reference

### Core Concepts

- **<Concept 1>**: <Brief description>
- **<Concept 2>**: <Brief description>

### Key Business Rules

1. <Rule 1>
2. <Rule 2>

<For domain experts, add:>

### Subscriptions

| Subscription | Category | Purpose |
|--------------|----------|---------|
| ... | ... | ... |

### File Locations

\```
<path>/
├── <dir>/          # <purpose>
├── <dir>/          # <purpose>
└── <file>          # <purpose>
\```

## Key Patterns

1. **<Pattern 1>**: <Description>
2. **<Pattern 2>**: <Description>

<For guides, replace Key Patterns with:>

## Conversation Flow

When helping users:

1. **<Step 1>**: <what to do>
2. **<Step 2>**: <what to do>

## Example Interactions

**User**: "<example question>"

**Response**: <example answer>

## Output Format

When helping with <domain>:

\```
## <Domain> Analysis: <Topic>

### Current Behavior
(Read the docs and relevant code first)

### Recommendation
Clear recommendation for the change.

### Implementation
Code changes with context.

### Testing
- Tests to add/modify
\```

## Before Answering

1. Read <primary documentation path> for full domain context
2. Check relevant source files in <source path>
3. Reference existing patterns in the codebase
```

**Important conventions to follow:**
- Always include `model: sonnet` in frontmatter
- Agents are read-only — never include Write or Edit in tools
- Domain experts should point to full docs rather than duplicating them
- Include file location trees so the agent knows where to look
- Guides should include example interactions
- Use tables for quick-reference data (states, subscriptions, rules)

Write the file using the Write tool.

### Phase 6: Register in kit.json

Read the current `kit.json`:

```bash
# Read current manifest
```

Use the Read tool to read `kit.json`.

**For skills**, add an entry to the `skills` array:

```json
{
  "name": "<name>",
  "description": "<description>",
  "path": "skills/<name>",
  "files": ["SKILL.md"]
}
```

**For agents**, add an entry to the `agents` array:

```json
{
  "name": "<name>",
  "description": "<description>",
  "path": "agents",
  "files": ["<name>.md"]
}
```

Use the Edit tool to insert the new entry into the appropriate array in `kit.json`. Ensure
the resulting JSON is valid.

### Phase 7: Summary

After creating the file and updating kit.json, show a summary.

**For skills:**

```
## Skill Scaffolded: <name>

### Files Created
- `skills/<name>/SKILL.md`

### Files Modified
- `kit.json` (added skill entry)

### Skill Summary
- **Name**: <name>
- **Description**: <description>
- **Allowed Tools**: <tools>
- **Workflow Phases**: <count> phases

### Next Steps
1. Review the generated SKILL.md and refine the workflow details
2. Test the skill by invoking `/<name>`
3. Commit when satisfied: `git add skills/<name>/ kit.json`
```

**For agents:**

```
## Agent Scaffolded: <name>

### Files Created
- `agents/<name>.md`

### Files Modified
- `kit.json` (added agent entry)

### Agent Summary
- **Name**: <name>
- **Description**: <description>
- **Tools**: <tools>
- **Model**: sonnet
- **Type**: <expert / guide / pattern expert>

### Next Steps
1. Review the generated agent file and refine the domain knowledge
2. Add more quick-reference data as you learn the domain
3. Commit when satisfied: `git add agents/<name>.md kit.json`
```

## Important Notes

- **Reference before generating**: Always read 1-2 existing examples from this repo (rei-kit)
  before writing. This ensures the output matches established conventions. Use the rei CLI
  help (`rei <command> --help`) for domain context, not for structural format.
- **Don't over-specify**: Generate a solid skeleton that the user can refine. Don't invent
  domain details you're unsure about — mark those spots with `<TODO: ...>` placeholders.
- **Respect the separation**: All generated files go in rei-kit, never in the rei repo.
- **Naming matters**: Follow the `rei-` prefix convention. Use `-expert` or `-guide` suffix
  for agents.
- **kit.json must stay valid**: After editing, the JSON must parse correctly. Read before
  editing to understand the current structure.
- **Skill content should be actionable**: Skills tell Claude exactly what to do — specific
  questions to ask, specific commands to run. Don't write vague instructions.
- **Agent content should be reference-oriented**: Agents provide quick-lookup knowledge.
  Point to full docs, don't duplicate them.
- **If the user provides all details upfront**, skip the interactive questions and generate
  directly. Mention which questions were skipped in the summary.

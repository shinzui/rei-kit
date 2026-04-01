---
name: rei-bootstrap
description: Interactively bootstrap intentions, habits, and reflections for Rei personal coaching. Use when the user wants to create a new life goal, project, or intention hierarchy. Guides through structured questions to define root intentions, break them into actionable children, establish supporting habits, set up focus areas with cycles, and configure reflection practices.
allowed-tools: AskUserQuestion, Bash, Read
---

# Rei Bootstrap

This skill helps users create a well-structured intention hierarchy with supporting habits, focus areas, cycles, and reflection practices through guided questioning.

## When to Use

Activate when the user says things like:
- "Help me bootstrap a new intention"
- "I want to set up a new goal/project"
- "Let's create an intention with habits"
- "Bootstrap my intentions"

## Workflow Overview

1. **Understand the Goal** - Ask about the high-level intention
2. **Define the Root Intention** - Clarify title, horizon, and success criteria
3. **Break Down into Children** - Identify 2-5 sub-intentions
4. **Set Up Focus Areas** - Import or create focus areas for daily time allocation
5. **Establish Habits** - Define supporting daily/weekly practices with focus alignment
6. **Start a Cycle** (Optional) - Begin tracking daily focus selections
7. **Set Up Reflections** - Configure review cadence
8. **Agent Guidance** (Optional) - Configure how the agent should coach this intention

## Instructions for Claude

### Phase 1: Discovery

Use AskUserQuestion to understand what the user wants to achieve:

```
Question: "What goal or project would you like to work on?"
Header: "Goal"
Options:
- A life area (health, career, relationships, learning)
- A specific project with a deadline
- A skill or habit I want to develop
- Something else
```

Follow up to get specifics about their intention.

### Phase 2: Root Intention

Ask about the planning horizon:

```
Question: "What timeframe are you thinking for this intention?"
Header: "Horizon"
Options:
- Short-term (days to weeks)
- Medium-term (1-3 months)
- Long-term (quarters to years)
- Ongoing/no specific end
```

Then ask about the starting status:

```
Question: "Should this intention be active now or deferred for later?"
Header: "Status"
Options:
- Active now (start working on it immediately)
- Future (plan it now, activate later when ready)
```

Then create the root intention:
```bash
# For Active intention (default)
rei intention create "TITLE" --horizon HORIZON --actor claude-code

# For Future intention (use --future flag)
rei intention create "TITLE" --horizon HORIZON --future --actor claude-code
```

Future intentions are useful for planning projects you're not ready to start yet. They:
- Won't appear in daily views or reviews until activated
- Allow structural setup (children, habits) but block operational actions
- Can be activated later with `rei intention activate <ID>`

#### Context Association

After creating the root intention, ask about context:

**First, check for existing contexts:**
```bash
rei intention contexts
```

```
Question: "Would you like to associate this intention with a context?"
Header: "Context"
Options:
- [If contexts exist, list them as options, e.g., "work", "personal"]
- Create a new context
- Skip (no context)
```

If associating with an existing context or creating a new one:
```bash
rei intention set-context <INTENTION_ID> "CONTEXT_NAME"
```

**Note**: Context can only be set on root intentions. Child intentions inherit context from their root. Common contexts include:
- `work` - Professional responsibilities
- `personal` - Personal development and life goals
- `family` - Family-related goals
- `health` - Health and fitness
- `learning` - Education and skill development

### Phase 3: Child Intentions

Ask the user to break down their goal:

```
Question: "What are the key areas or milestones needed to achieve this?"
Header: "Breakdown"
Options:
- Let me describe 2-3 key areas
- Help me brainstorm the breakdown
- I already have a clear structure in mind
```

For each child intention identified, create it with the parent:
```bash
rei intention create "CHILD_TITLE" --parent PARENT_ID --horizon HORIZON --actor claude-code
```

### Phase 3.25: Category Setup (Optional)

Categories classify the *type* of activities (meeting, coding, reading), distinct from context (life domain) and focus (daily activity nature). They enable filtering and analysis of actions and notes.

**First, check for existing categories:**
```bash
rei category tree
```

**If categories already exist**, acknowledge and offer to use or extend them:
```
Question: "Would you like to use existing categories or create new ones for this intention?"
Header: "Categories"
Options:
- Use existing categories (I'll assign them as needed)
- Create activity-specific categories for this intention
- Skip categories for now
```

**If no categories exist**, offer to set them up:
```
Question: "Would you like to establish activity categories for this intention?"
Header: "Categories"
Options:
- Yes, help me create categories
- Skip for now (can add later with `rei category create`)
```

If creating categories, guide the user:
```bash
# Create a parent category (optional)
rei category create "learning" --description "Learning activities" --actor claude-code

# Create child categories
rei category create "reading" --parent learning --description "Reading sessions" --actor claude-code
rei category create "coding" --parent learning --description "Hands-on coding practice" --actor claude-code
```

**Common category patterns:**
| Intention Type | Suggested Categories |
|---------------|---------------------|
| Learning goal | reading, coding, video-courses, mentoring |
| Work project | meeting, coding, review, planning |
| Health goal | exercise, meal-prep, tracking |
| Creative project | research, drafting, editing, publishing |

**Optionally link categories to focus areas:**
```bash
rei category link-focus "learning/reading" <READ_FOCUS_ID>
rei category link-focus "learning/coding" <STUDY_FOCUS_ID>
```

After creating categories, optionally assign the root intention to a category:
```bash
rei intention set-category <INTENTION_ID> "learning"
```

### Phase 3.5: Support Relationships (Optional)

After creating child intentions, ask if any intentions support other existing goals:

```
Question: "Do any of these intentions support other goals you're working on?"
Header: "Supports"
Options:
- Yes, let me set up some support relationships
- Skip for now (I can add supports later)
```

Support relationships are informational links (not dependencies) that help track how intentions relate across different hierarchies. For example:
- "Learn Haskell" might **support** "Build personal website" (learning enables building)
- "Exercise regularly" might **support** "Improve focus at work"

If the user wants to set up supports:

```bash
# Check what other intentions exist
rei intention list --all

# Add a support relationship
rei support add --from <SUPPORTING_ID> --to <SUPPORTED_ID> --note "DESCRIPTION" --actor claude-code
```

**Note**: Supports can be declared on both Active and Future intentions. Use the interactive FZF picker (`rei support add`) which lets you press `ctrl-f` to toggle between active-only and all intentions.

### Phase 4: Focus Areas

Focus areas represent categories of activity for daily time allocation. They describe the *nature* of activities (not goals).

**IMPORTANT: Always check for existing focus areas first and skip this phase if they exist.**

```bash
rei focus list
```

**If focus areas already exist**, acknowledge them and proceed to Phase 5 (Habits):
```
I see you already have focus areas set up:
  ⌘ Connect, ❥ Read, ❖ Study, ✿ Play, ✽ Create

We'll use these for habit alignment. Moving on to habits...
```

**Only if no focus areas exist**, ask if the user wants to set them up:

```
Question: "Would you like to set up focus areas for daily time tracking?"
Header: "Focus"
Options:
- Import Anatomy of Equanimity template (Connect, Read, Study, Play, Create)
- Create custom focus areas
- Skip focus areas for now
```

If importing the template:
```bash
rei focus import anatomy-of-equanimity
```

If creating custom focus areas, ask about their focus categories and create them:
```bash
rei focus create \
  --name "NAME" \
  --symbol "SYMBOL" \
  --description "DESCRIPTION" \
  --actor claude-code
```

**Note**: Focus describes the nature of an activity, not the goal. An intention like "Learn Haskell" spans multiple focuses:
- **Study** day: Read a chapter on monads
- **Create** day: Build a project applying what you learned
- **Play** day: Experiment with different approaches

### Phase 5: Habits

Ask about supporting practices:

```
Question: "What regular practices would support this intention?"
Header: "Habits"
Options:
- Daily practices (morning routine, evening review)
- Weekly practices (planning, review sessions)
- Both daily and weekly
- Skip habits for now
```

Create habits with appropriate schedules, linking them to the root intention:
```bash
# Daily habit
rei habit create --name "NAME" --purpose "PURPOSE" --daily --intention <ROOT_INTENTION_ID> --actor claude-code

# Weekly on a single day
rei habit create --name "NAME" --purpose "PURPOSE" --weekly monday --intention <ROOT_INTENTION_ID> --actor claude-code

# Weekly on multiple days (e.g., Mon, Wed, Fri)
rei habit create --name "NAME" --purpose "PURPOSE" --weekly-on mon,wed,fri --intention <ROOT_INTENTION_ID> --actor claude-code
```

To change a habit's schedule after creation, use the `update-schedule` command:
```bash
rei habit update-schedule <HABIT_ID> --daily
rei habit update-schedule <HABIT_ID> --weekly friday
rei habit update-schedule <HABIT_ID> --weekly-on tue,thu
```

**Note**: The `--intention` flag associates the habit with the intention it supports. This enables tracking which habits contribute to which goals.

**If focus areas were set up**, ask which focus each habit aligns with:

```
Question: "Which focus area does '[HABIT_NAME]' align with?"
Header: "Habit Focus"
Options:
- ⌘ Connect (social connection, staying informed)
- ❥ Read (absorbing information)
- ❖ Study (deep learning, skill building)
- ✿ Play (recreation, enjoyment)
- ✽ Create (making new things)
```

**Note**: Focus assignment for habits is managed through the cycle system, not directly on habits. Habits have a consistent nature, so focus alignment is conceptual. Examples:
- "Morning reading" → Read
- "Code practice" → Study
- "Music practice" → Play or Create
- "Journal writing" → Create

### Phase 6: Cycles (Optional)

Cycles are bounded time periods for tracking daily focus selections. If focus areas were set up (or already exist), offer to start a cycle.

**IMPORTANT: Always check for an active cycle first and skip this phase if one exists.**

```bash
rei cycle status
```

**If a cycle is already active**, acknowledge it and proceed to Phase 7 (Reflections):
```
You already have an active cycle:
  "January Focus" - Day 3/10 (7 days remaining)
  Today's focus: ❖ Study

We'll continue with this cycle. Moving on to reflections...
```

**Only if no active cycle exists** and focus areas are available, offer to start one:

```
Question: "Would you like to start a focus cycle to track daily focus selections?"
Header: "Cycle"
Options:
- Yes, start a 10-day cycle
- Yes, start a 7-day cycle (weekly)
- Yes, with a custom length
- Skip cycles for now
```

If starting a cycle:
```bash
# Start a cycle with optional name
rei cycle start --length 10 --name "CYCLE_NAME" --actor claude-code

# Start a 7-day cycle
rei cycle start --length 7 --actor claude-code
```

Explain the daily workflow:
```
Each day of the cycle, you'll select your focus:
  rei cycle select-focus

This creates a rhythm of intentional daily focus. You can:
  - Check progress: rei cycle status
  - Complete the cycle: rei cycle complete
  - Start a new cycle after completion
```

**Why cycles matter**: Cycles create accountability and rhythm. Instead of vague intentions to "balance" focus areas, you make one conscious choice each day: "Today, I focus on Study."

### Phase 7: Reflection Setup

Ask about reflection preferences:

```
Question: "How would you like to reflect on progress?"
Header: "Reflection"
Options:
- Daily quick check-ins
- Weekly deeper reviews
- Both daily and weekly
- I'll set up reflections later
```

Guide the user on using:
```bash
rei reflect today --prompt "PROMPT" --actor claude-code
rei reflect week --prompt "PROMPT" --actor claude-code
```

### Phase 8: Agent Guidance (Optional)

After completing the core setup, offer to configure agent guidance for the root intention. This guidance helps the agent provide better coaching during future reviews and check-ins.

```
Question: "Would you like to set up agent guidance for this intention?"
Header: "Guidance"
Options:
- Yes, help me configure how the agent should coach me
- Skip for now (I can add it later with `rei intention guidance`)
```

If the user wants guidance, ask clarifying questions to understand:

```
Question: "What coaching approach would work best for you with this intention?"
Header: "Style"
Options:
- Encouraging and supportive (celebrate wins, gentle on setbacks)
- Direct and challenging (push me, call out excuses)
- Analytical and structured (focus on metrics, patterns)
- Flexible and exploratory (adapt based on what's working)
```

Then ask about specific considerations:

```
Question: "Are there any constraints or things to avoid when coaching this intention?"
Header: "Constraints"
Options:
- Time constraints (limited hours available)
- Energy/health considerations
- Budget or resource limits
- Let me describe specific constraints
```

#### Generating Guidance Content

Based on the entire conversation, synthesize guidance that includes:

1. **Coaching Style** - How the agent should approach reviews (from user's preference + context)
2. **Success Indicators** - What progress looks like based on the child intentions and habits
3. **Review Focus Areas** - Key questions to ask during check-ins
4. **Action Patterns** - Preferred approaches based on habits and breakdown structure
5. **Constraints** - Boundaries and limitations to respect
6. **Motivation Context** - Why this intention matters (captured from Phase 1 discovery)

Create the guidance using the conversation context:
```bash
rei intention guidance <INTENTION_ID> --content "$(cat <<'EOF'
## Coaching Style
[Based on user's preference - e.g., "Be encouraging and celebrate small wins.
This is a long-term health goal, so focus on consistency over perfection."]

## Success Indicators
[Derived from child intentions - e.g., "Progress looks like:
- Regular workout sessions (3+ per week)
- Improved energy levels reported in reflections
- Completion of quarterly milestones"]

## Review Focus Areas
[Questions for check-ins - e.g., "During reviews, explore:
- What worked well this period?
- What got in the way?
- Is the current pace sustainable?
- Any habits need adjustment?"]

## Action Patterns
[From habits and breakdown - e.g., "User prefers morning workouts.
Weekly planning happens on Sundays. Focus on building one habit at a time."]

## Constraints
[From user input - e.g., "Limited to 30-minute sessions due to schedule.
Avoid high-impact exercises due to knee issues."]

## Motivation Context
[From discovery - e.g., "User wants to improve health to have more energy
for family activities and set a good example for kids."]
EOF
)"
```

**Important**: The guidance should be specific enough to help the agent provide personalized coaching, but flexible enough to adapt as the intention evolves. Focus on the "why" and "how" rather than rigid rules.

## Output Format

After bootstrapping, provide a summary:

```
## Bootstrap Complete

### Root Intention
- **Title**: [title]
- **ID**: [id]
- **Horizon**: [horizon]
- **Status**: [Active/Future]
- **Context**: [context name, or "None" if skipped]

### Child Intentions
1. [child1] (ID: [id])
2. [child2] (ID: [id])
...

### Support Relationships
- [If configured]: [intention1] → supports → [intention2] (note: "[note]")
- [If skipped]: Not configured (add later with `rei support add`)

### Categories
- [If already existed]: Using existing categories
- [If created]: [parent] (focus: [focus])
  - [parent]/[child1] (focus: [focus])
  - [parent]/[child2] (focus: [focus])
- [If skipped]: Not configured (add with `rei category create`)

### Focus Areas
- [If already existed]: Using existing focus areas (⌘ Connect, ❥ Read, ❖ Study, ✿ Play, ✽ Create)
- [If imported]: Anatomy of Equanimity (⌘ Connect, ❥ Read, ❖ Study, ✿ Play, ✽ Create)
- [If custom]: [list of custom focus areas]
- [If skipped]: Not configured (add later with `rei focus import anatomy-of-equanimity`)

### Habits Created
- [habit1] (daily) → linked to [root intention], focus: [focus]
- [habit2] (weekly on Monday) → linked to [root intention], focus: [focus]
- [habit3] (weekly on Mon, Wed, Fri) → linked to [root intention], focus: [focus]

### Active Cycle
- [If already existed]: Using existing cycle "[cycle name]" - [days remaining] days remaining
- [If started]: [cycle name] - [length] days starting [date]
- [If skipped]: Not started (start later with `rei cycle start --length 10`)

### Reflection Cadence
- Daily: [description]
- Weekly: [description]

### Agent Guidance
- [If configured]: Guidance set with [coaching style] approach
- [If skipped]: Not configured (add later with `rei intention guidance <ID>`)

### Next Steps
1. [If Future status]: When ready to start, activate: `rei intention activate <ID>`
2. [If cycle started]: Select today's focus: `rei cycle select-focus`
3. Record your first action: `rei action record --intention ID`
4. Start your first reflection: `rei reflect today`
5. [If guidance set]: Review and refine guidance: `rei intention guidance <ID>`
```

## Important Notes

- Always confirm with the user before creating entities
- **Always include `--actor claude-code`** on all entity creation commands to track that Claude created them
- Use horizons that match the scope (e.g., `3m` for quarterly goals, `1y` for annual)
- Keep habit names concise and action-oriented
- **Always link habits to their supporting intention** using `--intention <ID>` when creating habits
- Habits can also be linked/unlinked later with `rei habit link-intention` and `rei habit unlink-intention`
- Suggest realistic reflection cadences based on the user's goals
- **Always ask about context association** for root intentions - check existing contexts with `rei intention contexts`
- **Skip focus area setup if focus areas already exist** - check with `rei focus list` first
- **Skip cycle setup if a cycle is already active** - check with `rei cycle status` first
- Focus areas describe the *nature* of activities, not goals - an intention spans multiple focuses
- Only one cycle can be active at a time - reuse an existing active cycle
- Habits can be assigned to focus areas since they have consistent nature
- Recommend 7-10 day cycles for beginners, longer for experienced users
- **Future intentions** are useful for planning ahead without cluttering daily views
- Deferring a parent intention automatically defers all active children (cascade behavior)
- To list Future intentions: `rei intention list --future`
- To list both Active and Future: `rei intention list --all`
- **Support relationships** connect intentions across hierarchies - use when one intention helps another without being a child
- Supports can be declared on both Active and Future intentions
- View supports with `rei intention show --full` or `rei support show`
- **Categories** classify activity types (meeting, coding, reading) - distinct from context (life domain) and focus (daily nature)
- Check existing categories with `rei category tree` before creating new ones
- Categories can be linked to focus areas for alignment analysis
- Assign categories to intentions with `rei intention set-category <ID> SLUG`

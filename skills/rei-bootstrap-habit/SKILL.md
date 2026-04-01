---
name: rei-bootstrap-habit
description: Interactively bootstrap a new habit for Rei personal coaching. Use when the user wants to create a new daily or weekly practice. Guides through defining the habit name, purpose, schedule, linking to an intention, and optionally assigning a focus area.
allowed-tools: AskUserQuestion, Bash, Read
---

# Rei Bootstrap Habit

This skill helps users create a well-structured habit through guided questioning, ensuring all relevant details are captured.

## When to Use

Activate when the user says things like:
- "Help me create a new habit"
- "I want to start a new daily practice"
- "Bootstrap a habit"
- "Add a habit for..."
- "Create a weekly habit"
- "I want to stop doing X" (break habit)
- "Help me avoid X" (break habit)

## Key Concepts

### Polarity: Build vs Break

Habits have two polarities:
- **Build habits**: Cultivate positive behaviors (e.g., "Morning meditation", "Daily reading")
- **Break habits**: Avoid unwanted behaviors (e.g., "No social media before noon", "Quit snacking")

### Cadence: When to Track

There are two main cadence types:

**Recurring (time-triggered)**:
- Fixed schedules: Daily, Weekly, Monthly, Quarterly, Yearly, Every N days
- Free schedules: N times per week/month/quarter/year (flexibility in when)

**Cue-based (event-triggered)**:
- Triggered by events/situations, not calendar
- Examples: "After meals", "When feeling stressed", "When encountering good writing"
- Optional target frequency (e.g., "3 times per week")

## Workflow Overview

1. **Understand the Practice** - What habit and why
2. **Determine Polarity** - Build or Break
3. **Define Name and Purpose** - Clarify concise name and motivation
4. **Set the Cadence** - Recurring schedule or cue-based
5. **Link to Intention** - Optionally connect to a supporting intention
6. **Assign Focus Area** - If focus areas exist, align the habit
7. **Set Action Template** - Optional default description for actions
8. **Set Context** - If unlinked, optionally categorize the habit

## Instructions for Claude

### Phase 1: Discovery

Use AskUserQuestion to understand what practice the user wants to establish:

```
Question: "What habit or practice would you like to build?"
Header: "Practice"
Options:
- A morning routine (journaling, exercise, meditation)
- An evening routine (reflection, planning, wind-down)
- A learning practice (reading, studying, skill-building)
- Something I want to stop or avoid (break habit)
```

Follow up to get specifics about their habit.

### Phase 2: Polarity

Based on discovery, clarify polarity if needed:

```
Question: "Is this a behavior you want to cultivate or avoid?"
Header: "Type"
Options:
- Build: I want to do this regularly (Recommended)
- Break: I want to stop or avoid this behavior
```

**Note**: For Break habits, the workflow differs:
- Instead of "completed", we track abstinence
- Recording "occurred" means a relapse happened
- Use `rei habit start-abstinence` after creation to begin tracking

### Phase 3: Name and Purpose

Ask about the habit details:

```
Question: "What should this habit be called? (Keep it short and action-oriented)"
Header: "Name"
Options:
- Let me type the name
```

For **Build habits**, suggest active names: "Morning meditation", "Daily reading"
For **Break habits**, suggest avoidance names: "No social media before noon", "Skip late-night snacking"

Then ask about purpose:

```
Question: "What's the purpose of this habit? (Why does it matter to you?)"
Header: "Purpose"
Options:
- Let me describe the purpose
```

### Phase 4: Cadence

First, determine the cadence type:

```
Question: "How should this habit be triggered?"
Header: "Cadence"
Options:
- Time-based: On a regular schedule (daily, weekly, etc.) (Recommended)
- Cue-based: Triggered by events or situations
```

#### Time-based Schedules

If time-based, ask about frequency:

```
Question: "How often should you do this habit?"
Header: "Schedule"
Options:
- Daily (every day)
- Weekly (one or more specific days)
- Monthly (same day each month)
- Flexible (N times per week/month, any days)
```

**Daily**: Use `--daily`

**Weekly schedules**:
- Single day: Ask which day, use `--weekly DAY`
- Multiple days: Ask which days, use `--weekly-on DAY1,DAY2,...`

```
Question: "Which day(s) of the week?"
Header: "Days"
Options:
- Every Monday
- Mon, Wed, Fri (common pattern)
- Weekdays only (mon,tue,wed,thu,fri)
- Let me specify the days
```

**Monthly**:
```
Question: "Which day of the month? (1-31)"
Header: "Day"
Options:
- 1st of the month
- 15th (mid-month)
- Last weekday (use 28 for safety)
- Let me specify
```
Use `--monthly DAY`

**Quarterly**: Use `--quarterly`

**Yearly**:
```
Question: "Which date of the year?"
Header: "Date"
Options:
- Let me specify (month and day)
```
Use `--yearly-month MONTH --yearly-day DAY`

**Every N days** (fixed interval):
```
Question: "How many days between each occurrence?"
Header: "Interval"
Options:
- Every 2 days
- Every 3 days
- Let me specify
```
Use `--every N`

**Flexible schedules** (freedom in when):

```
Question: "How many times per period do you want to aim for?"
Header: "Target"
Options:
- N times per week (use --free-weekly N)
- N times per month (use --free-monthly N)
- N times per quarter (use --free-quarterly N)
- N times per year (use --free-yearly N)
```

#### Cue-based Habits

If cue-based:

```
Question: "What triggers this habit? (Describe the cue or situation)"
Header: "Cue"
Options:
- Let me describe the trigger
```

Examples: "After meals", "When feeling anxious", "When I see interesting code"

Optionally ask about target frequency:

```
Question: "Do you want to set a target frequency for this cue-based habit?"
Header: "Target"
Options:
- Yes, set a target (e.g., 3 times per week)
- No target, just track when it happens
```

If yes:
```
Question: "What's your target? (format: N/period, e.g., 3/week, 10/month)"
Header: "Frequency"
Options:
- 3/week
- 5/week
- 10/month
- Let me specify
```

Use `--cue "DESCRIPTION" --target "N/period"`

### Phase 5: Link to Intention (Recommended)

**First, check for existing intentions:**
```bash
rei intention list
```

Ask about linking:

```
Question: "Would you like to link this habit to an intention it supports?"
Header: "Link"
Options:
- Yes, let me pick from my intentions
- No, this is a standalone habit
```

If linking, use FZF or ask which intention:
```bash
rei habit link-intention <HABIT_ID> <INTENTION_ID>
```

**Note**: Linking habits to intentions enables:
- Automatic context inheritance
- Tracking which habits support which goals
- Recording actions that count toward the intention

### Phase 6: Focus Area (If Available)

**First, check for existing focus areas:**
```bash
rei focus list
```

**If focus areas exist**, ask about alignment:

```
Question: "Which focus area does this habit align with?"
Header: "Focus"
Options:
- [List existing focus areas, e.g., "Connect", "Read", "Study", "Play", "Create"]
- Skip (no focus alignment)
```

**Note**: Habits have a consistent nature, so focus assignment makes sense:
- "Morning reading" -> Read
- "Code practice" -> Study
- "Music practice" -> Play or Create
- "Journal writing" -> Create

**Skip this phase if no focus areas exist.**

### Phase 6.5: Category for Actions (Optional)

When you record actions for this habit, you can pre-assign a category so actions are automatically classified.

**First, check for existing categories:**
```bash
rei category list
```

**If categories exist**, ask about action categorization:
```
Question: "When you record actions for this habit, what category should they have?"
Header: "Category"
Options:
- [List relevant categories, e.g., "learning/coding", "meeting/1-on-1"]
- Create a new category
- No default category (I'll classify each action)
```

**Note**: This doesn't assign a category to the habit itself (habits don't have categories), but establishes the default category for actions recorded via `rei habit record-action`.

If creating a new category:
```bash
rei category create "CATEGORY_NAME" --actor claude-code
```

Store the category slug to mention in the output format for use with `--category` when recording actions.

**Skip this phase if no categories exist.**

### Phase 7: Action Template (Optional)

Ask if they want a default description:

```
Question: "Would you like to set a default action description?"
Header: "Template"
Options:
- Yes, let me define a template
- No, I'll describe each action individually
```

If yes, ask for the template:
```
Question: "What should the default action description be?"
Header: "Template"
Options:
- Let me type the template
```

The action template is used when recording actions with `rei habit record-action`.

### Phase 8: Context (Unlinked Habits Only)

**Only if the habit is NOT linked to an intention**, offer to set context:

```bash
rei intention contexts
```

```
Question: "Would you like to categorize this habit with a context?"
Header: "Context"
Options:
- [List existing contexts, e.g., "work", "personal"]
- Create a new context
- Skip (no context)
```

If setting context:
```bash
rei habit set-context <HABIT_ID> "CONTEXT"
```

**Skip this phase if:**
- The habit is linked to an intention (context is inherited)
- No contexts are in use

## Creating the Habit

Use the appropriate command based on gathered information:

### Build Habits (default)

**Daily with intention:**
```bash
rei habit create \
  --name "NAME" \
  --purpose "PURPOSE" \
  --daily \
  --intention <INTENTION_ID> \
  --actor claude-code
```

**Weekly on one day:**
```bash
rei habit create \
  --name "NAME" \
  --purpose "PURPOSE" \
  --weekly monday \
  --actor claude-code
```

**Weekly on multiple days:**
```bash
rei habit create \
  --name "NAME" \
  --purpose "PURPOSE" \
  --weekly-on mon,wed,fri \
  --actor claude-code
```

**Monthly:**
```bash
rei habit create \
  --name "NAME" \
  --purpose "PURPOSE" \
  --monthly 15 \
  --actor claude-code
```

**Quarterly:**
```bash
rei habit create \
  --name "NAME" \
  --purpose "PURPOSE" \
  --quarterly \
  --actor claude-code
```

**Yearly:**
```bash
rei habit create \
  --name "NAME" \
  --purpose "PURPOSE" \
  --yearly-month 6 --yearly-day 15 \
  --actor claude-code
```

**Every N days:**
```bash
rei habit create \
  --name "NAME" \
  --purpose "PURPOSE" \
  --every 3 \
  --actor claude-code
```

**Flexible schedule:**
```bash
rei habit create \
  --name "NAME" \
  --purpose "PURPOSE" \
  --free-weekly 3 \
  --actor claude-code
```

**Cue-based:**
```bash
rei habit create \
  --name "NAME" \
  --purpose "PURPOSE" \
  --cue "After meals" \
  --actor claude-code
```

**Cue-based with target:**
```bash
rei habit create \
  --name "NAME" \
  --purpose "PURPOSE" \
  --cue "When I see interesting code" \
  --target "3/week" \
  --actor claude-code
```

**With action template:**
```bash
rei habit create \
  --name "NAME" \
  --purpose "PURPOSE" \
  --daily \
  --action-template "TEMPLATE" \
  --intention <INTENTION_ID> \
  --actor claude-code
```

### Break Habits

Add `--break` flag to create a Break habit:

```bash
rei habit create \
  --name "No social media before noon" \
  --purpose "Protect my morning focus time" \
  --daily \
  --break \
  --actor claude-code
```

**After creating a Break habit, start abstinence tracking:**
```bash
rei habit start-abstinence <HABIT_ID>
```

## Output Format

After creating the habit, provide a summary:

```
## Habit Created

- **Name**: [name]
- **ID**: [habit_id]
- **Type**: Build / Break
- **Purpose**: [purpose]
- **Cadence**: [schedule description or cue description]
- **Linked Intention**: [intention_title] ([intention_id]) or "(standalone)"
- **Focus Area**: [focus_name] or "(not assigned)"
- **Action Category**: [category_slug] or "(not set)"
- **Action Template**: [template] or "(not set)"
- **Context**: [context] or "(inherited from intention)" or "(not set)"

### Next Steps

**For Build habits:**
1. Record your first action: `rei habit record-action` (add `--category SLUG` if category was set)
2. Check habit status: `rei habit show <HABIT_ID>`
3. View all habits: `rei habit list`

**For Break habits:**
1. Start abstinence tracking: `rei habit start-abstinence <HABIT_ID>`
2. If relapse occurs: `rei habit log-occurrence <HABIT_ID>`
3. Check habit status: `rei habit show <HABIT_ID>`
```

## Important Notes

- Always include `--actor claude-code` when creating habits
- If the user provides all details upfront, skip the questions and create directly
- **Polarity**: Default is Build; use `--break` for Break habits
- **Schedules**: `--daily`, `--weekly DAY`, `--weekly-on DAY1,DAY2,...`, `--monthly DAY`, `--quarterly`, `--yearly-month M --yearly-day D`, `--every N`
- **Flexible**: `--free-weekly N`, `--free-monthly N`, `--free-quarterly N`, `--free-yearly N`
- **Cue-based**: `--cue "DESCRIPTION"` with optional `--target "N/period"`
- Day formats: full name (monday) or short (mon)
- Link habits to intentions when possible for better tracking
- Only set context on unlinked habits (linked habits inherit from intention)
- Action templates save time when recording repeated actions
- Habits start as Active by default
- Use `rei habit pause` to temporarily pause a habit
- Use `rei habit retire` to permanently retire a habit
- **Categories** can be assigned to actions recorded from habits using `--category SLUG` with `rei habit record-action`
- Habits don't have categories directly - but actions from habits can be categorized

## Common Habit Examples

### Build Habits

**Morning practices:**
- Morning journaling (daily) -> Create focus
- Morning meditation (daily) -> Play focus
- Morning exercise (--weekly-on mon,wed,fri) -> Play focus

**Learning practices:**
- Daily reading (daily) -> Read focus
- Language study (--weekly-on tue,thu,sat) -> Study focus
- Code practice (daily) -> Study focus

**Evening practices:**
- Evening reflection (daily) -> Create focus
- Weekly planning (--weekly sunday) -> Study focus
- Gratitude journaling (daily) -> Create focus

**Flexible practices:**
- Read 3 books per month (--free-monthly 3) -> Read focus
- Exercise 4 times per week (--free-weekly 4) -> Play focus
- Write 2 blog posts per quarter (--free-quarterly 2) -> Create focus

**Cue-based practices:**
- Log interesting quotes (--cue "When I encounter good writing") -> Read focus
- Take walking breaks (--cue "After 2 hours of focused work") -> Play focus

### Break Habits

**Digital wellness:**
- No social media before noon (--daily --break)
- No phone during meals (--cue "During meals" --break)
- No screens after 10pm (--daily --break)

**Health:**
- No late-night snacking (--daily --break)
- No skipping workouts (--free-weekly 4 --break)

**Productivity:**
- No email first thing (--daily --break)
- No multitasking during deep work (--cue "During focus blocks" --break)

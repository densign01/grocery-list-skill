# Grocery List Skill

A [Claude Code](https://claude.ai/claude-code) skill that parses meal plans with recipe URLs, extracts and combines ingredients, and adds them to Apple Reminders.

## Installation

### Step 1: Install the Skill

Clone this repo into your Claude Code skills directory:

```bash
git clone https://github.com/densign01/grocery-list-skill ~/.claude/skills/groceries
```

Or manually copy the skill file:

```bash
mkdir -p ~/.claude/skills/groceries
curl -o ~/.claude/skills/groceries/SKILL.md \
  https://raw.githubusercontent.com/densign/grocery-list-skill/main/SKILL.md
```

### Step 2: Install Dependencies

**agent-browser** (required for recipe scraping):

```bash
npm install -g agent-browser
agent-browser install  # Downloads Chromium
```

### Step 3: Create a "Groceries" List in Apple Reminders

Open Apple Reminders and create a new list called **"Groceries"** (case-sensitive).

### Step 4: Verify Installation

In Claude Code, type `/groceries` — you should see the skill activate.

---

## Overview

This skill takes a weekly meal plan (mix of recipe URLs and plain-text items), scrapes ingredients from recipe websites using a headless browser, simplifies them for shopping, and adds everything to your Apple Reminders "Groceries" list.

## Features

- **Recipe scraping** via [agent-browser](https://github.com/vercel-labs/agent-browser) (headless Chromium)
- **Intelligent ingredient parsing** - keeps amounts for packaged items, simplifies bulk staples
- **Quantity combining** - merges garlic needs across recipes into one item
- **Duplicate detection** - checks existing list to avoid duplicates
- **Pantry staple filtering** - skips salt, pepper, olive oil (configurable)
- **Apple Reminders integration** via AppleScript

## Usage

Invoke with `/groceries` followed by your meal plan:

```
/groceries

Monday - Chicken soup - https://example.com/chicken-soup
Tuesday - Tacos - https://example.com/easy-tacos
Wednesday - Pasta night

Also add:
- milk
- eggs
- bread
```

## Workflow

```
┌─────────────────┐
│  Meal Plan      │
│  (URLs + text)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Check Existing  │  ← Reads current Groceries list
│ Reminders       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Scrape Recipes  │  ← agent-browser opens each URL,
│ via Browser     │    returns accessibility tree
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Parse & Simplify│  ← Keep amounts for packaged items,
│ Ingredients     │    simplify bulk staples, skip pantry items
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Combine Dupes   │  ← "1 onion" + "2 onions" = "3 onions"
│ Across Recipes  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Present to User │  ← Show what will be added vs skipped
│ for Confirmation│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Add to Apple    │  ← AppleScript batch insert
│ Reminders       │
└─────────────────┘
```

## Ingredient Parsing Rules

| Rule | Example |
|------|---------|
| Keep amounts for packaged items | "18oz chicken broth" → "18oz chicken broth" |
| Keep counts for produce | "3 onions" → "3 onions" |
| Drop measurements for bulk staples | "2 cups flour" → "Flour" |
| Skip pantry staples | salt, pepper, olive oil, vegetable oil |
| Simplify names | "boneless skinless chicken breast" → "Chicken breast" |
| Capitalize first letter | "eggs" → "Eggs" |

## Quantity Handling

**From recipes:** Include quantities (e.g., "2 leeks", "1 head garlic")

**Plain-text items:** Keep as-is, no quantities added (user decides at store)

## Why agent-browser?

Traditional web scraping (like `curl` or `fetch`) fails on modern recipe sites because:

1. **JavaScript rendering** - Most recipe sites load content dynamically
2. **Anti-bot measures** - Sites block automated requests
3. **Complex DOM** - Ingredients buried in nested components

`agent-browser` solves this by:
- Running a real Chromium browser
- Returning an **accessibility tree** (semantic structure, not raw HTML)
- Providing element refs for interaction if needed

### Example: Scraping a Recipe

```bash
# Navigate to recipe
agent-browser open "https://example.com/recipe"

# Wait for content to load
agent-browser wait 2000

# Get accessibility tree (AI-friendly format)
agent-browser snapshot
```

The snapshot returns structured data like:
```yaml
- heading "Ingredients":
  - listitem: "2 cups flour"
  - listitem: "1 tsp salt"
  - listitem: "3 eggs"
```

## Apple Reminders Integration

Uses AppleScript to interact with Reminders:

```applescript
tell application "Reminders"
    set groceryList to list "Groceries"
    make new reminder in groceryList with properties {name:"Eggs"}
end tell
```

**Batch insert** (more efficient):
```applescript
tell application "Reminders"
    set groceryList to list "Groceries"
    make new reminder in groceryList with properties {name:"Item 1"}
    make new reminder in groceryList with properties {name:"Item 2"}
    -- etc
end tell
```

## Configuration

The skill behavior can be adjusted:

| Setting | Default | Description |
|---------|---------|-------------|
| Skip duplicates | Yes | Check existing list before adding |
| Skip pantry staples | Yes | Omit salt, pepper, oils |
| Include quantities | Recipe items only | Plain-text items have no quantities |

## Limitations

- macOS only (Apple Reminders requirement)
- Some recipe sites may still block scraping
- Ingredient parsing is heuristic-based (may need manual cleanup)

## Related

- [agent-browser](https://github.com/vercel-labs/agent-browser) - Headless browser CLI for AI agents
- [Apple Reminders AppleScript](https://developer.apple.com/library/archive/documentation/AppleScript/Conceptual/AppleScriptLangGuide/) - Apple's scripting documentation

## License

MIT

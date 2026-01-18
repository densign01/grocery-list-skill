---
name: groceries
description: Parse recipes and add combined ingredients to Apple Reminders Groceries list
allowed-tools: [Bash, WebFetch, Read, Write, AskUserQuestion]
---

# Groceries Skill

Parse meal plans (recipes + URLs), combine ingredients intelligently, and add to Apple Reminders.

## When to Use

- User wants to add recipe ingredients to their grocery list
- User has a meal plan with multiple recipes
- User says "groceries", "grocery list", "meal plan", "shopping list"

## Instructions

### Step 1: Read Existing Grocery List

**ALWAYS start by checking what's already on the list** to avoid duplicates:

```bash
osascript -e 'tell application "Reminders"
    set groceryList to list "Groceries"
    set reminderNames to {}
    repeat with r in (reminders in groceryList whose completed is false)
        set end of reminderNames to name of r
    end repeat
    return reminderNames
end tell'
```

Store these existing items for comparison in Step 5.

### Step 2: Get the Meal Plan

If no meal plan was provided, ask the user:

"Paste your meal plan below. This can include:
- Recipe text with ingredients
- URLs to recipe websites
- A mix of both

I'll parse everything, combine duplicate ingredients, and add them to your Groceries list in Apple Reminders."

### Step 3: Extract URLs and Scrape Recipes

1. Identify any URLs in the input (look for http/https links)
2. **Primary method:** Use `agent-browser` to scrape (handles JavaScript-heavy recipe sites):
   ```bash
   agent-browser open "URL" && sleep 2 && agent-browser snapshot
   ```
   The snapshot returns an accessibility tree - look for the "Ingredients" heading and extract listitem contents.

3. **Fallback:** If agent-browser is unavailable, use WebFetch:
   ```
   WebFetch with prompt: "Extract the complete ingredients list from this recipe. Return ONLY the ingredients, one per line, with quantities."
   ```
4. Collect all scraped ingredients

### Step 4: Parse and Simplify Ingredients

From both the scraped content AND any plain text recipes, extract every ingredient and **simplify to shopping-friendly format**:

**Formatting Rules:**
- **Keep amounts for packaged items** - "18oz chicken broth" → "18oz chicken broth" (need to buy the right size)
- **Keep counts for produce** - "3 onions" → "3 onions" (need to grab multiple)
- **Drop measurements for bulk staples** - "2 cups flour" → "Flour" (you buy a bag, not measure at store)
- **Skip pantry staples** - omit salt, pepper, olive oil, vegetable oil (assume user has these)
- **Use simple names** - "boneless skinless chicken breast" → "Chicken breast"
- **Capitalize first letter** - "Eggs" not "eggs"

### Step 5: Combine Duplicate Ingredients

Group ingredients by their normalized item name:
- If multiple recipes need the same item, combine the counts only if it matters
- "1 onion" + "2 onions" = "3 onions" (meaningful count)
- "1 cup flour" + "2 cups flour" = "Flour" (just need to buy flour)

### Step 6: Check Against Existing List & Present

Compare parsed ingredients against the existing list from Step 1:
- **Match items case-insensitively** - "eggs" matches "Eggs"
- **Match partial names** - "Chicken breast" matches "Thin cut chicken breast"
- Items already on the list = **skip** (don't add duplicates)
- New items = **will be added**

Show the user what will happen:

```
## Adding to Groceries (X new items)

**Will add:**
- Ground beef
- Garlic
- Tahini

**Already on list (skipping):**
- ~~Chicken breast~~ (matches "Thin cut chicken breast")
- ~~Eggs~~
- ~~Onion~~
```

Ask: "I'll add X new items. Y items are already on your list. Sound good?"

### Step 7: Add to Apple Reminders

Upon confirmation, add each item to the "Groceries" list using AppleScript:

```bash
osascript -e 'tell application "Reminders"
    set groceryList to list "Groceries"
    make new reminder in groceryList with properties {name:"ITEM_NAME"}
end tell'
```

Run one command per item. For efficiency, you can batch them:

```bash
osascript <<'EOF'
tell application "Reminders"
    set groceryList to list "Groceries"
    make new reminder in groceryList with properties {name:"Chicken breast"}
    make new reminder in groceryList with properties {name:"3 onions"}
    make new reminder in groceryList with properties {name:"Eggs"}
    -- etc
end tell
EOF
```

### Step 8: Confirm Completion

After adding all items, confirm:
"Added X new items to your Groceries list. (Y items were already there.) Happy cooking!"

## Example Usage

User: `/groceries`

Then pastes:
```
Chicken Stir Fry:
- 1 lb chicken breast
- 2 tbsp soy sauce
- 1 onion
- 2 cloves garlic

https://www.somerecipe.com/beef-tacos

Simple Salad:
- 1 head lettuce
- 2 tomatoes
- 1/4 cup olive oil
```

Claude scrapes the URL, parses all three recipes, and presents a simplified list:

```
**Meat:**
- Chicken breast
- Ground beef

**Produce:**
- 3 onions (combined from recipes)
- Lettuce
- Tomatoes
- Garlic

**Others:**
- Soy sauce
- Taco shells
```

(Note: Salt, pepper, olive oil are skipped as pantry staples)

## Notes

- The "Groceries" list must already exist in Apple Reminders
- Items are added as unchecked reminders
- **Duplicates are automatically detected and skipped** - existing items won't be added again
- Partial matching prevents duplicates like "Chicken breast" when "Thin cut chicken breast" exists

## Quantity Handling

- **Recipe items:** Include quantities from the recipe (e.g., "2 leeks", "1 head garlic")
- **Plain-text items:** Keep as-is without adding quantities - user decides at the store

## Requirements

- macOS (for Apple Reminders AppleScript integration)
- [agent-browser](https://github.com/vercel-labs/agent-browser) CLI (recommended for recipe scraping)
  ```bash
  npm install -g agent-browser
  agent-browser install
  ```

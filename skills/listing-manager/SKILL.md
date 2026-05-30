---
name: listing-manager
description: >
  Manage existing listings across Craigslist and Facebook Marketplace. Use this skill
  when the user wants to mark items as sold, sync sold status to platforms, remove old
  listings, revise prices on stale items, check what's still listed, or generally manage
  their selling inventory. Trigger phrases include: "I sold the...", "mark as sold",
  "update my listings", "lower the price", "revise prices", "what hasn't sold yet",
  "remove that listing", "delete the posting", "sync my inventory", or "clean up sold items".
---

# Listing Manager — Sync Sales & Revise Prices

This skill manages existing listings after they've been posted. It complements the `sell-stuff` skill (which handles creating and posting new listings) by handling the lifecycle after posting: syncing sold items across platforms and revising prices on stale listings.

## Overview

Two core workflows:

1. **Sync Sold Items** — User marks an item as sold → update tracker → delete from Craigslist → mark as sold on Facebook Marketplace → clean up photos
2. **Revise Stale Prices** — Identify items that haven't sold → suggest price drops based on age → update platforms with approved changes

## Prerequisites

- The `inventory-tracker.xlsx` spreadsheet must exist in the workspace with the standard column layout (see below)
- The user must be logged into Craigslist and Facebook in the browser
- Items should have been previously posted using the `sell-stuff` skill workflow

### Tracker Column Reference

| Col | Header | Key Values |
|-----|--------|------------|
| A | Item # | Sequential number |
| B | Item Name | Descriptive name |
| C | Brand | Brand/manufacturer |
| D | Model | Model name/number |
| E | Condition | Good, Fair, New, etc. |
| F | Suggested Price | Current asking price |
| G | Photos Folder | Relative path (e.g., `04-epson-v370-scanner`) |
| H | Status | Draft / Listed / Sold |
| I | FB Marketplace | Listed / Sold / Not Listed |
| J | Craigslist | Listed / Sold / Not Listed |
| K | Date Listed | YYYY-MM-DD |
| L | Date Sold | YYYY-MM-DD |
| M | Sold Price | Actual selling price |
| N | Notes | Change log and details |

---

## Workflow 1: Sync Sold Items

### When to use

The user says something like:
- "I sold the Epson scanner for $35"
- "Mark item #4 as sold"
- "The dresser sold for $70, update everything"
- "Someone bought the keyboard and the toaster"

### Step 1: Identify the item and sale details

Parse the user's message to extract:
- **Which item** — by name, item number, or description
- **Sale price** — what the buyer paid
- **Which platform it sold on** — if mentioned (optional, for notes)

If any detail is missing, ask the user. At minimum you need the item and the sale price.

Look up the item in `inventory-tracker.xlsx` to confirm it exists and has Status = "Listed". If the item is already "Sold" or "Draft", warn the user.

### Step 2: Update the tracker spreadsheet

```python
import openpyxl
from datetime import date

wb = openpyxl.load_workbook('inventory-tracker.xlsx')
ws = wb.active

# Find the item row by Item # or Item Name
target_row = None
for row in range(2, ws.max_row + 1):
    item_num = ws.cell(row=row, column=1).value
    item_name = ws.cell(row=row, column=2).value
    if item_num == ITEM_NUMBER or (item_name and SEARCH_TERM.lower() in str(item_name).lower()):
        target_row = row
        break

if target_row:
    ws.cell(row=target_row, column=8).value = 'Sold'                    # Status
    ws.cell(row=target_row, column=9).value = 'Sold'                    # FB Marketplace
    ws.cell(row=target_row, column=10).value = 'Sold'                   # Craigslist
    ws.cell(row=target_row, column=12).value = date.today().isoformat() # Date Sold
    ws.cell(row=target_row, column=13).value = SOLD_PRICE               # Sold Price

    # Append to Notes
    existing_notes = ws.cell(row=target_row, column=14).value or ''
    new_note = f"Sold {date.today().isoformat()} for ${SOLD_PRICE}."
    ws.cell(row=target_row, column=14).value = (existing_notes + ' ' + new_note).strip()

wb.save('inventory-tracker.xlsx')
```

### Step 3: Remove the Craigslist listing

**Navigate to the user's Craigslist account postings:**

1. Go to `https://accounts.craigslist.org/login/home`
2. If not logged in, ask the user to log in first and wait for confirmation
3. Look for "my postings" or a list of active postings
4. Use `read_page` to find the listing by matching the title from the tracker
5. Click "delete" next to the matching listing
6. If a confirmation dialog appears, click "OK" or "confirm"
7. Take a screenshot to verify deletion

**If the listing is not found:** It may have expired (CL posts expire after ~30 days) or been manually deleted. Log this in Notes: "CL listing not found (may have expired)." and continue — don't fail the whole workflow.

### Step 4: Mark as sold on Facebook Marketplace

**Navigate to the user's Facebook Marketplace listings:**

1. Go to `https://www.facebook.com/marketplace/you/selling`
2. If not logged in, ask the user to log in first
3. Use `read_page` or `find` to locate the listing by title
4. Click the three-dot menu (⋯) on the listing
5. Click "Mark as Sold" from the dropdown menu
6. If Facebook shows a confirmation dialog or "Who did you sell to?" prompt, click through it (select "Sold to someone else" or skip if possible)
7. Take a screenshot to verify

**If "Mark as Sold" is not available**, try deleting the listing instead via the same three-dot menu → "Delete". Either outcome removes it from active listings.

**If the listing is not found:** Log "FB listing not found (may have been removed)." in Notes and continue.

### Step 5: Clean up photos

After both platforms are synced, clean up the photos folder:

1. Read the Photos Folder path from the tracker (column G)
2. Tell the user: "I'm about to delete the photos folder `inventory/[folder-name]/` since this item is sold. The photos will be permanently removed. Confirm?"
3. **Wait for explicit user confirmation** — do NOT delete without it
4. If confirmed:

```python
import shutil
import os

folder_path = f'inventory/{PHOTOS_FOLDER}'
if os.path.exists(folder_path):
    shutil.rmtree(folder_path)
    print(f"Deleted {folder_path}")
```

5. Update Notes: "Photos deleted [date]."

### Step 6: Confirm completion

Summarize what was done:
- "Item #4 (Epson Scanner) marked as sold for $35"
- "Craigslist: listing deleted ✓"
- "Facebook Marketplace: marked as sold ✓"
- "Photos folder cleaned up ✓"
- "Tracker updated ✓"

---

## Workflow 2: Revise Stale Prices

### When to use

The user says something like:
- "Some of my stuff hasn't sold yet, can you suggest new prices?"
- "Revise prices on stale listings"
- "What should I reprice?"
- "Lower the prices on things that aren't selling"
- "Help me move these items faster"

### Step 1: Analyze the inventory

Read `inventory-tracker.xlsx` and find all items where Status = "Listed":

```python
import openpyxl
from datetime import date, datetime

wb = openpyxl.load_workbook('inventory-tracker.xlsx')
ws = wb.active
today = date.today()

stale_items = []
for row in range(2, ws.max_row + 1):
    status = ws.cell(row=row, column=8).value
    if status != 'Listed':
        continue

    item_num = ws.cell(row=row, column=1).value
    item_name = ws.cell(row=row, column=2).value
    current_price = ws.cell(row=row, column=6).value
    date_listed_raw = ws.cell(row=row, column=11).value

    # Parse date — could be string or datetime object
    if isinstance(date_listed_raw, datetime):
        date_listed = date_listed_raw.date()
    elif isinstance(date_listed_raw, str):
        date_listed = datetime.strptime(date_listed_raw, '%Y-%m-%d').date()
    else:
        continue  # Skip if no date

    days_listed = (today - date_listed).days
    stale_items.append({
        'row': row,
        'item_num': item_num,
        'name': item_name,
        'price': current_price,
        'date_listed': date_listed,
        'days_listed': days_listed,
    })
```

### Step 2: Calculate suggested prices

Apply progressive discount tiers based on how long the item has been listed:

| Days Listed | Suggested Discount | Rationale |
|-------------|-------------------|-----------|
| 0-6 days | No change | Too early to revise |
| 7-13 days | 10-15% off | First price drop to generate renewed interest |
| 14-20 days | 20-25% off | More aggressive — item is getting stale |
| 21+ days | 30-40% off | Need to move it — consider below-market pricing |

```python
import math

def suggest_new_price(current_price, days_listed):
    """Returns (suggested_price, discount_pct) or None if too fresh."""
    if days_listed < 7:
        return None  # Too fresh
    elif days_listed < 14:
        discount = 0.12  # ~12% off
    elif days_listed < 21:
        discount = 0.22  # ~22% off
    else:
        discount = 0.35  # ~35% off

    new_price = math.floor(current_price * (1 - discount))

    # Floor: never suggest less than $5 for items originally $10+
    if current_price >= 10 and new_price < 5:
        new_price = 5

    return (new_price, round(discount * 100))
```

### Step 3: Present suggestions to the user

Show all stale items in a clear summary:

```
Here's what I'd suggest for your unsold items:

| # | Item | Current | Days Listed | Suggested | Drop |
|---|------|---------|-------------|-----------|------|
| 1 | De'Longhi Heater | $30 | 6 days | — (too fresh) | — |
| 2 | B&D Toaster | $8 | 6 days | — (too fresh) | — |
| 4 | Epson Scanner | $40 | 12 days | $35 | 12% |
| 9 | IKEA Mattress | $150 | 0 days | — (too fresh) | — |

Items needing revision: #4 (Epson Scanner)

Want me to update the price on #4 to $35 across Craigslist and Facebook Marketplace?
```

**Bundle awareness:** If related items are both stale (e.g., MALM nightstand and MALM chest), mention it: "Items #11 and #12 are both IKEA MALM birch pieces. You could also consider a deeper bundle discount to move them together."

Wait for the user to approve, decline, or set a custom price for each item.

### Step 4: Update platforms with approved prices

For each approved price revision:

#### Update Craigslist price

1. Go to `https://accounts.craigslist.org/login/home`
2. Find the listing in "my postings" by matching the title
3. Click "edit" next to the listing
4. On the edit form, find the `price` field and update it:

```javascript
// Use form_input for the price field, or if that fails:
const priceInput = document.querySelector('input[name="price"]');
priceInput.value = '';
priceInput.focus();
```

Then type the new price, or use `form_input` with the ref for the price field.

5. Also update the description if it mentions the old price (search for the dollar amount in `PostingBody`)
6. Click "continue" or "save" to submit the edit
7. Walk through any map/image confirmation pages
8. Take a screenshot to verify the updated price

#### Update Facebook Marketplace price

1. Go to `https://www.facebook.com/marketplace/you/selling`
2. Find the listing by title
3. Click the three-dot menu (⋯) → "Edit listing" or "Edit"
4. Find the price field and update it. Use `form_input` if possible, otherwise use JavaScript:

```javascript
// For FB's React-controlled price input:
const priceInput = document.querySelector('input[aria-label="Price"]')
    || document.querySelector('input[placeholder*="Price"]');
if (priceInput) {
    const nativeSetter = Object.getOwnPropertyDescriptor(
        window.HTMLInputElement.prototype, 'value'
    ).set;
    nativeSetter.call(priceInput, 'NEW_PRICE');
    priceInput.dispatchEvent(new Event('input', { bubbles: true }));
    priceInput.dispatchEvent(new Event('change', { bubbles: true }));
}
```

5. Click "Save" or "Update" or "Next" → walk through any confirmation pages
6. Take a screenshot to verify

### Step 5: Update the tracker

For each revised item:

```python
# Update price
ws.cell(row=target_row, column=6).value = NEW_PRICE  # Suggested Price

# Log the change in Notes
existing_notes = ws.cell(row=target_row, column=14).value or ''
revision_note = f"Price revised {date.today().isoformat()} from ${OLD_PRICE} to ${NEW_PRICE} ({DAYS_LISTED} days listed)."
ws.cell(row=target_row, column=14).value = (existing_notes + ' ' + revision_note).strip()

wb.save('inventory-tracker.xlsx')
```

### Step 6: Show summary

```
Price revisions complete:

| # | Item | Was | Now | CL | FB |
|---|------|-----|-----|----|----|
| 4 | Epson Scanner | $40 | $35 | Updated ✓ | Updated ✓ |
| 8 | Dressing Stick | $8 | $5 | Updated ✓ | Updated ✓ |

Items declined: #3 (Keyboard)
Tracker updated with all changes.
```

---

## Quick Status Check

If the user just wants to know what's going on with their inventory (e.g., "what's still listed?" or "show me my inventory status"), read the tracker and present a summary without taking any action:

```
Your inventory status:

Listed (11 items): $296 total asking price
- Items #1-5, #7-12 — listed on both CL and FB

Sold (1 item): $10 revenue
- Item #6 (Resistance Bands) — sold 2026-03-02 for $10

Oldest unsold: Items #1-8 (listed 6 days ago)
Newest: Items #9-12 (listed today)
```

---

## Error Handling

- **User not logged in:** If `read_page` shows a login form instead of account postings, ask the user to log in first. Don't attempt to enter credentials.
- **Listing not found on platform:** Log it, warn the user, but continue with other steps. The tracker is the source of truth.
- **Platform UI changed:** If the expected buttons/forms aren't found, use `read_page` to inspect the current page structure and adapt. If you truly can't find the right elements, give the user a direct link to manually update.
- **Multiple items with similar names:** Always confirm with the user by showing Item # and full name before taking action.
- **Date parsing:** The tracker may store dates as Excel date objects or strings. Handle both:
  ```python
  from datetime import datetime
  if isinstance(val, datetime):
      d = val.date()
  elif isinstance(val, str):
      d = datetime.strptime(val, '%Y-%m-%d').date()
  ```

## Tips

- **Batch operations:** If the user wants to mark multiple items as sold, process them one at a time to avoid confusion. Confirm each one.
- **Price floor:** Don't suggest prices below $5 for items that were originally $10+. At that point, suggest donating or bundling instead.
- **Seasonal awareness:** Some items sell better at certain times. If it's winter and an outdoor item isn't selling, mention that waiting might be better than a deep discount.
- **Re-listing:** If an item has been listed 30+ days and hasn't sold even with price drops, suggest deleting and re-posting as a "new" listing — platforms often boost visibility for new posts.
- **Always screenshot** before and after making changes on platforms for verification.

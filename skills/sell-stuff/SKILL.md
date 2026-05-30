---
name: sell-stuff
description: >
  End-to-end workflow for listing items for sale on Craigslist and Facebook Marketplace.
  Use this skill whenever the user wants to sell something, list items online, post to
  Craigslist or Facebook Marketplace, organize photos of things to sell, create selling
  listings, manage an inventory of items for sale, or track what they've listed and sold.
  Also trigger when the user drops photos of items and mentions selling, pricing, or
  getting rid of stuff. Even casual phrases like "I want to get rid of this",
  "how much could I get for this", "help me sell my old furniture", or "post this on
  marketplace" should activate this skill.
---

# Sell Stuff — Listing Items for Sale

This skill walks you through the complete process of helping someone sell their stuff online. The workflow has been refined through real-world use and includes hard-won workarounds for browser automation quirks on both Craigslist and Facebook Marketplace.

## Overview

The end-to-end flow:

1. **Receive & identify items** — User provides photos (often HEIC from iPhone) and basic info
2. **Organize photos** — Convert to JPG if needed, store in numbered inventory folders
3. **Write listing descriptions** — Craft compelling, honest listings with titles, descriptions, pricing
4. **Post to platforms** — List on Craigslist and Facebook Marketplace via browser automation
5. **Track inventory** — Maintain a spreadsheet tracking status across platforms
6. **Clean up inbox** — Remove source photos from `inbox/` once items are successfully posted

## Step 1: Receive and Identify Items

When the user provides photos of items to sell:

- Look at the photos carefully to identify the item (brand, model, condition, notable features)
- Ask the user for any details you can't determine from the photos: brand, model, how old it is, any defects or issues, and their asking price
- If the user hasn't given a price, suggest one based on what the item retails for new and its condition — but always defer to the user's preference
- Note the condition honestly. If there are flaws visible in the photos, mention them — buyers appreciate transparency, and it prevents disputes later

## Step 2: Organize Photos

### HEIC to JPG Conversion

iPhone photos are often in HEIC format. Convert them to JPG for compatibility with listing platforms:

```python
import pillow_heif
from PIL import Image

heif_file = pillow_heif.read_heif("photo.HEIC")
image = Image.frombytes(heif_file.mode, heif_file.size, heif_file.data, "raw")
image.save("photo.jpg", "JPEG", quality=92)
```

Install the library if needed: `pip install pillow-heif --break-system-packages`

Note: ImageMagick's `convert` command often fails with HEIC files on this system — use the Python approach above instead.

### Inventory Folder Structure

Organize photos into numbered folders inside `inventory/`:

```
sell-stuff/
├── inventory/
│   ├── 01-item-name/
│   │   ├── IMG_1234.jpg
│   │   └── IMG_1235.jpg
│   ├── 02-another-item/
│   │   └── IMG_1236.jpg
│   └── ...
├── inventory-tracker.xlsx
└── listings.md
```

Use the next available number. Check what folders already exist to determine the next number. Use lowercase-kebab-case for the item name portion.

## Step 3: Write Listing Descriptions

Save all listings to `listings.md` with this format for each item:

```markdown
## [Number]. [Item Name] — $[Price]

**Title:** [Concise, searchable title — include brand, key specs]

**Description:**
[2-4 paragraphs. Lead with what it is. Include brand, model, key features,
dimensions if furniture. Be upfront about any flaws. End with pickup location
and payment methods.]

**Condition:** [Used — Good | Used — Fair | New In Box | etc.]
**Photos:** [Number] photos (inventory/[folder-name]/)
```

### Writing good listings

The goal is to help the user sell quickly at a fair price. Good listings are:

- **Honest about flaws** — "Full disclosure: one drawer slides open on its own" builds trust and prevents returns/complaints
- **Specific about what's included** (and what's NOT) — "Power cord NOT included" saves everyone time
- **Searchable** — Put brand and model in the title so people searching can find it
- **Locally relevant** — Include pickup location, payment methods (Cash, Venmo, etc.)
- **Bundle-friendly** — If the user is selling matching items, suggest bundle deals

## Step 4: Post to Platforms

### Critical: Photo Uploads

The browser file upload tool does NOT work for uploading photos from the VM filesystem to websites — it returns a permissions error. Instead:

1. Make sure the photos are organized in the user's workspace folder (the `inventory/` subfolders)
2. When you reach the photo upload step on any platform, ask the user to drag and drop the photos from their inventory folder into the browser
3. Wait for the user to confirm they're done before proceeding
4. Tell the user exactly which folder to drag from, e.g.: "Please drag the photos from your `inventory/09-ikea-anneland-mattress/` folder into the upload area, then let me know when you're done."

This is the most reliable approach and has been tested extensively. Do not attempt to use the file_upload tool for listing photos.

### Craigslist

**Posting URL:** Navigate to `https://post.craigslist.org/` and select the appropriate city/area.

**Category selection:** Choose "for sale by owner" → appropriate subcategory (furniture, electronics, etc.)

**Form fields** (these are the actual HTML field names):
- `PostingTitle` — The listing title
- `price` — Price in dollars (numbers only, no $ sign)
- `PostingBody` — The full description text
- `sale_manufacturer` — Brand name (if applicable)
- `sale_model` — Model name/number (if applicable)
- `sale_size` — Dimensions or size (if applicable)
- `condition` — A dropdown select element

**Condition dropdown workaround:** The `form_input` tool sometimes doesn't reliably set the Craigslist condition dropdown. Use JavaScript instead:

```javascript
const sel = document.querySelector('select[name="condition"]');
sel.value = '[VALUE]';
sel.dispatchEvent(new Event('change', {bubbles: true}));
```

Condition values: `10` = new, `20` = like new, `30` = excellent, `40` = good, `50` = fair, `60` = salvage

**Craigslist posting flow:**
1. Fill in all form fields on the posting details page
2. Click "continue" → Map page (verify location, click continue)
3. Image upload page → **Ask the user to drag photos** → Click "done with images"
4. Preview page → Review → Click "publish"
5. Confirm the listing was published successfully

### Facebook Marketplace

**Posting URL:** Navigate to `https://www.facebook.com/marketplace/create/item`

**The description typing problem:** Facebook Marketplace uses React-controlled form fields. The regular `type` action frequently fails partway through with a "Detached while handling command" error, causing text duplication and corruption. This happens because Facebook's React event handlers interfere with synthetic keyboard input on textareas.

**The fix — use JavaScript to set text values for descriptions:**

```javascript
const textarea = document.querySelector('textarea');
const nativeInputValueSetter = Object.getOwnPropertyDescriptor(
  window.HTMLTextAreaElement.prototype, 'value'
).set;
nativeInputValueSetter.call(textarea, `YOUR DESCRIPTION TEXT HERE`);
textarea.dispatchEvent(new Event('input', { bubbles: true }));
textarea.dispatchEvent(new Event('change', { bubbles: true }));
```

This approach bypasses React's synthetic event system by calling the native HTMLTextAreaElement value setter directly, then dispatching native DOM events that React's reconciler picks up. It works reliably where keyboard simulation doesn't.

For regular input fields (title, price), the normal `type` action or `form_input` tool usually works fine — the textarea/description is the problematic one.

**Facebook Marketplace posting flow:**
1. Fill in: Title, Price, Category (search and select from dropdown), Condition, Description (via JS), optionally other fields like bed size for furniture
2. Scroll down to the photo upload area → **Ask the user to drag photos**
3. Click "Next" → Delivery method page (select "Door pickup" or appropriate option, set location)
4. Click "Next" → Audience/visibility page
5. Click "Publish"
6. Confirm success

**Category selection on Facebook:** Type the category name into the category search field, wait for the dropdown to appear, then click the matching result. Common categories: "Bedroom Furniture Sets", "Dressers", "Nightstands", "Electronics", "Home Appliances", "Sporting Goods".

## Step 5: Track Inventory

Maintain an `inventory-tracker.xlsx` spreadsheet with these columns:

| Column | Header | Description |
|--------|--------|-------------|
| A | Item # | Sequential number |
| B | Item Name | Descriptive name |
| C | Brand | Brand/manufacturer |
| D | Model | Model name/number |
| E | Condition | Good, Fair, New, etc. |
| F | Suggested Price | Asking price |
| G | Photos Folder | Relative path to inventory folder |
| H | Status | Draft → Listed → Sold |
| I | FB Marketplace | Listed / Sold / Not Listed |
| J | Craigslist | Listed / Sold / Not Listed |
| K | Date Listed | YYYY-MM-DD |
| L | Date Sold | YYYY-MM-DD (when sold) |
| M | Sold Price | Actual selling price |
| N | Notes | Any additional notes |

Add summary rows at the bottom with formulas for: Total Estimated Value (SUM of prices), Items Listed (COUNTIF status="Listed"), Items Sold (COUNTIF status="Sold"), Total Revenue (SUM of sold prices).

Use `openpyxl` to create/update the spreadsheet:

```python
import openpyxl
wb = openpyxl.load_workbook('inventory-tracker.xlsx')
ws = wb.active
# Update status for newly listed items
for row in range(start_row, end_row + 1):
    ws.cell(row=row, column=8).value = 'Listed'      # Status
    ws.cell(row=row, column=9).value = 'Listed'       # FB Marketplace
    ws.cell(row=row, column=10).value = 'Listed'      # Craigslist
    ws.cell(row=row, column=11).value = '2026-03-08'  # Date Listed
wb.save('inventory-tracker.xlsx')
```

If the spreadsheet doesn't exist yet, create it with the headers and formatting above.

## Step 6: Clean Up Inbox

The `inbox/` folder is where users drop raw photos (often as zip files or loose HEIC/JPG files) before processing. Once items have been successfully posted to all requested platforms, clean up the source files from the inbox:

1. **Track which inbox files belong to which items.** When you receive photos in Step 1, note which inbox file(s) they came from (e.g., `inbox/Photos-march-8-2026.zip` contained the mattress and dresser photos).
2. **Only delete after successful posting.** Wait until the item is confirmed posted on all requested platforms (status changed to "Listed" in the tracker) before removing source files.
3. **Delete the source files** from `inbox/` — these are the zip archives or raw photo files the user originally dropped in. The converted JPGs in the `inventory/` folders are the permanent copies and should NOT be deleted.
4. **Confirm the cleanup** to the user, e.g.: "I've cleaned up `Photos-march-8-2026.zip` from your inbox since all those items are now listed."

```python
import os

# After all items from a source file are posted:
inbox_file = 'inbox/Photos-march-8-2026.zip'
if os.path.exists(inbox_file):
    os.remove(inbox_file)
    print(f"Cleaned up {inbox_file}")
```

If the inbox file contains photos for multiple items and only some have been posted, do NOT delete it yet — wait until all items from that file are posted.

## Workflow Flexibility

Not every run needs to hit all five steps. Common variations:

- **"Just post these to Craigslist"** — Skip Facebook, but still organize photos and track
- **"I already have listings written, just post them"** — Skip description writing, go straight to posting
- **"Add these items to my inventory"** — Just organize photos and update the spreadsheet, no posting yet (set status to "Draft")
- **"Mark this item as sold"** — Update the tracker spreadsheet with sold date and price
- **"I have new photos for an existing item"** — Add to the existing inventory folder

Adapt to what the user needs rather than rigidly following every step.

## Tips and Pitfalls

- **Always ask the user for photo upload** on both platforms. Never try to automate this — it will fail silently or throw permission errors
- **Take screenshots frequently** during the posting process to verify forms are filled correctly before submitting
- **If a platform's UI changes**, the form field names or page flow might be different. Use `read_page` to inspect the current form structure and adapt
- **Bundle deals** are great for moving multiple related items (e.g., matching furniture pieces)
- **CL posts expire** — remind the user they may need to renew listings after ~30 days
- **Facebook Marketplace** sometimes requires the user to set a delivery radius or accept terms — handle these as they come up

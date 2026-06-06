# Sell Stuff

A plugin for selling household items online. It bundles two skills that cover the full lifecycle of a listing, from first photo to sold.

## Installing in Claude Cowork

This repo is itself a plugin marketplace, so installing is two commands. In Claude (Cowork or Claude Code), run:

```
/plugin marketplace add mohammadshamma/sell-stuff-plugin
/plugin install sell-stuff@sell-stuff-marketplace
```

The first command registers this GitHub repo as a marketplace; the second installs the `sell-stuff` plugin (which includes both skills below) from it. After installing, start a new session — or reload — and the skills are available.

Two requirements for the skills to actually do their work:

- **A project folder.** The skills read and write a few files (an inventory spreadsheet, listing copy, photo folders) in whatever folder you point Cowork at. Create a folder for your selling project and select it when you start.
- **Browser logins.** Posting and updating listings happens through the browser, so you'll need to be logged into Facebook and Craigslist.

To update later, re-run `/plugin marketplace add` (it refetches), or use `/plugin` to manage installed plugins. To remove it, `/plugin uninstall sell-stuff@sell-stuff-marketplace`.

## Skills

The plugin includes two skills. You don't call them by name — they activate automatically when what you say matches what they do. Just describe what you want in plain language and the right skill kicks in.

### Sell Stuff — list new items end to end

Handles a listing from first photo to posted. Drop in photos and it identifies the item, converts iPhone HEIC photos to JPG, organizes them into numbered inventory folders, writes honest and searchable listing copy, posts to Facebook Marketplace and Craigslist, and records everything in your inventory tracker.

Invoke it with anything about selling or listing a new item, for example:

- "I want to sell these" / "post this on marketplace" / "help me sell my old furniture"
- "how much could I get for this?"
- "add these to my inventory"

### Listing Manager — manage items after they're posted

Handles the lifecycle after a listing goes live. Mark items as sold and sync that across both platforms (removing from Craigslist, marking sold on Facebook), and review unsold items to suggest and apply price drops on stale listings.

Invoke it with anything about updating or cleaning up existing listings, for example:

- "I sold the scanner for $35" / "mark item 4 as sold"
- "what hasn't sold yet?"
- "lower the prices on stale stuff" / "revise prices"
- "remove that listing" / "clean up sold items"

## Working files

The skills read and write three things in your selling project folder:

- `inventory-tracker.xlsx` — the source-of-truth spreadsheet (item, brand, condition, price, status, platform status, dates, sold price, notes)
- `listings.md` — the human-readable listing copy for review before posting
- `inventory/NN-item-name/` — per-item photo folders; raw drops go in `inbox/`

## Notes

Posting uses the browser, so you'll need to be logged into Facebook and Craigslist. Photo uploads are done by you dragging the photos into the upload area — this is the most reliable method and the skill will prompt you at the right moment.

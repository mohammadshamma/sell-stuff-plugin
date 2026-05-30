# Sell Stuff

A plugin for selling household items online. It bundles two skills that cover the full lifecycle of a listing, from first photo to sold.

## What's inside

**Sell Stuff** — listing new items end to end. Drop in photos, and it identifies the item, converts iPhone HEIC photos to JPG, organizes them into numbered inventory folders, writes honest and searchable listing copy, posts to Facebook Marketplace and Craigslist, and records everything in your inventory tracker.

**Listing Manager** — managing items after they're posted. Mark items as sold and sync that across both platforms, and review unsold items to suggest and apply price drops on stale listings.

## How it triggers

You don't need to call these by name. The skills activate from natural requests like:

- "I want to sell these" / "post this on marketplace" / "how much could I get for this"
- "add these to my inventory"
- "I sold the scanner for $35" / "mark item 4 as sold"
- "what hasn't sold yet" / "lower the prices on stale stuff"

## Working files

The skills read and write three things in your selling project folder:

- `inventory-tracker.xlsx` — the source-of-truth spreadsheet (item, brand, condition, price, status, platform status, dates, sold price, notes)
- `listings.md` — the human-readable listing copy for review before posting
- `inventory/NN-item-name/` — per-item photo folders; raw drops go in `inbox/`

## Notes

Posting uses the browser, so you'll need to be logged into Facebook and Craigslist. Photo uploads are done by you dragging the photos into the upload area — this is the most reliable method and the skill will prompt you at the right moment.

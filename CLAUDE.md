# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page marketing/ordering website for **Asia To Go** (Skewers & Noodles), an Asian street food stall inside New Era Supermarket, Al Muraqqabat, Deira, Dubai. There is no build system, package manager, or framework — the entire site is one static HTML file plus a handful of images.

## Repository layout

- `index.html` — the entire site: `<head>` with all CSS in a single `<style>` block, then markup, then all JS in a single `<script>` block at the end of `<body>`. ~2000 lines total.
- `images/` — sauce photography (`bangkok-heat.png`, `tokyo-fire.png`, `shanghai-black-gold.png`, `sauces-grid.png`) referenced via relative `/images/...` paths.
- `favicon.png`, `favicon-32.png` — site favicon (also base64-inlined for the logo elsewhere).

There is no `package.json`, no bundler, no test suite, and no CI config. Development is done by editing `index.html` directly.

## Running / previewing locally

There's no dev server or build step. Serve the directory with any static file server and open it, e.g.:

```bash
python3 -m http.server 8000
```

Then visit `http://localhost:8000/index.html`.

## Deployment

The site is deployed on **Vercel** as a static site (root-served `index.html`; asset paths like `/images/...` and `/favicon.png` assume this). Vercel Web Analytics and Speed Insights are wired in via `<script defer src="/_vercel/insights/script.js">` and `/_vercel/speed-insights/script.js` at the bottom of `index.html` — these are injected by Vercel's platform, not files in this repo. There is no `vercel.json`; deployment relies on Vercel's default static-site detection.

## Architecture

Everything lives in `index.html` in three parts, in this order:

1. **`<style>` (head)** — all CSS, driven by CSS custom properties defined on `:root` (colors like `--red`, `--gold`, `--bangkok`/`--tokyo`/`--shanghai`/`--seoul` per-sauce accent colors, spacing, etc.). No CSS framework; everything is hand-written, mobile-first with a burger nav.

2. **Markup (body)** — page sections in this order: `nav` → `header.hero` → `#build` (cup builder) → `#sauces` (4 signature sauces) → an unlabeled rewards-teaser/stamp section → `#rewards` (loyalty card signup) → `#find` (location, socials, coupon capture) → footer. A coupon modal is also in the DOM, hidden until triggered.

3. **`<script>` (end of body)** — all site behavior, in this order:
   - **Supabase config**: a `supabase-js` client (loaded from a CDN `<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2">`) is created at the top with a hardcoded `SUPABASE_URL` and `SUPABASE_ANON` anon key. The anon key is intentionally public (Supabase anon keys are meant to be client-exposed; row-level security in Supabase is what actually protects data — there is no RLS config in this repo, it lives in the Supabase project itself).
   - **Constants**: `WA` (WhatsApp number for orders) and `PRICES` (skewer/juice add-on prices) are hardcoded here — this is the single place to update pricing or the order number.
   - **Nav burger toggle** and **scroll-reveal** (`IntersectionObserver` adding `.in` to `.reveal` elements).
   - **"Build Your Cup" order builder**: a plain JS state object `S` (`base`, `baseP`, `sauce`, `sk` skewer count, `juice`) mutated by click handlers on `[data-base]` / `[data-sauce]` elements and +/- buttons, re-rendered into the summary panel by a single `render()` function. Submitting builds a WhatsApp deep link (`wa.me/<WA>?text=...`) with the order summary URL-encoded — **there is no backend order system; ordering is "generate a WhatsApp message."**
   - **Stamp card UI**: `renderStamps(n)` draws a 9-stamp + 1-free grid.
   - **Rewards registration**: on submit, validates a UAE phone number (`uaeValid`, regex `^(?:\+?971|0)?5\d{8}$`), generates a random `CARD #NNNN` client-side (`makeCardNum`), and inserts a row into the Supabase `rewards` table (`name`, `phone`, `card_number`, `stamps`, `free_meals`). Duplicate phone numbers are caught via Postgres unique-violation error code `23505`.
   - **Coupon popup**: appears after a 14s timeout or on exit-intent (`mouseout` near the top of the viewport, `clientY < 10`), captures name/phone, inserts into the Supabase `leads` table (`name`, `phone`, `coupon_code`), and shows a fixed `ASIA15` code. The claimed coupon is stashed on `window.__coupon` and appended to the WhatsApp order message if the customer later orders.

There are two Supabase tables this code depends on: `rewards` and `leads`. Their schemas are **not defined in this repo** — they live in the Supabase project (accessed via `SUPABASE_URL`). When changing the fields inserted into either table, the corresponding table/columns must already exist in Supabase or the insert will fail.

## Conventions to follow when editing

- Keep everything in `index.html` — this project deliberately has no build step or module system. Don't introduce a bundler, framework, or split the CSS/JS into separate files unless explicitly asked.
- Bilingual code comments (English + Arabic) appear in the Supabase/config section of the script — match that style if adding comments near those blocks.
- Per-sauce colors are threaded through via CSS custom properties (`--bangkok`, `--tokyo`, `--shanghai`, `--seoul`) and referenced inline (`style="--sc: var(--bangkok)"` / `style="background: var(--tokyo)"`). Add new sauces by extending this pattern rather than hardcoding colors.
- Pricing and the order WhatsApp number are centralized in the `WA` and `PRICES` constants near the top of the main `<script>` block — update there, not by hunting through the markup.
- Images are referenced as root-relative paths (`/images/...`, `/favicon.png`), consistent with Vercel's root-served static deployment.

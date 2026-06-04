# TRMNL Affirmations Plugin — Design Spec

**Date**: 2026-06-03
**Status**: Approved (brainstorm complete)
**Target**: First TRMNL Private Plugin for `MattAltermatt/affirmations`

---

## Overview

A TRMNL e-ink Private Plugin that displays one of 13 personal affirmations on a 400×480 half-vertical canvas, rotating daily based on day-of-year. Content lives in a static JSON file served by GitHub Pages; the Liquid template lives in the TRMNL dashboard and renders server-side.

## Goals

- Display one affirmation per day on a TRMNL device, rotating through a curated list of 13 personal mantras.
- Self-contained: no backend, no scheduled jobs, no LLM, no API keys. Pure static hosting + Liquid.
- Iterable: editing an affirmation = one commit + push to GitHub. Editing the layout = one paste in the TRMNL dashboard.
- Shareable: a public preview URL anyone can click to see what the device is showing today.

## Non-goals

- Public TRMNL marketplace plugin (this is a Private Plugin for personal use).
- Multiple layouts (only `half_vertical` 400×480; other tabs in dashboard left empty / unused).
- LLM-generated or randomly-rotating content (fixed daily picker).
- Multi-device sync, user personalization, themes.
- Image / icon decoration on the card (text only).

---

## Architecture

```
+--------------+         poll          +----------------------+        fetch       +----------------------------------+
| TRMNL device | <---- every ~15 min --| TRMNL backend        | --- polling URL -->| GitHub Pages                     |
| (ESP32 e-ink)|                       |  - runs Liquid       |                    | MattAltermatt/affirmations       |
|              | <-- BMP image URL ----|  - renders HTML->BMP | <-- 13-item JSON --| affirmations.json (static)       |
+--------------+                       +----------------------+                    +----------------------------------+
```

Source: `help.trmnl.com/en/articles/9510536-private-plugins` — *"If you choose Polling, TRMNL will fetch this content on your behalf."*

Key consequences:
- No backend to run, no CORS, no auth — public static JSON is sufficient.
- Refresh interval is configured per-plugin in the TRMNL dashboard.
- If GitHub Pages ever has an outage, the device shows the last successful frame (TRMNL caches).

---

## Repo layout

```
MattAltermatt/affirmations/
├── index.html              # GH Pages root - preview/landing page (mock device frame + cycler)
├── affirmations.json       # the data (polling URL points here)
├── template/
│   └── half_vertical.html  # Liquid markup; mirror of what is pasted into TRMNL dashboard
├── docs/
│   └── superpowers/
│       └── specs/
│           └── 2026-06-03-trmnl-affirmations-design.md   # this file
├── README.md
└── .gitignore
```

## Public URLs (once GitHub Pages is enabled)

- Preview / landing → `https://mattaltermatt.github.io/affirmations/`
- Polling URL (paste into TRMNL dashboard) → `https://mattaltermatt.github.io/affirmations/affirmations.json`

---

## Data shape — `affirmations.json`

Wrapped under an `items` key so it surfaces as a Liquid variable. Periods added for consistent end-stop.

```json
{
  "items": [
    "Moderation.",
    "Be present.",
    "Be positive.",
    "Assume good intentions.",
    "Feelings over efficiency.",
    "Stick with it.",
    "You won't remember, but someone will.",
    "Celebrate firsts, celebrate lasts.",
    "Say nothing until you are sure.",
    "Ask, don't assume.",
    "Something is usually better than nothing.",
    "Strength is impressive, kindness is remembered.",
    "Take the hint."
  ]
}
```

## Rotation logic

```liquid
{% assign idx = 'now' | date: '%j' | minus: 0 | modulo: items.size %}
{% assign current = items[idx] %}
```

- `'now' | date: '%j'` — day-of-year (001–366) as string
- `minus: 0` — coerce string → int
- `modulo: items.size` — wrap to valid index (0–12 for 13 items)

Each affirmation surfaces ~28 days/year.

## Fit-to-box sizing

Bucket by string length, map to TRMNL framework `value--*` class:

```text
chars    -> size class           -> approx px    bucket count
-------------------------------------------------------------
<= 12    -> value--xxxlarge      -> ~120 px      3 entries
13-24    -> value--xxlarge       -> ~80 px       4 entries
25-40    -> value--xlarge        -> ~56 px       4 entries
41+      -> value--large         -> ~40 px       2 entries
```

## Liquid template — `template/half_vertical.html`

This file in git is the mirror of what gets pasted into the TRMNL dashboard's "Edit Markup" → `half_vertical` tab.

```liquid
{%- assign idx = 'now' | date: '%j' | minus: 0 | modulo: items.size -%}
{%- assign current = items[idx] -%}
{%- assign len = current | size -%}

{%- if len <= 12     -%}{%- assign size_class = 'value--xxxlarge' -%}
{%- elsif len <= 24  -%}{%- assign size_class = 'value--xxlarge'  -%}
{%- elsif len <= 40  -%}{%- assign size_class = 'value--xlarge'   -%}
{%- else             -%}{%- assign size_class = 'value--large'    -%}
{%- endif -%}

<div class="view view--half_vertical">
  <div class="layout layout--col layout--center gap--large">
    <span class="value {{ size_class }} text--center">{{ current }}</span>
  </div>
</div>
```

---

## Local dev preview — `index.html`

A single HTML page that:
- Fetches `./affirmations.json` (single source of truth shared with the polling URL).
- Picks today's index via the same `day_of_year % 13` math, implemented in 10 lines of JavaScript.
- Renders the card inside a 400×480 mock-device frame.
- Provides prev/next buttons to cycle through all 13 entries for visual sanity-check.
- Pulls TRMNL's `plugins.css` so styling matches the device render (CDN URL verified during impl).

Page chrome around the device frame: light shadow + rounded corners, a small index counter (`3 / 13`), prev/next buttons, and a "view source on GitHub" link.

Doubles as the public landing page for the repo — anyone visiting `mattaltermatt.github.io/affirmations/` sees today's affirmation in the mock device.

---

## TRMNL dashboard setup (one-time, manual)

1. Create new Private Plugin → "Screen Templating" type.
2. Set Strategy → Polling.
3. Set Polling URL → `https://mattaltermatt.github.io/affirmations/affirmations.json`.
4. Set Refresh interval → 15 min (or any; doesn't change rotation cadence since picker is day-based).
5. Click "Edit Markup" → `half_vertical` tab → paste contents of `template/half_vertical.html`.
6. Verify dashboard's live preview pane shows today's affirmation rendered.
7. Add plugin to a Playlist assigned to the device.

---

## Error handling

| Failure | Behavior | Mitigation |
| --- | --- | --- |
| GH Pages 404 / outage | TRMNL caches last successful frame; device shows it | None needed — auto-recovers |
| Liquid syntax error | Dashboard preview pane shows error before save | Catch in dashboard preview |
| Index out of range | Bounds-handled by `modulo: items.size` | Built-in |
| Empty `items[]` | Never ship empty list (modulo-by-zero) | Maintain ≥1 entry |

No retry logic, no fallback UI, no error logging. Static content, no auth, no rate limits.

---

## Verification approach

Three layers, increasing fidelity:

1. **Local preview** (`index.html` in Chrome) — eyeball all 13 entries; check fit, alignment, line breaks.
2. **TRMNL dashboard preview** — "Edit Markup" live-renders against the polling URL; compare against local preview.
3. **Real device** — install plugin via dashboard, watch a refresh cycle, confirm sharpness on e-ink.

No unit tests. Visual preview *is* the test.

---

## Open verification points (resolved during implementation)

- Exact framework class names: `layout--center`, `text--center` — confirm in `trmnl.com/framework/docs/3.1`. If missing, substitute with correct class or inline `style="text-align: center"`.
- Liquid response variable scoping: confirm `items` surfaces as a top-level Liquid variable (vs `data.items` or other wrapper). May require renaming the JSON top-level key.
- TRMNL `plugins.css` public CDN URL for use in `index.html` — confirm.

---

## Out of scope (deferred to backlog)

- Multi-layout support (`full`, `half_horizontal`, `quadrant`) — only `half_vertical` for v1.
- Custom typography beyond what the TRMNL framework provides.
- Per-affirmation size overrides in JSON (currently auto-computed from string length).
- Affirmation-specific decoration (icons, borders, dates, attribution).
- Time-of-day variants (greeting prefix, etc.).
- Public TRMNL marketplace submission.
- GitHub Actions to rotate / regenerate content on a schedule.

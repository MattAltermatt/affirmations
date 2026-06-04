# Affirmations — TRMNL Plugin

A personal TRMNL e-ink display plugin: one of 13 affirmations, rotated daily.

🌐 **Preview / landing:** https://mattaltermatt.github.io/affirmations/
📡 **Polling URL:** https://mattaltermatt.github.io/affirmations/affirmations.json

## How it works

- `affirmations.json` is a static list of 13 mantras.
- The TRMNL backend polls the JSON URL on the configured refresh interval.
- A Liquid template (pasted into the TRMNL dashboard) picks today's mantra by day-of-year and renders it as a 400×480 half-vertical card.
- The device receives a BMP image from the TRMNL backend and displays it.

See [`docs/superpowers/specs/2026-06-03-trmnl-affirmations-design.md`](docs/superpowers/specs/2026-06-03-trmnl-affirmations-design.md) for the full design.

## Editing the affirmations

1. Edit `affirmations.json`.
2. Commit and push to `main`.
3. GitHub Pages redeploys within ~1 minute.
4. The TRMNL backend pulls the new JSON on its next refresh cycle.

The Liquid template's fit-to-box logic auto-sizes based on string length (4 buckets — `value--xxxlarge` / `value--xxlarge` / `value--xlarge` / `value--large`).

## TRMNL dashboard setup (one-time)

1. In TRMNL dashboard, click **+ New Plugin**.
2. Choose **Screen Templating** type.
3. **Name:** `Affirmations`.
4. **Strategy:** Polling.
5. **Polling URL:** `https://mattaltermatt.github.io/affirmations/affirmations.json`
6. **Refresh interval:** 15 minutes (or any — doesn't affect rotation cadence since the picker is day-based).
7. Save.
8. Click **Edit Markup**.
9. Switch to the **half_vertical** tab.
10. Paste the contents of [`template/half_vertical.html`](template/half_vertical.html).
11. Save. Confirm the live preview pane shows today's affirmation rendered correctly.
    - **If the preview is blank or shows an error:** the polling response variable may be exposed under a different name than `items`. Check the dashboard's variable sandbox for the actual name (it may be `data.items`, `data[0]`, etc.) and adjust the first `assign` line of `template/half_vertical.html` accordingly — e.g. `{%- assign items = data.items -%}` at the top.
12. Add the plugin to a Playlist assigned to your device.

## Local development

```bash
# Serve the preview locally
python3 -m http.server 8080

# Open in Chrome:
# http://localhost:8080/
```

Use the prev/next buttons to cycle through all 13 entries and verify rendering for each.

# Dispatch R&D

R&D space for proposed changes to the Dispatch app (production repo lives at git.customd.com/urgent-couriers/despatchweb — never pushed to from here).

## Workflow
1. Build clickable mockups / prototypes here.
2. George reviews and tests via GitHub Pages preview.
3. Final deliverable to the developer is a Markdown spec describing the change — not code commits to despatchweb.

## Current mockup
- **Live preview:** https://deliver-different-testing.github.io/Dispatch/ (once Pages is enabled)
- **Source:** [index.html](./index.html)
- **Screenshots:** [mockups/](./mockups/)

### What's being mocked
Inbound-depot view for Dispatch:
- Left **Overview** lists inbound destination depots with collection job counts
- Clicking a depot updates:
  - **Run List** → inbound schedules heading into that depot
  - **Associated Jobs** → jobs on the selected schedule

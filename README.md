# Grist

**Convert coffee grind settings between grinders — using microns (median particle size) as the common currency.**

You know a recipe's setting on one grinder and want the equivalent on another. Grist normalizes every grinder's setting to a particle size in microns, then converts to the target grinder's **native notation** — clicks, numbered sub-steps, rotations, stepless dials, and more.

It's a **single, self-contained HTML file**: inline CSS + vanilla JS, zero external requests, zero build step. All your data lives in your browser's `localStorage`. It works fully offline — save `index.html` to disk and open it.

## Live demo

**https://soyboycore.github.io/grist/**

## Features

- **Instant converter** — pick source + target grinder, then set the value by typing, by ± detent buttons, or by dragging a slider, get the equivalent live (no "convert" button). Handles five setting schemes: simple steps, numbered sub-steps (e.g. Ode `3.2`), clicks, rotations-clicks (e.g. 1Zpresso `1.3.2`, Kingrinder `2.42`), and stepless.
- **Microns as a first-class value** — every conversion shows the intermediate particle size, and you can convert microns ⇄ any grinder directly.
- **Snapping + residual** — converting into a stepped grinder snaps to the nearest legal detent and shows the micron residual and any clamp warning.
- **Piecewise-linear calibration** — each grinder carries micron *anchors*; non-linear grinders (Encore ESP's two-zone burr, Niche's sensitive espresso end) are modeled with extra anchors, all user-editable.
- **Multi-target mode** — see one setting across every favorite grinder at once ("I'm at a friend's place").
- **Brew-method targets** — pick a method, see the recommended setting range (native + microns) for your grinder; send the midpoint into the converter.
- **Dial-in helper** — sour/weak → finer, bitter/harsh → coarser, sized in your grinder's native steps.
- **Visual micron scale** — inline SVG chart on a shared 0–1500 µm axis: one row per brew-method target and one per grinder, with a marker for the current setting. Sized to the measured container width so labels stay legible on a phone.
- **Custom grinder wizard** — add any grinder you own in under a minute (name → scheme → config → calibration).
- **Saved setups** — named recipes (grinder + setting + method + dose/ratio + notes).
- **Export / import** — back up or share everything as a versioned JSON file.
- **Light / dark / auto** — follows your system by default, with an explicit override in the header that persists.
- Mobile-first, large touch targets, no external fonts.

## Adding a custom grinder (3 lines)

1. Open **My grinders → + Add**.
2. Enter a name, pick the setting scheme (with plain-language examples), and fill in its config (ranges / sub-steps / clicks-per-rotation).
3. Calibrate — give min/max microns, µm-per-step, a free-form anchor list, or a "reference brew" shortcut ("my V60 setting is 24"). Two anchors minimum; add more for non-linear grinders.

Your grinder then appears in every picker and survives reloads.

## Accuracy disclaimer

Results are **starting points, not gospel.** Burr wear, alignment, zero-point drift, and bean density mean the final 10% is always dialed by taste. Hand grinders measure clicks from the point where the burrs touch — that zero point varies unit to unit, so recalibrate the `0` anchor to your grinder. Grinders marked with a **community estimate** badge use best-available published figures; edit their anchors to correct them for your setup.

## Data sources & credits

Calibration ranges based on published data from Honest Coffee Guide (honestcoffeeguide.com) and manufacturer manuals; values are community estimates — dial in by taste. Grist does not scrape or hotlink any external data at runtime; all figures are baked into the file and are fully user-editable.

## License

MIT — © 2026 soyboycore. See [LICENSE](LICENSE).

## Deploying

No build step, no CI: push to `main` and GitHub Pages serves `index.html` at the repository root.

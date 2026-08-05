# Implementation Brief: Grind Size Converter

Build **GrindBridge** — an interactive coffee grind-size converter as a **single, self-contained HTML file** (inline CSS + vanilla JS, zero external requests, zero build step). English UI. All user data in `localStorage`. It must work fully offline once saved to disk.

## 1. Product goal

A home barista owns several grinders (e.g. Fellow Ode Gen 2, Comandante C40, xBloom Studio). They know a recipe's setting on one grinder and want the equivalent setting on another. The tool converts between grinders using **microns (median particle size) as the common currency**, supports **user-defined custom grinders**, and helps dial in.

Success = a conversion takes < 5 seconds and < 3 interactions, results always display in the target grinder's **native notation**, and the user can add any grinder they own in under a minute.

## 2. The hard part: setting schemes are not uniform

Grinders express settings in incompatible notations. The tool MUST model these as first-class scheme types, each with a parser and formatter ("notation codec"). Internally, every setting normalizes to a single number: the **position index** (0 … N−1 for stepped grinders, continuous float for stepless).

| Scheme type | Notation example | Grinders | Codec rules |
|---|---|---|---|
| `simple-steps` | `18` | Baratza Encore, Wilfa Uniform | Integers `min…max`, step 1 |
| `numbered-substeps` | `3.2` | Fellow Ode Gen 2 (main 1–11, 2 sub-detents between mains), Fellow Opus | Main numbers with `k` sub-detents between consecutive mains. Ode Gen 2: `1, 1.1, 1.2, 2, 2.1, 2.2, … 10.2, 11` = 31 positions (the last main has no sub-steps). Position index = `(main − firstMain) × (k+1) + sub`. Reject `1.3` when k=2. |
| `clicks` | `24 clicks` | Comandante C40, Timemore C2/C3, Hario Skerton Pro | Integer clicks from user-calibrated burrs-touching zero |
| `rotations-clicks` | `1.3.2` or `2.42` | 1Zpresso (rotation.number.tick), Kingrinder K6 (rotation.click) | Configurable: clicks per rotation, and display style (`R.N.T` with numbersPerRotation × ticksPerNumber, or `R.C`). Position index = total clicks from zero. |
| `stepless` | `17.3` | Niche Zero, DF64, Eureka Mignon Specialita | Continuous float `min…max`, configurable display precision (0.1). No snapping. |

**Codec requirements:** `parse(string) → position | error` (with a helpful message, e.g. "Ode Gen 2 has no setting 1.3 — sub-steps are .1 and .2") and `format(position) → string`. Input fields must accept the native notation, not raw indices. For stepped grinders, conversion results **snap to the nearest legal detent** and show the residual: `≈ 2.1 (≈ 490 µm, target was 505 µm)`.

## 3. Conversion model

Each grinder carries an ordered list of **calibration anchors**: `[{ position, microns }, …]` (minimum 2). Map position→microns with **piecewise-linear interpolation** between anchors; microns→position with the inverse. Conversion = source setting → position → microns → target position → snap → format.

Why anchors, not a µm-per-step coefficient: several grinders are confirmed non-linear (Baratza Encore ESP has a fine espresso zone ~20 µm/click for settings 0–20 and a coarse zone ~90 µm/click for 21–40; Niche Zero is far more sensitive at the espresso end; Fellow Opus has a dual macro/micro mechanism at the fine end). Anchors make non-linearity a data problem, not a code problem.

Rules:
- Out-of-range results: clamp to the grinder's min/max and show a clear warning ("finer than the Ode Gen 2 can go — it bottoms out at ~275 µm").
- Every conversion displays the intermediate micron value prominently. Microns are also directly enterable (micron ⇄ any grinder).
- All calibration data — including built-in grinders — is **user-editable** (add/move/delete anchors), with a per-grinder "reset to defaults" action. Published micron data is approximate; editing is the correction mechanism.
- Show a permanent, small disclaimer: results are starting points; burr wear, alignment, zero-point drift, and bean density mean the final 10% is dialed by taste. Hand grinders (clicks from zero) get an extra hint that their zero point varies unit to unit.

## 4. Seed grinder database

Ship these built-in. Endpoint ranges and brew ranges for the first five were verified against honestcoffeeguide.com; entries flagged *(est.)* are best available estimates — ship them, but mark estimated grinders with a subtle "community estimate" badge in the UI.

Default anchors = two endpoints unless a middle anchor is given. Positions below are given in native notation; convert to position indices when encoding.

| Grinder | Scheme | Settings range | Micron anchors | Brew ranges (native settings) |
|---|---|---|---|---|
| **Fellow Ode Gen 2** | numbered-substeps (mains 1–11, 2 subs, 31 positions) | 1 … 11 | 1 → 275, 11 → 1160 | Espresso 1–2 (not recommended, filter grinder); AeroPress 1.2–8.2; V60 2.2–5.2; Pour over 2.2–8.1; Drip 1.1–8; French press 6–11; Cold brew 7–11 |
| **Fellow Opus** | numbered-substeps (mains 1–11, 3 subs, 41 positions) | 1 … 11 | 1 → 230, 11 → 1160 | Espresso 1–2.5; AeroPress 2–8.75; V60 3–6; Pour over 3–8.5; Drip 2–8; French press 6–11; Cold brew 7.25–11 |
| **xBloom Studio** | simple-steps | 0 … 80 | 0 → 180, 80 → 1050 | Espresso 0–18; AeroPress 13–71; V60 21–47; Pour over 22–68; Drip 12–66; French press 47–80; Cold brew 58–80 |
| **Comandante C40 MK4** | clicks | 0 … 40 | 0 → 0, 40 → 1090 | Espresso 7–13; AeroPress 12–35; V60 15–25; Pour over 16–34; Drip 12–33; French press 26–40; Cold brew 30–40 |
| **Baratza Encore ESP** | simple-steps | 0 … 40 | 0 → 230, 20 → ~480 *(est., two-zone)*, 40 → 1380 | Espresso 0–13; AeroPress 8–30; V60 15–25; Pour over 16–30; Drip 7–29; French press 25–38; Cold brew 27–40 |
| Baratza Encore *(est.)* | simple-steps | 1 … 40 | 1 → 250, 40 → 1200 | Filter-only grinder |
| Comandante C40 + Red Clix *(est.)* | clicks | 0 … 80 | 0 → 0, 80 → 1090 | Same micron range as C40 at double click resolution |
| 1Zpresso JX-Pro *(est.)* | rotations-clicks (R.N.T, 40 clicks/rotation, 10 numbers × 4 ticks) | 0 … 2 rotations (80 clicks) | 0 → 0, 80 → 915 | Espresso 12–20 clicks; AeroPress 28–32; V60 32–34; Chemex 36–38; Pour over 32–40 |
| 1Zpresso K-Ultra *(est., LOW confidence — sources conflict on clicks/rotation vs µm/click; verify or let user recalibrate)* | rotations-clicks | 0 … ~4 rotations | endpoint ~0 → 0, max → 760 | Espresso ~2 rotations + 7; Pour over ~7.0–9.5 |
| Timemore C3 *(est.)* | clicks | 0 … 36 | 6 → ~250, 30 → ~950 (avoid < 6 clicks — burr contact) | Espresso 7–9; AeroPress 11–15; Pour over 13–15; French press 16+ |
| Kingrinder K6 *(est.)* | rotations-clicks (R.C, 60 clicks/rotation, 3 rotations) | 0 … 180 clicks | 0 → 0, 180 → 1350 (≈16 µm/click) | Espresso 15–25 clicks; French press 110+ |
| Niche Zero *(est., non-linear at fine end)* | stepless | 0 … 50 | 0 → 0, 15 → ~300, 50 → 1400 | Espresso 10–20; AeroPress 20–28; Pour over 30–40; French press 45–50 |
| DF64 Gen 2 *(est.)* | stepless | 0 … 90 | 0 → 180, 90 → 1050 | Espresso 10–25 |
| Wilfa Uniform *(est.)* | simple-steps | 1 … 41 | 1 → 220, 41 → 1010 | Espresso ~5 |
| Eureka Mignon Specialita *(est.)* | stepless (display as dial number, ~25 µm per mark) | 0 … 10 (per rotation dial; treat as continuous 0–48 marks) | 0 → 195, max → 1400 | Espresso-focused |
| Hario Skerton Pro *(est.)* | clicks (24/rotation) | 0 … ~11 | 0 → 350, 11 → 1400 (~100 µm/click) | Filter only |

Encode this as a `SEED_GRINDERS` JSON array with fields: `id`, `name`, `scheme` (type + config), `anchors`, `brewRanges` (in native notation), `confidence` (`verified` | `estimate`), `notes` (e.g. zero-point caveat for hand grinders).

## 5. Brew-method micron targets (global defaults, editable)

| Method | Target µm |
|---|---|
| Turkish | 40–220 |
| Espresso | 180–380 |
| Moka pot | 360–660 |
| AeroPress | 480–800 |
| Pour over / V60 | 400–950 (sweet spot ~500–700) |
| Drip machine | 500–900 |
| Chemex | 600–900 |
| Cupping | 600–800 |
| French press | 690–1300 |
| Cold brew | 900–1500 |

When a grinder has its own verified `brewRanges`, prefer those for that grinder; otherwise derive the setting range from these micron targets via the anchors.

## 6. Features

1. **Converter (main view).** Source grinder + setting in native notation (validated as you type) → live result for a chosen target grinder. No "convert" button — updates instantly. Show: target setting (snapped, native notation), micron value, and the snap residual. Support "microns" as a pseudo-grinder on either side. A swap button reverses direction.
2. **Multi-target mode.** Optionally show the conversion to *all* saved/favorite grinders at once as a list — this is the "I'm at a friend's place" use case.
3. **Brew-method targets.** Pick a method → see the recommended setting range on the selected grinder (native notation) with the micron range. One tap sends the range midpoint into the converter.
4. **Dial-in helper.** User picks current brew method + taste problem: *sour/weak/underextracted* → go finer; *bitter/harsh/overextracted* → go coarser. Suggest a concrete adjustment in the grinder's native steps, sized from the anchors: ≈30–50 µm for espresso range, ≈80–150 µm for filter range (e.g. "go 2 clicks finer on the Comandante" / "move from 5 to 4.2 on the Ode"). Include the standard rule of thumb: change one variable at a time.
5. **Visual micron scale.** A horizontal micron axis (0–1500 µm, log or linear — pick what reads better with the espresso zone visible) showing: brew-method target bands as background zones, each selected grinder as a row with its covered range, and a marker for the current setting on each row. This makes every conversion visually verifiable. Pure inline SVG.
6. **Custom grinder wizard.** Stepper flow: name → scheme type (the 5 types above, with plain-language descriptions and examples) → scheme config (ranges, sub-steps, clicks per rotation…) → calibration. Calibration accepts any of: (a) min/max microns, (b) µm per step + finest-setting µm, (c) free-form anchor list, or (d) "reference brew" shortcut — "my V60 setting is 24" pins an anchor at the V60 midpoint (~600 µm). At least 2 anchors required; more anchors = better accuracy for non-linear grinders.
7. **Saved setups.** Named recipes: grinder + setting + brew method + dose/ratio (free text) + notes. List view with one-tap "load into converter". localStorage.
8. **Data management.** Export/import everything (custom grinders, edited calibrations, setups) as a JSON file for backup/sharing. Version the schema (`schemaVersion: 1`).

## 7. UI/UX

- Mobile-first (this gets used standing at a kitchen counter). Large touch targets, numeric keyboards (`inputmode`), grinder pickers as searchable selects.
- Clean, warm coffee-shop aesthetic; light + dark mode via `prefers-color-scheme`. No external fonts.
- For stepped grinders, offer stepper buttons (−/+ one detent) next to the setting input — faster than typing on mobile, and it guarantees legal values.
- Tabs or segmented nav: **Convert · Targets · Dial in · My grinders · Setups**.
- Empty states that teach: first run shows a worked example (Ode Gen 2 "3" → Comandante).

## 8. Acceptance tests (must pass before done)

1. **Notation:** Ode Gen 2 accepts `3.2`, rejects `3.3` and `11.1` with helpful errors; the position after `2.2` is `3`. JX-Pro formats 54 clicks as `1.3.2`.
2. **Round trip:** any setting → microns → same grinder returns the identical setting for every seed grinder.
3. **Cross-check:** Ode Gen 2 `2.2` and Comandante `15 clicks` both land within ±40 µm of each other (~410–425 µm) — the verified V60 range bottoms for both.
4. **Snapping:** converting into a stepped grinder always yields a legal detent and displays the µm residual.
5. **Clamping:** converting an espresso setting (e.g. 250 µm) into the Ode Gen 2 clamps to setting 1 with a visible warning.
6. **Non-linear:** Encore ESP with 3 anchors interpolates piecewise (setting 10 ≈ 355 µm, not the linear-two-point value).
7. **Custom grinder:** create a clicks-type grinder with anchors (0→0, 30→900); setting 15 converts to ~450 µm; it appears in all pickers and survives reload.
8. **Persistence:** edited built-in calibration survives reload; "reset to defaults" restores it; export→wipe→import restores everything.
9. The file works from `file://` with network disabled; no console errors.

## 9. Publishing & repo deliverables

The repo will be public on GitHub and served via GitHub Pages. Alongside the app, produce:

- **`index.html`** — the app itself (named exactly this so Pages serves it at the repo root URL).
- **`LICENSE`** — MIT, exactly `Copyright (c) 2026 soyboycore`. Do not use any other name anywhere in the repo.
- **`README.md`** — what the tool is, a "Live demo" link (`https://soyboycore.github.io/grind-converter/`), feature list, how to add a custom grinder (3 lines), the accuracy disclaimer, a **Data sources & credits** section ("Calibration ranges based on published data from Honest Coffee Guide (honestcoffeeguide.com) and manufacturer manuals; values are community estimates — dial in by taste"), and the license note.
- **In-app footer** — small credits line with the same attribution, a link to `https://github.com/soyboycore/grind-converter`, and the app version (e.g. `v1.0.0`, kept in one constant).
- **Attribution constraint:** the author is `soyboycore` everywhere — LICENSE, README, footer, meta tags, code comments. Never write a real personal name or any company name into this repo.
- Keep the "community estimate" badges and disclaimer from §4 — they matter for publishing honestly.

No build step and no CI needed: deployment is "push to main, Pages serves the root".

## 10. Non-goals

No accounts, no backend, no analytics, no particle-photo analysis, no non-English locales, no grinder images. Don't scrape or hotlink external data at runtime.

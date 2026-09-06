# AGENTS.md — kalkulator_pizza.html

Technical notes for future development of `kalkulator_pizza.html`. Written right after the session in which the tool was built (2026-09-06) — covers not just "what is" but "why", plus mistakes already made and fixed once so they don't come back.

## What this is

A self-contained HTML+CSS+JS file (no external dependencies, no build step, no node) that computes:
1. **Dough composition** (Baker's %) from sliders: number of balls, ball weight, flour type, hydration, yeast modifier.
2. **Fermentation schedule** worked backward from bake time: mixing → bulk (kitchen) → divide into balls → main phase (fridge OR pantry) → finish (kitchen) → bake.

Tied to `pizza_napoletana.md` in the same directory — that's where every reference value in the model comes from. Opened directly in a browser (`file://...`), no server needed.

## Fermentation model — the most important part to understand

Base rule: **fermentation rate roughly doubles every ~10°C** (Q10≈2, an approximation, not hard physics).

```
rateRatio(T) = 2 ^ ((T - Tref) / doublePer)
```

defaults: `Tref=14°C`, `doublePer=10°C` (editable in the "Advanced settings" section).

### Three phases, INDEPENDENT of each other

This is the key part, and **it was the source of a real bug** found during verification (see the comment in `computeSchedule()` in the code):

- **Bulk** (kitchen): base `2h @ 21°C` — from the recipe (`pizza_napoletana.md`, "Fermentacja Zbiorcza" section).
- **Main phase** (fridge or pantry): base `21h @ 14°C` — from the recipe ("Fermentacja w Spiżarni" section, 20-22h, midpoint of 21h used). Also scaled by the yeast modifier: `mainEquiv = 21 × (100 / yeastPct)`.
- **Finish** (kitchen, before baking): base `2.5h @ 24.5°C` — **this is not from the recipe, it's a real calibration point from a kitchen test** (good cornicione, dough easy to stretch — see "Lessons learned" below).

**The first version of the code subtracted bulk+finish from a shared 21h pool**, treating 21h as the budget for the WHOLE process. That was wrong — the recipe explicitly states 20-22h is the duration of the main phase *alone*; bulk and finish are counted separately and ADD to it, they are not subtracted from it. Verification caught the bug: with fridge=14°C (i.e. identical to the recipe's own temperature), the schedule should yield ~21h for the main phase, but it produced ~12.5h. If anyone (human or agent) ever "simplifies" this model by merging the three phases into one shared pool of hours to divide up — **that is exactly the same bug, don't do it**. Always verify by setting the main phase temperature = Tref and checking that the result is exactly the base value (21h).

### Dough composition (Baker's %)

```
totalDough = balls × ballMass
yeastEffectivePct = 0.3 × (yeastPct / 100)   // 0.3% is the recipe baseline at 100% on the slider
saltPct = 3                                    // fixed, from the recipe, no UI for it
multiplier = 1 + hydration/100 + saltPct/100 + yeastEffectivePct/100
flour = totalDough / multiplier
water = flour × hydration/100
salt  = flour × saltPct/100
yeast = flour × yeastEffectivePct/100
```

Flour type affects **only** the default hydration (and shows a warning for type 450), not the fermentation-time model — that's a deliberate decision from planning, not a gap. Presets: tipo "00" → 62%, typ 500 → 60%, typ 450 → 58% (cake flour, weak gluten, warning shown in UI).

## Application state (`state`)

A single global object, entirely persisted to `localStorage` under the key `pizzaCalcState` (see `loadState()`/`saveState()`, called at the end of `render()`). Any field added to `state` is automatically saved/restored with no extra work — `Object.assign(state, saved)` in `loadState()`.

Fields: `lang, balls, ballMass, flour, hydration, yeastPct, kitchenT, pantryT, fridgeT, bakeTime, path, advTref, advDouble, advBaseMain, advBulkH, advBulkT, advFinalH, advFinalT`.

`render()` is the only rendering function — called after EVERY change (every input has an `addEventListener("input", ...)` that mutates `state` and calls `render()`). There's no diffing/reactivity — the whole DOM is rewritten from scratch on every render (simple, and sufficient at this UI scale).

## i18n system

`I18N = { pl: {...}, en: {...} }` — a text dictionary, 61 keys, **must stay symmetric between pl and en** (if you add a key, add it in both). Check with:

```bash
python3 -c "
import re
c = open('kalkulator_pizza.html').read()
pl = set(re.findall(r'^\s*(\w+):', re.search(r'pl: \{(.*?)\n  \},\n  en:', c, re.S).group(1), re.M))
en = set(re.findall(r'^\s*(\w+):', re.search(r'en: \{(.*?)\n  \}\n\};', c, re.S).group(1), re.M))
print('only pl:', pl-en, 'only en:', en-pl)
"
```

Two ways it's used:
- **Static labels** (headings, field labels, buttons) — `data-i18n="key"` attribute in the HTML, swapped automatically by `applyStaticI18n()` at the start of `render()`.
- **Dynamic sentences** (schedule, warnings, calculation details) — the `t(key, {param: value})` function, substitutes `{param}` inside the string.

Flour type names (tipo "00", typ 500, typ 450) and the season preset buttons **are deliberately left untranslated / with values baked into the text** — those are product names / fixed numbers, not sentences.

Polish grammatical cases: `pathWords.fridge/pantry` have three forms `{nom, gen, loc}` (nominative/genitive/locative) — because "Balls into: Fridge" / "Remove from the fridge" / "12h in the fridge" each need a different case in Polish. In English all three forms are identical (fridge/pantry don't decline), but the structure has to stay the same for both languages because the code (`pw.nom`, `pw.gen`, `pw.loc`) doesn't branch on language.

**A trap I already fell into once**: the `dayTag()` function had a local variable `const t = new Date(...)`, which **shadowed the global `t()` function** (the i18n helper) for that function's whole scope. Result: even writing `t("day_prev")` inside `dayTag` would have blown up (trying to call a Date as a function). The variable was renamed to `td`. **If you add a new helper function, watch out for naming local variables `t` — that name is taken globally.**

## Bake time — native `<input type="time">`

Deliberately NOT implemented as a custom HH:MM input (the user didn't want that — preferred the simplicity of the native input over custom time-handling logic). Problem: the 12h/24h format of a native `<input type="time">` depends on the input's `lang`, not the page's language state, and `en-US` defaults to showing AM/PM. Fixed in `render()`:

```js
$("bakeTime").lang = state.lang === "en" ? "en-GB" : "pl";
```

`en-GB` (not `en-US`) forces 24h even in English mode. This depends on how the browser interprets the element's `lang` attribute (Chrome/Edge respect it; behavior may differ across engines — not tested in Firefox/Safari during this session).

## Known environment limitations (not the file's fault)

Headless Chromium in this dev environment is **unreliable** — `chromium-browser --headless --dump-dom` returned empty output with no error, intermittently (worked once, mostly didn't). Don't waste time chasing this on future changes. Instead:
- Re-derive the formulas independently in Python (see this session's history for an example — a plain-Python replica of `rateRatio`/`computeSchedule`, compared against the recipe's expected result).
- Check statically: `grep`/`python3 -c` for balanced `{}/()/<div></div>`, no dangling references after a refactor (`grep` for old variable/key names), i18n key symmetry.
- Ask the user to open it in a browser manually for a visual check — that worked reliably in this session, unlike automation inside this sandbox.

## Deliberately left out / rejected during planning

- **Ambient humidity** (winter 40% / summer 60%) — was in the original request, removed at the user's request: no reliable coefficient linking humidity to hydration, and pseudo-precision wasn't wanted.
- **A separate "physical warm-through" model** for a dough ball (mass → heat-conduction time, independent of fermentation) — considered after the cornicione lesson (cold dough stretches poorly even with good fermentation), but the user chose the simpler, single Q10 model over a two-dimensional one. The finish base (2.5h @ 24.5°C) partly compensates for this empirically, but doesn't scale correctly with ball mass — if `ballMass` in the UI ever grows significantly (e.g. >300g), this is worth revisiting.
- **Custom HH:MM time picker** — implemented, then deliberately rolled back at the user's request (preferred the native input plus a forced locale over custom logic).

## Ideas for future development (not discussed, just suggestions)

- Export/import settings as JSON (useful when moving between devices — `localStorage` is per-browser).
- Logging a series of real outcomes (date, settings, quality rating) to calibrate against — currently calibration (`advBulkH/T`, `advFinalH/T`) is manual, editable in the UI, but nothing suggests NEW values based on history.
- The "inconvenient time" warning only checks `bulkStart` (mixing at night) — could also check `mainOut` (removing from fridge/pantry at night).
- If another flour type is added, add it to `FLOUR_PRESETS` (hydration + optional `warnKey` pointing to an i18n key with the warning) — the rest (composition table, UI) wires up automatically.
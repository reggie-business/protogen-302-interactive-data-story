# BUILD SPEC — The Coin-Flip Winter (P302)

How to use: hand each stage to Copilot agent mode as one block. Review what comes back,
redirect in plain language, then commit with the message at the end of the stage and deploy.
Stages are large and self-contained on purpose — fewer round-trips. The teaching notes (›) are
for you, not Copilot; they explain the layer underneath so you can direct and review with intent.

Pre-flight: `BRIEF.md` and `winters.json` in hand. GitHub repo already created (yours, named
by you). Keep this file at `docs/BUILD_SPEC.md` — it counts as AI scaffolding in the rubric.

---

## DESIGN TOKENS (grounded in a Minneapolis winter night, not generic dark mode)

The signature element is the **widening band** — spend all boldness there, keep the rest quiet.

```
Color (named hex):
  --night    #0E141B   deep overcast winter-night sky — the canvas (NOT pure black)
  --frost    #EDF1F5   snow-under-cloud off-white — primary text/surface (NOT warm cream)
  --slate    #6E8CA8   muted overcast blue — default data color (the dots, the calm years)
  --deep     #2E4A66   frozen-lake navy — structure, axes, the reference line
  --amber    #E8A13C   sodium-streetlight amber — the ONE warm color. Means "warming" every
                       time it appears: the recent swing, the warm edge of the band, 2024.
  --ice      #BFE3F2   bright cold cyan-white — reserved for the record-cold accent (2014)
```
› One warm accent, one cold accent, used only to *mean* something. If amber ever appears as
  mere decoration, it's wrong.

```
Type (three roles, each justified):
  Display : a wide/strong grotesque (e.g. Archivo or Archivo Expanded) — beat headlines.
            NOT a high-contrast Didone serif (that's the default look).
  Body    : a readable serif (e.g. Source Serif 4) — narration. Editorial, calm.
  Data    : a monospace (e.g. IBM Plex Mono) — every number/readout. Mono = instrument
            readings, which is literally what these are. This is the subject-grounded choice.
  Scale   : 13 / 15 / 18 / 24 / 40 / 64. Body line-height 1.6.
```

› The one aesthetic risk worth taking (optional, go-further): render the band's recent edge as
  slightly *fractured* rather than a smooth ribbon — ice cracking as the system loses its
  footing. Justified by the subject. Only if it reads as intentional, never as noise.

Quality floor (don't announce it, just hit it): responsive to phone width, visible keyboard
focus, `prefers-reduced-motion` respected.

---

## STAGE 0 — Bootstrap (run by hand in terminal)

```bash
# Scaffold Vite + Vue + TS. Say NO to pinia, testing, jsx, eslint, prettier.
npm create vite@latest        # name it yours; framework: Vue; variant: TypeScript
cd <your-app>
npm install

mkdir -p src/data docs
# Move your two artifacts in:
#   winters.json  -> src/data/winters.json
#   BRIEF.md      -> repo root
#   this file     -> docs/BUILD_SPEC.md

git init
git add -A && git commit -m "Scaffold Vite+Vue+TS, add brief and winter dataset"
git branch -M main
git remote add origin <your-repo-url>
git push -u origin main
```
› If Copilot later scaffolds into a subfolder instead of root, tell it to move everything to root.

---

## STAGE 1 — Story shell + tokens + scroll structure

> Build the shell of an interactive scrollytelling data story in Vue 3 + TS. Read BRIEF.md and
> docs/BUILD_SPEC.md first and follow the design tokens exactly.
>
> Layout: a single centered column. A **sticky chart stage** (position: sticky, full viewport
> height) on top of which a chart will later render. Below/over it, **four full-height text
> beats** in order: (1) The fortress, (2) The step, (3) The swing, (4) The payoff. Use the beat
> intent from BRIEF.md for placeholder copy — short, declarative, active voice, sentence case.
>
> Wire an **IntersectionObserver** that sets a reactive `activeBeat` (1–4) as each text beat
> scrolls into view. For now render a placeholder box in the chart stage that prints the current
> `activeBeat`. Apply the color and type tokens as CSS custom properties on :root and load the
> three Google fonts.

› What's happening: `activeBeat` is one reactive number. The whole chart will key off it — each
  beat just changes what the same chart shows. Sticky-positioning the chart is what makes the
  visual hold still while the words scroll past it; that's the entire scrollytelling trick.

Commit: `Add story shell, design tokens, and scroll-driven beat tracking`. Deploy to Vercel.

---

## STAGE 2 — The static SVG chart

> Create `WinterChart.vue`. Import `src/data/winters.json` (75 rows of
> {year, avgLow, avgHigh, subzero}). Render a hand-built **SVG** scatter, not a chart library.
>
> - Use a `viewBox` so it scales responsively. Define padding for axes.
> - Two linear scales: x maps year (1951→2025) across the plot width; y maps temperature
>   (domain about -2°F→24°F) up the plot height — remember SVG y grows *downward*, so invert it.
> - Plot all 75 winters as small circles in `--slate`.
> - Sparse axis ticks (every ~15 years on x; a few °F gridlines on y) in `--deep`, hairline.
> - Draw a faint reference line at the overall mean winter low.

› The "scale" mystique is just this: `x = padLeft + (year - 1951)/(2025 - 1951) * plotWidth`,
  and `y = padTop + (maxT - temp)/(maxT - minT) * plotHeight`. Two ratios. That's all D3's
  `scaleLinear` does under the hood — you can read every dot's position by hand now.

Verify: the scatter is visible; recent dots are visibly more spread out than mid-century ones
(your thesis should be faintly legible even before any interaction). Commit:
`Add SVG winter scatter chart with linear scales`.

---

## STAGE 3 — Scroll-driven reveal layers

> Make `WinterChart.vue` take `activeBeat` as a prop and reveal layers as it changes. Same data,
> layers toggled — do not redraw from scratch.
>
> - Beat 1 (fortress): dots faint. Draw a single straight **least-squares trend line** through
>   avgLow vs year, labeled as the "steady ramp" everyone expects.
> - Beat 2 (step): fade the straight trend line down. Overlay the **eight decade-average points
>   as a step line** (1950s 8.1, 60s 7.7, 70s 7.0, 80s 10.0, 90s 11.5, 2000s 11.4, 2010s 10.1,
>   20s 10.4). The jump-then-plateau shape should be obvious. This is reveal #1.
> - Beat 3 (swing): draw the **spread band** — for each year, the min–max of the trailing 15
>   winters' avgLow, as a filled ribbon in translucent `--slate` with an `--amber` warm edge.
>   Highlight **2024** (warmest, 20.9°) in `--amber` and **2014** (coldest, 0.1°) in `--ice`,
>   each labeled. This is reveal #2 — the real story.
> - Beat 4: leave the band drawn; the scrubber (next stage) takes over.

› One straight line vs. the decade steps vs. the band — three ways of looking at the *same 75
  numbers*. The story is literally "here's what you assumed the shape was; here's the truer
  shape." The chart is the argument; the prose just narrates it.

Commit: `Add beat-driven reveal layers (trend, decade step, spread band)`. Deploy, scroll the
live site, confirm each beat changes the chart.

---

## STAGE 4 — The scrubber (the payoff)

> In beat 4, add an interactive **scrubber**: a range input (or custom drag handle) over the
> x-axis, bound to a reactive `scrubYear` (1951→2025).
>
> - The spread band is drawn **only up to `scrubYear`** — so as the reader drags right, the band
>   grows, and visibly *fattens* once it passes ~1990. This widening is the whole point.
> - A **playhead**: a vertical line at `scrubYear`, the current winter's dot enlarged, and a
>   **callout card** (mono type) showing `year · avg low · N nights below 0°F`, plus a tag when
>   it's the record warm (2024) or cold (2014) winter.
> - A **readout**: "Swing of the last 15 winters: X°F" = the band's height at `scrubYear`
>   (trailing-15 max − min). It should climb as the reader moves into recent decades.
> - A **play / pause** button that sweeps `scrubYear` automatically (requestAnimationFrame or a
>   timer), so the band blooms on its own. Disable the sweep under `prefers-reduced-motion`.
> - Edge state: before 15 winters exist (years < ~1965) the trailing window is shorter; let the
>   band be naturally thin there. If the reader hasn't moved yet, the readout shows the earliest
>   value, not an error.

› The elegant part: `scrubYear` is *one reactive number*. The band extent, the playhead, the
  callout, and the readout are all **computed from it** — change the number, everything
  recomputes. You're not wiring four things; you're wiring one source of truth and letting Vue's
  reactivity fan it out. That's the mental model worth keeping.

Commit: `Add interactive scrubber with growing spread band and playhead`.

### ⚠ Verification check (this is the pass/redo line)
On the **live** Vercel site, drag the scrubber end to end and confirm **all four** respond:
band widens, readout climbs, playhead moves, callout updates. A scrubber that moves the playhead
but leaves the band static is this project's version of the P301 "filter that didn't drive the
deltas" bug — and it's a Redo. Verify it live, not just on localhost.

---

## STAGE 5 — Polish, scaffolding, deploy

> - Responsive: chart `viewBox` scales; the scrubber stays thumb-usable at phone width; beats
>   stack cleanly. Test at ~380px.
> - Accessibility floor: visible focus ring on the scrubber; reduced-motion disables auto-sweep.
> - Footer (frost on night, small): "Temperature data: Open-Meteo Historical Weather API (ERA5),
>   CC BY 4.0. Corroborating context: Minnesota DNR State Climatology Office."
> - `README.md` in root: what the piece is, the finding in two sentences, the data source, and
>   how to run it. `LICENSE` in root.
> - Deploy to Vercel; optional password protection; confirm the live URL renders the final build.

Commit: `Add attribution footer, README, license; finalize responsive + a11y`.

---

## RUBRIC MAP (why this passes)

- **Does it work?** Live, gated, the scrubber drives all four elements end to end. ✓
- **Repo set up right?** BRIEF.md + docs/BUILD_SPEC.md (AI scaffolding) + README + LICENSE in
  root; staged, descriptively-committed history. ✓
- **Looks right / shows thinking?** Subject-grounded editorial design, one signature element;
  the BRIEF starts from a derived point of view and the build performs it. ✓ Go-further:
  fractured band edge, the "your winter" lookup, handled edge states.

## A note on how it'll really go

The build is staged for speed, but a data story lives in the editing — and here the editing is
mostly the **copy** and the **band's feel**. Expect to spend your real time tightening the four
beat headlines so the assume→reveal→reveal→payoff lands, and tuning the band so the widening is
unmistakable (opacity, the trailing-window size, the amber edge). That's the part worth doing
with a thought partner. Front-load the build; spend what's left making it *land*.

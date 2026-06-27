# BRIEF.md — The Coin-Flip Winter

*An interactive data story about how Minneapolis stopped being able to count on the cold.*
*(Working title — swappable.)*

## The point of view

Everyone "knows" the climate story: winters are warming, steadily, everywhere. This piece
shows that for Minneapolis the real story is stranger and sharper than the cliché.

> **Assume:** Minnesota winters are warming in a steady, predictable ramp.
> **Reveal:** They didn't ramp. Winter overnight lows stepped up about 4°F a generation ago
> (warming concentrated in the 1980s–early '90s), and then stopped trending — the last three
> decades barely move on average. What changed instead is the *spread*: winters started
> swinging far harder from one year to the next.
> **So-what:** The all-time coldest winter (2014) and the all-time warmest (2024) in 75 years
> both landed in the last decade. The deep freeze didn't vanish. It became a coin flip —
> and you can't plan, budget, or brace for a coin flip the way you can for a cold that's
> reliably there.

Every claim above is derived from this project's own data pull (see The Data). Nothing is
borrowed. Where an outside source corroborates a point (e.g., the Minnesota DNR's finding that
the state has seen the most winter warming in the contiguous US), it is cited as corroboration,
not as the source of the claim.

## Audience & context

- **Industry framing:** Media & Communications — public-interest data journalism.
- **Primary reader:** a Twin Cities resident who lives these winters and assumes the only
  trend is "milder." The piece should land emotionally for them in under two minutes of
  scrolling, no expertise required.
- **The stake (why it matters beyond interest):** unpredictability is a planning problem.
  A city that right-sizes its snow, salt, and heating-assistance budgets for "warmer winters"
  is planning for the wrong thing — the risk now is variance, not warmth. The story stays
  reader-facing, but this is the "so-what" that gives it teeth.

## The data

- **Source:** Open-Meteo Historical Weather API (ERA5 reanalysis), daily min/max temperatures
  for Minneapolis (44.98, -93.27), winters 1951–2025. No API key. Licence: CC BY 4.0.
- **What ERA5 is, stated plainly:** a ~25 km modeled reanalysis grid, not a single station
  thermometer. It is built for long-term consistency, which is exactly right for trend
  analysis — and it deliberately has no urban-heat-island effect baked in. (This is why our
  numbers differ from station-based figures like the DNR's, and naming that distinction is
  part of the point.)
- **Dataset shape:** `winters.json` — 75 rows of
  `{ year, avgLow, avgHigh, subzero }`, where a winter = Dec–Feb labeled by its January year,
  and `subzero` = nights at or below 0°F. Pre-computed locally so the app reads clean numbers.
- **The findings the build must reflect (all from this dataset):**
  - Winter overnight low: **+4.0°F since 1951** (daytime high +4.2°F — nearly identical, which
    is why we *dropped* a "nights warm faster than days" angle: it isn't true in this data).
  - Decade averages of winter low (°F): 1950s 8.1 · 1960s 7.7 · 1970s 7.0 · 1980s 10.0 ·
    1990s 11.5 · 2000s 11.4 · 2010s 10.1 · 2020s 10.4. → a **step up, then a plateau.**
  - Year-to-year whiplash roughly **doubled** (~2.7–3.9°F mid-century → 4.5–6.8°F since the '80s).
  - Decade range of winter lows widened from ~8–10°F to 14–20°F.
  - **Coldest winter: 2014 (avg low 0.1°F). Warmest: 2024 (20.9°F). Both in the last decade.**

## The story (four beats, top to bottom)

1. **The fortress.** Open on the belief: the eternal, brutal Minnesota winter, warming
   steadily like everywhere else. Set the cliché up to knock it down.
2. **The step.** Show the decade averages. Yes, it warmed ~4°F — but as a jump that ended
   around 1990, then flatlined. Not a ramp. First reveal.
3. **The swing.** The scatter blows open. Whiplash and range doubling; the coldest-ever and
   warmest-ever winters, both recent, on opposite ends. The real story.
4. **The payoff.** Hand the reader the unpredictability to feel for themselves — the scrubber.

## The interaction (one element, and it *is* the argument)

A **scrubber** the reader drags across the 75 winters.

- The chart shows winter avg overnight low by year (x = year, y = °F), all winters present as
  dots, with a faint reference line.
- Behind the line, a **shaded band shows the spread (min–max) of the trailing ~15 winters**,
  drawn up to the scrubber's position. Early decades: a thin ribbon. Past ~1990: it visibly
  fattens. The widening band is the evidence, made physical.
- A **playhead callout** names the scrubbed winter and its stats (avg low, sub-zero nights,
  and a tag when it's the record warm/cold). A **readout** shows the trailing-15 swing in °F,
  climbing as the reader moves right.
- A **play/pause** control sweeps the playhead automatically — for drama, and so a reviewer
  who doesn't drag still witnesses the band bloom.
- **Why this and not a filter/toggle:** the reader should *discover* the chaos by their own
  hand, not be told it. The interaction performs the thesis.

## Layout

Scrollytelling: four full-height beats stacked vertically, the chart pinned and evolving as
the reader scrolls into each beat, ending in the fully interactive scrubber. One column,
generous whitespace, the chart always the protagonist.

## Tech

- **Stack:** Vue 3 + Vite + TypeScript (continuing the P200/P301 toolchain).
- **Charting:** hand-built **SVG** inside a Vue component — full control over the band, the
  playhead, and the scrubber, which off-the-shelf chart libraries fight you on. Scales are
  simple linear maps (year→x, °F→y); the band is a filled polygon; the scrubber is one
  reactive number. D3 optional, for scale math only.
- **Deps kept minimal** so the AI build stays on rails.
- **Data:** `src/data/winters.json`, imported directly.

## Style (deliberate, swappable)

Editorial data-journalism, not a dashboard. The look should feel like a serious piece you'd
read, not a tool you'd operate.

- **Palette:** deep midnight/ice-blue base (winter, cold, night), off-white text, a single
  warm **amber/ember accent** reserved for the warming signal and the recent swing — so the
  one warm color *means* something every time it appears.
- **Type:** a strong serif for headlines (editorial weight) over a clean sans for labels and
  UI. Big, confident headline type per beat.
- **Tone of copy:** declarative, plainspoken, a touch literary. Short captions that state the
  finding, never hedge it into mush.
- *(This intentionally departs from the Avenir/Slalom look of P301 — a public data story wants
  its own skin. Swap if a consistent portfolio through-line matters more.)*

## Nice-to-haves / go-further

- Auto-play sweep on the scrubber (above).
- Responsive down to phone width (the rubric rewards it; the scrubber must stay thumb-usable).
- Graceful empty/edge states (e.g., the band before 15 winters exist).
- Optional secondary interaction: a "your winter" lookup — reader enters a meaningful year and
  sees where it fell. Only if it doesn't dilute the single clean scrubber.

## Attribution

Footer + README: "Temperature data: Open-Meteo Historical Weather API (ERA5), CC BY 4.0."
Corroborating context: Minnesota DNR State Climatology Office.

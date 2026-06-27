# The Coin-Flip Winter

An interactive data story about how Minneapolis winters stopped being predictable.

## The Finding

Minnesota winters warmed roughly 4°F over 75 years — but not as a steady climb. The warming concentrated in the 1980s and early '90s, then plateaued. What changed instead is volatility: winters now swing far harder from year to year. The coldest winter in 75 years (2014) and the warmest (2024) both arrived in the last decade. The deep freeze didn't vanish; it became a coin flip.

## Data Source

**Temperature data:** [Open-Meteo Historical Weather API](https://open-meteo.com/) (ERA5 reanalysis), daily min/max for Minneapolis 1951–2025. CC BY 4.0.  
**Corroborating context:** Minnesota DNR State Climatology Office.

Dataset: 75 winters (Dec–Feb) of average overnight lows, highs, and nights below 0°F.

## Stack

- **Vue 3** + TypeScript + Vite
- **Hand-built SVG** scatter (no chart library)
- **Responsive** design (desktop side-by-side, mobile stacked)

## Run Locally

```bash
npm install
npm run dev
```

Open http://127.0.0.1:5173/ in your browser.

### Build for production

```bash
npm run build
```

Output in `dist/`.

---

See [BRIEF.md](BRIEF.md) for the detailed story intent and [docs/BUILD_SPEC.md](docs/BUILD_SPEC.md) for the technical build stages.

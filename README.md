# DataVizTCO — The Monitoring Mirage
 
A scrollytelling data visualization about **hydrometeorological sensor inequality** in Ontario and across Canada, built for the Toronto Climate Observatory.
 
The core argument: where we don't monitor, we don't measure — and where we don't measure, disasters go undetected, unwarned, and under-recorded. The piece walks a reader from a handful of famous floods, through a game that exposes how blind our records can be, to a province-wide **W-score** (Hydrometeorological Intelligence Score) that quantifies the gaps.
 
🔗 **Live:** [https://<username>.github.io/DataVizTCO/](https://infiniteinbox.github.io/DataVizTCO/)
 
---
 
## The narrative
 
The story runs as a single scroll- and click-driven sequence (`ScrollStory.svelte`), moving through numbered scenes:
 
1. **Rain intro** — a Canvas 2D rainfall field with clickable droplets sized by damage cost, each a historical flood event (Hurricane Hazel, July 2013, Lake Ontario freshet).
2. **Guess the Flood** — the reader is shown real gauge readings for four days and asked whether a flood occurred. One round is deliberately unanswerable (no record exists), capping the best score at 3/4. That ceiling is the point.
3. **"Where a lack of monitoring caused harm"** — an interstitial, then a map of Canada with markers for real events where a radar gap, broken station, or missing gauge delayed or prevented a warning.
4. **Benefits sequence** — a typed-out list of what better monitoring unlocks, each with supporting media.
5. **Ontario stations** (`OntarioStations.svelte`) — a MapLibre map of the actual monitoring network and radar coverage.
6. **"It's not what it seems"** — interstitial into the W-score.
7. **W-score heatmap** (`WScoreMap.svelte`) — the province rendered as a PMTiles vector heatmap of monitoring intelligence, computed in the notebook.
8. **Disaster database** (`DisasterReports.svelte`) — external records showing disasters cluster where monitoring is densest, not where risk is highest.
---
 
## Tech stack
 
**Frontend**
- SvelteKit with Svelte 5 runes (`$state`, `$derived`, `$effect`)
- JavaScript with JSDoc (no TypeScript)
- [Scrollama](https://github.com/russellgoldenberg/scrollama) for scroll-driven scenes
- [D3.js](https://d3js.org/) for the Canada map projection (`geoConicConformal`)
- [MapLibre GL JS](https://maplibre.org/) for the interactive Ontario and W-score maps
- Canvas 2D API for the rainfall intro
**Data pipeline**
- Python / Jupyter (`sensorinequity_v2.ipynb`) — computes AHP-weighted W-scores across ~1M one-kilometre grid cells
- Shapely + GeoPandas for spatial work
- [tippecanoe](https://github.com/felt/tippecanoe) for `.pmtiles` generation
**Deployment**
- GitHub Pages via GitHub Actions
- `@sveltejs/adapter-static`
- Git LFS for `.pmtiles` files
---
 
## Project structure
 
```
tco_flood_history/
├── src/
│   ├── lib/
│   │   ├── ScrollStory.svelte      # main narrative + scene state machine
│   │   ├── OntarioStations.svelte  # MapLibre station/radar map
│   │   ├── WScoreMap.svelte        # PMTiles W-score heatmap
│   │   ├── DisasterReports.svelte  # disaster database scene
│   │   └── assets/                 # favicon, etc.
│   └── routes/
│       ├── +layout.svelte
│       └── +page.svelte            # mounts ScrollStory
├── static/
│   ├── canada.geojson
│   ├── data/
│   │   ├── game_rounds.json
│   │   ├── stations.geojson
│   │   ├── radar_coverage.geojson
│   │   ├── ontario_wscore.pmtiles  # tracked via Git LFS
│   │   └── ontario_wscore_bounds.json
│   └── *.jpg / *.gif               # benefit-card media
├── notebooks/
│   └── sensorinequity_v2.ipynb     # W-score computation
├── .nojekyll
└── README.md
```
 
---
 
## Data pipeline
 
The AHP (Analytic Hierarchy Process) pipeline lives entirely in Python and is **not** ported to JavaScript. The web app only ever renders pre-computed outputs.
 
The notebook (`sensorinequity_v2.ipynb`) builds a W-score for each ~1 km grid cell by combining several monitoring networks under AHP weights. Precipitation, hourly, and daily networks use a Gaussian distance-decay kernel; **streamflow is handled separately** — broadcast across its watershed rather than by distance from the gauge, since a gauge speaks for its whole basin.
 
Exports feed the frontend as static files:
 
| Output | Format | Used by |
|---|---|---|
| Ontario W-score surface | `.pmtiles` (via tippecanoe) | `WScoreMap.svelte` |
| W-score bounds | `.json` | `WScoreMap.svelte` (`fitBounds`) |
| Monitoring stations | GeoJSON | `OntarioStations.svelte` |
| Radar coverage | GeoJSON | `OntarioStations.svelte` |
| Game rounds | `.json` | `ScrollStory.svelte` |
 
---
 
## Running locally
 
```bash
npm install
npm run dev
```
 
Then open the printed localhost URL.
 
To preview the production build (recommended before deploying — the base path only applies here):
 
```bash
npm run build
npm run preview
```
 
---
 
## Deployment
 
Pushing to `main` triggers a GitHub Actions workflow that builds with `adapter-static` and publishes to GitHub Pages.
 
Two things that must stay correct for the deployed site to work:
 
- **Base path.** The site lives under `/DataVizTCO`, so `svelte.config.js` sets `paths.base` to the repo name in production. **Every asset fetch must prefix `base`** from `$app/paths`:
```js
  import { base } from '$app/paths';
  const geo = await fetch(`${base}/canada.geojson`).then(r => r.json());
```
  Root-absolute paths like `/data/stations.geojson` work in `dev` but 404 in production. This currently needs fixing in `ScrollStory.svelte`, `OntarioStations.svelte`, and `WScoreMap.svelte`.
- **Git LFS.** `.pmtiles` files are stored in LFS. The Actions checkout must pull LFS objects, and `submodules: false` is set on checkout.
- **`.nojekyll`** is present so GitHub Pages doesn't strip files/folders beginning with `_`.
---
 
## Accessibility
 
- Map markers are keyboard-focusable with `role="button"`, `tabindex`, and Enter/Space handlers.
- The animated radar SVG respects `prefers-reduced-motion`.
---
 
## Credits
 
Built for the **Toronto Climate Observatory**. Historical flood figures, monitoring network data, and disaster records are drawn from public sources cited within the piece.
 

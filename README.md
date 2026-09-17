# Jackson Weather

A single self-contained HTML file that charts daily weather for any city, 2000 to the
current year. No build step, no dependencies, no API keys, no backend — everything is
fetched client-side and rendered as hand-built SVG.

**Live:** https://jackson-weather-corley.vercel.app

## Running it

Open `index.html` in a browser. That's the whole workflow. If you want a local server
(not required), `python3 -m http.server` works fine.

## Editing it

See [CLAUDE.md](CLAUDE.md) — it covers the architecture, the decisions that are
load-bearing, and the gotchas that will bite you otherwise.

## Data

Observations and soil moisture come from ERA5 reanalysis via Open-Meteo. The US
forecast comes from the National Weather Service. City search is Open-Meteo geocoding.
All free tier, all keyless. Open-Meteo's free tier is non-commercial and requires
CC BY attribution.

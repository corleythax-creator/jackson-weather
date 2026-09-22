# Jackson Weather

A weather dashboard for Brandon, Mississippi — and for any other city you search.
Daily weather from 2000 to the current year, a forecast compared across every model
that publishes one, and a statewide map. Hand-built SVG, no charting library and no
framework.

**Live:** https://jackson-weather-corley.vercel.app

## What's in it

- **Forecast** — one city against every source, as a **Daily** view out to 10 days and
  an **Hourly** view out to 48. Band for the spread, a thin line per model, the average
  bold, and a table underneath so a forecast you saw elsewhere can be matched to the
  model it came from.
- **Mississippi** — 12 cities against the next 7 days, as a table and as a map.
- **Verification** — how far each model's *published* forecast landed from what the
  archive later recorded, by lead time.
- **Timeline** — temperature and rainfall against the 1991–2020 normals, with optional
  soil-moisture, solar/UV and thermostat-runtime panels.
- **Hot days** — days over a threshold by month, against the 30-year average and the
  record years.

## Running it

Open `index.html` in a browser. That's the whole workflow — no build step, no install,
no watch process. If you want a local server (not required), `python3 -m http.server`
works fine, and is the only way to see the favicon resolve.

## Editing it

See [CLAUDE.md](CLAUDE.md) — the architecture, the decisions that are load-bearing, and
the gotchas that will bite you otherwise. It is long on purpose: most of it is why
something is the way it is, written down at the point where changing it back would
break something.

## Data

All from free, keyless tiers, fetched client-side at runtime:

- **Observations, normals and soil moisture** — ERA5 reanalysis via Open-Meteo.
- **Forecasts** — the US National Weather Service by default, plus the 13 models
  Open-Meteo carries, for the comparison.
- **City search** — Open-Meteo geocoding.

Open-Meteo's free tier is non-commercial and requires CC BY attribution.

**One exception to "no backend":** the HVAC panel reads a Supabase table holding daily
rollups from an ecobee thermostat. It is off by default and appears only on the home
city. The key in the source is a publishable one, limited by row-level security to
reading two tables. See **The ecobee panel** in CLAUDE.md — it needs credentials that
are not in the repo, and draws nothing without them.

## Files

`index.html` is the application. `icon-32.png` and `icon-180.png` are the tab and
home-screen icons — the only other files, because a favicon cannot be inlined at usable
quality. Nothing here needs a build step.

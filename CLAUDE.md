# CLAUDE.md — Jackson Weather

Working notes for picking this up cold. Written for Claude Code, useful to humans.

## What it is

One self-contained `index.html` that charts daily weather for any city, 2000 to the
current year. No build step, no dependencies, no API keys, no backend. Everything is
fetched client-side at runtime and drawn as hand-built SVG — there is no charting
library and no framework.

- **Live:** https://jackson-weather-corley.vercel.app
- **Vercel project:** `jackson-weather`, team `corley`
- **Repo:** https://github.com/corleythax-creator/jackson-weather

## Running and deploying

Open `index.html` in a browser. That is the entire dev loop — no server needed, no
install, no watch process.

The Vercel project currently deploys by direct file upload, not from this repo. To get
push-to-deploy, connect it in the Vercel dashboard under **Settings → Git → Connect Git
Repository**. Connecting the existing project (rather than creating a new one) keeps the
`jackson-weather-corley.vercel.app` alias intact. After that, pushing `index.html` to
`main` deploys production.

## Data sources

All keyless. All free tier.

| Purpose | Endpoint | Notes |
|---|---|---|
| Observations | `archive-api.open-meteo.com/v1/archive` | ERA5 reanalysis, ~5-day lag |
| Recent + model forecast | `api.open-meteo.com/v1/forecast` | `past_days=14`, `forecast_days=16` |
| **US forecast (authoritative)** | `api.weather.gov` | 2 calls: `/points/{lat},{lon}` then the returned forecast URL |
| City search | `geocoding-api.open-meteo.com/v1/search` | name, admin1, country, lat/lon |
| Temperature/rain/solar normals | archive, 2001–2025 daily | one request, cached per city |
| Soil baseline | archive, 2016–2025 **hourly** | large; fetched only on demand |
| Daily condition | rides the archive + forecast calls | WMO `weather_code`, no extra request |

**Open-Meteo's free tier is non-commercial only** and requires CC BY attribution. Fine
for a personal dashboard; it becomes a licensing question if this ever monetises.

## Feature map

- **Timeline tab** — temperature (high/low/mean + 25-year normal band), rainfall,
  and two optional panels: soil moisture percentile and solar/UV.
- **Day condition symbols** — one glyph per day in a row above the panels, from the
  WMO `weather_code` the feeds already return. Day aggregation only, one city only,
  and only while the glyphs have room. The condition is also named in the readout.
- **Hot days tab** — days reaching a threshold (85–110°F) by month, against the
  25-year average for that month.
- **City search** — up to 5 cities. One city gets the full detail view; two or more
  switches to comparison (mean lines + cumulative rainfall).
- **Year picker** — 2000 to current year.
- **Day / Week / Month aggregation**, independent of zoom.
- **Zoom** — drag, scroll, presets on desktop; pinch on touch.
- **CSV export** at the current aggregation.

## Architecture

One `<script>`, no modules. In order:

1. **Date axis** — `setYear(y)` rebuilds `DATES`, `TODAY_IDX`, `OBS_END`, `DATE_POS`,
   and sets the opening view. Everything downstream keys off these. `OBS_END` is the
   last *observed* day: today for the current year, 31 Dec for a past one.
2. **Buckets** — `buildBuckets(gran)` groups the date axis into day/week/month.
3. **Per-city data** — `city.byDate`, a `Map` of ISO date → daily record.
4. **Derived series** — normals (`buildClim`), soil percentile (`buildSoil`),
   antecedent precipitation index (`buildApi` / `fitApi`).
5. **Render** — `draw()` for the timeline, `drawHot()` for the Hot days tab,
   `renderReadout(i, select)` for the figures above the chart.
6. **Interaction** — one IIFE at the bottom wires mouse, touch, and all controls.

### State

`year`, `mode` (`chart` | `hot`), `gran` (`day` | `week` | `month`), `thresh`,
`view` (a date-index range), `selDate`, `activePanel` (mobile), `soilOn`, `solarOn`,
`cities[]`.

## Decisions that are load-bearing

Changing these without understanding why breaks correctness, not just appearance.

- **NWS wins the forecast where it has coverage.** Open-Meteo's model blend ran 5–9°F
  hot against NWS for Jackson, which is what a user compares against. `applyNws()`
  overwrites forecast highs, lows and rain chance for the ~7 days NWS publishes;
  Open-Meteo fills days 8–16. NWS never overwrites an observation.
- **Observed beats modelled.** Where archive and forecast overlap, the archive wins —
  *except* UV index and precipitation probability, which the archive doesn't carry, so
  they're merged onto the archive record. See `absorb()`.
- **Buckets are built over the whole year, then filtered to the view.** A month bar
  always sums its whole month even when half off-screen. Building buckets from only the
  visible days would mean zooming silently changed what a bar represented.
- **Rainfall sums, temperature averages, solar stays per-day, rain chance takes the
  max.** Aggregation rules differ per series because the quantities differ. Solar stays
  per-day so a 28-day February compares to a 31-day July.
- **The viewed year is excluded from its own baseline** — the normals, the soil
  percentile pool, and the hot-day average. Otherwise an extreme year drags its own
  reference toward itself and looks less extreme than it was.
- **Normals are smoothed ±7 days, rainfall needs it more than temperature.** Any one
  calendar date across 25 years is dominated by a few wet years.
- **The correlation strip always uses daily values**, whatever `gran` is set to.
  Weekly buckets would wash out the 1–2 day lags it exists to measure.
- **Hot days excludes forecast days.** A modelled 101°F is not an observation.
- **Condition symbols are day-only.** A week or a month has no single condition, so
  `aggCity` carries `wc` only when the bucket is one day, and the row disappears at
  coarser aggregation. Picking a "dominant" code would be inventing a summary the
  feed never gave. They're also single-city — in comparison mode the glyph has no
  unambiguous owner — and they hide once spacing drops below the glyph width.
- **`weather_code` is not merged from the model onto an observed day.** UV and rain
  chance are merged in `absorb()` because the archive doesn't carry them; the archive
  *does* carry `weather_code`, so the exception doesn't apply. A day the archive gives
  no code for draws no symbol rather than borrowing the model's.
- **The soil gradient is `userSpaceOnUse` and vertical.** Percentile maps to y, so each
  point's colour is its own value with no path splitting. The area fill between the line
  and the 50th percentile inherits this for free.
- **Labelled circles only render when they can't touch.** `spacing >= 2 * radius`,
  else it falls back to peak/trough labels via `pickLabels()`. The circle font is sized
  from the digit count so three-digit values still fit the ring.

## Gotchas

- **Never let `draw()` touch anything inside `#readout`.** `renderReadout()` replaces
  that subtree. A previous bug put an element there, which `draw()` then threw on after
  the first hover — silently killing the soil button's handler, because the throw
  happened before `maybeLoadSoil()` ran. Data loads now fire *before* `draw()` in
  handlers for the same reason.
- **UV and rain chance are current-year only.** They come from the forecast feed, which
  reaches ~14 days back. Past years get solar but no UV; the renderer skips null days
  rather than faking them.
- **Soil moisture is the least trustworthy field here.** It's a model state variable
  responding to the model's own rainfall, not an observation, and it depends on how ERA5
  parameterises soil type at that grid cell. Read the percentile, not the raw m³/m³, and
  cross-check anything decision-grade against the US Drought Monitor.
- **The rainfall/soil correlation is partly circular** — both series come from ERA5, so
  it measures ERA5's soil physics rather than actual ground. This caveat is printed in
  the UI so it travels with the numbers; keep it there.
- **Soil depth bands differ between archive and forecast models**, which is why the soil
  line stops at today instead of extending into the forecast.
- **Native `<select>` popups need their own colours.** `color-scheme:dark` on the root
  isn't enough — the popup is a separate surface and will render light text on white.
  `select option` has explicit background/color.
- **Mobile is a different layout, not a squeeze.** Below 720px the viewBox narrows to
  380 units and panels show one at a time via `#panelpick`. The opening window is 9 days
  on a phone (yesterday, today and 7 forecast days) and a week-plus-forecast on desktop,
  because the labelled circles need room. Touch: one finger scrubs, two fingers pinch-zoom.
- **The condition glyph and the numbers under it can disagree on NWS days.** NWS wins
  the forecast high, low and rain chance but publishes no WMO code, so those days keep
  Open-Meteo's condition. On a marginal day that shows up as a rain glyph sitting over
  a 20% chance. Mapping NWS's free-text `shortForecast` instead would trade one kind of
  wrong for a fuzzier one.
- **`api.weather.gov` is US-only.** Non-US points 404 on the `/points` call; the
  `try/catch` in `loadCity()` leaves the model blend in place. That is the intended
  fallback, not an error to fix.

## House style

- Single file. Resist adding a build step or dependencies.
- Comments explain *why*, not what. Keep the ones marking non-obvious decisions.
- Prose in the UI is plain and declarative; caveats sit next to the numbers they qualify
  rather than in a footnote nobody reads.
- Real data only. If something can't be fetched, say so in the UI — never fill with
  placeholder or invented values.

## Verifying a change

There is no test suite. What has actually caught bugs here:

```bash
# 1. Does the script parse? (catches most edit mistakes instantly)
sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d' > /tmp/app.js
node --check /tmp/app.js

# 2. Does every element the code reaches for exist in the markup?
#    This caught a null-reference that silently disabled a whole feature.
grep -o '\$("[a-zA-Z0-9_]*")' index.html | sort -u
grep -o 'id="[a-zA-Z0-9_]*"' index.html | sort -u
```

For anything computational — bucket coverage, lag detection, hot-day counting, label
spacing — copy the function into a scratch `.js` file and run it against synthetic data
with a known answer. Every numeric feature in here was checked that way before shipping,
and it found real errors.

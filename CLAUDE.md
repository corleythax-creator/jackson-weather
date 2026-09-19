# CLAUDE.md — Jackson Weather

Working notes for picking this up cold. Written for Claude Code, useful to humans.

## What it is

One self-contained `index.html` that charts daily weather for any city, 2000 to the
current year. No build step, no dependencies, no API keys, no backend. Everything is
fetched client-side at runtime and drawn as hand-built SVG — there is no charting
library and no framework.

It opens on **Brandon, MS** (32.2803, −89.9983) despite the repo name — the name is
from the original default and the Vercel project, and renaming either would cost the
`jackson-weather-corley.vercel.app` alias. Search any city to change what's shown; the
opening city is the single `addCity()` call at the very bottom of the file.

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
| **US forecast (default)** | `api.weather.gov` | 2 calls: `/points/{lat},{lon}` then the returned forecast URL; fetched every load so it can be compared even when not driving |
| Candidate forecast models | `api.open-meteo.com/v1/forecast` with `models=` | one extra call returns ECMWF, GFS, ICON, GEM and the blend, each suffixed with its id |
| City search | `geocoding-api.open-meteo.com/v1/search` | name, admin1, country, lat/lon |
| Temperature/rain/solar normals | archive, 1991–2020 daily | WMO 30-year window; one request, cached per city |
| Day history + records | archive, 1940–last complete year | max, min and precipitation, in 20-year slices; fetched on first tap, on the Hot days tab, or on week/month aggregation, then cached in `localStorage` (~380 KB) so it is pulled once per browser rather than once per load |
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
- **Day history fly-out** — tap or click a day and a card shows that calendar date
  across the whole ERA5 record, 1940 to the last complete year: a low-to-high bar per year,
  then a gap and the viewed year's own bar (solid when observed, dashed and washed
  out while it's still a forecast), with this year's pair also drawn across as
  dashed rules. A "Higher than N% of years" badge sits top right, and the averages
  and extremes underneath.
- **Hot days tab** — days reaching a threshold (85–110°F) by month, against the
  30-year average for that month, plus dashed outlines for the hottest and coldest
  years in the full archive, named with their annual totals in the key.
- **Rainfall records** — the same encoding on the Timeline's rain panel: dashed
  outlines for the wettest and driest years in the full archive, at week and month
  aggregation only.
- **Forecast source picker** — the chart's forecast days can be driven by NWS
  (default) or by any of the raw models Open-Meteo exposes. A comparison table under
  the chart shows every source's daily high side by side, so a forecast that
  disagrees with whatever app you trust can be matched to the model it came from.
  The choice persists per browser.
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
- **"Observed" means the archive published it, not that the date is past.** ERA5 runs
  about five days behind, so the archive answers for today with nulls and the forecast
  feed's `past_days` window fills them. `city.obsEnd` is the last date the archive
  actually returned; days after it but on or before today are marked `est` — kept,
  because they are the best estimate available, but flagged, because they are not
  measurements. `est` days are excluded from the observed-day tally and the hot-day
  count, tagged in the readout, and drawn with the forecast styling. They are still
  the numbers on screen, because they are the best estimate available — the flag is
  about provenance, not suppression.
- **NWS still does not overwrite today, only tomorrow onward.** Today's model figure
  is an analysis of a day largely already elapsed; NWS's "today" period is a forecast
  issued that morning. Measured on 18 Sep 2026, the model had 99°F and that is what
  Jackson reached. An earlier change let NWS take today on the theory that a
  non-observation should defer to NWS; it was reverted, because "not measured" does
  not mean "worse".
- **Every forecast source keeps its own map, so switching is lossless.** `applySource()`
  lays the chosen source over the forecast days from `city.nwsMap` or `city.alt[id]`;
  nothing is overwritten irrecoverably, so flipping the picker needs no refetch. NWS
  is fetched on every load even when it is not driving, because the comparison table
  has to be able to show it. Sources only ever touch days after `OBS_END`.
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
- **The normals window and the history window are deliberately different.** Normals,
  the band and the hot-day average use `CLIM_FROM`–`CLIM_TO` = 1991–2020: the WMO
  standard 30 years, which is also what NWS and the consumer apps quote, so "vs
  normal" here matches what a user reads elsewhere. The fly-out's day history uses
  `HIST_FROM`–`HIST_TO` = 1940 to the last complete year, the whole ERA5 record.
  Do not collapse them. A 30-year window is a *baseline*, and climate drift is
  exactly why it is short — averaging 86 years folds a cooler mid-century into the
  reference and overstates every present-day reading. The fly-out is *descriptive*,
  so there the long record is the point.
- **The long record is lazy and sliced.** `maybeLoadHist()` runs on the first fly-out
  tap, not at load, and `histTried` makes it once per city per session. It goes out
  as `HIST_CHUNK`-year requests, not one 86-year span: a single span that long was
  rejected by the live archive while the 30-year normals call beside it succeeded, so
  each slice is kept near a size that demonstrably works. Slices settle independently
  via `allSettled` — partial coverage beats none. Whatever failed is named in the
  card, and the "Across N years (a–b)" line is derived from the rows that actually
  arrived, so a short record never passes as the whole one. `dayHistory()` filters `histRaw`,
  never `climRaw`. The fly-out opens only for a single city at day aggregation, and
  its summary excludes the viewed year — the same rule the normals and the soil
  percentile follow, so a year is never ranked against itself. The badge ranks the
  daily **high** against exactly the years the averages use, so the two never
  disagree.
- **The long record is cached in `localStorage`, and only when complete.** Three
  triggers each firing five slice requests on every page load got the live archive
  answering 429. The record does not change, so `cacheHist()`/`cachedHist()` keep it
  per browser under `jw:hist:<v>:<lat,lon>:<from>-<to>`; a warm load makes **zero**
  archive requests for it. Rules that matter: a partial pull is never cached, or a
  rate-limited gap would freeze in instead of being retried next visit; values are
  stored verbatim, because rounding to 1dp saved ~50 KB but could nudge 99.96 to
  100.0 and count a hot day that never happened; values are aligned to a day index
  from `HIST_FROM`, so the date array costs nothing; the key carries `HIST_TO`, so
  the turn of the year invalidates it by itself. Every `localStorage` touch is
  wrapped — it throws in private mode, and on quota the code evicts its own other
  entries, retries once, then gives up quietly. Verified: blocked storage, quota
  exceeded and a corrupt entry all fall back to fetching with no page error.
- **The hot-days record years come from `histRaw`, the average from `climRaw`.**
  `hotExtremes()` ranks whole years by their annual count at the current threshold
  over 1940–last complete year, while the dashed monthly rule stays the 1991–2020
  average — the way a forecast quotes "normal high 86, record 100 in 2010". Both
  windows are named in the hint text because they differ. The viewed year is
  excluded, ties keep the most recent year *and* report how many share the mark, a
  threshold no year ever reached draws nothing, and a coldest year that is zero every
  month is never announced in the key because it has no outline to point at.
- **Rain records are hidden at day aggregation.** `rainExtremes()` is only consulted
  when `gran !== "day"`. One specific year's rainfall on one calendar date is noise —
  the normal it would sit beside is defensible only because it is smoothed ±7 days
  across 30 years, and a single record year gets no such smoothing. Week and month
  totals are real quantities, so it draws there. `bucketSum()` matches on month-day,
  so 29 February missing from a non-leap record year is a *missing* day, not a dry
  one, and a bucket with no matching dates returns null rather than zero.
- **Record outlines paint over the bars, not behind them.** A record year lower than
  the viewed year is the interesting case, and behind a solid bar it is invisible.
  They carry no fill, so the bar still reads as the subject. Verified: of the
  overlapping outline/bar pairs, none is painted underneath.
- **Baseline years are never written as literals in UI strings.** They were twice,
  in the key and in the standfirst, and both silently kept saying 2001–2025 after
  the window moved. Interpolate the constants.
- **`b.label` is what the x axis prints, `b.long` is the full date.** The axis shows
  the day of month alone; the readout and the fly-out carry the month. Don't collapse
  the two fields back together.
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
- **Open-Meteo can fail with HTTP 200.** An error comes back as
  `{error:true, reason:"…"}` in an otherwise fine response, which `grab()` sees as
  success — it only checks `r.ok`. `maybeLoadHist()` therefore checks the *shape*
  (`daily.time` present) as well, and keeps `reason` for the card. Anything else
  parsing a response straight from `grab()` has the same blind spot.
- **The fly-out repaints itself when the archive lands.** `openFly()` records
  `flyAt` and calls `maybeLoadHist()` *before* rendering, so the first paint can say
  it is loading; the fetch calls `openFly()` again on completion. A live `flyAt`
  means no redraw happened, so the bucket index is still valid — `closeFly()` clears
  it and every redraw closes the card.
- **`draw()` closes the fly-out, on purpose.** The card is anchored to a bucket's x
  position, so any zoom, aggregation, year or tab change would leave it pointing at
  the wrong day. It lives in `#fig` and never inside `#readout`, so the rule below
  still holds both ways.
- **UV and rain chance are current-year only.** They come from the forecast feed, which
  reaches ~14 days back. Past years get solar but no UV; the renderer skips null days
  rather than faking them.
- **ERA5 rainfall is a modelled field, not a gauge reading.** It is the reanalysis
  model's own precipitation on a ~31 km cell, so it produces light-rain days that a
  rain gauge in that cell would call dry, and it will disagree with what you saw out
  of the window. This is inherent to the source, not a bug to fix; it is stated in the
  footer so the caveat travels with the numbers. The same applies to soil moisture
  below, and it is why the rainfall/soil correlation is partly circular.
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

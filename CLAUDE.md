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
| Candidate forecast models | `api.open-meteo.com/v1/forecast` with `models=` | one call returns all 13: NBM, Open-Meteo blend, ECMWF IFS + AIFS, GFS, ICON, GEM, ARPEGE, UKMO, JMA, KMA, CMA, ACCESS-G, each suffixed with its id |
| **Statewide table** | `api.open-meteo.com/v1/forecast`, comma-separated coordinates | 11 cities × 13 models in 3 requests; one object per location, on demand |
| **Forecast verification** | `previous-runs-api.open-meteo.com/v1/forecast` | `temperature_2m_max_previous_day1…7` for all 13 models; one call, on demand |
| City search | `geocoding-api.open-meteo.com/v1/search` | name, admin1, country, lat/lon |
| Temperature/rain/solar normals | archive, 1991–2020 daily | WMO 30-year window; one request, cached per city |
| Day history + records | archive, 1940–last complete year | max, min and precipitation, in 20-year slices; fetched on first tap, on the Hot days tab, or on week/month aggregation, then cached in `localStorage` (~380 KB) so it is pulled once per browser rather than once per load |
| Soil baseline | archive, 2016–2025 **hourly** | large; fetched only on demand |
| Daily condition | rides the archive + forecast calls | WMO `weather_code`, no extra request |

**Open-Meteo's free tier is non-commercial only** and requires CC BY attribution. Fine
for a personal dashboard; it becomes a licensing question if this ever monetises.

## Feature map

- **Forecast tab (the landing view)** — every forecast source's daily high, day by day.
  The chart's headline line is the **average of all sources**, with the full spread as
  a band, every model a thin line and the chosen source drawn alongside in green; the
  overnight low gets the same treatment in blue beneath it. A
  table beneath with an **average** row on top, every source's high and
  low shaded blue-to-red by where it falls among the sources that day, then spread and
  source-count rows. The point is to find the row matching whatever forecast you trust
  and pick it.
- **Mississippi tab** — a fixed list of 11 cities (Southaven, Tupelo, Columbus,
  Meridian, Natchez, Jackson, Greenville, Clarksdale, Cleveland, Greenwood, Grenada)
  against the next 7 days. Each cell is the mean of the 13 Open-Meteo models for that
  city and day, high over low, shaded by where the city falls among the others that
  day. Tapping a figure opens the model-by-model breakdown behind it, anchored to the
  cell. No chart on this tab.
- **Verification tab** — how far each model's *published* forecast landed from what
  the archive later recorded, by lead time. A chart of average miss against days of
  warning (band, thin lines, bold average and bold best), and a table of every model
  × lead 1–7 with the average miss above its signed bias, shaded per column. NWS is
  absent: nothing archives what it said last week.
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
   `drawFc()` for the forecast comparison, `drawVer()` for verification,
   `drawMs()` for the statewide table,
   `renderReadout(i, select)` for the figures above the chart.
6. **Interaction** — one IIFE at the bottom wires mouse, touch, and all controls.

### State

`year`, `mode` (`fc` | `ms` | `chart` | `hot` | `ver`), `gran` (`day` | `week` | `month`), `thresh`,
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
- **The cache key is derived from the request URL, not from the city.** `grabCached()`
  hashes the URL into `jw:<tag>:<hash>`. Keying on city alone meant that growing the
  model list from five to thirteen left every browser holding the old cached call
  showing five sources while a fresh phone showed thirteen — the same app, visibly
  different data, for as long as the TTL lasted. Any change to what a request *asks
  for* must change its key, and deriving the key from the URL is the only way to get
  that for free. Verified: two URLs produce two entries, each serves its own response,
  and a repeat is served from storage without a call.
- **The Forecast chart's line is the average, not the chosen source.** One number a
  day beats asking the eye to pick one model out of a dozen, and the labelled circles
  ride it. The chosen source stays drawn in green so the picker still means something
  on this tab — and because it is what the Timeline tab uses.
- **The forecast table is squeezed on mobile, and that costs content.** Seventeen
  columns will not fit a phone, so below 720px it drops the agency sub-labels, halves
  the padding, prints the month only where it changes, omits the degree sign (the
  heading carries °F) and rounds the average to whole degrees. That took the table
  from 1133px to 542px — still scrolling, but half as far. A tenth of a degree on a
  forecast mean is false precision anyway; do not "restore" it on mobile without
  measuring what it costs.
- **The Forecast tab draws a plume, not fourteen coloured lines.** Thirteen equally
  weighted colours would be unreadable and would need a palette the app does not have.
  Instead: a band for the full spread, every model a thin muted line, NWS in the today
  green when it is not the choice, and the chosen source bold with labelled circles.
  The chart answers "how much do they agree", the table answers "which one is this".
- **The table's colour ramp is `SOIL_STOPS` reversed, not a new palette.**
  `TEMP_STOPS` is the soil scale read the other way — coolest blue, middle grey,
  warmest red — so the page keeps one colour language instead of gaining a second.
  Shading is computed **per column**: a cell's position is its rank among the sources
  *for that day*, not against the whole table, which is the only reading that answers
  "is this source warm or cool for this day". A day where every source agrees sits at
  the neutral middle rather than being ranked on noise. The tint is 50% alpha, which
  keeps white text at 4.99:1 at its worst point across the ramp — checked, not assumed.
- **The average row is measured against the sources, not against itself.** It is
  excluded from the per-day min/max used for shading, so adding it cannot move the
  scale. It is a mean over whatever sources reach that day, which shrinks as NWS and
  the shorter models run out — hence the **Sources** row, so an average over five
  models is never mistaken for one over twelve.
- **A source that does not cover the point never becomes a row.** `absorbAlt()` drops
  a model whose arrays are all null, and `fcRows()` only lists maps with entries, so a
  model with no coverage is absent rather than a row of dashes. Verified by stubbing
  two models to null and watching 14 sources become 12.
- **Eleven cities are three requests, not eleven.** Open-Meteo takes comma-separated
  `latitude`/`longitude` and answers with one object per location, in the order asked.
  A city-at-a-time loop would have been eleven calls against a free tier that has
  already collapsed this app once. It goes out in `MS_CHUNK`-city slices rather than
  one request because a slice that fails costs four cities instead of all eleven —
  the same reasoning as `HIST_CHUNK` — settled with `allSettled` so partial coverage
  beats none. A single-location response comes back as a bare object rather than an
  array, so `maybeLoadMs()` normalises with `Array.isArray(...) ? ... : [...]`; a
  final chunk of one would otherwise parse as nothing.
- **The statewide list is fixed, and kept in the order it was asked for.** It is not
  `cities[]` — searching, removing and the 5-city limit do not apply, and the tab
  renders with no city loaded at all, which is why its dispatch sits *before* the
  `if(!cities.length) return` in `draw()`. The coordinates are town centres to about
  a mile, far inside any model's grid cell. Alphabetising them would be a change to
  what was asked for, not a tidy-up.
- **NWS is not in the statewide average.** It is two calls per city — 22 more
  requests — which is exactly the budget this table was designed around. The footnote
  says so and gives the number, so the omission reads as a decision rather than a
  gap.
- **Missing cities are named, and counted as cities.** A failed slice produces one
  error message covering four cities; reporting `msErr.length` said "1 did not load"
  when four were gone. The footnote is built from
  `MS_CITIES.filter(c => !msByCity.has(c.name))` and lists them by name.
- **The statewide tab ignores `year`, so the year picker is hidden on it.** It is
  always the next `MS_DAYS` days from today. Leaving the control visible would invite
  the reading that 2015 shows 2015.
- **The breakdown's toggle lives in the cell's own handler, not the dismiss
  listener.** The document-level listener runs *after* the button's handler, by which
  point `msSel` already names the cell just clicked — so "close when the click is on
  the selected cell" closed the card on the same click that opened it. The listener
  now ignores `button.msbtn` entirely and the button decides open-or-close itself.
- **Verification scores what a model *said*, not what it now says.** This is the
  whole reason it uses `previous-runs-api` rather than the ordinary archive.
  `temperature_2m_max_previous_day3` is the high a model published three days before
  the date it applies to; asking the archive for that model's current numbers on a
  past day would grade its *analysis*, which is a different and far easier test and
  would flatter every model. The endpoint goes back to January 2024 for most models,
  which is ample for the rolling `VER_DAYS` window.
- **Truth is the archive day already on hand, so verification costs one request.**
  `verScore()` reads `city.byDate` and accepts a day only when `src === "archive"`
  and `!r.fc` — a forecast day, or one inside the ERA5 lag window, has nothing
  measured to grade against. That keeps the tab to a single lazy call. The cost is
  that ERA5 is the yardstick: a reanalysis on a ~31 km cell, not a thermometer in
  Brandon, and a model tuned to the same reanalysis gets a small edge. Both caveats
  are printed under the table, because the number is meaningless without them.
- **NWS is not in the verification table and must not be faked into it.** Nobody
  publishes an archive of past NWS forecasts, so it cannot be scored the way the
  models are. Grading its *current* seven-day output against future observations
  would take a week per data point. The table says so rather than leaving a gap.
- **The verification chart plots only the models covering every lead.** A mean over
  a shrinking set *falls* when a short-coverage model drops out, which on an error
  chart reads as the forecasts improving with less warning — the exact opposite of
  the truth. So `plot` is `sc.rows.filter(r => r.leads.every(Boolean))`, and the
  band, the average and the bold lines all use it; the table still lists everything,
  with a **Models** row giving the count per lead. If fewer than two models are
  complete it falls back to all of them rather than drawing nothing.
- **`TTL_VER` is twelve hours because the verification window slides.** The request
  says `past_days=VER_DAYS`, so the same URL means something different tomorrow —
  `TTL_STATIC` would freeze a month-old window in place. Twelve hours keeps repeat
  visits free without that.
- **Lower is better, so the cool end of the ramp is the accurate end.** The
  verification table reuses `TEMP_STOPS` and the per-column ranking from the
  forecast table rather than gaining a second palette; blue reads as good because
  the number is a distance from the truth, not a temperature.
- **The verification y axis starts at zero.** Error is a magnitude; a floating
  baseline would make a one-degree gap between two models look like a chasm.
- **The Forecast chart draws the low as a second plume, not a second chart.** Same
  band / thin lines / bold average, in `--cold`, sharing one y range. The low's
  labelled circle is dropped on any day where it would collide with the high's —
  the same `spacing >= 2 * radius` rule the rest of the app uses, applied
  vertically. High and low are pooled separately in `agg()`, because a source can
  cover one and not the other.
- **Year and forecast source sit above the chart.** They decide what is drawn, so
  they belong before it; the rest of the controls (aggregation, zoom, panels) act on
  what is already there and stay beneath. `.controls.topctl` is the top row.
- **The Forecast tab does not load the normals.** It is the landing view and the
  normals are a 30-year pull it never reads, so `maybeLoadClim()` is deferred until a
  tab that needs it. A cold landing is three Open-Meteo calls, not four. Verification
  does not read them either, so the gate is `mode === "chart" || mode === "hot"` —
  not `mode !== "fc"`, which would have quietly pulled them on the new tab.
- **Every Open-Meteo response goes through `grabCached()`.** Reloading used to refetch
  everything, which exhausted the free tier and left the app unable to load *at all* —
  the archive call 429s, `byDate` comes back empty, the city is dropped and no chart
  ever appears. Now: this year's archive and forecast are kept 30 minutes, normals,
  past years and the soil baseline a month, under `jw:` keys. A warm reload makes
  **zero** Open-Meteo requests. A 200 carrying `{error:true,reason}` has no
  `daily`/`hourly` block and is never cached.
- **Stale beats nothing.** When a request fails and any copy exists in storage — at any
  age — it is served and `usedStale` puts a line in the footer saying the numbers may
  be hours behind. A rate limit should degrade the dashboard, not blank it.
- **Request budget is a design constraint, not an afterthought.** A cold load is three
  Open-Meteo calls. Before adding a fourth, check what it does to a user who reloads:
  the model comparison was added as an unconditional load-time call and had to be made
  lazy within the hour. The default source is NWS, so a normal load costs nothing for it.
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
- **`.key` needed `[hidden]` spelled out too.** `.key{display:flex}` is the same trap
  `.grp` fell into: the statewide tab hides the key, and without `.key[hidden]
  {display:none}` an empty flex row kept its margins. That is now two elements caught
  by the same rule — check any element with an author `display` before relying on
  `hidden`.
- **`#dl` is toggled two ways, not hidden once.** The CSV button is revealed once at
  wiring time, so a one-way `hidden = true` on the statewide tab removed it for the
  rest of the session. `syncButtons()` sets `$("dl").hidden = ms` both ways.
- **Verification needs the current year.** The window is the last `VER_DAYS` days,
  but the truth comes from `city.byDate`, which only holds the viewed year. On a past
  year there is no overlap, so `drawVer()` says so and stops rather than scoring
  against nothing.
- **The previous-runs response has the same HTTP-200 blind spot.** `maybeLoadVer()`
  checks for `daily.time` and keeps `reason` for the message, exactly as
  `maybeLoadHist()` does.
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
- **An author `display:` rule beats the browser's `[hidden]`.** `.grp{display:inline-flex}`
  kept the Day/Week/Month group on screen on tabs that hide it, for as long as the Hot
  days tab has existed. `.grp[hidden]{display:none}` is now explicit. Anything given a
  `display` in CSS needs the same treatment before `.hidden` will work on it.
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

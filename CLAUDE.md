# CLAUDE.md — Jackson Weather

Working notes for picking this up cold. Written for Claude Code, useful to humans.

## What it is

One self-contained `index.html` that charts daily weather for any city, 2000 to the
current year. No build step, no dependencies, no charting library and no framework —
everything is drawn as hand-built SVG and fetched client-side at runtime.

**There are now exactly two exceptions, and both are worth knowing before you read
further.** The first is `icon-32.png` and `icon-180.png`, the only files in the repo
besides `index.html`: a favicon cannot be inlined at usable quality, and ~85 KB of
base64 re-parsed on every load — a third of `index.html` again — is a bad trade for
something the browser fetches once and caches. They need no build step, which is the
rule that actually matters. The second:
the ecobee thermostat panel reads from a Supabase table rather than from a weather API.
Not out of preference — ecobee caps a pull at 31 days and keeps only ~15 months, and its
refresh token rotates on every single use, so a static page can neither accumulate the
history nor hold the credential. Everything else in the file still obeys the original
rules, and the Supabase key in the source is a publishable one that can only read two
tables. See **The ecobee panel** below.

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

The Vercel project is connected to this repo, so **pushing to `main` deploys
production by itself** — no manual deployment step. Confirmed on 19 Sep 2026: the push
of `3ed7213` produced `dpl_HUPZzuQceESDWiPkeyLTjzmzLgZp`, target production,
`source: "git"`, with the `jackson-weather-corley.vercel.app` alias attached. A push to
any other branch gets a preview deployment, and Vercel's bot comments the URL on the PR.
Confirmed again on 19 Sep 2026: `9d093c9` produced `dpl_GbwVUSsabTktt2ctUhG5hpadXb2z`,
target production, `source: "git"`, `aliasError: null`, with the
`jackson-weather-corley.vercel.app` alias attached.

**Do not confirm a deploy by fetching the site from an agent session.** The sandbox
proxies outbound HTTPS and serves a *cached* copy, so the page comes back without the
change and it looks like the deploy failed. The tell is in the headers: a
deployment-specific URL minutes old answering `x-vercel-cache: HIT` with an `age` of
twelve hours, and a byte count that does not match the local file. Check the Vercel API
instead — `readyState`, `aliasError` and the `alias` list on the deployment are the
authoritative answer, the same way RLS has to be checked in SQL rather than over HTTP.

## Data sources

All keyless. All free tier.

| Purpose | Endpoint | Notes |
|---|---|---|
| Observations | `archive-api.open-meteo.com/v1/archive` | ERA5 reanalysis, ~5-day lag |
| Recent + model forecast | `api.open-meteo.com/v1/forecast` | `past_days=14`, `forecast_days=FC_DAYS` (10) |
| **US forecast (default)** | `api.weather.gov` | 2 calls: `/points/{lat},{lon}` then the returned forecast URL; fetched every load so it can be compared even when not driving. Both cached — the grid lookup for a month, the periods for half an hour |
| Candidate forecast models | `api.open-meteo.com/v1/forecast` with `models=` | one call returns all 13: NBM, Open-Meteo blend, ECMWF IFS + AIFS, GFS, ICON, GEM, ARPEGE, UKMO, JMA, KMA, CMA, ACCESS-G, each suffixed with its id |
| **Hourly comparison** | `api.open-meteo.com/v1/forecast` with `hourly=` + `models=` | every model's hourly temperature in one call, suffixed by model id; on demand |
| Hourly NWS | `api.weather.gov` `/forecast/hourly` | one temperature per hour, 156 of them; shares the cached grid lookup |
| **Statewide table** | `api.open-meteo.com/v1/forecast`, comma-separated coordinates | 11 cities × 13 models in 3 requests; one object per location, on demand |
| Statewide NWS | `api.weather.gov` | 2 calls per city, 4 at a time; folded into the same average |
| **Forecast verification** | `previous-runs-api.open-meteo.com/v1/forecast` | `temperature_2m_max_previous_day1…7` for all 13 models; one call, on demand |
| City search | `geocoding-api.open-meteo.com/v1/search` | name, admin1, country, lat/lon |
| Temperature/rain/solar normals | archive, 1991–2020 daily | WMO 30-year window; one request, cached per city |
| Day history + records | archive, 1940–last complete year | max, min and precipitation, in 20-year slices; fetched on first tap, on the Hot days tab, or on week/month aggregation, then cached in `localStorage` (~380 KB) so it is pulled once per browser rather than once per load |
| Soil baseline | archive, 2016–2025 **hourly** | large; fetched only on demand |
| Daily condition | rides the archive + forecast calls | WMO `weather_code`, no extra request |
| **Thermostat (home only)** | `wcrsomethkjlfhhuurmh.supabase.co/rest/v1/ecobee_daily` | one row per local day, written by a cron; one call, on demand, home city only |

**Open-Meteo's free tier is non-commercial only** and requires CC BY attribution. Fine
for a personal dashboard; it becomes a licensing question if this ever monetises.

## Feature map

- **Forecast tab (the landing view)** — a **Daily** and an **Hourly** view of the same
  question, switched above the chart. Both are one city against every source.
  - **Hourly** — every source's temperature for the next 48 hours (24 on a phone),
    starting at the current hour. One plume rather than two, because an hour has a
    temperature and not a high and a low: band for the spread, a thin line per model,
    the average bold with labelled rings, the chosen source in green. A table beneath
    with the same per-column shading as the daily one, a rule and a date at each
    midnight, and a footnote naming how many sources it actually ran over.
  - **Daily** — every forecast source's daily high, day by day, with a row of
    condition glyphs above the plot, the weekday's first letter under every column and
    Saturday and Sunday tinted green behind the chart.
  The chart's headline line is the **average of all sources**, with the full spread as
  a band, every model a thin line and the chosen source drawn alongside in green; the
  overnight low gets the same treatment in blue beneath it. A
  table beneath with an **average** row on top, every source's high and
  low shaded blue-to-red by where it falls among the sources that day, then spread and
  source-count rows. The point is to find the row matching whatever forecast you trust
  and pick it.
- **Mississippi tab** — a fixed list of 12 cities (Southaven, Tupelo, Clarksdale,
  Grenada, Cleveland, Greenwood, Columbus, Greenville, Kosciusko, Meridian, Jackson,
  Natchez, ordered north to south) against the next 7 days. Each cell is the mean of the 13
  Open-Meteo models *and NWS* for that city and day, high over low, shaded by where
  the city falls among the others that day. Above it, a **map**: a hand-traced
  Mississippi outline with the cities at their real coordinates, each a marker
  carrying that day's average high and coloured on the same ramp, plus a
  warmest-to-coolest ranking beside it on desktop. A day picker drives the map and
  marks the matching table column. Tapping either a marker or a figure opens the
  source-by-source breakdown. The map costs no extra requests — it is the data the
  table already holds, drawn a second way.
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
- **HVAC runtime panel (home city only)** — a Timeline panel of the house's own ecobee:
  cooling, heating and aux runtime as stacked bars in hours, with indoor temperature on
  a right-hand axis over them. Off by default, one more button beside Soil and Solar,
  and absent entirely on any city that is not the house. The reading it exists for is
  runtime against the day's heat — a bar growing while the indoor line drifts up is a
  system losing ground, which no thermostat app will tell you.
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
  Open-Meteo fills the rest of the horizon. NWS never overwrites an observation.
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
- **The statewide list is fixed, and sorted north to south at definition.** It is not
  `cities[]` — searching, removing and the 5-city limit do not apply, and the tab
  renders with no city loaded at all, which is why its dispatch sits *before* the
  `if(!cities.length) return` in `draw()`. The coordinates are town centres to about
  a mile, far inside any model's grid cell. The `.sort()` is on the constant rather
  than at render time, so the chunking, the table and the breakdown all agree on one
  order. Adding Kosciusko took it to twelve, which is exactly three `MS_CHUNK`
  slices — the model side still costs three requests, and NWS went from 22 calls to
  24. Both numbers are interpolated into the footnote rather than written out.
- **NWS is in the statewide average, and it is the expensive part.** Three requests
  fetch 13 models for all of them; NWS is two calls per city on top, because it takes
  one point at a time. They go four at a time through `pool()` rather than
  twenty-two at once, and they land *after* the table has already drawn from the
  models, so the tab is useful immediately and fills in. What makes it affordable is
  that `nwsForecast()` now caches: the grid lookup is a fixed property of a
  coordinate so it keeps for a month, the periods for half an hour. A warm reload
  makes **zero** requests to either host.
- **NWS's night low is filed against the next day.** NWS pairs a daytime high with
  the night that *follows* it — "today 95, tonight 70" — while `temperature_2m_min`
  is the minimum inside a calendar day, which is the *morning* low, the same air mass
  one day later. Averaging them as published would put two different quantities in
  one mean. `mergeMsNws()` therefore takes the low from `prevDay(d)` and the high
  from `d`. The visible consequence is that the first column's low has one fewer
  source behind it, which the footnote explains rather than hiding. The high needs no
  shift. Note this is *not* what `applyNws()` does on the Timeline, and deliberately
  so: there a single source is being displayed in its own convention, which is what a
  phone app shows; here sources are being averaged against each other.
- **A cell's source count is not constant, so the copy gives a range.** NWS reaches
  about seven days and never the first column's low, so `drawMs()` computes `nMin`
  and `nMax` across every cell and prints `13–14` rather than letting an average over
  thirteen pass as one over fourteen. The breakdown card names the exact count, and
  the low's count too when it differs.
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
- **The readout only reserves height on the Timeline.** `min-height` on `.readout`
  exists so hovering does not shunt the chart up and down under the pointer — but
  only the Timeline rewrites the readout on hover. On the other four tabs it is a
  static line, and the floor was reserving 38px of nothing on every one of them.
  `syncButtons()` puts the `hoverable` class on only when `mode === "chart"`.
- **Vertical space is budgeted from a measurement, not by eye.** The chart used to
  start 490px down on desktop and 525px on a phone. Trimming the title, the
  standfirst, the control padding, the readout floor and the two forecast chart
  heights — and cutting a standfirst sentence the readout beneath it already said —
  brought that to 347px and 383px, about 28% off each. The scratch harness prints
  the top edge of every landmark, so the next trim can be aimed at whatever is
  actually costing the most rather than guessed at.
- **The Mississippi standfirst is written to fit one line.** It is the tab's
  headline and it sits above the fold on every visit, so the sentence is kept short
  enough to stay on one line at desktop width rather than being allowed to wrap. On a
  phone it takes two, which is the floor for that much information at 390px.
- **The search box keeps a 16px font however small it gets.** Padding and max-width
  are fair game; the font size is not. Below 16px, iOS zooms the whole page when the
  field takes focus, and the user has to pinch back out.
- **Five tabs fit one row at 390px, and that drove the tab metrics.** They need 356px
  of the 362px a 390px phone offers, at 13.5px with a 6px margin. Anything narrower
  wraps to two rows, which is the intended graceful failure — do not shorten the
  labels to chase it.
- **The Forecast and Verification charts are shorter than they look like they should
  be.** Adding the overnight low widened the Forecast y range from about 10° to 40°,
  so most of the panel is empty air between the two plumes; the height came down to
  match rather than paying full price for the gap.
- **The state line and the county lines are real, and they come out of one source.**
  `MS_OUTLINE` (127 points) and `MS_COUNTIES` (202 polylines, 549 points) are both
  derived from the US Census county polygons, at build time, by a topological split:
  an edge exactly two counties share is an interior border, an edge only one county
  has is on the state line. Because both fall out of the same vertex set they can
  never disagree, and a shared border is drawn once rather than twice. Decimated with
  Douglas-Peucker at ~0.01° and rounded to 3dp — about 100m, far inside a forecast
  model's grid cell — which costs ~11.5 KB of source. They replaced a 42-point
  outline I had traced by hand; the real one is better in every way and the caption
  now credits it rather than apologising for it. To regenerate: fetch a US counties
  GeoJSON, keep the features whose FIPS id starts with `28`, count every rounded
  edge, chain the count-2 edges into the mesh and the count-1 edges into the ring,
  then simplify. There is no runtime fetch and no dependency — the single-file rule
  still holds, the data is just baked in.
- **`MS_ROADS` is still approximate, and the caption says so.** Eleven major routes —
  I-55, I-20, I-59, I-10, I-22, US 61, 45, 49, 82, 84 and the Natchez Trace Parkway —
  as hand-traced centrelines, good enough to place a city against a road and not to
  navigate by. Each end is extended along its own heading until it meets the real
  border, so no route dangles short of the state line. **Two ends are exempt, and any
  new route must be checked for the same thing**: a road that *stops* inside
  Mississippi must not be extended to the state line. I-59's last vertex merges with
  I-20 at Meridian and the two run concurrently into Alabama — extending it threw the
  line 0.66° off. The Trace's southern end is a terminus at Natchez, not a crossing;
  extending it ran the parkway out into Louisiana.
- **The Natchez Trace is on the map because of Kosciusko.** The Trace runs Natchez –
  Jackson – Kosciusko – Tupelo and is both the direct and the obvious drive from
  Jackson, so it is the route that answers "how would I get there". It is a parkway
  rather than an interstate, so it draws in the thinner US-highway style.
- **`MS_PLACES` are markers with no forecast behind them.** Bruce, Amory, Houston,
  Eupora, Starkville, Belzoni, Louisville, Macon, Rolling Fork, Yazoo City,
  West Point, Vicksburg, Raleigh, De Kalb, Bude and Lucedale are there for
  orientation only: a smaller, dimmer dot and a smaller name,
  deliberately unlike a station so a reader cannot mistake one for a station whose
  number failed to load. They are not in `MS_CITIES`, so they cost no requests and
  never reach the table, the ranking or the breakdown, and nothing about them is
  tappable. The harness asserts a place never renders a second text node — that is
  what a stray temperature would look like.
  The same trap caught the shield font: `.msshield text` sets `font-size`, so the
  per-badge size has to go on an inline `style`, not a `font-size=` attribute — and
  then `.msname` and `.msplace text` for the same reason, once the station and place
  fonts had to differ between phone and desktop. A size that varies by breakpoint
  cannot live in a CSS rule that also wins the cascade; the rules now carry colour
  only and `nfs`/`pfs` go out inline.
- **Station labels are placed before place labels.** Both go through the same
  collision placer, which takes a radius, a text size, a line height and a gap, but
  the stations carry the numbers so they get the clear ground and a place name gives
  way. All twelve markers and all five place dots are obstacles before any label is
  placed, so order within each group cannot change the result.
- **Label widths are measured, not estimated.** `textW()` sizes a name with a cached
  canvas 2D context at the same font, because a px-per-character guess was rejecting
  positions that were in fact clear: at 4.2px/char it made "Starkville" a 48px box
  around 31px of text, which is why that name sat below-left of its dot with open
  space above it. Two labels had already been nudged by hand-tuning that constant
  before it was worth measuring instead. There is a `str.length * px * 0.52` fallback
  if canvas is unavailable, which is the only path that can still misjudge a box.
- **A place label can slide sideways, above *or* below.** Places try eight positions
  — above, above-right, above-left, below, below-right, below-left, right, left —
  where a station tries four. None of this is decoration. Amory's plain "above" box
  overlaps Tupelo's marker by about 4px, so the raised-and-shifted pair is what keeps
  its name above the dot; and on a phone Starkville is boxed in by Columbus's marker
  about 28px east and Eupora's already-placed name to the north-west. All eight are
  then tried again pushed 7px out, and again at 14px, before a place is given up on.
  That second ring is what keeps Starkville drawn on the phone map once a dozen
  places are competing, and in practice nothing lands more than about 20px from its
  dot, so a label still reads as belonging to the nearest one. Shrinking the place
  font (now 7px) was not enough on its own for any of it.
- **For a place, *every* raised ring outranks dropping below the dot.** The candidate
  list is built across all three rings and then reordered so the nine raised
  positions come first: pushing a name a few pixels further out reads better than
  flipping it to the other side. A station keeps the simple order, because its label
  is bigger and a long push would detach it from the number.
- **`pad` is 1px, and that is deliberate.** `hw` already extends 3px past the text on
  each side, so two labels whose boxes merely touch still have 6px of clear air
  between them; `pad` adds to that when a placed label becomes an obstacle. It was
  2px, which put 10px between neighbours and blocked Houston's "above" against
  Bruce's name by half a pixel. Now that widths are measured rather than estimated,
  1px is enough — but do not take it to 0, or names will render edge to edge. On a phone Starkville's name sits
  below its dot rather than above: the map is 83px per degree there against 99.5 on a
  desktop, so Columbus's marker is proportionally closer and "above" misses by about
  half a pixel. A name below the dot beats a name touching a marker, so that is left
  rather than tuned to the pixel.
- **A place dot is an obstacle before any label is placed, which is why `wide`
  exists.** Rolling Fork sits about 33px from Yazoo City on a phone, and because
  every place's dot goes into `boxes` up front, it crowded Yazoo City's name out even
  when its own label was placed last — demoting it made no difference, because the
  dot blocks regardless of label order. `wide:1` drops a place from the phone map
  entirely, which is the only thing that frees the space. Proven by removing Rolling
  Fork outright and watching Belzoni and Yazoo City both fit. Reach for it only when
  a place is provably the one crowding another out, not as a way to thin a busy map.
- **A place that cannot be placed clear is dropped, not drawn on top.** `place()`
  returns false when no candidate clears, and `drawMs()` renders only the places that
  fit. A station always draws — it carries a number, so it falls back to below and
  takes the overlap. A dot with no name tells a reader nothing, which is why the
  whole place goes rather than just its label. `MS_PLACES` sorts itself by latitude
  the way `MS_CITIES` does, so a new entry can be written anywhere in the list; the
  order still decides who gets the clear ground, north first.
- **Check a new place against the nearest station before adding it.** The scratch
  harness prints that distance; anything under about 14px vanishes beneath an 11.5px
  marker. Bruce, Vicksburg, Macon and Raleigh all came in at 40px or more.
- **A place closer than a marker radius to a station is not worth adding.** Flowood
  was tried and removed: seven miles from Jackson is 8.9px against an 11.5px marker
  radius, and because places draw *under* the stations its dot rendered invisibly
  beneath Jackson's chip, leaving a name pointing at nothing. Drawing places over the
  markers instead would put a grey dot on a temperature chip, which reads as a defect
  rather than a second town. Check the pixel separation before adding a suburb.
- **A shield may carry `dx`/`dy` and `sm`.** The nudge shifts the *candidate* before
  the collision test, not the badge afterwards, so a hand-placed label still cannot
  overlap anything — US 61 uses `dy:-11`. `sm` is for a route named rather than
  numbered: "Natchez Trace" at full badge size is a banner, so it drops to 6px. The
  Trace also carries `dy:-18`, and needs it: it is the longest route and it runs
  diagonally through the busiest part of the map, so once sixteen place names were
  down nothing on its own line was clear and its badge was dropped entirely. More
  anchor points did not recover it — lifting the badge off the route did.
- **Road geometry is checked against the outline, not against the eye.** Every road
  vertex *and* 50 sampled points along every segment are tested point-in-polygon
  against `MS_OUTLINE` — 3,610 points in total, all of which must fall inside. The
  vertex check alone is not enough: a straight hop between two interior points can
  still cut a corner off a concave boundary, which is exactly how I-59's crossing of
  the Pearl River was caught running over open water, and how I-10 was caught cutting
  across the Bay of St. Louis once the real coastline replaced the straight one.
  Re-run it after touching either array — swapping in the Census outline alone put
  three routes outside. Watch the sign when nudging an endpoint: longitudes here are
  negative, so *east* is the larger number, and "move it inside" on the Alabama line
  means more negative, not less.
- **`MS_FILL` tints the counties that hold a city, and it is generated, not curated.**
  The 26 counties containing a station or a place dot are emitted as closed rings by
  the same build step that produces `MS_OUTLINE` and `MS_COUNTIES`, from the same
  Census source, by testing each city point against each county polygon. Simplified
  a shade tighter (0.008° against 0.012°) because the county lines draw *over* the
  fills: a fill sitting a hair inside its own outline is hidden, one that spills past
  it is not. **Regenerate it whenever `MS_CITIES` or `MS_PLACES` changes**, or a new
  city sits in an untinted county — the scratch harness checks both directions, that
  every city is inside a tinted ring and that no tinted ring is empty.
- **County lines are the faintest thing on the page, and one element.** All 202 of
  them are a single `<path>` rather than 202 polylines: the same picture, a fraction
  of the DOM, and nothing about them needs to be addressable. They are stroked at
  `rgba(190,214,220,.085)` — texture, not information. Anything more assertive and 82
  county borders bury the eleven markers that the page is actually about.
- **Shields yield to everything and are dropped rather than squeezed.** Roads are
  context, not data, so the badges are placed only after the markers and the city
  names, against the same collision test, trying every interior vertex and the
  quarter, half and three-quarter points of every segment. A route whose badge finds no clear spot keeps its line
  and loses its label — which is what happens to US 82 on a phone. The lines
  themselves are drawn under the markers and are deliberately dim; if they ever start
  competing with the temperature ramp for attention, they are wrong.
- **`msProject()` corrects longitude by cos(lat).** At Mississippi's latitude a degree
  of longitude is 0.84 of a degree of latitude, so plotting lon and lat on the same
  scale comes out a third too wide and the state reads as the wrong shape. The scale
  is `min(w/dx, h/dy)` so the aspect ratio survives whatever box it is given.
- **The phone map is bigger than the desktop one, and the marker radius is the
  thing that caps it.** A 390px phone was drawing a 330×400 map with 10.5px names
  — legible at arm's length only just, and leaving a third of the screen to the
  table below. It is now 352×500 with 11.5px station names and 8px place names,
  against 400×480 and 10.5/7 on desktop, with the body gutter cut to 11px to pay
  for the width. The markers grew with it, but only to `rr=13`: 13.5 put Kosciusko's
  marker through Greenwood's name, which is the clash the harness catches, so 13 is
  the largest value that is provably clear. The extra room is worth more than it
  cost — the phone map now places **15** of the 16 places (Starkville came back) and
  **all 11** shields, where the smaller one dropped Starkville and US 82. Rolling
  Fork is still absent, which is `wide:1` by design, not a casualty of the scale.
  Two strings were shortened to buy the same space: the readout under the map and
  the Census credit in the key.
- **Map labels are placed against collisions, not at a fixed offset.** Greenville,
  Cleveland, Greenwood and Grenada sit within about fifty pixels of each other, and a
  fixed side put names straight through neighbouring markers. Each name takes the
  first of above / below / right / left that clears every marker and every name
  already placed. All eleven markers go in as obstacles *before* any name is placed,
  so the order cannot change the outcome, and the strip above the panel reserved for
  the date caption is excluded — Southaven is within a hundredth of a degree of the
  state's northern edge and its name landed on the caption before that rule. The
  harness asserts no name overlaps another name or another city's marker; placement
  depends only on coordinates and name length, so checking one day checks all seven.
- **The marker ring is `--paper`, and wide enough to cut the marker out of the map.**
  With roads under them the markers needed a heavier ring — 2.2px, and 2.8px when
  hovered or selected — so a chip reads as sitting on top of the state rather than
  merging into whatever line passes behind it.
- **The number inside a map marker is pure black, and that is a measured value.**
  The markers are solid ramp colours at full opacity, which is a much harder
  background than the table's 50%-alpha tint. Measured across the ramp at 5% steps:
  `--ink-soft` gives 1.00:1 at the middle (invisible), `--ink` 2.15:1, white 2.56:1,
  and a dark grey like #10191F 3.89:1 at the hot end. Picking black-or-white per
  marker still bottoms out at 4.26:1 around the crossover. Plain **#000 is the only
  single colour that clears 4.5:1 on every stop**, worst case 4.60:1 on the hottest
  red. Do not soften it to a dark grey to match the page furniture — that is exactly
  the change that fails.
- **The map carries the high only.** A low under every marker was a third line of text
  per city and it pushed the names into each other. The low is in the ranking beside
  the map, in the table and in the breakdown; the map answers "where is it hot today"
  and hands the rest off.
- **A breakdown opened from a marker clears the whole map, not just the marker.**
  Anchoring it under the dot buried the map it came from, and anchoring it beside the
  dot buried whichever half of the state the dot was in. `drawMsMap()` returns where
  the map ends as a fraction of the svg, and `openMsFly(..., beside)` puts the card
  past that edge, vertically level with the marker. A table cell still gets the card
  underneath, which is right there. On a phone nothing has room beside it, so both
  fall back to underneath.
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
- **`grabCached()` takes an optional `ok` and `keep`.** `ok` decides whether a
  response is worth storing — the default is the Open-Meteo shape, and anything that
  is not that shape must say so or it silently never caches. `keep` trims a response
  before it is stored *and* returned, which is how eleven NWS forecasts fit in ~28 KB
  instead of ~250: the app reads five fields per period and `detailedForecast` is not
  one of them. A caller using `keep` must not read fields it did not keep.
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
- **`FC_DAYS` is 10, and the number comes from source coverage rather than taste.**
  Coverage thins with lead time and does not thin evenly. Measured for Brandon on 20
  Sep 2026: **11** sources through day 4, then 9, 8, 7, and **6 at day 10** — falling
  to 4 from day 11 and to **2** on day 16. It was 16, then 15 to shed the two-source
  column, now 10, which stops at the last day still carrying six. An average over two
  is not the same quantity as an average over eleven, and a "spread" between two
  sources is noise with a number attached. Every request and every horizon interpolates
  the constant, so moving it is the entire change — and re-running that measurement is
  how to decide whether it should move again, because these reach limits shift as
  providers change their runs. Note this also sets the Timeline's forecast horizon,
  which is the same `HORIZON`.
- **The Forecast chart draws the low as a second plume, not a second chart.** Same
  band / thin lines / bold average, in `--cold`, sharing one y range. The low's
  labelled circle is dropped on any day where it would collide with the high's —
  the same `spacing >= 2 * radius` rule the rest of the app uses, applied
  vertically. High and low are pooled separately in `agg()`, because a source can
  cover one and not the other.
- **Year and forecast source sit above the chart.** They decide what is drawn, so
  they belong before it; the rest of the controls (aggregation, zoom, panels) act on
  what is already there and stay beneath. `.controls.topctl` is the top row.
- **Daily and Hourly are one tab with a switch, not two tabs.** Five top-level tabs
  already need 356px of the 362px a 390px phone offers; a sixth wraps to a second row.
  The switch sits in `.controls.topctl` because it decides *what is drawn*, which is
  the same reason the year and source pickers live there.
- **Only the view on screen is fetched.** `maybeLoadHourly()` and `maybeLoadAlt()` are
  separate lazy loads, and the tab handler, the view switch and the cold-load path each
  pick exactly one. Landing on Hourly must not also pull the daily model set — a cold
  load is three Open-Meteo calls plus the one that view needs, and that budget is the
  constraint the whole app is built around.
- **Two of the thirteen models have no hourly data at all.** KMA and ACCESS-G answer
  the hourly endpoint with nothing but nulls. They are dropped rather than drawn as a
  row of dashes — the rule `absorbAlt()` already applies daily — so the hourly
  comparison honestly runs over fewer sources than the daily one, and the table's
  footnote prints the count rather than letting a mean over eleven pass as one over
  fourteen. Do not "fix" this by filling their gaps from another model.
- **Night is shaded from real sunrise and sunset, at fractional hour positions.**
  `nightSpans()` returns fractional indices, not whole hours: a sunrise at 06:46 falls
  between two columns and at roughly twenty pixels an hour, snapping it to one would be
  visibly wrong. `X()` is linear in its argument, so a fraction projects with no extra
  maths. The band draws *before* the gridlines — it is the ground the chart sits on,
  not a layer over the data. Which side of the window opens in darkness is decided by
  the last event before the left edge, or, when there is none, by the first event after
  it: a sunrise ahead means the window opened in the dark. A window wholly inside one
  night shades end to end, and one wholly in daylight shades nothing.
- **Sunrise and sunset ride the plain forecast call, and only that one.** Asked for
  beside `models=` they come back suffixed per model — thirteen identical copies of the
  same astronomy, since no model computes it. The main forecast call carries no model
  list, already spans the hourly window, and is made on every load, so the two fields
  cost nothing. Adding them changed that URL, which changed its cache key by itself, so
  no warm browser can be served an entry missing them.
- **Dates are parsed at midday, never at midnight.** `dayOf()` builds
  `new Date(t+"T12:00")`. An ISO date at 00:00 can land on the previous day once a
  daylight-saving shift is applied, which would letter the axis wrong and paint the
  weekend band on the wrong columns, twice a year.
- **The weekday letter goes on every column; the date number keeps its cadence.** The
  date row is thinned to `maxLab` labels, which on a 16-day desktop axis means every
  other day. A single character is narrow enough to fit where a date is not, and a
  weekday letter on every *other* day would be worse than none. Weekend letters take
  the band's own colour so the tint and the letters read as one marking rather than two.
- **The weekend band is drawn half a step past its outer columns.** Stopping at the
  column centres would tint half of Saturday and half of Sunday; the band exists to
  cover those days. Consecutive weekend days are grouped into one rect, and a run that
  reaches the last column is flushed there — a window ending on a Saturday still gets
  its band.
- **The rain chance is printed for every day that has one, zero included.** A blank
  above a glyph has to mean "no figure published", not "nought per cent", or the row
  stops being readable at a glance. Rather than hiding the dry days, the ink is scaled
  to the number — `0.55 + 0.45 * pop/100` — so the wet days find the eye without the dry
  ones being silently dropped. **The floor is 0.55, and it was 0.3 first, which was
  wrong**: a 5% day rendered as a ghost, which is decoration rather than information,
  and this is a figure somebody asked to be able to read. It reads `r.pop`, which the
  forecast call already carries, so it costs no request; on the ~7 days NWS drives, that
  value is NWS's own.
- **The rain chance drops its per-cent sign on a phone, and the label is what sets the
  row's limit.** `"100%"` is wider than the glyph beneath it, so the text, not the icon,
  decides how tight the row can get — and it is tight: measured at 390px, `"100%"` is
  23.3px against 22.4px of column, so two wet days running would collide. Dropping the
  sign takes it to 15.2px, which leaves 7.2px of clearance with *every* day at 100%.
  That is the same trade the comparison table makes with the degree sign, and the key
  names the row either way. Measure this again before changing the font: it was checked
  with a fifteen-day run of 100% so the widest possible neighbours were adjacent.
- **The tab icon and the home-screen icon are different crops of the same picture.**
  At 32px the whole card turns to mud — the chart lines below the cloud become noise —
  so `icon-32.png` is the sun and cloud alone, the part that survives the downscale,
  and `icon-180.png` is the full artwork, which reads properly at that size. Both were
  produced by stepping the 1254px source down in halves rather than in one draw: a
  single 1254→32 resample samples too few source pixels per destination pixel and
  aliases badly. iOS applies its own squircle mask over the apple-touch-icon, so the
  artwork's own rounded corners get clipped a second time; the corners are dark and
  empty, so nothing is lost, but do not add detail out there.
- **The Daily view's glyphs come from `weather_code` already in `byDate`.** No extra
  request: the forecast call carries the code, and the tab reads it. They obey the same
  spacing rule as the Timeline's, so a cramped axis drops them rather than overlapping,
  and they carry the same caveat — NWS publishes no WMO code, so an NWS-driven day keeps
  Open-Meteo's condition and can disagree with the numbers under it.
- **The hourly axis starts at the current hour, not at midnight.** The hours already
  elapsed today are not a forecast, and including them would put a model's analysis of
  6am beside its forecast of 6pm in the same row. Forty-eight hours is the desktop
  window and twenty-four the phone's: past two days the models diverge faster than an
  hour-by-hour reading is worth, and 48 columns is already the most a table can carry.
- **The hourly rings go on every nth hour, solved rather than switched off.** The rest
  of the app drops labelled circles entirely once `spacing >= 2 * radius` fails, which
  at 48 columns would mean never drawing one. Here the same rule is solved for n —
  `cstep = ceil(2 * radius / spacing)` — so the signature look survives a dense axis.
  Verified at both widths: the resulting spacing clears `2 * radius` in each case.
- **Local wall-clock is the join key between the two hourly sources.** Open-Meteo under
  `timezone=auto` returns `2026-09-19T14:00` and NWS returns
  `2026-09-19T14:00:00-05:00`, so NWS is sliced to 13 characters and keyed alike.
  **`units` already ends with `&timezone=auto`** — appending another one produced a
  duplicate parameter that Open-Meteo rejected outright, which cost an hour to find
  because the failure surfaced as a bare "Failed to fetch". Do not add a timezone to a
  URL that already interpolates `units`.
- **Changing what a response *keeps* needs a new cache tag, exactly as changing what it
  *asks for* needs a new key.** The hourly view reads `properties.forecastHourly` from
  the NWS points response, which the old `keep` threw away. The URL is identical, so
  the key would have been identical, and every browser holding a month-old entry would
  have been served a response missing the field. The tag moved from `nwspt` to
  `nwspt2`, and `nwsPoint()` now serves both the daily and hourly forecasts from one
  cached lookup.
- **The year picker is hidden on the Hourly view.** It is always the next two days from
  now, so leaving the control visible would invite the reading that 2015 shows 2015 —
  the same reasoning that hides it on the statewide tab.
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
- **The ecobee integration is server-side because the credential forces it, not because
  a table was convenient.** ecobee rotates the refresh token on every `/token` call and
  kills the old one the instant it answers, so whatever holds that token must be able to
  write a new one back. A static page cannot, and neither can a stateless serverless
  function — it cannot rewrite its own env vars. `public.ecobee_auth` is that writable
  place. The ecobee *API key* is not in it: that is an edge function secret, because it
  never changes. This is also why a Vercel function was the wrong shape for the job even
  though the repo already deploys there.
- **`ecobee-sync` writes the rotated token before it fetches anything.** If the report
  call runs first and fails, the token that was just issued is gone and the old one is
  already dead — the integration is bricked until the PIN dance is repeated by hand. So
  the order is refresh, store, *then* fetch, and a failed store aborts rather than
  carrying on. A refresh token also expires after 30 days unused, which is the real
  reason the cron matters: miss a month and it has to be re-authorized.
- **ecobee fails with HTTP 200 too.** An error comes back as a non-zero `status.code` in
  an otherwise fine response — the same blind spot Open-Meteo has with
  `{error:true,reason}`, and checking `r.ok` catches neither. Both ecobee calls in the
  sync check the shape, exactly as `maybeLoadHist()` does.
- **`intervals` is the thermostat's `est` flag.** A complete day is 288 five-minute
  rows; fewer means the runtime totals are *floors*, not totals. Today always is one,
  and so is any day the thermostat was offline. Partial days are still drawn and still
  counted — they are the best figure available — but washed out to 42% opacity, tagged
  "part day" in the readout, and marked `partial` in the CSV. Provenance, not
  suppression: the same rule `est` days follow on the weather side.
- **Runtime sums across a bucket, indoor temperature averages.** Different quantities,
  different aggregation, the same way rainfall sums while temperature averages. A bucket
  with no thermostat rows returns `null` rather than zeros — a zero would read as a
  system that never ran, which is a different claim from "no data".
- **Indoor temperature is on the HVAC panel's right-hand axis, not on the temperature
  panel.** It was on the temperature panel first, and rendering it showed why that
  fails: a Mississippi summer holds the house near 73°F and the outdoor low between 71
  and 76°F, so the two lines sit on top of each other through the entire cooling season
  — unreadable exactly when it matters. On the runtime panel it sits next to the load it
  explains. Its axis also has a **10°F minimum span**, for the same reason the
  verification axis starts at zero: a thermostat holds a degree or two, and a range
  fitted tight to that turns ordinary cycling into a mountain range.
- **`--indoor` is `#C98A9B`, and the distance was measured.** The existing temperature
  lines sit 39–95 ΔE apart, so a new one had to clear roughly that. `#C98A9B` is ≥33 ΔE
  from everything on its panel and 5.91:1 against `--paper`. Its one close neighbour is
  `--uv` at ΔE 24, which never shares a panel with it — the two meet only in the key,
  where they are a swatch apart and labelled.
- **The thermostat panel exists only on the home city, and is absent rather than
  disabled elsewhere.** `HVAC_HOME` is a `city.key`, so searching Memphis simply does
  not offer the control. A greyed-out button would read as something that failed to
  load; there is nothing to load, because there is no thermostat in Memphis.
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

- **An SVG `fill="..."` attribute loses to *any* CSS rule, and this stylesheet has
  one.** `text{fill:var(--ink-soft)}` at the top of the CSS beats every
  `<text ... fill="var(--hot)">` in the file, because a presentation attribute sits
  below all author CSS in the cascade. That silently painted every labelled circle on
  the Forecast, Timeline and Verification charts grey instead of its accent colour,
  the hot-days counts, the fly-out's year label and the solar/UV axis — and on the new
  map it put 1.04:1 grey numbers on grey markers. Colour a label with an inline
  `style="fill:..."`, which does win, never with the attribute. Nothing in the
  existing checks catches this: it parses, the element exists, and the attribute is
  right there in the DOM. Only `getComputedStyle(el).fill` shows it, so that is what
  the harness now asserts. The same trap as `.grp` and `.key` with `[hidden]`, one
  layer down.
- **`font-size` as an attribute loses to the stylesheet too, and ten call sites still
  do it.** `text{fill:var(--ink-soft);font-size:12px}` at the top of the CSS beats a
  presentation attribute on *both* properties, and while the `fill` half is documented
  above, the `font-size` half is live: every `font-size="…"` attribute in the file
  renders at 12px, whatever the code computed. It is not only cosmetic. The labelled
  circles size themselves to fit their ring — `f = min(10.5, (2r - 3) / (digits * 0.6))`
  — and that result is being discarded, so on a **phone** a three-digit temperature
  measures 20.6px inside a 20px ring and spills out of it; 15 of 30 labels overflowed in
  a 103°F render at 390px. Desktop fits (20px in a 23px ring), which is why it survived.
  Brandon clears 100°F most summers and the Hot days tab goes to 110, so the case is
  reached. Known and unfixed, because correcting it visibly resizes labels on four
  charts: the ring labels on the Forecast, Hourly, Timeline and Verification charts, the
  hot-day counts and the fly-out's axis labels. Anything new must put its size in an
  inline `style`, which `.poplab` and the map's shields and place names already do.
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
- **A multi-location response is an array, and the default cache test missed it.**
  `grabCached()`'s default `ok` looks for `j.daily`, which an array never has, so the
  statewide tab's three calls were refetched on *every page load* — quietly breaking
  the "a warm reload makes zero Open-Meteo requests" rule for weeks' worth of visits.
  It now passes its own `ok`. Anything that requests a shape other than a single
  `{daily}`/`{hourly}` object has to do the same; check a cold load against a reload
  when adding one, because nothing else surfaces it.
- **The thermostat response is an array, which is the array-cache trap a second time.**
  `grabCached()`'s default `ok` looks for `j.daily`, and a PostgREST response is a bare
  array, so `maybeLoadHvac()` passes `j=>Array.isArray(j)`. This is the same bug the
  statewide tab shipped with for weeks. Any new request whose shape is not a single
  `{daily}`/`{hourly}` object has to say so, and the only way to notice is to compare a
  cold load against a reload.
- **The Supabase key is in the query string on purpose.** `grab()` is a bare
  `fetch(u)` with no headers, so `?apikey=` is the only way to authenticate without
  changing it — and it keeps the cache key derived from the whole request, which is the
  rule. The key is the publishable one; RLS gives it `select` on `ecobee_daily` and
  `ecobee_sync` and nothing else, and `ecobee_auth` has RLS on with **no policies at
  all**, which is what makes the token unreadable. Supabase's linter reports that last
  one as "RLS enabled, no policy" — that is the intended state, not a finding.
- **Do not verify RLS with curl from an agent session.** The environment proxies
  requests to this Supabase host and injects privileged credentials, so a browser-shaped
  test silently runs as a privileged role: an anon write that must fail returns 201, and
  an anon read of the token table returns `[]` for want of rows rather than for want of
  permission. Check it in SQL under `set local role anon` instead.
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

## The ecobee panel

The code is in place and the schedule is built; it draws nothing until it is given
credentials, and says so in the UI rather than showing zeros. Four steps, once:

1. **An ecobee API key.** This is the blocker, and it may not be surmountable: ecobee's
   developer page currently reads *"we are not currently accepting new developer
   registrations at this time."* If you already have a developer account, the key is
   under My Apps. If not, there is no API route at present — the Home IQ web portal's
   CSV export (System Monitor → Download Data, 31 days a time, ~15 months back) is the
   fallback, and `ecobee_daily` is shaped to receive it.
2. `supabase secrets set ECOBEE_API_KEY=…` on project `wcrsomethkjlfhhuurmh`.
3. **Authorize**, read-only. `POST` to the `ecobee-auth` function with
   `{"action":"pin"}`, enter the PIN it returns at ecobee.com → My Apps → Add
   Application, then `POST {"action":"claim","code":"<the code it returned>"}` within
   ~9 minutes. The scope requested is `smartRead` deliberately: this panel only ever
   displays numbers, and a read-only grant cannot change a setpoint even if the token
   leaks. Then call `ecobee-sync` once with `{"days":460}` to backfill everything
   ecobee still holds.
4. **Turn on the cron.** Put the service role key in Vault as
   `ecobee_sync_service_key`, then
   `select cron.alter_job(job_id := (select jobid from cron.job where jobname='ecobee-daily-sync'), active := true);`
   It is scheduled 05:17 UTC — just after local midnight, so the day it re-reads has
   finished — and left **inactive** until that secret exists, because an active job
   would only fail nightly.

Pieces, so a later reader can find them: tables `ecobee_daily` (the rollups, anon
readable), `ecobee_auth` (the rotating token, unreadable), `ecobee_sync` (last outcome,
anon readable, so the page can say when it last refreshed); functions `ecobee-sync` and
`ecobee-auth`; client entry point `maybeLoadHvac()`.

Two caveats that belong next to the numbers. The `outdoor_mean` ecobee reports is a
nearby *station* relayed by ecobee, not a sensor at the house — it is an independent
check on ERA5's 31 km cell, which this app otherwise has none of, but it is not a
backyard thermometer. And `heat_minutes` is compressor heat while `aux_minutes` is
auxiliary or furnace heat, so on a gas furnace *all* heating lands in aux; the two are
kept apart rather than merged because merging them would invent a number the feed
never gave.

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

For anything with a colour on it, read the *painted* value back out of the browser
(`getComputedStyle(el).fill`) and compute the contrast ratio, rather than trusting the
markup — an attribute that the cascade throws away looks perfectly correct in the DOM.

For anything computational — bucket coverage, lag detection, hot-day counting, label
spacing — copy the function into a scratch `.js` file and run it against synthetic data
with a known answer. Every numeric feature in here was checked that way before shipping,
and it found real errors.

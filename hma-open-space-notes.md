# HMA Open Space Acquisitions — field survey-status app

New app, built 2026-09-11, forked from the same PREDS Summary / Districts app pattern
(OAuth PKCE sign-in, Leaflet + marker clustering, filter drawer, detail sheet). Single
self-contained `index.html`. Not yet deployed — delivered to Tema as a file; needs a
GitHub Pages deployment (see "Still needed before this can go live").

## What it does
Lists the ~1,300 properties TEMA has acquired under the HMA Open Space program (from
`TN_properties_acquired.csv`: Region/Address/City/Lat/Lng) and cross-references each one
against the live HMA Open Space Monitoring survey layer to show, in the field, which
acquisitions have and haven't been surveyed yet — with a one-tap link to launch the
Survey123 form for any property that still needs a visit.

## Data sources
- **Acquired properties**: embedded directly in the page as `ACQUIRED_RAW` (not
  fetched — there's no live service for this list). Regenerate/re-embed if the
  acquisitions list itself changes; it does not update from the two feature layers.
  Now sourced from `TN_properties_with_county_2.csv` (Region/Address/City/Lat/Lng/
  **County**, 1,300 rows — same 1,300 properties as the original
  `TN_properties_acquired.csv`, just with a County column added; verified row-for-row
  identical region/address/city/lat/lng, no blank counties). County is shown on every
  property's card and in its detail sheet directly from this list now — it's no longer
  limited to properties that already have a matched survey record (survey-record
  county is still used as a fallback if a given row's list county were ever blank).
- **Survey layer** (`HMA_Open_Space_Monitoring_VL_dashboard/FeatureServer/0`,
  `services1.arcgis.com/kILp9lqGUeOhnDbI` — same AGOL org as PREDS): token-secured.
  Field schema (`SURVEY_FIELDS` in the code) was confirmed against a live export the
  user provided (`HMA_Open_Space_Monitoring_Field_App_View1.xlsx`, 653 rows), not
  guessed. Notable data-quality findings from that export, baked into the app:
  - `property_id` is free text typed by field inspectors ("Not Available", "Project
    number is 49", "Unknown. Project number is 6.", etc.) on 638 of 653 rows — **not**
    a usable key. Shown in the detail sheet when present, never used for matching.
  - The exported `x`/`y` attribute columns were `0`/`0` (missing) on ~70% of rows. The
    app never trusts those two columns — it always re-queries the live layer with
    `returnGeometry=true&outSR=4326` and uses `feature.geometry.x/.y` — but the same
    gap may well exist in the live geometry too, which is exactly why address-text
    matching is the primary correlation signal, not proximity.
  - `address` is one field holding a full geocoded string ("106 Riverside Dr,
    Columbia, TN, 38401, USA"); the street portion (before the first comma) is what's
    compared against the acquired CSV's own Address column.
  - `region` values in the survey layer ("East"/"Middle"/"West") already match the
    acquired CSV's Region column casing/spelling; no "Southeast" surveys existed yet
    in the reviewed export, which is fine — same field, just not yet used.
- **Parcels layer** (`Tennessee_Property_Boundaries_State_Use/FeatureServer/0`,
  `utility.arcgis.com`): statewide TN parcel boundaries, shown as an optional
  viewport-scoped map layer (`PARCELS_MIN_ZOOM = 15`) and used for a per-property
  point-intersection lookup in the detail sheet's Parcel Info section. **Its field
  schema was never confirmed** — every attempt (WebFetch, curl through the sandbox
  proxy, the linked desktop's browser signed out) got a 403 with no anonymous access,
  and no export was provided for it the way one was for the survey layer.
  - The **map-click popup** (`buildParcelPopupHtml()`) is now curated: Address, City,
    Calc. Acres, and a link out to the county's TN Property Assessment (TPAD) page
    (`https://assessment.cot.tn.gov/TPAD/Parcel/GIS?gislink=...`). Since the schema
    was never live-verified, each value is found by matching a short list of likely
    real field names case-insensitively (`PARCEL_FIELD_CANDIDATES` /
    `findParcelField()`) rather than one hardcoded guess — e.g. address tries
    `PropertyAd`, `SITUS_ADDR`, `Address`, etc. in order, first match wins. This
    guards against the ~10-character truncated field names typical of
    shapefile-derived TN parcel services (e.g. `PropertyAd`, `CalcAcreag`,
    `GISLINK`). **Worth a live spot-check once deployed** — if none of the candidate
    names match the real schema, that value is just omitted rather than shown wrong,
    so a silently-empty popup field is the sign to add the real name to the list.
  - **Live example spotted 2026-09-14**: a real parcel popup showed Address
    "HAPPY VALLEY ST" and City "233 ELIZABETHTON" — a house number stuck
    onto the city value. That's not a blank field (which the code handles
    fine), it's a **wrong match** — whatever field
    `PARCEL_FIELD_CANDIDATES.city` matched on this parcel service actually
    holds something like a site address, not a clean city name. Not fixed
    yet — needs the real field list from this layer (a live export, like
    the one that fixed the survey layer's schema) to pick the right
    candidate name rather than guess again.
  - The **detail sheet's Parcel Info section** (`loadParcelInfoInto()`) was left as
    the original generic renderer (every populated attribute, labeled via
    `humanizeFieldName()`) — only the map popup was asked to be curated.

## Map layers — always on, no toggle
- The Layers button/menu (which let you turn Parcels and Survey Points on/off
  individually) was removed entirely — both layers are just always on now,
  rendered once at startup (`showParcelsLayer()` / `renderSurveyPointsLayer()`
  in `startApp()`). No `showParcels`/`showSurveyPoints` state anymore; there's
  nothing left to toggle them off with. Parcels is still zoom-gated
  (`PARCELS_MIN_ZOOM = 15`) and still refetches on pan/zoom past that.
- One pre-existing limitation, unchanged by this: the Survey Points overlay
  is built once at startup from whatever `surveyRecords` looked like then —
  it does **not** re-render on the periodic 5-minute survey refresh, so a
  long-running session's dots can go stale relative to the list/map markers
  (which do refresh). Worth fixing if it turns out to matter in the field.

## Survey Points map overlay + match highlighting
- The **Survey Points** layer's markers are color/glyph-coded by address
  match, not a single fixed blue: a **green circle with a white check**
  (`surveyPointIcon()`, `STATUS_COLORS.surveyed`) when a survey record's
  normalized street+city matches something on the acquired list, a
  **yellow circle with a black X** (`SURVEY_NOMATCH_COLOR`, `#FBBF24`) when
  it doesn't — computed in `correlateAll()` as `r.addressMatched`,
  independent of which acquired property (if any) ends up "claiming" that
  record as its best match. Lets a field user spot address-quality issues
  (typos, wrong addresses entered by inspectors) directly on the map. The
  popup states the same thing in words.
  - Changed 2026-09-14 from plain colored dots (`L.circleMarker`) to a
    checkmark/X glyph (`L.marker` + `L.divIcon`, via `surveyPointIcon()`),
    per user request, so the survey-point layer is the one that "owns" the
    checkmark now — see the acquisition-marker note right below.
- Reverted later the same day, per user preference: the acquisition-list
  marker keeps its **larger** checkmark (`makeIcon()`, 26px/38px when
  selected) — both marker types are checkmarks now when matched/surveyed,
  and the **size** (26-38px acquisition vs 17px survey point) is what
  tells them apart on the map, not the glyph. Not-surveyed acquisition
  markers are unchanged (red circle with a white "!").
- Selecting a **matched** property (from the list or the map) draws a
  dashed connector line plus a pulsing halo (`.match-halo`) on its matched
  survey point (`showMatchHighlight()` in `selectProperty()`) — shown
  regardless of whether the Survey Points overlay toggle is on, since the
  point is to show *that one* link, not the whole layer. Cleared
  automatically when a different property is selected.

## Acquisition-list coordinate corrections (2026-09-14)
User spotted 8 acquired properties whose points were badly wrong on the
map (via the location-suspect warning + just eyeballing the map) and
listed them for correction. Fixed 7 directly in `ACQUIRED_RAW`; the root
cause was different for each, which is worth recording since more bad
rows likely exist in the other ~1,300:

| # | Address / City | Bad value | Root cause | Fix |
|---|---|---|---|---|
| `i:826` | 1075 Willow Industrial Ct, Cookeville | lat 85.54, lng -36.14 (North Pole) | **lat/lng swapped** | Swapped back: lat 36.143056, lng -85.536944 |
| `i:823` | 517 West Front St, Erin | lng +87.70 (China) | **longitude sign dropped** (should be negative — US) | Flipped sign: lng -87.70309412 |
| `i:844` | 251 Midway Dr, Erin | lat/lng 0, 0 (Null Island) | **duplicate row** for the same address — a second, separate `i:822` entry already had good coordinates | Copied `i:822`'s coordinates (36.314313, -87.7021). Note: this address still appears **twice** in the list (`i:822` and `i:844`) — a genuine duplicate acquisition-list entry, not just a bad-coordinate one. Left both in place since fixing coordinates was the ask, but worth a follow-up decision on whether to actually be two entries. |
| `i:709` | 1140 A & B Thompson Alley, Franklin | lat 32.93, lng -86.73 (near Montgomery, AL) | bad geocode, no obvious pattern | Re-geocoded via US Census Bureau geocoder (see below): lat 35.9149906574, lng -86.866700840824 |
| `i:55` | 4517 Fagan St, Chattanooga | lat 34.60 (~30 mi south, in GA) | bad geocode, no obvious pattern | Re-geocoded: lat 34.998654327825, lng -85.311518882985 |
| `i:167` | 207  Academy Street, Elizabethton (note double space in the address string) | lat 35.35 (~1° south, into NC), county "Polk County" | **whole-degree latitude typo** (36→35) — neighboring rows `i:166`/`i:168`/`i:169` on the same street all read ~36.3496; county field was also wrong, presumably a side effect of the same bad point (Polk County, TN is a real but wrong county down near the GA/NC line) | Fixed lat to 36.34956421 (matching the street's other entries) and county back to Carter County |
| `i:827` | 1152 Tuckahoe Dr, Nashville | lat 39.26 (in Indiana) | **whole-degree latitude typo** (36→39) — longitude was already correct (-86.76) | Re-geocoded via Census to confirm: lat 36.261342874671, lng -86.760621450925 |
| `i:68` | 18 Mona Lane, Oak Ridge | lat 34.00, lng -84.31 (Atlanta, GA area) | bad geocode | Census geocoder had no record of Mona Ln at all. **Fixed 2026-09-14** once the user supplied the correct coordinates from Google Maps: lat 36.00228282302554, lng -84.30536433195616. |

Corrections for `i:709`, `i:55`, and `i:827` came from the **US Census
Bureau's public geocoder**
(`geocoding.geo.census.gov/geocoder/locations/onelineaddress`), a free,
no-key, authoritative source for US street addresses — worth reusing for
any future one-off corrections like this rather than guessing. It has
gaps for smaller/newer streets (didn't have Mona Ln at all), so it's not
a substitute for the survey layer's own geocode notebook when doing this
at scale, just a good tool for a handful of one-off fixes.

All 8 of the originally-flagged rows are now fixed.

**Still worth doing**: this was 8 rows found by one person skimming the
map/warnings, out of ~1,300 — the same lat/lng-swap, sign-drop, and
whole-degree-typo patterns found here are mechanical enough that a script
could scan the whole list for candidates (e.g., points outside a rough TN
bounding box, or `|lat| > 90`/`|lng| > 180` outright) rather than relying
on someone spotting each one visually. Not built yet.

## Survey points getting visually buried under acquisition markers
User report: survey points for matched properties only seemed to appear
once the acquisition marker was selected. Both map layers were already
always-on (see "Map layers — always on, no toggle" above) — nothing was
actually conditional on selection. The real cause: both layers' markers
live in the same Leaflet pane (`markerPane`), and without an explicit
`zIndexOffset` a marker's default z-index is just its screen-Y position —
the same rule for every marker regardless of which layer it's logically
in. Two points sitting only a few feet to ~150ft apart (the common case
for a good match, by construction — see the Correlation logic section)
have near-identical screen-Y, so which one drew on top was effectively a
coin flip. The **larger** acquisition-list circle (26-38px) could easily
end up fully covering the **smaller** 17px survey dot underneath it —
selecting the property just happened to draw the pulsing halo + dashed
line right on top of everything, making it look like the survey point
had appeared for the first time.

Fixed 2026-09-14: `renderSurveyPointsLayer()`'s markers now set
`zIndexOffset: 500`, guaranteeing survey points always render above
acquisition markers (default offset 0) regardless of screen position —
so the checkmark/X is visible at a glance without selecting anything.
Trade-off worth knowing: when a survey point sits exactly on an
acquisition marker, the survey point (now always on top) can make the
acquisition marker itself harder to click directly at that exact pixel —
the existing near-miss click-priority fallback (`nearestPointFeature()`,
`POINT_CLICK_PRIORITY_PX`) helps, and the acquisition marker's visible
edge/ring outside the smaller survey dot's footprint is still directly
clickable.

## Filters & list UI
- Filter drawer has three single-select filters on the **acquired-property
  list/map**: **Survey Status**, **Region**, and **County**
  (`activeStatus`/`activeRegion`/`activeCounty`, each `'all'` == no
  filter). County is a native `<select>` rather than a button grid — there
  are 42 distinct counties in the acquired list, computed fresh in
  `buildFilterGrid()` from `properties.map(p => p.county)` — too many for
  the button-grid pattern the other two use.
- **Bug fixed 2026-09-14**: filtering never actually changed what showed
  on the *map* — only the side list. `refreshMarkers()` was looping over
  `properties` (the full, unfiltered array) instead of `renderedList`
  (what `getFiltered()` produces and what the list is built from), so the
  map markers ignored Survey Status/Region/County/search entirely while
  the list below correctly narrowed. Fixed by having `refreshMarkers()`
  loop over `renderedList` instead — it's always set immediately before
  `refreshMarkers()` is called (both happen inside
  `applyFiltersAndRender()`), so this is safe.
- **County dropdown now cascades from Region** (2026-09-14): picking a
  Region narrows the County `<select>` to only the counties that actually
  occur in that region (`buildFilterGrid()`'s `countySource`), instead of
  always listing all 42 statewide. If the currently-selected county isn't
  valid under the newly-picked region, `setRegionFilter()` resets County
  back to "All" rather than silently keep filtering on a county that's no
  longer even in the list. `setRegionFilter()` now calls `buildFilterGrid()`
  to rebuild the whole grid (previously it just toggled CSS classes on the
  region buttons) so the county options stay in sync.
- **New: Survey Points map filter** (2026-09-14, `activeSurveyMatch`,
  `setSurveyMatchFilter()`) — All / No Address Match, in its own row in the
  filter drawer. This is deliberately **separate** from the three filters
  above: it only controls which dots `renderSurveyPointsLayer()` draws on
  the map (survey records with no address match on the acquired list —
  the yellow-X markers), and has no effect on the acquired-property list,
  its markers, or its count. Re-renders just that one layer via
  `renderSurveyPointsLayer()` rather than going through
  `applyFiltersAndRender()`.
- **Survey Points filter buttons now show counts** (2026-09-14): "All" and
  "No Address Match" each display a live number (`.fgt-count`,
  `surveyMatchCounts` in `buildFilterGrid()`) — how many survey points that
  option would put on the map, using the exact same scope
  `renderSurveyPointsLayer()` itself applies (valid geometry + the active
  Region/County), minus the match filter itself, since that's the thing
  being counted per-option. `buildFilterGrid()` (which rebuilds these
  counts along with everything else in the drawer) is now also called
  right after every `correlateAll()` in `fetchSurveyRecords()` — both the
  live-fetch success path and the cached/offline fallback paths — so the
  counts stay current on the periodic 5-minute survey refresh too, not
  just when a filter changes.
- **Region/County now also scope the Survey Points layer** (2026-09-14,
  follow-up to the above): a survey record's own `region`/`county`
  attributes (`SURVEY_FIELDS.region`/`.county`) are checked in
  `renderSurveyPointsLayer()`'s filter chain, same active state
  (`activeRegion`/`activeCounty`) the acquisition list/map already use.
  `applyFiltersAndRender()` now calls `renderSurveyPointsLayer()` too (it's
  a no-op before the first survey fetch, since `surveyRecords` is still
  empty then) so picking a Region/County re-renders both layers together.
  Survey Status deliberately does **not** apply to this layer — every
  record in it is already a completed survey, so the acquisition list's
  surveyed/not-surveyed distinction has no meaning here.
- The **"Distance from me" buffer filter was removed** (along with the
  `p.distance` calculation and the distance-based list sort) — the list
  now always sorts alphabetically by address. The **Locate/re-center
  button and the user's blue dot on the map are unaffected** — that's
  still there for map navigation (`requestLocation()`/`recenterOnUser()`),
  it just no longer computes or uses a per-property distance for anything.
- **Survey date on the list, sorted most-recent-first** (2026-09-14):
  each surveyed property's card now shows a small date badge
  (`surveyDateLabel` in `renderCard()`) from the matched survey record's
  `inspection_date` (`prop.surveyDate`, set in `correlateAll()` alongside
  `prop.surveyRecord`). The list's default sort (`getFiltered()`) changed
  from always-alphabetical to **most-recently-surveyed first**: properties
  with a usable survey date sort newest-first at the top
  (`surveyDateMs()` handles the blank/unparseable-date edge cases so
  those never silently corrupt the sort); anything without one — not
  surveyed, or surveyed but the matched record's `inspection_date` is
  blank — falls to the bottom, alphabetical among themselves, same as the
  old default behavior for the whole list.
- Each list card's footer now has a single **Details** button only — the
  **Survey123 launch button was removed from the list card** since the
  detail sheet already has its own "Launch Survey123" button
  (`survey-launch-btn` in `openDetail()`); no reason to offer it twice.
  `launchSurvey123()` itself is untouched, just no longer wired to a
  list-card button.

## Flagging likely-wrong acquisition-list locations
A real, live example surfaced this: a property's acquisition-list point sat
nowhere near where its (address-matched) survey record actually was —
visibly obvious once `showMatchHighlight()`'s dashed connector line was
implemented, since the line ran clear off the visible map. The acquired
CSV's lat/lng is generated separately from the survey data (see Data
sources above) and clearly has some bad geocodes/typos in it.

- `correlateAll()` now also sets `prop.locationSuspect = true` whenever
  `matchDistanceFt > CONFIG.suspectLocationFeet` (default **5280 ft / 1
  mile**, tunable). This can only fire for an **address-only** match — a
  proximity match is by definition within `CONFIG.proximityMatchFeet` (150
  ft) of the acquired point — so when it fires, the address text lined up
  but the two points are genuinely far apart, meaning the *acquired-list*
  coordinate (not the survey point, which came from an inspector standing
  there) is the more likely culprit.
- Surfaced in three places, all sharing the same wording
  (`locationWarningText()`): the map marker popup (`buildPopupHtml()`), the
  list card — as a warning-styled distance pill plus an explicit warning
  line (`renderCard()`) — and a prominent warning banner at the top of the
  detail sheet (`openDetail()`), right under the Surveyed/Not Surveyed
  banner. Color is `LOCATION_WARNING_COLOR` (`#B54708`, amber), chosen to
  be visually distinct from both status colors (green/red) and the
  compliance palette.
- This only flags properties that already have a survey match — an
  unsurveyed property's location can't be cross-checked this way yet, so no
  warning shows for those (a bad geocode on an unsurveyed property is
  currently invisible until it gets surveyed and correlates).

### Map popup header: which kind of pin is this?
With survey points and acquisition-list markers both visible on the map at
once (see "Survey Points map overlay" above), a field user could click
either kind of pin and get a popup with no label saying which one it was.
Fixed 2026-09-14: `buildPopupHtml()` (the acquisition-list marker) now opens
with a bold "Acquisition List Location" header in violet (`#6941C6`,
`ACQUISITION_KIND_COLOR` — chosen to be distinct from the survey popup's
blue, the surveyed/unsurveyed green/red, and the location-warning amber),
mirroring how `buildSurveyPointPopupHtml()` already opens with "Survey
Record" in blue. The Surveyed/Not Surveyed status (previously the popup's
top line) moved down into a badge alongside the compliance badge, so the
very first thing a user reads now identifies the pin type, not its status.

## Detail-sheet wording: separating "how matched" from "how far apart"
User feedback on a screenshot of the detail sheet's SURVEYED banner:
"Matched by address match and within 115 ft of a survey point" — asked
whether that meant the survey location is 115 ft from the acquisition-list
location. It does (it's the same `p.matchDistanceFt` used everywhere else
— the card's 📍 pill, the location-suspect warning), but the sentence
didn't say so; it read like "within 115 ft" was describing *how* the
proximity match was found, not stating a distance between two points.

Fixed 2026-09-14 by splitting the two ideas apart everywhere the detail
sheet shows them:
- `matchMethodLabel(p)` now says ONLY how the match was found — "address
  text", "location proximity", or "address text and location proximity" —
  no distance figure in it at all.
- New `matchDistanceLine(p)` states the distance on its own, worded so
  it's unambiguous: `"115 ft between the list location and the survey
  point"`. Shown as its own line in the SURVEYED banner (skipped there
  when `p.locationSuspect` is true, since the amber "Location May Be
  Wrong" banner right below already states the same number — no reason to
  say it twice) and as an always-visible **"List-to-Survey Distance"** row
  in the Status section of the detail sheet, so it's there even when
  there's no banner to catch it.
- Added a `title` tooltip ("Distance between the acquisition-list location
  and the matched survey point") to the 📍/⚠ distance pill on list cards
  and map popups (`renderCard()`'s `dist`), for the same reason — it was
  showing the right number with no label at all.

## Map zoom on load
`initMap()` starts the map at a static statewide view (`[35.85, -86.4]`,
zoom 7), but `startApp()` immediately calls `fitToProperties()`
(`map.fitBounds()` over all 1,300 acquired properties, capped at
`FIT_ALL_MAX_ZOOM = 13`) right after — so the intended default view has
always been "fit to every collected point," not the static statewide view.

That fit-all view was being **immediately undone**: `startApp()` also
kicks off an automatic, silent `navigator.geolocation.getCurrentPosition()`
call to place the user's location dot, and its success handler
(`onLocationSuccess()`) unconditionally called `map.flyTo(userLocation,
10)` — flying the map away from the just-set fit-all view to the user's own
location, every single load. Fixed 2026-09-14: `onLocationSuccess()` now
takes a `{ recenter }` option (default `true`); the automatic startup
lookup passes `recenter: false` (drops the dot, doesn't move the map), while
the explicit **Locate** button (`recenterOnUser()` → `requestLocation()`)
still gets the default `true` and flies there as before, since that's a
deliberate user action.

## Survey Report export (.xlsx)
Added 2026-09-14: an **"Export Survey Report"** button at the bottom of the
filter drawer (`generateSurveyReport()`), for a downloadable summary of the
raw survey data separate from the app's own list/map view.

- **Scope**: every survey record whose own `region`/`county` attributes
  match the currently-active **Region** and **County** filters —
  deliberately *not* Survey Status or the Survey Points match filter, since
  the point of the report is "every record collected in this area," not a
  further-filtered subset. `buildSurveyReportData()` does the filtering and
  all the derived calculations.
- **What it computes per survey record**: whether its normalized
  street+city matches something on the acquired list (`r.addressMatched`,
  same logic `correlateAll()` uses), which acquired address(es) it matches,
  and two duplicate flags:
  - **Survey duplicate**: more than one survey record shares the exact same
    normalized address — almost always a duplicate field submission (the
    same property surveyed/submitted twice).
  - **Acquisition duplicate**: a survey record's address matches more than
    one row on the acquired list — almost always a duplicate address on
    the acquisition list itself (the 1,300-row CSV isn't guaranteed unique
    by address).
- **Output**: a 3-sheet workbook — **Summary** (counts + which
  Region/County filter was active), **Survey Records** (one row per record
  in scope, sorted by address), and **Duplicates** (just the rows flagged
  either way, for a quick review list). Built client-side with SheetJS
  (`window.XLSX`, loaded from cdnjs — see `<head>`), no server involved.
  Filename includes the active region/county and the date, e.g.
  `HMA_Survey_Report_Middle_DavidsonCounty_2026-09-14.xlsx`.
- **Offline fallback**: if the SheetJS CDN script didn't load (no signal in
  the field), `generateSurveyReport()` detects `typeof XLSX === 'undefined'`
  and exports a plain CSV of the Survey Records sheet instead (via a Blob +
  temporary `<a download>` link) — same data, just one sheet, and a
  same-session alert tells the user why they got a CSV instead of an xlsx.
- Not yet tested against a real large survey dataset in a real browser —
  worth confirming file size/row count stays comfortable and that the CDN
  actually resolves from a field device before relying on this.
- **Fixed 2026-09-14 — downloads blocked when embedded**: the user reported
  the export doing nothing when this app is loaded inside an ArcGIS
  Experience Builder widget's `<iframe>`. Root cause: both `XLSX.writeFile()`
  and the original CSV export built a hidden `<a>` and triggered it with
  `.click()` from script — a **script-initiated** download, which is
  exactly what a sandboxed embedding iframe tends to block (browsers gate
  that on the iframe's own `sandbox="allow-downloads"` token, which an app
  running *inside* the iframe has no way to add itself). Fixed by never
  auto-triggering a download at all: `exportSurveyReportXlsx()` now calls
  `XLSX.write(wb, {type:'array'})` (bytes only, no auto-download) and both
  it and `exportSurveyReportCsv()` hand their Blob to a shared
  `presentDownloadLink()`, which renders a real, visible `<a href download
  target="_blank">` inside `#report-download-ready` (in the filter drawer,
  under the Export button) for the user to click **themselves**. A genuine
  user click on a real link is a much less restricted action than a
  script-triggered one, so this should get through even where the old
  auto-download didn't — though if the embed's sandbox is locked down
  enough (no `allow-downloads` at all, even for user gestures), no amount
  of client-side JS can force a download through it; that requires whoever
  configures the Experience Builder embed to add `allow-downloads` (and
  `allow-popups`, for the new-tab link below) to the iframe's `sandbox`
  attribute.
  - Also added a general escape hatch for the same problem: an "Open the
    app in a new tab" link (`#open-newtab-link`, always visible in the
    filter drawer, `href` set to `location.href` in `startApp()`) — since
    it's a real link with `target="_blank"`, clicking it opens this same
    app in its own top-level browser tab, outside the iframe's sandbox
    entirely, where downloads (and anything else the embed might restrict)
    should just work normally.

## Validate Acquisition Locations export (.xlsx)
Added 2026-09-14, alongside the manual coordinate fixes above: the user asked
whether the "scan the whole acquisition list for likely-bad coordinates"
idea (originally floated as a one-off notebook) could instead be a
downloadable export inside the app itself, like the Survey Report. It is —
a second button, **"Validate Acquisition Locations"** (`#validate-locations-btn`,
outlined/secondary style so it doesn't compete with the primary Survey
Report button), right below Export Survey Report in the filter drawer,
sharing the same scoping note, `presentDownloadLink()` mechanism, and
online-doesn't-matter offline-CSV-fallback pattern.

- **Scope**: same as the Survey Report — properties whose own
  `region`/`county` match the active Region/County filters.
  `buildLocationValidationData()` does the filtering and flagging.
- **What it checks per property** (`validateAcquisitionLocation(p)`),
  modeled directly on the actual bad-coordinate patterns found and fixed
  by hand this session (see "Acquisition-list coordinate corrections"
  above):
  - Missing/non-numeric coordinates, or exactly `0,0` (Null Island).
  - Latitude/longitude outside their valid ranges (`>90`/`>180`).
  - Positive longitude (should always be negative for a US location) —
    catches the sign-dropped case (e.g. Erin → China).
  - Outside a coarse Tennessee bounding box (`TN_BBOX`, lat 34.9–36.75,
    lng -90.4 to -81.5) — a deliberately loose rectangle, not a precise
    state-boundary check, so it flags "obviously nowhere near TN" without
    false-flagging real border-adjacent addresses.
  - When a point is out-of-bbox, also checks whether swapping lat/lng
    would land it back in the box, and if so calls it out specifically as
    a likely **swap** (the exact Cookeville → North Pole pattern).
  - Duplicate coordinates and duplicate addresses across the in-scope
    list, reusing the same normalized-address comparison the Survey
    Report's duplicate detection uses — this surfaces a lot more than the
    8 originally-reported rows (spot-checked against the full 1,300-row
    list: ~55 duplicate-coordinate groups and ~62 duplicate-address
    groups list-wide, including one Jackson-area cluster of 100+ rows
    sharing a single coordinate, and ~20 rows all sharing the "500 Steam
    Plant Road, Gallatin" coordinate) — worth a look, not necessarily all
    errors, since some acquisitions genuinely share a site.
- **Output**: a 2-sheet workbook — **Summary** (counts + active
  Region/County filter) and **Flagged Locations** (only the rows with at
  least one issue, sorted by address, with an "Issues Flagged" column
  listing every issue found for that row in plain English). Properties
  with no issues aren't listed at all — the report is meant to be a short
  punch list, not a full dump of all 1,300 rows.
- If nothing in scope is flagged, `generateLocationValidationReport()`
  alerts the user instead of producing an empty file.
- Validated against the live 1,300-row `ACQUIRED_RAW` dataset via a
  throwaway Node script before shipping: zero false positives, and all 8
  of the previously-fixed rows (see above) now come back clean.
- Not yet run by the user against the real list — the duplicate counts
  above are informational, surfaced here so they're not a surprise the
  first time this export is used.

## Third correlation tier: partial address match
Added 2026-09-14. Real, live example the user flagged with screenshots: an
acquisition-list point ("7505 ANTIETAM", Murfreesboro) sat right next to a
survey point ("7505 Antietam Ln") — obviously the same property — but
showed **Not Surveyed** on the acquisition side and **No address match on
acquired list** on the survey side. Root cause: the acquisition list's
address is missing its street-type suffix entirely ("ANTIETAM" vs.
"Antietam Ln"), so `correlationKey()`'s exact normalized-string comparison
never lined the two up, and the two points were apparently just far enough
apart that the existing 150 ft plain-proximity check also missed it.

`correlateAll()` now has a third, last-resort correlation tier for exactly
this gap:

- **`streetCore(streetAddr)`** (near `correlationKey()`) splits an address
  into `{ houseNum, core }` — the leading house number, and everything
  else with a trailing street-type suffix (ST/AVE/DR/RD/LN/CT/BLVD/CIR/PL/
  HWY/ALY/TRL/PKWY/TER, i.e. `STREET_TYPE_SUFFIXES`) stripped off if
  present. Deliberately does **not** strip directionals (N/S/E/W/NE/NW/SE/
  SW) — "700 N Main St" and "700 S Main St" are genuinely different
  streets, not a formatting gap, so those must NOT be treated as a match.
- **The PARTIAL ADDRESS MATCH pass** (inside `correlateAll()`'s per-
  property loop, precomputing each survey record's `streetCore()` once up
  front rather than per pair): for each acquired property, finds the
  nearest survey record within `CONFIG.partialMatchFeet` (default **600
  ft** — looser than the 150 ft plain-proximity tier, tunable) whose
  `houseNum` and `core` both exactly match the property's, AND whose city
  matches. Only used as a last resort — if the property already matched by
  exact address text or plain proximity, the partial tier never overrides
  it (`prop.matchMethods` gets `'partial'` added only when neither of the
  other two fired).
- Surfaces the same way the other two tiers do: `p.surveyed`,
  `p.matchMethods` (now `'address'` | `'proximity'` | `'partial'`),
  `p.matchDistanceFt`, `p.surveyRecord` all populate normally.
  `matchMethodLabel(p)` describes it as "a close, partial address match —
  house number and street name agree, but the address text doesn't line
  up exactly" wherever match method is shown (detail sheet banner/Matched
  By row).
- **Survey Points map layer** now has a matching third visual state — see
  `surveyMatchState(r)` (`'matched'` | `'partial'` | `'none'`, the single
  source of truth `surveyPointIcon()`, `buildSurveyPointPopupHtml()`, and
  the Survey Points filter all key off): a survey record picked up by
  *some* property's partial match gets `r.partialMatched = true`
  (re-derived every `correlateAll()` run) and now shows **blue**
  (`SURVEY_PARTIAL_COLOR`, `#2563EB`) instead of yellow, with its own
  popup label ("Partial match — nearby acquisition record, address text
  differs"). The Survey Points (map) filter drawer now has three buttons
  instead of two — All / Partial Match / No Address Match — each with a
  live count. Deliberately did **not** fold partial matches into
  `r.addressMatched` itself — that flag stays a pure exact-text-match
  signal, since the Survey Report export's "Matches Acquisition List"
  column and its duplicate-detection logic are specifically about address
  **text** quality and shouldn't quietly change meaning.
- Validated the matching predicate (house number + core street name
  comparison) against the exact reported case plus edge cases (different
  house number, different directional, different street entirely) via a
  standalone Node script before shipping — all behaved as intended,
  including correctly rejecting "700 N Main St" vs. "700 S Main St".

## Survey point marker style + click behavior
Two related fixes, 2026-09-14, both from user feedback on a screenshot
where a selected acquisition marker (solid green circle, white check) sat
right next to its matched survey point (also, at the time, a solid green
circle with a white check) — hard to tell apart at a glance, especially
selected/halo-highlighted.

- **Survey points are now a white circle with a colored ring + colored
  check** (`surveyPointIcon()`) instead of a solid colored circle — green
  ring for an exact match, blue for partial (see above), amber ring +
  black X for no match at all. Acquisition-list markers (`makeIcon()`)
  keep their existing solid-fill look, so the two marker families now read
  as visually distinct shapes rather than same-style circles that happen
  to differ only in size.
- **Clicking a survey point now selects its linked acquisition property**
  instead of just popping up the survey record's own info in isolation.
  New `selectSurveyRecord(r)` (in `renderSurveyPointsLayer()`'s marker
  click handler) looks up the acquired property that actually claimed this
  survey record as its match (`properties.find(p => p.surveyObjectId ===
  r.objectId)`) and routes straight through the existing `selectProperty()`
  — the same function the acquisition marker/list card already use — so
  it clears whatever acquisition selection was previously active and shows
  this one instead (halo, dashed connector line, map fly-to, list card
  highlight, its own popup), exactly like clicking that property's own
  marker would. Falls back to just clearing the old acquisition selection
  (leaving the survey point's own popup, opened automatically by
  `bindPopup`) when the record has no linked property at all — a true "No
  Address Match" point has nothing on the acquisition side to jump to.

## Correlation logic (the actual point of this app)
Per user decision, a property counts as "surveyed" if **either** of two independent
checks matches — not just one — because the real survey export shows why both are
needed:
1. **Address text match**: normalize both the acquired CSV's Address+City and the
   survey layer's `address` (street portion only) + `city` — uppercase, strip
   punctuation, expand common abbreviations (ST/STREET, DR/DRIVE, NW/NORTHWEST, etc.)
   — and compare.
2. **Proximity match**: within `CONFIG.proximityMatchFeet` (default **150 ft**, easy
   to retune) of a survey record's point geometry, via haversine distance. Only
   considers survey records with valid (non-zero) geometry.

Validated against the real 653-row export + the real 1,300-row acquired list before
shipping (not just reasoned about): **536 of 1,300** acquired properties correlate to
a survey record — 385 by address only, 27 by proximity only, 124 by both. The 27
proximity-only matches are why proximity isn't optional: those are real surveyed
properties whose address text didn't line up closely enough to match on its own
(different abbreviation, minor typo, etc.).

`correlateAll()` re-runs on every survey-layer refresh (every 5 min by default, plus on
manual retry) against the full, static acquired list. It's an intentionally brute-force
O(properties × geo-tagged survey records) nested loop — realistic volumes here (~1,300
× a few hundred) are trivial for the browser; no spatial index was worth the complexity.

## Survey123 deep link — removed 2026-09-14
The detail sheet used to have a "Launch Survey123 for This Property" button
(and, before that, a matching one on the list card, removed in an earlier
session) that opened:
`https://survey123.arcgis.app/?itemID=e59afef666a2407085c631d624b89c02&field:address=...&field:city=...&field:region=...&center=lat,lng`

Removed at the user's request, along with the "Apple Maps" link that sat
next to "Google Maps" in the detail sheet's nav row (Google Maps stays).
Since the Survey123 button was the only caller of `launchSurvey123()`/
`survey123Url()`, those functions were deleted outright rather than left
as dead code, along with their `CONFIG.survey123ItemId`/`survey123BaseUrl`
entries and the `.survey-launch-btn`/`.survey-btn-wrap` CSS. The
`.nav-btns` row is now a single-column layout (`grid-template-columns:
1fr`) since only the Google Maps link remains. `.nav-btn.apl` CSS was
removed too; `.nav-btn`/`.nav-btn.goo` stay.

## OAuth / hosting
- New, separate AGOL app registration from PREDS Summary/Districts (per user
  decision): `clientId: 'd4YpqfdNJGpPJqsS'`, `redirectUri:
  'https://temagis.github.io/HMA_Open_Space/'`.
- Same PKCE authorization-code flow, same iframe-popup fallback, same silent-refresh
  pattern as the other two apps.
- **Both the survey layer and the parcels layer are queried with this app's own AGOL
  token** — confirmed the survey layer needs it (499 "Token Required"); the parcels
  layer only ever returned a bare 403 with no token-required message, so it's an
  assumption (consistent with it being in the same broader AGOL environment) that this
  app's org token will also satisfy it. Needs live verification once deployed.

## Filter drawer: full-screen instead of a capped dropdown
Fixed 2026-09-14. User feedback (screenshot): opening Filter showed the
Survey Status/Region rows, then cut off mid-way through Survey Points (map)
with the export buttons entirely off-screen — the drawer was a dropdown
capped at `max-height:480px` with its own internal scroll, and on most
phone screens the full filter grid (Status + Region + County + the 3-option
Survey Points row, added earlier this session) plus both export buttons
just didn't fit in 480px.

- `#filter-drawer` is now `position:fixed` covering the full viewport
  height (capped/centered to 480px wide the same way `#app` and
  `#detail-content` already are, so it still reads as the same phone-width
  panel on a wide desktop screen instead of stretching edge-to-edge) —
  slides down from `translateY(-100%)` to `translateY(0)` instead of
  animating `max-height`. Added its own header row (`.filter-drawer-header`
  — "Filters" title + a close `✕` button) since, full-screen, the "Filter"
  toggle button in the list header is now covered by the drawer itself and
  isn't available to close it.
- **Had to move `#filter-drawer` in the DOM** — it used to be nested inside
  `.list-controls` (itself `position:sticky`, inside `#list-section`,
  inside `#app`). That nesting broke the full-screen `z-index` the moment
  it was tried: `.list-controls` is `position:sticky`, which — sticky
  positioning always does this, even at a modest `z-index:10` — creates
  its own CSS stacking context, and `#app`'s `#header` is a flex item with
  `z-index:100` (flex items get their own stacking context from `z-index`
  even at `position:static`). A nested `position:fixed` descendant's
  `z-index` is only ever compared *within its nearest stacking-context
  ancestor* — so no matter how high `#filter-drawer`'s own `z-index` was
  set (tried 1900), it was still capped inside `.list-controls`'s
  context (z=10), and `#header` (z=100, a sibling context) kept painting
  on top of it. `getBoundingClientRect()` showed the drawer correctly
  sized to the full viewport the whole time — this was purely a paint-order
  bug, not a sizing one, which is why it wasn't obvious from just the CSS.
  Fixed by moving `#filter-drawer` in the HTML to be a sibling of `#app`
  (same place `#detail-sheet` already lived, for the exact same reason).
- Verified with a headless-Chromium script (`playwright`, since a live
  ArcGIS sign-in isn't available in this environment) across a handful of
  phone sizes: fits with zero scrolling at iPhone 12/11-ish heights
  (844px/896px); shorter screens (iPhone SE 667px, a small-Android 640px)
  still need a short scroll to reach the two export buttons — a reasonable
  fallback, not a regression, and far less scrolling than the old 480px cap
  needed on any screen. Also checked the ≥900px desktop split-view layout —
  the drawer centers correctly over the list column.

## Still needed before this can go live
- **Hosting**: `https://temagis.github.io/HMA_Open_Space/` needs a real GitHub Pages
  deployment, and that exact URL needs to be added to the AGOL app item's
  (`d4YpqfdNJGpPJqsS`) Redirect URIs list.
- **Parcels layer live check**: confirm the parcels service actually accepts this
  app's AGOL token (never confirmed — see above), and sanity-check a few real popups/
  detail-sheet Parcel Info sections against the generic attribute rendering to see if
  a curated, friendlier field list is worth building once the real schema is visible.
- **Proximity threshold**: 150 ft is a starting default (`CONFIG.proximityMatchFeet`)
  — revisit once the team has used it in the field for a bit.
- **Attachments**: survey-record photos are fetched the same way PREDS Summary does
  (`queryAttachments`), but this wasn't confirmed against a live token — worth a check
  once deployed.

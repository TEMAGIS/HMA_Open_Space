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

## Selecting a distant/suspect match cut the connector line off-screen
Fixed 2026-09-14. User feedback (screenshots): clicking a survey point
whose matched acquisition property is miles away (a `p.locationSuspect`
case — the banner reads "Acquisition list location may be wrong — matched
survey point is 3.5 mi away") zoomed the map in on the acquisition point
as usual, but that meant the OTHER end of the dashed connector line
(`showMatchHighlight()`) — 3.5 miles away — was off-screen, with no way to
see where the line actually led without manually panning/zooming out.

`selectProperty(id)` (used by both the acquisition marker/list card click
path and, since the click-behavior fix above, the survey-point click path)
now looks up the matched survey record's own coordinates up front, and
adds a corrective step, `ensureMatchVisible()`, that runs after the map
settles wherever the normal selection flow put it (the
`clusterGroup.zoomToShowLayer()` callback, or a `map.once('moveend', ...)`
after the plain `flyTo()` — needed because `flyTo` is animated, so the
check has to wait for the animation to actually finish rather than
inspecting the pre-flight bounds): if the matched survey point isn't
already inside the current view (`map.getBounds().contains(...)`), it
pulls back out with `map.flyToBounds()` over both points (capped at
`maxZoom: map.getZoom()` so it only ever zooms OUT to fit both, never in
past where the marker already was). For a normal nearby match the survey
point is already on screen, so this is a no-op — only a genuinely distant/
suspect match triggers the extra zoom-out. One accepted trade-off: for a
far-enough match, the acquisition marker can end up re-covered by its
marker-cluster bubble again after the zoom-out (expected at a more zoomed-
out view) — the dashed line and the survey point's own pulsing halo marker
(not clustered) stay visible either way, which is the part that mattered
here.

## Address normalization: apostrophes, and inconsistent survey-point symbols
Fixed 2026-09-14. User feedback (screenshots): "323 Neely's Bend Road"
showed Surveyed/Full Compliance on the acquisition side (a connector line
drawn to a real nearby survey point), but the survey point at the other
end of that line — and other clearly-related points nearby — still showed
the plain yellow "No Address Match" X, reading as an outright
inconsistency ("matched survey points showing up as unmatched", "some
matched addresses showing up with the wrong symbol"). Two separate root
causes, both fixed:

1. **`normAddr()` was splitting possessive street names on the
   apostrophe.** The final cleanup step (`.replace(/[^A-Z0-9 ]/g, ' ')`)
   turns every non-alphanumeric character into a SPACE, not just strips
   it — so "NEELY'S" became two tokens, `"NEELY"` and `"S"`, while the
   survey layer's "Neelys" (no apostrophe) stayed one token, `"NEELYS"`.
   The two addresses then normalized to different strings no matter how
   well the rest agreed, so neither the exact address-text match nor the
   newer partial-match tier (`streetCore()`, built on top of the same
   `normAddr()`) could ever line them up. Fixed by stripping apostrophes
   (straight `'` and curly `’`) entirely, as a dedicated step BEFORE that
   general replacement, so "NEELY'S" and "NEELYS" now both normalize to
   "NEELYS". Same fix benefits any other possessive TN street name
   (O'Brien, etc.) — verified with a standalone Node script for the exact
   reported case plus a curly-apostrophe variant and an "O'Brien"-style
   case; all now normalize identically on both sides.
2. **Survey Points map coloring only reflected the exact-text-match tier,
   not what a property actually matched by.** `r.addressMatched` (green
   check) was always meant as a pure address-text-quality signal,
   deliberately independent of which property (if any) claims a record —
   but that meant a survey record that WAS the real reason a property
   showed "Surveyed" (matched via plain proximity, or the partial-address
   tier) still painted as a plain yellow "no match" X, since neither of
   those tiers touched `r.addressMatched`. To a field user comparing the
   two layers side by side, that reads as a bug, not a subtle distinction.
   `correlateAll()` now also flags `r.proximityMatched` (mirroring the
   existing `r.partialMatched`) whenever a survey record is within
   `CONFIG.proximityMatchFeet` of a property, regardless of which tier
   ultimately "wins" as that property's official match — and
   `surveyMatchState(r)` now treats `r.partialMatched OR
   r.proximityMatched` as the blue "partial" state (not just
   `partialMatched` alone). So a survey record now shows green only when
   its own address text is an exact match, blue when it correlated to a
   property some other way (proximity or partial-address), and yellow
   only when it truly matched nothing at all — consistent with whatever
   the connected acquisition property's own Surveyed/Not Surveyed status
   says.

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
  needed on any screen.
- **Fixed 2026-09-14 (same day, second pass)** — at the ≥900px desktop
  split-view width, the drawer's own `max-width:480px; margin:0 auto;`
  (matching how `#app` centers itself) centered it across the FULL
  viewport width, not over the 420px list column specifically — obvious
  once the user saw it in a wide ArcGIS Experience Builder embed (well
  over 900px), where it floated in the middle of the screen, disconnected
  from both the Filter button that opened it (which lives in the list
  column) and the list itself. `#detail-content` already solves the exact
  same problem for the detail sheet with a `left:0; right:auto; width:
  420px; margin:0;` override inside the existing `@media (min-width:
  900px)` block — added the identical override for `#filter-drawer`, so it
  now sits flush over the list column at desktop widths instead of
  centered across the whole window. Verified via the same headless-
  Chromium approach at a wide (1148px) viewport matching the user's embed
  screenshot: drawer's `getBoundingClientRect()` now reports `{x:0,
  width:420}`, flush against the list column.

## Filter drawer: Survey Points moved to the top, "Survey Status" renamed
Fixed 2026-09-14. User feedback (screenshot): "For the filters can we have
the Survey Points at the top of the filters ... the survey status should be
titled TN Properties Acquired." Two small changes in `buildFilterGrid()`:

- The **Survey Points (map)** filter section (the All/Partial Match/No
  Address Match row that controls which dots show on the map — see
  `activeSurveyMatch`/`setSurveyMatchFilter()` above) now renders **first**
  in the drawer, ahead of Status/Region/County. A field user opening Filter
  is most often checking the survey layer's data-quality first, so that's
  now the first thing they see rather than something scrolled/tapped past.
- The old **"Survey Status"** label (the Surveyed/Not Surveyed/All row —
  `activeStatus`/`setStatusFilter()`) is now titled **"TN Properties
  Acquired"**, so it reads as its own distinct section (the acquisition
  list itself) now that it's no longer first/assumed to be the primary
  filter in the drawer.

Purely a `buildFilterGrid()` markup/ordering change — no filter logic,
state, or counts changed.

## List toggle: browse Survey Records directly, not just the acquisition list
Added 2026-09-14. Same user request as above continued: "can the list be of
the survey points since those are the details that are being shown" — the
list only ever showed acquisition properties, even though the detail sheet,
map layer, and filter drawer all treat survey records as their own
first-class thing. Ambiguous enough (filter the existing list by survey
data? replace it outright? something else?) that this went through
`AskUserQuestion` rather than guessing — the user picked **"Add a toggle to
switch between both lists"**: keep today's acquisition-property list as the
default, add a second mode that lists survey records instead, independent
of any filter choice.

- **`listMode`** (`'properties'` | `'surveys'`, default `'properties'`) is
  the new piece of state, switched via a small segmented control
  (`.list-mode-toggle`, two buttons: "TN Properties Acquired" / "Survey
  Records") now sitting above the list-count/Filter row in `.list-controls`.
  `setListMode()` flips it and re-runs `applyFiltersAndRender()` — it
  doesn't reset Region/County/Survey Points/search, so switching modes mid-
  filter keeps whatever scope was already active; only which dataset the
  list (and its count line) renders changes. The acquisition map/markers
  are unaffected by which mode the list is in either way.
- **`baseFilteredSurveyRecords()`** is `renderSurveyPointsLayer()`'s old
  inline filter chain (valid geometry + Region + County + the Survey Points
  match filter), pulled out into its own function so the map layer and the
  new list can never drift apart — both call it. **`getFilteredSurveyRecords()`**
  wraps that with search-term filtering and most-recent-inspection-first
  sorting (`inspectionDateMs()`, same null-safe pattern as the acquisition
  list's `surveyDateMs()`) — list-only, so typing in the search box doesn't
  also thin out the map's Survey Points layer.
- **`renderSurveyCard(r)`** parallels `renderCard(p)`'s exact markup/CSS
  classes (`.asset-card`, `.card-strip`, `.type-icon`, `.meta-badges`, the
  Details button) so the two list modes look and feel like the same
  component: color/icon/label come from `surveyMatchState(r)` (green check
  = Address Match, blue check = Partial Match, yellow X = No Address
  Match — same glyphs `surveyPointIcon()` already uses on the map, via
  `SURVEY_MATCH_ICONS`), badges show inspection date, compliance status,
  and inspector name. Cards use `data-survey-id="s<objectId>"` (the `s`
  prefix keeps them unambiguous from acquisition cards' `data-asset-id`,
  which is a plain property id) so `bindListHandlers()`'s existing single
  click/keydown listeners on `#asset-list` could be extended with a second
  branch rather than needing a whole separate event-binding function.
- **Selecting a survey card** (`selectSurveyCard()` → `selectSurveyRecord(r)`,
  the same function survey map-marker clicks already used) now also tracks
  `selectedSurveyObjectId` and highlights/scrolls to that record's own list
  card — independent of `selectedId` (the acquisition selection), so a
  survey card and an acquisition card can each show their own highlighted
  state at once without clobbering each other. (`selectProperty()`'s card-
  clearing line was narrowed from `.asset-card` to `.asset-card[data-asset-id]`
  so it no longer wipes out a survey card's highlight when a linked property
  gets selected underneath it.) When the record has a linked acquisition
  property, `selectSurveyRecord()` still just delegates straight into
  `selectProperty()` as before (halo, connector line, map fly-to, its own
  popup — unchanged). When it has **no** linked property, `selectSurveyRecord()`
  now also flies the map to and opens the popup of that record's own marker
  — via a new **`surveyMarkers`** lookup (objectId → Leaflet marker,
  populated in `renderSurveyPointsLayer()` alongside the existing layer
  build) — since there's no `selectProperty()` call to do that for it.
- **Details button, no linked property**: `openSurveyDetails()` checks for a
  linked acquisition property first — if one exists, it opens that
  property's existing, unchanged detail sheet (`openDetail()`, zero new
  code for that path) exactly as if its own card's Details button had been
  clicked. If there's no linked property, it opens **`openSurveyOnlyDetail(r)`**
  — a new, reduced detail view built from the same detail-sheet DOM/CSS as
  `openDetail()`: a match-state banner (colored/labeled from
  `surveyMatchState(r)`), the address header, `buildSurveyDetailRows()` for
  the Survey Results section (unchanged, already record-based), Parcel Info
  and Survey Photos sections reusing `loadParcelInfoInto()`/
  `loadAttachmentsInto()` via a small synthetic object
  (`{id:'s'+objectId, lat, lng}` / `{surveyed:true, surveyObjectId}` — both
  loaders already only key off those few fields, so no changes were needed
  to either), and a Google Maps link. Deliberately omits the Acquisition
  Info section and the `locationSuspect` banner — there is no acquisition
  record to compare this point against.
- Verified with a headless-Chromium script (Playwright, real ArcGIS sign-in
  isn't available in this environment) using a small hand-built Leaflet
  stub (`L`/`map` fakes covering just the marker/layerGroup/flyTo/popup
  calls this app makes) so `renderSurveyPointsLayer()`/`selectSurveyRecord()`
  could run without a live map: confirmed the toggle renders survey cards
  with correct counts/badges, clicking a card selects/highlights it, a
  card's Details button opens the full acquisition detail sheet when a
  linked property exists (checked "Acquisition Info" section is present)
  and the reduced survey-only sheet when it doesn't (checked that section
  is absent and the correct match-state banner shows), and toggling back to
  "TN Properties Acquired" restores the untouched acquisition list.

## Fourth correlation tier: city mismatch (address matches, city doesn't)
Added 2026-09-15. User feedback (screenshots): "412 Brook View Estates Dr"
showed up as a "No Address Match" yellow-X survey point right next to its
own acquisition-list marker showing "Not Surveyed" — same address text on
both sides, same region (Middle) — with no obvious explanation, since the
survey point's popup didn't even display its own city field to compare
against. "i'm not sure why these don't match."

Root cause: the exact-match tier (`correlationKey()`) requires the
normalized STREET text **and** the CITY string to both agree — and for
Nashville-area addresses in particular, the city field on a survey
submission very often doesn't match the acquisition list's (a USPS/mailing
city like Old Hickory, Hermitage, Antioch, Whites Creek, etc. gets typed
instead of the incorporated city, or the field is blank/mis-typed). The
partial-match tier (`streetCore()`) also requires city agreement, so it
couldn't rescue this either — only plain GPS proximity (150 ft) could, and
in this case the two points were apparently far enough apart that even that
missed it.

Asked the user how to handle this via `AskUserQuestion` (visibility-only
vs. also changing "Surveyed" status vs. a uniqueness-based version of the
latter) — the user's own answer supplied the actual rule to use: **"if the
address matches and the proximity is within 500 or so feet then it should
be the same city"** — i.e., trust the address text over the city label
when the two points are demonstrably the same physical spot.

- **`CONFIG.cityMismatchFeet: 500`** — new threshold, alongside
  `proximityMatchFeet` (150) and `partialMatchFeet` (600).
- **`correlateAll()`** now precomputes `r._streetNorm = normAddr(r.street)`
  for every geo-tagged survey record (same precompute-once pattern as
  `r._core` for the partial tier) and, per property, tracks the nearest
  survey record whose `_streetNorm` matches the property's own
  `normAddr(address)` **exactly** and is within `CONFIG.cityMismatchFeet` —
  city ignored entirely for this check. Only ever consulted (and only ever
  becomes the property's official match) when the city-qualified exact
  tier came up empty — that's a strictly higher-confidence version of the
  same signal, so this never overrides it. New method tag: `'citymismatch'`
  in `prop.matchMethods`, ranked **above** plain proximity but **below**
  the full address+city tier (an exact address-text match nearby is more
  certain than just "something nearby" with no text agreement at all).
  The distance requirement is what keeps this safe — an identical street
  address with no distance check at all would be too easy to false-positive
  on for a repeated street name somewhere else in the state; requiring it
  to also be right next to *this* acquired-list point is not.
- **`r.cityMismatchMatched`** (new, alongside `r.partialMatched`/
  `r.proximityMatched`) is flagged on the survey record whenever this tier
  finds a candidate — regardless of whether it ends up "used" as the
  official match — same pattern as the other two flags, so the map/list
  coloring reflects every real correlation. `surveyMatchState()` now folds
  it into the blue "partial" bucket alongside the other two.
- **More specific labeling for this exact case** (rather than reusing the
  generic "partial match" wording, which reads as "the address text didn't
  quite line up" — misleading here, since it lined up exactly): the map
  popup (`buildSurveyPointPopupHtml`), the Survey Records list card
  (`renderSurveyCard`), and the survey-only detail view
  (`openSurveyOnlyDetail`) all now check `r.cityMismatchMatched` and show
  "Address matches acquired list — submitted city differs" (or the card's
  short form, "Address Match, City Differs") instead. The detail sheet's
  "Matched By" line (`matchMethodLabel()`) does the same for a *property*
  matched this way: "address text — house number and street name line up
  exactly, but the submitted city differs from the acquisition list."
- **Survey Record popup now also shows the survey's own City** (it
  previously only showed Inspected/Inspector/Region) — the missing piece
  that made this bug impossible to diagnose from the map in the first
  place; now a city mismatch like this is visible at a glance even before
  reading the match-state badge.
- Verified with a headless-Chromium script exercising `correlateAll()`
  directly against the real acquired-list record for "412 Brook View
  Estates Dr" (Nashville, Davidson County): (a) same street text, city
  "Old Hickory", ~330 ft away → now matches via `citymismatch`, blue/
  partial state, "Surveyed"; (b) same setup but ~2,200 ft away (outside
  `CONFIG.cityMismatchFeet`) → correctly stays unmatched, confirming the
  distance safeguard actually holds; (c) same street text AND same city,
  far away → unchanged, still matches via the original full-confidence
  `address` tier regardless of distance (that tier never had a distance
  requirement, by design — see the correlation-tier docs above).
- Deliberately left untouched: `r.addressMatched` (used by the Survey
  Report export's "Matches Acquisition List" column) still requires the
  city to agree too — that field is specifically meant as a pure
  address-*and*-city text-quality signal for that report, independent of
  which property (if any) ends up correlating to a record; a city mismatch
  like this one is exactly the kind of data-quality issue that column
  exists to surface for the exported spreadsheet, so loosening it here
  would defeat its purpose.

## Partial-match tier: also ignore spacing inside a compound street name
Fixed 2026-09-15. User feedback (screenshots): a "1211 WOODSGREEN DR"
acquisition property already showed Surveyed/Full Compliance (matched to a
nearby survey point via some other tier), but a second, separately-
submitted survey record for the exact same address — "1211 Woods Green
Rd" — still painted as a plain yellow "No address match on acquired list"
X right next to it. Same house number, same city (Murfreesboro on both
sides this time — not another city-mismatch case), same core street name —
just "WOODSGREEN" as one word on the acquisition list vs. "Woods Green" as
two words on the survey side. "is there any logic tha[t] capture[s]
these?"

Root cause: `streetCore()` (used by the PARTIAL ADDRESS MATCH tier) already
stripped the trailing street-type suffix before comparing two addresses'
"core" street name — but it compared the core as a literal, space-joined
string, so "WOODSGREEN" (one token) and "WOODS GREEN" (two tokens, joined
with a space) never matched even though they're clearly the same name
split differently.

- `streetCore()` now also returns `coreCompact` — the same core with ALL
  internal spaces removed ("WOODS GREEN" → "WOODSGREEN") — alongside the
  original space-joined `core` (left as-is for anything that wants a
  readable version). The partial-match comparison in `correlateAll()` now
  compares `coreCompact` instead of `core`, so a compound street name
  written as one word on one side and multiple words on the other is
  recognized as the same street.
- Still gated by the same safeguards as before — same house number AND
  within `CONFIG.partialMatchFeet` (600 ft) — so this stays a tight,
  last-resort tier rather than a loose "sounds similar" match; it only
  loosens the specific "one word vs. several" formatting gap, nothing else.
- Verified against the real reported case (`1211 WOODSGREEN DR` /
  Murfreesboro, real acquired-list coordinates) with a synthetic survey
  record `1211 Woods Green Rd`, same city, ~220 ft away (outside the 150 ft
  proximity threshold, inside the 600 ft partial threshold): now correctly
  correlates via the `partial` tier (blue, "Surveyed") instead of showing
  as an unmatched yellow X. Re-ran the city-mismatch tier's own test suite
  (see above) afterward to confirm this didn't regress that fix.

## Partial-match tier: small edit-distance tolerance for near-miss spellings
Fixed 2026-09-15. User feedback (screenshots): "529 KINGS HILL BLVD"
(acquisition list) sat right next to "529 Kings Hills Blvd" (survey record)
— same house number, same city (Pigeon Forge), same suffix (Blvd) — yet the
survey point still showed "No address match on acquired list," while the
acquisition property was already shown Surveyed via a completely different
survey record 0.9 miles away that happened to match its address text
exactly. "why is this not being caught?"

Root cause: the singular/plural difference — "HILL" vs. "HILLS" — meant
even the already-fixed `coreCompact` comparison (exact string equality)
failed; the previous two fixes this session (apostrophes, then internal
spacing) each patched one specific kind of near-miss, and this is a third,
different kind (a one-letter spelling variant) that neither covers.

- Added a small **edit-distance tolerance** on top of the exact
  `coreCompact` match: `levenshtein()` (a plain single-character
  insert/delete/substitute distance function) plus `coresMatch()`, which
  first tries exact equality (the common case, cheap) and falls back to
  "edit distance ≤ `CONFIG.coreFuzzyMaxEdits`" (set to 2) when the shorter
  of the two core strings is at least 4 characters (guards against a very
  short/generic core matching too loosely). `correlateAll()`'s partial-tier
  comparison now calls `coresMatch()` instead of comparing `coreCompact`
  directly.
- Deliberately NOT a general fuzzy-address search: `coresMatch()` is only
  ever reached after the same house-number, city, and
  `CONFIG.partialMatchFeet` (600 ft) gates every other check in this tier
  already requires — so by the time it runs, the candidate is already
  narrowed down to "a survey point right at this one specific address."
  That's what keeps a distance-2 tolerance safe rather than opening the
  door to unrelated streets matching on vague similarity.
- Verified against the real reported case (529 Kings Hill(s) Blvd, real
  acquired-list coordinates): a synthetic survey record ~365 ft away now
  correctly matches via the `partial` tier; the same record moved to
  ~1,100 ft (outside `CONFIG.partialMatchFeet`) correctly does NOT match,
  confirming the distance gate still holds regardless of the new fuzzy
  tolerance. Re-ran the city-mismatch and compound-street-name test suites
  from the two fixes above afterward to confirm neither regressed.

## Map popups, list cards: show the acquisition-list address a survey record actually matched
Added 2026-09-15, same user session as the fixes above — after three
different address-normalization gaps in a row were only diagnosable by
manually comparing two separate popups (or, worse, not visible in the UI
at all — the "409 Brook View" city-mismatch case had no way to see the
survey's own city until that fix), the user asked more generally: "can
this show the corresponding address names used to match."

- **`matchedAcquisitionAddr(r)`** (new) resolves the acquisition-list
  `{ address, city }` a survey record correlates to — for direct side-by-
  side display. Prefers the property that's actually claimed this record
  as its official match (`properties.surveyObjectId`, the authoritative
  answer); falls back to whichever looser-tier candidate address
  `correlateAll()` stashed on the record (new fields, set alongside the
  existing `partialMatched`/`proximityMatched`/`cityMismatchMatched`
  booleans and reset the same way each run: `r.partialCandidateAddr`,
  `r.proximityCandidateAddr`, `r.cityMismatchCandidateAddr`) for a record
  that was flagged as a correlation but didn't end up "winning" as any one
  property's official match (see the "flag BOTH candidates" comment in
  `correlateAll()` — this can genuinely happen, e.g. a property that
  already had a full address+city hit to a DIFFERENT, distant record still
  flags a nearby partial/city-mismatch candidate too).
- **Survey Record map popup** (`buildSurveyPointPopupHtml`): for a
  'partial' (blue) record only — 'matched' is already identical text, and
  'none' has nothing to show — a new "Acquisition List" row shows the
  matched property's address/city directly above Inspected/Inspector/City/
  Region, so the two address strings sit side by side without needing to
  also open the acquisition marker's own popup.
- **Acquisition List Location popup** (`buildPopupHtml`): the reciprocal —
  a new "Survey Record" row shows the survey's own submitted address/city,
  shown whenever the property matched by anything OTHER than a clean full
  address+city hit (i.e. `matchMethods` isn't exactly `['address']']` —
  proximity, city-mismatch, or partial all mean the two texts don't read
  identically).
- **Survey Records list card** (`renderSurveyCard`): same idea, a small
  new `.card-addr-compare` row ("vs. <acquisition address>, <city>") under
  the badges for a 'partial' card, so the mismatch is visible while
  scrolling the list without opening Details either.
- Verified all three render correctly for the "529 Kings Hill(s) Blvd"
  case from the fix above: the survey popup's "Acquisition List" row and
  the card's "vs." row both showed "529 KINGS HILL BLVD, Pigeon Forge",
  and the acquisition popup's "Survey Record" row showed "529 Kings Hills
  Blvd, Pigeon Forge" — confirming the comparison surfaces in both
  directions.

## Toggle default changed to Survey Records, moved to the left
Changed 2026-09-15 per user request. `listMode` now defaults to
`'surveys'` instead of `'properties'`, and the "Survey Records" button is
now the first (left) button in `.list-mode-toggle`, with "TN Properties
Acquired" second — both the HTML's initial `active`/`aria-selected` state
and the JS default were updated together so they agree on load (previously
the acquisition list was both the default and the left button; now Survey
Records is both). `setListMode()` itself is unchanged — it already looks
up both buttons by id regardless of DOM order.

## Detail sheet: removed the Parcel Info section
Changed 2026-09-15 per user request ("no need to include parcel data on
the details panel"). Both `openDetail()` (acquisition properties) and
`openSurveyOnlyDetail()` (survey-only records) no longer show a "Parcel
Info" section — that whole point-in-polygon lookup against the parcels
service (`queryParcelAtPoint`, `getParcelInfoCached`, `loadParcelInfoInto`,
and the small `parcelPointCache`) was removed entirely as dead code once
nothing called it. Parcel attributes are still available the other way
this app has always shown them: clicking a parcel boundary directly on the
map still opens its own popup (`buildParcelPopupHtml`, under the PARCELS
BOUNDARY LAYER section) — that overlay and its popup are untouched, this
only removed the *second*, redundant copy that used to also appear inside
the detail sheet.

## Removed the "Validate Acquisition Locations" export
Changed 2026-09-15 per user request ("remove the acquisition location
validation, it is confusing people"). This was the filter-drawer button
that exported an .xlsx/.csv report of acquisition properties whose point
fell outside a rough Tennessee bounding box — a data-QA tool, not
something a field user filtering survey points needs to see, and
apparently confusing in practice. Removed the button and its entire JS
implementation (`TN_BBOX`, `inTnBbox()`, `validateAcquisitionLocation()`,
`buildLocationValidationData()`, `LOCATION_VALIDATION_HEADER`,
`locationValidationRow()`, `exportLocationValidationXlsx()`,
`exportLocationValidationCsv()`, `generateLocationValidationReport()`).
**Note**: this is a different feature from the per-property "Location May
Be Wrong" warning banner/triangle that can show on an individual
property's popup or detail sheet (`locationSuspect` /
`CONFIG.suspectLocationFeet`) — that one was left in place, since the
request named the export specifically. Flag if that one should go too.

## Filter drawer: every row now shows live counts
Changed 2026-09-15 per user request ("can all of the filters show the
counts"). Previously only the Survey Points row had per-option counts
(`surveyMatchCounts`); now Status ("TN Properties Acquired"), Region, and
the County dropdown all do too, following the same rule: each option's
count is scoped by every OTHER active filter, never by itself, so the
number always answers "how many would picking this leave on screen." A
new shared helper, `propertyMatchesStatus(p, status)`, is now the single
place that defines what each status option means — used by both
`getFiltered()` (the actual filter) and `buildFilterGrid()` (the status
and region counts, since region counts need to already respect whatever
status is active) — added specifically so the count logic and the filter
logic can't drift apart the way it's easy to accidentally let happen.

## "TN Properties Acquired": added a Partial Match option
Changed 2026-09-15 per user request ("should acquired also show partial
matches"). New 4th status option alongside All/Surveyed/Not Surveyed,
using the blue checkmark icon already defined for the Survey Points row
(`STATUS_ICONS.partial`). It's a subset of Surveyed — a property counts
as "Partial Match" when it's surveyed but its match wasn't a full, clean
address+city hit (i.e. it also or only involved the proximity,
city-mismatch, or fuzzy-partial-street correlation tier). Backed by a new
`isCleanAddressMatch(p)` helper (`matchMethods.length === 1 &&
matchMethods[0] === 'address'`) that `propertyMatchesStatus()` calls for
the `'partial'` case — the same helper is also reusable by the "Survey
Record" address-comparison row in the acquisition popup, though that row
still has its own inline equivalent check for now (not yet refactored to
call the helper — low-priority cleanup).

## Parcel popup: no more TPAD link on an empty popup
Fixed 2026-09-15 per user report (screenshot: a parcel popup reading "No
address/acreage data returned for this parcel." still showed a "View on
TN Property Assessment (TPAD) →" link below it). Root cause: the parcel
layer's field schema was never confirmed against the live service (see
the PARCEL_FIELD_CANDIDATES comment), so the link's field lookup had a
fallback list that included ParcelID-style fields (PARCELID/ParcelId/
MapParcelID) alongside true GIS-link fields — and a raw parcel ID is
present on nearly every feature, including ones with no address/acreage
at all, so the link kept appearing even when there was nothing else to
show. `PARCEL_FIELD_CANDIDATES.gislink` now only matches genuine
GIS-link-named fields (GISLINK/GIS_Link/GisLink/GIS_LINK); a parcel with
only a bare ID and no real link field now renders the popup with no link,
same as it already did for address/city/acres.

## Acquisition list: fixed "138 INDUSTRIAL DIRVE" typo
Fixed 2026-09-15 per user report. Record `i:253` (Carthage, Smith County)
read "138 INDUSTRIAL DIRVE" in the embedded acquisition data
(`ACQUIRED_RAW`) — a typo of "DRIVE". Beyond just looking wrong, this
typo actively broke survey correlation for that property: `normAddr()`'s
street-type abbreviation table only recognizes the correctly-spelled
"DRIVE" (→ "DR"), so "DIRVE" passed through unrecognized and
`streetCore()` couldn't strip it as a suffix — the property's core street
name computed as "INDUSTRIAL DIRVE" instead of "INDUSTRIAL", which is
too different (edit distance > `CONFIG.coreFuzzyMaxEdits`) from a
correctly-spelled survey-side "138 Industrial Dr" for even the fuzzy
partial-match tier to catch. Corrected the address text directly in
`ACQUIRED_RAW` rather than patching the normalizer for this one typo —
see the note below about a systematic geocode-based QA pass for catching
others like it across the full list.

## Geocoding QA pass for the acquisition list + a "geocoded" fallback tier
Added 2026-09-15, following the "138 INDUSTRIAL DIRVE" typo report above.
That typo wasn't just a cosmetic problem — it silently broke BOTH the
exact address-text tier and the fuzzy partial-match tier (the misspelled
word wasn't recognized as a street-type suffix, so it never got stripped
for the fuzzy core comparison either), which is exactly the kind of
failure text-only matching can't systematically catch. Discussed the
options and went with: run a geocoding pass over the whole acquisition
list, and have the app also try the geocoder's standardized address as a
fallback whenever the raw text doesn't match anything.

- **`hma-acquisition-geocode.py` / `.ipynb`** (new project docs, same
  pattern as `hma-survey-geocode.py`/`.ipynb`): since the acquisition list
  isn't a live AGOL layer (it's the static `TN_properties_with_county_2.csv`
  embedded as `ACQUIRED_RAW`), this script reads that CSV (a fresh export
  is included alongside it, `TN_properties_with_county_2.csv`, reflecting
  the current embedded data including the Industrial Drive fix), geocodes
  every address against the ArcGIS World Geocoding Service, and writes a
  NEW CSV (`TN_properties_with_county_2_geocoded.csv`) with `geo_address`/
  `geo_city`/`geo_score`/`geo_distance_ft`/`geo_flag` columns added —
  nothing is written back to any live service, and the original columns
  are never touched. Only rows where the standardized address actually
  differs from the source (`geo_flag == "corrected"`) are the ones worth
  reviewing; run it from ArcGIS Online/Enterprise Notebooks like the
  survey one, then send the output CSV back so it can be merged into
  `ACQUIRED_RAW`.
- **App-side fallback tier** (`correlateAll()`, `index.html`): each
  acquisition property now optionally carries `geoAddr`/`geoCity` fields
  (absent on every record until the script above has been run and merged
  in). When present and different from the raw address, and the raw
  address's exact-match tier found nothing, `correlateAll()` also probes
  the same address index with the corrected address — tagged `'geocoded'`
  in `matchMethods`, ranked just below a direct address hit and above
  every distance-based tier (city-mismatch/proximity/partial). A
  `'geocoded'` match does NOT count as a "clean" address match (see
  `isCleanAddressMatch()`), so it shows up under the "Partial Match"
  filter too — the whole point is to surface these for a human to
  eventually clean up in the source list, not to quietly paper over them
  forever.
- Verified with a synthetic test mirroring the real Industrial Drive case:
  with the typo and no `geoAddr`, the property stayed "Not Surveyed" (bug
  reproduced); adding `geoAddr: "138 Industrial Dr"` and re-running
  correlation flipped it to Surveyed via `matchMethods: ['geocoded']`,
  with the correct computed distance and match label, and it correctly
  appeared under the Partial Match filter. Re-ran the city-mismatch/
  Woods-Green/Kings-Hill regression tests from the earlier fixes too — all
  still pass unchanged.
- **Next step**: run `hma-acquisition-geocode.py` (or the `.ipynb`) in
  ArcGIS Notebooks and send back `TN_properties_with_county_2_geocoded.csv`
  — the `geoAddr`/`geoCity` fields won't do anything until that's done and
  the results get merged into `ACQUIRED_RAW`.

## Survey Records list: card icons now match the map, dropped the surveyor's name
Changed 2026-09-15 per user report (screenshot: the list showed a SOLID
green circle w/ white check for "Address Match" and a SOLID amber circle
w/ black X for "No Address Match" — visually its own thing, not what the
same records look like as dots on the map). The map's `surveyPointIcon()`
draws a WHITE circle with a colored RING and a colored check (or black X)
— added a matching `surveyCardMatchIcon(state)` for `renderSurveyCard()`
that reproduces that exact look (same ring colors: `STATUS_COLORS.surveyed`
green / `SURVEY_PARTIAL_COLOR` blue / `SURVEY_NOMATCH_COLOR` amber, same
black X for no match) as a static SVG instead of a Leaflet divIcon, so a
card and its marker now read as the same symbol. Also removed the small
"🧑 <inspector name>" badge that used to sit on every card — per user
request, the card is just the match/date/compliance summary now; the
inspector's full name is still shown in Details (unchanged —
`buildSurveyDetailRows()` already has its own Inspector row).

## Geocoding QA pass: results merged in, plus a safety gate the real data exposed
2026-09-15. The user ran `hma-acquisition-geocode.py` in ArcGIS Notebooks
and sent back `TN_properties_geocoded.csv` (1,300 rows). Breakdown:
942 `matches source` (geocoder confirms the address as-is — includes
i=253, "138 Industrial Drive", the hand-fixed typo from earlier), 17
`low confidence` (geocoder couldn't place it confidently — left alone,
not merged), and 341 `corrected` (standardized address differs from the
source).

Of those 341, only rows where the geocoded point landed within 1 mile of
the acquisition list's own stored coordinate (`CONFIG.suspectLocationFeet`
— same bar the app already uses elsewhere to flag a suspect location)
were trusted and merged in as `geoAddr`/`geoCity` on `ACQUIRED_RAW`: 258
rows. The other 83 were left out and saved to a new project doc,
`TN_properties_needs_review.csv` — the geocoder found a confident,
differently-spelled match, but far enough from the property's own point
that trusting it automatically felt wrong (could mean the geocoder
matched a same-named street somewhere else in the state, or — just as
plausible — that the acquisition list's OWN coordinate for that property
is the one that's off). One row in this set was a genuinely garbage match
(`i=28`, "1526 LOCKHART LANE" → geocoded address literally `"Lane"`,
~9,500 miles away) — the distance cutoff catches cases like this
automatically, which a score-only filter (the `MIN_MATCH_SCORE` the
script already had) didn't; the confidence score on that one was 80.76,
just over the 80 threshold.

**Real cases the merge surfaced, beyond the original "138 INDUSTRIAL
DIRVE" report**: acquisition-list records i=863–866, four consecutive
"WOODSGREEN DR" addresses on the same Murfreesboro street including the
one from the earlier "1211 WOODSGREEN DR" report — the geocoder says the
real suffix is "Rd", not "Dr", for all four. That report was originally
fixed by the fuzzy partial-match tier (suffix-and-spacing-insensitive
core comparison); it now also resolves through the cleaner `geocoded`
tier, ranked higher, since the corrected address is now on file.

**Distance-gating the `geocoded` tier** (`correlateAll()`): building the
test for this exposed a real gap — the `geocoded` tier was modeled after
the plain `address` tier (no distance limit at all, since two sides
typing the identical full address+city is trusted regardless of
distance). But a `geoAddr` value isn't literally what either side typed —
it's a third, inferred address from an external geocoding service, one
extra layer of "might be wrong" on top of the usual text-match risk. Real
test case that caught it: a synthetic survey point 1,100 ft from "529
KINGS HILL BLVD" (outside `partialMatchFeet`) still matched via its
`geoAddr` fallback, purely because the corrected address text happened to
line up exactly — clearly too loose. Added `CONFIG.geocodedMatchFeet`
(600 ft, same radius as the partial-match tier) and gated `geoAddrHits` by
it. Also had to back out an earlier attempt at flagging a survey record's
own address-text-match color (`r.addressMatched`, used for the Survey
Points map layer's green/amber coloring) via `geoAddr` — that set has no
distance concept at all, so it bypassed the new gate entirely. Replaced
with `r.geocodedMatched`/`r.geocodedCandidateAddr`, set inside the
distance-gated loop, same pattern as `partialMatched`/`proximityMatched`/
`cityMismatchMatched` — and, consistent with those three, a
`geocoded`-only match colors as amber "partial" on the map/list, not
green, even when it's the officially winning tier for the property. That
fits the whole point of tagging it `'geocoded'` instead of `'address'` in
the first place: keep it visibly flagged for review, not quietly treated
as fully clean.

Re-ran every regression test from this session's earlier fixes
(city-mismatch, Woods Green spacing, Kings Hill fuzzy-edit-distance, the
geocoded-fallback synthetic test, the filter-grid counts, the card-icon
mirroring) after this change — all still pass.

- **Hosting**: `https://temagis.github.io/HMA_Open_Space/` needs a real GitHub Pages
  deployment, and that exact URL needs to be added to the AGOL app item's
  (`d4YpqfdNJGpPJqsS`) Redirect URIs list.
- **Parcels layer live check**: confirm the parcels service actually accepts this
  app's AGOL token (never confirmed — see above), and sanity-check a few real map
  popups (`buildParcelPopupHtml`) against the generic attribute rendering to see if
  a curated, friendlier field list is worth building once the real schema is visible.
  (The detail sheet no longer has its own separate Parcel Info section as of
  2026-09-15 — see above — so this is now the only place parcel data shows up.)
- **Proximity threshold**: 150 ft is a starting default (`CONFIG.proximityMatchFeet`)
  — revisit once the team has used it in the field for a bit.
- **Attachments**: survey-record photos are fetched the same way PREDS Summary does
  (`queryAttachments`), but this wasn't confirmed against a live token — worth a check
  once deployed.

## Survey Records list card: label text wrapping + clearer "no match" wording
Fixed 2026-09-15 per user report (screenshot: "No Address Match" ran into
the address title on the same line — `.card-type-label` was a fixed
92px, `white-space: nowrap`, so any label wider than that column just
overflowed sideways into "153 Industrial Dr" instead of wrapping).
`.card-type-label` now wraps (`white-space: normal`, `word-break:
break-word`, tightened `line-height`) instead of forcing one line —
affects the same label on the acquisition-list card too, but those
labels ("Surveyed"/"Not Surveyed") are short enough that it's a no-op
there. Also reworded the "No Address Match" state to "No Address Match
with Acquisitions" (both `renderSurveyCard()` and the matching label in
`openSurveyOnlyDetail()`'s Details view, kept in sync) — spells out what
it's being compared against, which matters more now that it wraps onto
its own lines rather than reading as one short phrase next to the icon.

## New URL parameter: `?mode=surveyonly` — Survey Points only, no acquisition list
Added 2026-09-15 per user request: a URL parameter for embedding/sharing
a version of the app scoped to just the survey layer, for an audience
that shouldn't see (or doesn't need) the acquisition list at all. Same
"`?key=value` flag" pattern as the existing `?layout=full` (Experience
Builder embed sizing, top of the file).

Append `?mode=surveyonly` to the URL and:
- **Acquisition markers never render on the map** (`refreshMarkers()`
  returns immediately) — the Survey Points layer, parcels layer, and
  everything else map-related is unaffected.
- **The "TN Properties Acquired" list/toggle is hidden** — the
  `.list-mode-toggle` bar (`startApp()`) is hidden entirely rather than
  left as a button that would flip to a list the rest of the UI is
  hiding, and `setListMode()` refuses to leave `'surveys'` as a backstop
  even if something else calls it directly.
- **The "TN Properties Acquired" filter row is omitted** from the filter
  drawer (`buildFilterGrid()`) — Survey Points, Region, and County stay,
  since Region/County still scope the survey layer too.

Deliberately NOT changed: `buildProperties()` and `correlateAll()` still
run exactly as normal, so match state (green/amber/red, all four+
correlation tiers, the geocoded fallback) and every count still work
correctly — this parameter only hides the acquisition-list UI, it
doesn't change what a survey point's color means or skip any scoring.
Verified with a Playwright test that compares normal-mode vs.
`?mode=surveyonly` side by side: acquisition marker count goes from
1,300 (all properties) to 0, the filter drawer and toggle both disappear,
and a matched property still correctly shows `matchMethods` and
`surveyed: true` under the hood in both modes. Also caught and fixed a
real bug in the same pass: `.list-mode-toggle` sets its own `display:
flex`, which silently overrides the `hidden` attribute's default
`display: none` unless a `.list-mode-toggle[hidden] { display: none; }`
override exists too (same trap `#app[hidden]` already has its own
`!important` override for) — added that rule alongside it.

Example: `https://temagis.github.io/HMA_Open_Space/?mode=surveyonly`

## Fixed: signing in dropped `?mode=surveyonly` from the URL
Reported 2026-09-15, same day as the parameter above: "when I login it
redirects and removes the mode=surveyonly." Root cause was the OAuth
sign-in flow itself. When the app isn't embedded in an iframe (the normal
case for a `?mode=surveyonly` link opened directly, as opposed to an
Experience Builder embed), clicking Sign In does a **full-page redirect**
— `window.location.href` — to ArcGIS's authorize page, and ArcGIS later
sends the browser back to a fixed, exact-match Redirect URI registered on
the AGOL app item (`CONFIG.redirectUri`, no query string of its own,
just `?code=...&state=...` tacked on). Landing back on that bare URL is
what erased `mode=surveyonly` — the URL the app re-evaluates
`SURVEY_ONLY_MODE` from on that fresh page load simply no longer had it.
(The popup-based sign-in flow, used automatically when the app *is*
embedded in an iframe, never touches the main tab's URL at all, so it was
never affected — only the direct/full-page case was broken.)

Fix mirrors the pattern the OAuth code already uses for its own PKCE
verifier/state across this exact redirect round-trip (`sessionStorage`):
- `startOAuthFlow()`'s full-page-redirect branch now saves
  `window.location.search` to `sessionStorage['oauth_return_search']`
  right before navigating away.
- A new early inline script at the very top of `<head>` (same spot and
  pattern as the existing `?layout=full` parser) runs before anything
  else on the page: if the URL it lands on looks like an OAuth return
  (`code` or `error` present) and a saved search exists, it merges the
  saved params back into `window.location.search` via
  `history.replaceState` — before the main script's `SURVEY_ONLY_MODE`
  constant (or anything else that reads the URL) ever evaluates. This is
  what makes it work even though `SURVEY_ONLY_MODE` is computed once at
  script-parse time: the merge happens earlier still.
- `handleOAuthCallback()`'s three `history.replaceState(...)` calls (on
  success, on error, and on a state mismatch) now go through a new
  `_urlWithoutOAuthParams()` helper that strips only `code`/`state`/
  `error`/`error_description`, instead of wiping the whole query string
  — so the URL bar ends up clean (no OAuth crumbs) but keeps
  `?mode=surveyonly` (or anything else) visible and correct afterward,
  including on a later page refresh.

Verified with a new Playwright test (`test_oauth_redirect.js`) that stubs
`fetch` to fake a successful token exchange and simulates the full
round-trip three ways: (a) a saved `?mode=surveyonly` present before a
full-page OAuth redirect restores correctly — `SURVEY_ONLY_MODE` is
`true` and the URL ends up exactly `?mode=surveyonly` with no leftover
`code`/`state`; (b) a plain login with nothing saved still lands on a
clean, empty URL exactly as before; (c) loading `?mode=surveyonly`
directly, with no login involved, is unaffected by the new restore
script. Re-ran the full existing regression suite (`test_kingshill`,
`test_woodsgreen`, `test_citymismatch`, `test_geoaddr`, `test_filtergrid`,
`test_cardicons`, `test_labelwrap`, `test_surveyonly`) — no regressions.

## Fixed: `?mode=surveyonly` Region/County counts still described the hidden acquisition list
Noticed 2026-09-15, looking at a filter-drawer screenshot from `?mode=
surveyonly`: the Region row ("All 1300", "East 167", "Middle 749",
"Southeast 60", "West 324") and "All Counties (1300)" were unchanged from
normal mode — still counting the 1,300-property acquisition list, even
though that list is entirely hidden in survey-only mode (no map markers,
no list, no filter row for it). The numbers next to Region/County didn't
describe what was actually on screen.

`buildFilterGrid()`'s Region and County blocks now branch on
`SURVEY_ONLY_MODE`: in that mode both are computed from `surveyRecords`
(matching each button's `SURVEY_FIELDS.region`/`SURVEY_FIELDS.county`,
same geometry check `surveyMatchScope` already uses just below) instead
of from `properties`. Normal mode is untouched — still scoped to the
acquisition list exactly as before. The "Survey Points (map)" row's own
counts were already survey-based and needed no change.

Verified with a new Playwright test (`test_surveyonly_counts.js`): a
synthetic 3-record survey set is used in both modes so a
properties-based count (1300) and a surveys-based count (3) are
unmistakably different. Normal mode's Region "All" still reads 1300;
survey-only mode's reads 3, East reads 2, "All Counties (3)" and "Test
County (2)" match the synthetic set exactly, and the acquisition section
stays hidden. Re-ran the full regression suite (`test_kingshill`,
`test_woodsgreen`, `test_citymismatch`, `test_geoaddr`, `test_filtergrid`,
`test_cardicons`, `test_labelwrap`, `test_surveyonly`,
`test_oauth_redirect`) — no regressions.

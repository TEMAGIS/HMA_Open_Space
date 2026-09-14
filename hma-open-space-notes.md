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
- The **acquisition-list marker's own checkmark was removed** the same
  day: a surveyed acquisition property (`makeIcon()`) is now a **plain
  green circle**, no glyph — the checkmark previously shown there was
  easy to confuse with the survey layer's own (now more prominent)
  checkmark. Not-surveyed acquisition markers are unchanged (red circle
  with a white "!").
- Selecting a **matched** property (from the list or the map) draws a
  dashed connector line plus a pulsing halo (`.match-halo`) on its matched
  survey point (`showMatchHighlight()` in `selectProperty()`) — shown
  regardless of whether the Survey Points overlay toggle is on, since the
  point is to show *that one* link, not the whole layer. Cleared
  automatically when a different property is selected.

## Filters & list UI
- Filter drawer has three single-select filters now: **Survey Status**,
  **Region**, and **County** (`activeStatus`/`activeRegion`/`activeCounty`,
  each `'all'` == no filter). County is a native `<select>` rather than a
  button grid — there are 42 distinct counties in the acquired list,
  computed fresh in `buildFilterGrid()` from `properties.map(p => p.county)`
  — too many for the button-grid pattern the other two use.
- The **"Distance from me" buffer filter was removed** (along with the
  `p.distance` calculation and the distance-based list sort) — the list
  now always sorts alphabetically by address. The **Locate/re-center
  button and the user's blue dot on the map are unaffected** — that's
  still there for map navigation (`requestLocation()`/`recenterOnUser()`),
  it just no longer computes or uses a per-property distance for anything.
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

## Survey123 deep link
Each property's "Survey123" button/action opens:
`https://survey123.arcgis.app/?itemID=e59afef666a2407085c631d624b89c02&field:address=...&field:city=...&field:region=...&center=lat,lng`
- `itemID` and the base URL are exactly what the user provided.
- `field:address`, `field:city`, `field:region` prefill those exact questions — safe
  to hardcode because those are the field names confirmed against the real survey
  export, not a guess.
- `center=lat,lng` centers the form's map on the property regardless of what the
  form's own geopoint question is named (a documented, schema-independent Survey123
  URL parameter).
- Not yet live-tested end-to-end (need the Survey123 app/web fallback to actually
  open with those fields prefilled as expected) — worth a real-device check.

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

## Still needed before this can go live
- **Hosting**: `https://temagis.github.io/HMA_Open_Space/` needs a real GitHub Pages
  deployment, and that exact URL needs to be added to the AGOL app item's
  (`d4YpqfdNJGpPJqsS`) Redirect URIs list.
- **Parcels layer live check**: confirm the parcels service actually accepts this
  app's AGOL token (never confirmed — see above), and sanity-check a few real popups/
  detail-sheet Parcel Info sections against the generic attribute rendering to see if
  a curated, friendlier field list is worth building once the real schema is visible.
- **Survey123 prefill live check**: confirm `field:address`/`field:city`/
  `field:region` actually populate those questions as expected, and that `center`
  positions the map correctly, on a real device with the Survey123 app installed.
- **Proximity threshold**: 150 ft is a starting default (`CONFIG.proximityMatchFeet`)
  — revisit once the team has used it in the field for a bit.
- **Attachments**: survey-record photos are fetched the same way PREDS Summary does
  (`queryAttachments`), but this wasn't confirmed against a live token — worth a check
  once deployed.

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
  and no export was provided for it the way one was for the survey layer. Its popup
  and detail-sheet section render **every populated attribute the query returns**,
  generically labeled (`humanizeFieldName()` turns `PARCEL_ID`/`ownerName`-style raw
  names into "Parcel Id"/"Owner Name"), rather than a curated field list. Live
  verification once deployed and signed in is still needed — see below.

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

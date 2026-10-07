# Meridian — build process log

This document is the engineering record of how this product was built: what was
decided, what broke, and what was fixed. It is written for review, not marketing.

---

## 0. Starting conditions

The working directory was **completely empty** — no repository, no `git`, no existing
stack to reuse. Node 24 / npm 11 on macOS (Apple Silicon).

One environment problem had to be solved before anything else: `~/.npm` was root-owned, so
`npm install` failed with `EPERM`. The project ships an `.npmrc` that redirects the cache into
the repository, and `npm install --cache=./.npm-cache` works around an overriding env var.

---

## 1. Stack decisions, and why

| Decision | Reason |
| --- | --- |
| **Next.js 15 (App Router) + React 19 + TypeScript** | Requested. App Router gives static pre-rendering for all 10 destinations and keeps the map out of the server bundle. |
| **MapLibre GL + vector tiles** | See §3. Raster tiles bake their labels into pixels; only vector tiles let us cut label noise. |
| **Zustand with `persist`** | Two small stores (trips, UI) persisted to `localStorage`. Manual hydration (`skipHydration` + explicit `rehydrate()`) keeps server and first client render identical, which avoids hydration mismatches. |
| **No component library, no icon font, no utility CSS framework beyond Tailwind** | The whole UI kit is ~5 files. This was a deliberate constraint: a travel planner's UI surface is small, and pulling in a design system would have dictated the visual language rather than the product doing so. |
| **Tailwind 3 with CSS custom properties** | Tokens live in `app/globals.css` as CSS variables and are mirrored in `tailwind.config.ts`, so the map style and the UI can read the same palette. |
| **OSRM for routing, with a geodesic estimator fallback** | Real road geometry and drive times, free, keyless. Distance and travel time are separate abstractions and never conflated. |

Runtime dependencies at the end: `maplibre-gl, next, react, react-dom, zustand` — **five**.

---

## 2. Data strategy, and what verification caught

The brief demanded real coordinates and forbade inventing live data. Rather than write the
datasets from memory, the data went through a **coordinate-verification pass**: bulk Wikidata
bounding-box queries, the Wikipedia coordinates API, and OpenStreetMap element ids, with every
record keeping its source.

That pass caught six things that would otherwise have shipped wrong:

1. **Bali has only five Hilton-branded hotels.** There is no DoubleTree, Curio, Tapestry,
   Canopy or Waldorf Astoria trading in Bali (Waldorf Astoria Nusa Dua is announced for 2027).
   The first draft contained a DoubleTree that does not exist. It was deleted.
2. **Siem Reap's airport code is `SAI`, not `REP`** — REP closed in October 2023.
3. **Phnom Penh moved to Techo International (`KTI`)** in September 2025; the old `PNH` field
   no longer serves commercial traffic.
4. **El Nido's IATA code is `ENI`**; "LIO" is only the local airstrip name.
5. **Le Méridien Angkor is closed.** Marriott's marketing pages are still live, which is why it
   is widely listed as open. It is excluded.
6. **Four Points by Sheraton Palawan is in Sabang**, about two hours from Puerto Princesa city.

Final dataset: **10 destinations · 48 areas · 43 loyalty hotels · 119 places**, of which
**5 places are marked `approximate`** and drawn with a visible badge. There are **no prices
anywhere** — hotel cost is a brand-positioning tier, and the basis is shown on every card.

A data validator (`npm run validate:data`) runs schema, coordinate-range, duplicate-id, NaN-radius,
brand-registry and unknown-area-reference checks, plus a "no price in a description" assertion.
Every check in it exists because the corresponding failure actually happened once.

---

## 3. The basemap — the central design problem

### The problem

The first build used OpenStreetMap raster tiles with a CSS desaturation filter. It looked like a
map-library demo: dense place names, motorway shields, POI clutter, mixed-language labels — all
competing with our own markers. **Raster tiles bake labels into the pixels, so no CSS can fix it.**

### The decision

Migrate the map engine to **MapLibre GL** and **author our own style** over free, keyless vector
tiles (`components/map/basemap-style.ts`). We deliberately do **not** load a vendor style JSON at
runtime: a remote style can change or start watermarking without warning, and we would lose control
of label density.

Tile hosts, both free and requiring no account:

- **primary** — CARTO `carto.streets` vector tiles. Note: CARTO's *style JSON* is now watermarked
  and their keyless *raster* tiles return an "API KEY REQUIRED" image, but the **vector tiles** are
  open and CORS-enabled. We take the tiles and supply the cartography.
- **fallback** — OpenFreeMap planet tiles, swapped in automatically after sustained tile failures.

The authored style cuts labels to **country → region → city → town**. No villages, suburbs,
hamlets, POIs, road names, house numbers or waterway names. Base city labels do not begin until
zoom 6, because below that the destination markers *are* the city labels. English is preferred via
`name_en`. Land, water, borders and roads all resolve to the product's own tokens.

### Two failures worth recording

**Blank canvas.** MapLibre parses tiles in a web worker resolved at runtime through
`import.meta.url` — something a bundler cannot follow. Under Next.js the worker never loaded and
the map rendered as an empty canvas. `scripts/vendor-maplibre-worker.mjs` now copies the worker
into `public/` before every dev run and build, and fails loudly if a MapLibre upgrade moves those
files rather than shipping a silently blank map.

**A grey veil over the planning map.** A 3.5% grey fill on inactive areas looked harmless in
isolation. At destination zoom the viewport sits inside ten or more overlapping stay areas and
day-trip zones, and the fills compounded into a flat grey wash that made the planning map look
like mud. Inactive areas are now outline-only. Isolating it took an A/B test (rendering the map
with the area layer mounted and unmounted, then sampling rendered pixels) because the cause was
not visible by inspection.

---

## 4. Other real bugs found by testing

These were genuine defects, not polish:

1. **The region bounds excluded Bali.** A hard-coded bounding box had drifted out of date and left
   a destination entirely off-screen. Bounds are now derived from the data, so adding a destination
   later cannot reproduce the bug.
2. **`minZoom` clamped the mobile fit**, pushing Singapore — the origin — off the left edge on a
   390 px viewport. The minimum zoom must stay below the fitted zoom for the narrowest viewport.
3. **Marker `z-index` escaped the map's stacking context.** Markers set a z-index for ordering
   inside the map; without isolation those values competed with page chrome, and a map marker
   floated above the mobile bottom sheet. Fixed with a stacking context on the map root.
4. **A silently rejected style expression was invisible.** The map's error handler only counted
   tile failures, so a bad style expression produced no diagnostic at all. All map errors are now
   logged in development. This cost real debugging time and was worth fixing properly.
5. **Area labels repeated four or five times.** Placing labels from large radius polygons let the
   placement engine find several valid positions inside one shape. Labels now come from a dedicated
   one-point-per-area source.

---

## 5. Product decisions worth arguing about

- **No prices, anywhere.** No fares, no nightly rates, no availability. There is no pricing source
  in V1 and inventing numbers would be worse than omitting them. Provider adapters for Amadeus and
  a Skyscanner-compatible API exist and are inert without credentials.
- **Selecting a destination never leaves the map.** Comparison is a spatial task; navigating away
  to a detail page destroys the thing the product is for.
- **Adding a place does not switch panels.** Adding three stops while browsing a list should not
  yank the user out of the list. The button flips to "Added to Day N" and the map gains a numbered
  marker instead.
- **`4D3N` shorthand was removed.** Travellers read "4–7 days"; the shorthand reads like a database
  enum. The same reasoning removed `FULL` badges, `16M/5H` counts and airport-code-first rows.
- **Area boundaries are dashed radius circles, not invented polygons.** The dash pattern and the
  tooltip say "approximate extent". Real boundaries are a data task, and drawing a guess as if it
  were official would be dishonest.
- **Efficiency analysis is geometric.** It flags long transfers, spread-out days, backtracking,
  hotel/activity mismatch and over-packed days, and proposes concrete relocations. It never claims
  to know that a temple closes at 17:00. Every message quotes the number it is based on.

---

## 6. Verification

`npm run test:e2e` drives a real Chromium against the running app through the whole V1 workflow and
asserts on the DOM. It fails on console errors, uncaught page errors and failed requests.

**Result: 67/67 checks pass, 0 console errors, 0 page errors, 0 failed requests.**

Coverage: homepage → region map → Singapore origin → destination selection (including that all ten
destinations are genuinely rendered, queried through the map engine) → preview card → planner →
layer toggles → Marriott/Hilton filtering → price tiers → marker → detail card → trip date
generation → add to itinerary → reorder → move between days → day/map emphasis sync → route
geometry → efficiency panel → transport panel → refresh persistence → mobile layout → all nine
other destinations load without crashing.

`npm run build` succeeds: 10 destinations pre-rendered, 202–224 kB first load.

---

## 7. Known limitations

1. **No prices, at all.** Deliberate.
2. **No backend and no accounts.** Trips live in `localStorage` on one browser.
3. **Only Bali is deep.** The other nine destinations have enough data to be selectable and to
   demonstrate the architecture; they are not yet good enough to plan a real trip from.
4. **Destination labels collide at overview zoom.** Phnom Penh can lose its label to Phu Quoc and
   Ho Chi Minh City. Hovering always reveals it and the dot never disappears, but a proper label
   priority pass is future work.
5. **Traffic is not modelled.** Routing returns a normal-traffic drive time.
6. **Area shapes are radius circles**, not real administrative boundaries.
7. **No photography.** Descriptions and the map carry the product.
8. **The public OSRM demo server** is rate-limited and not for production.

---

# Iteration 2 — the destination experience

The Southeast Asia homepage was left alone. This pass rebuilt **the Bali destination
experience**, which worked technically and failed as a product: it opened on a trip form, drew every
category at once, and had no photography at all.

## What changed, and why

### The navigation became a progression

`Trip · Places · Areas · Route · Flights` described the system's nouns. It is now
**EXPLORE → STAY → DO → PLAN**, which are the traveller's verbs:

| Tab | Question | Map shows |
| --- | --- | --- |
| EXPLORE | Which part of Bali suits me? | Areas labelled with their tagline |
| STAY | Which property? | Loyalty hotels, individual markers |
| DO | What should I actually do? | One category at a time |
| PLAN | How does the trip fit together? | The active day's route |

The destination now **opens on EXPLORE at whole-island scale**, not on a form. The previous build
asked for dates before the traveller understood the island.

### The map shows one thing at a time

The single biggest cause of the old "marker soup" was drawing every category simultaneously. Each
tab now owns its layer set, and **each tab re-frames the camera**. That second part turned out to
matter more than expected: switching to STAY previously left the camera at whole-island scale, where
all 21 hotels collapsed into four clusters and neither loyalty programme was distinguishable. The
camera is part of the navigation.

### Areas are the primary content of EXPLORE

Six headline regions carry an authored two-or-three word tagline — `CANGGU / Surf · Cafés`,
`ULUWATU / Cliffs · Sunsets` — drawn directly on the map as a two-line label with an anchor dot.

Making all six legible at island scale took two attempts. A fixed label anchor meant Canggu, Nusa Dua
and Sanur were dropped by the collision engine at exactly the zoom where they matter most, because
south Bali has six named regions inside ~30 km. `text-variable-anchor` lets each label choose
top/bottom/left/right of its dot, and a `symbol-sort-key` gives the headline regions collision
priority over excursion zones.

Area order is curated rather than computed. Sorting by "amount of stuff" put the Nusa Dua resort
enclave first, which is a reasonable metric and the wrong editorial choice.

### Photography

`lib/images/` is a provider layer. Components ask for images by `(entityKind, entityId)` and never
construct a URL.

**Source: Wikimedia Commons** — free by policy, machine-queryable, and it returns licence and author
with every file. `scripts/fetch-bali-images.mjs` resolves 80 subjects, scores candidates on
subject-token overlap, aspect ratio and resolution, rejects known-bad matches, then downloads and
resizes them into a committed manifest.

**The `subject` field is the honesty mechanism.** Commons has almost no hotel photography for Bali.
Three properties have genuine photos of themselves; the rest borrow their area image. Rejecting bad
matches mattered: the first pass picked a bird for a Hilton Garden Inn, a competitor's resort for a
Renaissance, and a fashion shoot for the Ritz-Carlton. Those subjects are now forced to the area
fallback and the card prints *"Area photo — not this specific property"*.

Fallbacks are a first-class state: no image, a failed image and a representative image each have a
designed treatment. The test suite forces every image on the page to fail and asserts the page
survives.

### Transport became a first-class itinerary item

`TransportLeg` carries mode, rationale, alternatives, distance, duration, geometry, source,
confidence and a multimodal descriptor. The timeline alternates PLACE and TRANSPORT LEG.

**Routing and recommendation are deliberately separate.** `lib/routing/` answers how far and how
long by road (OSRM by default — free, keyless; server-only adapters for OpenRouteService, Mapbox and
Google Routes behind `/api/route`, so keys never reach the browser). `lib/transport/recommend.ts`
answers what the traveller should actually do, and states its reasoning in the UI.

**Nothing is fabricated.** `RouteResult` cannot carry a number unless a provider produced one, so
the UI cannot print an invented duration by accident. An unanswered route renders as *"Route
unavailable"* and shows only the straight-line distance, explicitly labelled as not a driving
distance. The geodesic estimator still exists but is fenced off to the route-efficiency heuristics,
where it is labelled a heuristic.

Water crossings are modelled as data, so `Ubud → Nusa Penida` correctly renders as a two-mode leg
("car to Sanur Harbour, then a fast boat") in the tests.

## Bugs this pass surfaced

1. **`hotel is not defined`** — a crash on adding a place. A bulk text replacement had written the
   hotel branch's identifier into the place and custom-stop branches. It only fired after a place
   was added, which is why the map handle disappeared in later assertions.
2. **Card → marker hover did nothing.** Emphasis compared the hovered id against the *itinerary
   item* id only, so hovering a hotel or place card in STAY/DO never highlighted its marker.
3. **Focusing an area dropped to street level.** The focus controller honoured its zoom ceiling for
   bounds but not for a single point, where it forced zoom ≥ 14.
4. **The area filter was invisible.** Choosing Uluwatu in EXPLORE silently carried into STAY and
   produced an empty list with no way to see why. There is now a removable filter chip, and empty
   states offer the way out rather than just describing it.
5. **The region fit was capped the wrong way.** A `fitMaxZoom` intended to prevent over-zoom-out
   was in fact zooming the island *out*, because it caps zoom-in.

## Verification

**70/70 checks pass, 0 console errors, 0 page errors, 0 failed requests.**

New coverage this pass: whole-island framing on arrival, all six headline areas present *and*
actually rendered at island scale, area photography loading, area focus zoom staying in range,
inherited area filter being visible and removable, hotel photography, Marriott/Hilton filtering,
card → marker hover synchronisation, category switching changing the place set, the first-run form
containing exactly three fields, one transport leg per consecutive pair, legs declaring their data
source, a measured leg showing distance and duration, the map drawing the same day as the timeline,
day switching changing the route, and broken images not crashing the page.

## Known limitations after this pass

1. **Only three Bali hotels have real photography.** The rest show their area image, disclosed as
   such. This is a genuine Commons coverage limit, not a bug.
2. **Photography is licensed CC BY / CC BY-SA / CC0 and credited, but not curated by hand.** Some
   images are competent documentation rather than beautiful travel photography.
3. **Attributes in image URLs are not decoded**, so an `alt` can contain `&amp;`.
4. **Multimodal routing is not solved.** Water crossings are curated data for Bali only; a
   multi-leg journey across a harbour is described, not routed.
5. **Traffic is not modelled**, so a measured leg is a free-flow estimate and says so.
6. **Area shapes remain radius circles**, not administrative boundaries.
7. **Non-Bali destinations have no photography** and show the fallback state throughout.

---

# Iteration 3 — making Bali good enough to actually use

The brief for this pass was narrow and unforgiving: **do not redesign the information
architecture again, do not add features, and do not expand to another destination.** Fix
photography, fix the remaining visual weakness of EXPLORE / STAY / DO, represent the travel areas
honestly, make transport legs first-class in PLAN, and clean up the small data-quality problems.
The standard to hit was "I would use this instead of Google Maps plus six browser tabs".

What follows is what the work actually turned up.

## 1. The basemap had no island in it

The first thing a fresh screenshot showed was that Bali had no silhouette. Land was `#F6F5F1`
and sea was `#DCE4E8` — a colour difference of about twenty values — and nothing in the style
drew a coastline at all. At island zoom the map read as a blank cream rectangle with some blurred
blobs on it.

Three changes fixed it, and none of them added a label or a marker:

- **A deeper, cooler sea** (`#D2E1E8`) so land has something to be lighter than.
- **A coastline stroke.** Neither tile host ships a coastline layer, so the water polygons are
  stroked instead. Lakes and rivers get an edge too, which at planning zoom is useful rather than
  noise. This single layer did more for the map than anything else in the pass.
- **Roads that exist at island scale.** Primary-road widths at z6–9 were 0.6–1.6 px of white on a
  near-white ground: invisible. Secondary and tertiary roads now start at z9.5 rather than z11.
  At planning zoom the road network *is* the useful context — it is what tells you whether two
  stops are realistically connected.

Vegetation opacity was also raised, because the interior of Bali is rice terrace and forest and
the map was claiming it was empty.

## 2. Travel zones, third attempt — and the honest answer

The previous pass replaced radius circles with a **radial envelope** around each area's centroid.
It was still wrong, and the screenshot said so plainly: a radial envelope around a centroid
degenerates into a circle whenever the content is sparse, so the map showed twenty-two interlocking
grey circles with blurred edges. Worse, the blur made them read as smudges — the whole south of
the island was one dirty cloud, and the sea had pale halos floating on it.

The zone is now built from three honest parts:

1. **A convex hull of the area's own mapped content**, offset outward and smoothed. A hull keeps
   the *shape* of the content: Canggu stays an elongated coastal strip, Uluwatu stays a cliff
   line, and only a genuinely point-like area comes out round. A minimum thickness stops a
   two-point hull from rendering as a sliver.
2. **A fine dashed edge.** Dashes are the conventional cartographic signal for "approximate", so
   the boundary never pretends to be an administrative one.
3. **A soft halo that only selected and active zones get.** Emphasis is earned by interacting,
   not applied to everything by default.

Two further decisions matter as much as the geometry:

- **The zones are inserted *below* the basemap's water layer.** A padded hull derived from coastal
  content will always spill past the shoreline. Left on top, the result was dashed arcs floating
  out over the sea. The tile source carries real ocean polygons, so drawing the zones beneath the
  water clips them to land for free — the same trick a paper map uses when it prints the sea last.
- **The map now draws one scope at a time.** EXPLORE lists nine stay bases or six day-trip zones,
  never all fifteen at once. The panel's segmented control and the map share that state through
  the UI store, so the list and the map can no longer disagree. Drawing everything was the single
  biggest reason the island map looked busy.

## 3. Photography: four hotels have it, sixteen honestly do not

Photography was the weakest part of the product and it is the part the brief put first. The
resolver was rebuilt around one rule: **a hotel photo may only be used if it is provably of that
property.**

The old resolver matched on the property's name appearing in a filename, which is not the same
thing at all. Running the hardened rules against the live sources showed exactly what that had
been letting through:

| Candidate | What it actually is |
| --- | --- |
| `Four Points by Sheraton Taipei Bali 01.jpg` | A hotel in **New Taipei City**, in a district called Bali |
| `Blanco Renaissance Museum Ubud Bali.jpg` | A **museum** in Ubud, matched on the word "Renaissance" |
| `Le Meridien Nirwana Bali Pool garden.jpg` | A **different resort**, in Tabanan, 30 km from Jimbaran |
| `The Pond with Stepping Stones, Karangasem Palaces.jpg` | A palace pond, matched on "stones" |
| `Four Points By Sheraton Bali Ungasan, Infinity Pool.jpg` | Deleted from Commons; Openverse still advertises it |

There is now a per-property rule table requiring the file title to name both the property *and*
the right part of Bali, a global reject list (Taipei, museums, the Nirwana resort), and a link
check that fetches every candidate and drops it if the source file no longer exists. The last one
is why the Four Points Ungasan pool photo — which looks like a genuine win — is **not** in the
product: the file is gone, and shipping a 404 dressed as a resort pool is worse than shipping
nothing.

The result is **4 subjects with useful, verified property photography out of 20 hotels** — St. Regis
(beach), W Bali (pool, exterior), Conrad (room, grounds) and Umana (room, exterior). Sixteen
hotels have no entry in the manifest at all and say so:

> No property photography available for this hotel

They never borrow their area's beach photograph. That is the whole point of the state.

Two corrections came from *looking at the images* rather than trusting their filenames:

- **Conrad's hero was a dark photograph of the word "CONRAD" on a wall.** The filename contains
  "resort & spa", so the classifier called it a spa photo. The room photo that was sitting in the
  gallery slot is now the hero; the signage is last.
- **Signage was then removed altogether.** The Westin's only photo, and one of Conrad's three, are
  close-ups of the resort's name on a wall. They are real, correctly licensed and provably of the
  property — and they are not property photography: a traveller choosing between two Nusa Dua
  resorts learns nothing from a dark photograph of the word "WESTIN", and a card led by one is
  worse than a card that says plainly that no photography is available. They are excluded from the
  manifest, and the exclusion is logged by the generator so the coverage number stays honest.

Places fare far better than hotels: **41 of 48 Bali places and all 15 areas have photography**, and
the seven that do not are three beach clubs and four logistical waypoints.

Photography is also no longer a fixed 3:2 hole when it is missing. A place with no photo gets a
compact text-led card instead of an empty box the same size as the photograph it replaces — which
had been pushing two cards off the screen and making the list look broken.

## 4. Transport legs, first-class

PLAN now reads as a schedule rather than a list. Between each pair of stops:

```
09:00  The Westin Resort Nusa Dua, Bali
         ↓
       Car / Grab · ≈25 min · 21 km
       About 21 km by road distance. Ride-hailing works…
       OSRM road route · free-flow estimate
       or a private car and driver / a taxi
09:25  Uluwatu Temple (Pura Luhur Uluwatu)
```

The mode, the measured duration and the measured distance are on one line; the rationale and the
data source are underneath; the alternatives are named. Where no routing engine answered, the leg
says **No route data** and the day's clock is marked `approx` rather than shown with a confidence
it has not earned. Traffic is not modelled and the leg says so by calling itself a free-flow
estimate.

## 5. Three real data bugs, and the checks that now catch them

**The same hotel was in the product twice.** `four-points-by-sheraton-bali-ungasan` and
`four-points-bali-ungasan` were two records for one property, at identical coordinates, with two
different Marriott property codes (`DPSFG` and `DPSFP`). The duplicate-id check could not see it
because the ids differed — the STAY list simply showed Four Points Ungasan twice. Only `DPSFG` is
real (verified against marriott.com); the other record is deleted. The validator now compares
every hotel against every other in the same destination within 150 m and fails if two records name
the same physical site.

**An entity could sit outside its own map bounds.** The destination map fits `mapBounds` on
arrival, so an entity outside them is one the traveller can never see without panning into empty
space — and it is the signature of a transposed lat/lng. The validator now asserts every hotel and
place falls inside its destination's bounds. It immediately proved that the other nine
destinations are clean.

**A third of the photography was filed under ids that no longer existed.** The image
generator keeps its own hand-written search list, and that list had drifted from the dataset:
`jatiluwih-rice-terraces` against `jatiluwih-rice-terrace`, `amed-beach` against `jemeluk-beach`,
`old-mans-canggu` against `old-mans`, and so on for eleven places and three areas. The manifest is
keyed by string, and a key that matches nothing looks exactly like an entity that has no
photography — so the product showed Jatiluwih Rice Terraces, Mount Batur, Kelingking, Jimbaran,
Amed, Goa Gajah and the rest as **"no photo yet"** while their photographs sat unused on disk. The
same drift had left five subjects in the generator for places that no longer exist, so images were
being downloaded for entities nobody could reach.

Seventeen places and two areas regained their photography once the ids were realigned. The
validator now fails if any manifest key is not a real entity, and fails again if the generator's
own id list names anything the dataset does not contain. That is the durable half of the fix: the
drift cannot come back silently.

Two things are deliberate rather than fixed. `sunset-road`, `ubung-bus-terminal`,
`beachwalk-kuta-pickup` and `seminyak-village-pickup` are logistical waypoints — a road strip, a
bus station and two meeting points. Any photograph we could find for them would be a stand-in for
"somewhere in Kuta", which is exactly the borrowed imagery this pipeline exists to prevent, so
their cards say "no photo of this place yet" and the generator documents why.

Smaller fixes in the same pass:

- **Alt text is entity-clean.** A truncated HTML description had been producing an `alt` that
  began with a bare ampersand ("& JIWA spa treatment room"). Zero `&amp;`, `&quot;` or `&#`
  sequences remain, verified by grep against the generated manifest.
- **A stale detail card followed you between tabs.** A hotel's card stayed open over the DO list
  and over the itinerary. It is now dismissed when you move between steps.
- **The resolver threw away a twenty-minute run on one dropped connection.** Every HTTP call now
  retries with backoff, and Openverse results whose Commons file has since been deleted are
  re-resolved against Commons before they are dropped.

## 6. What the verification actually covers

`npm run test:e2e` — **73/73 checks pass, 0 console errors, 0 page errors, 0 failed requests.**

Three of those checks had to be rewritten in this pass because they were asserting on the wrong
thing:

- **`querySourceFeatures('route')` returned zero while the route was plainly painted on screen.**
  It reads the tiles that happen to be loaded, and a GeoJSON line can be visible while that call
  returns nothing. The check now asserts on *rendered* features, and cross-checks the line against
  the panel: a solid line must be backed by a named routing engine, and a dashed one must be a day
  where no leg had route data. If the map and the legs ever disagree, the test fails.
- **The route check sampled one frame.** `next dev` compiles on first request and the camera
  animates; a single sample turned a working route into a failing test. There is now a shared
  `until()` poller.
- **The inherited-area-filter check raced the panel's first paint.**

## Known limitations after this pass

1. **Sixteen of twenty Bali hotels have no property photography at all.** This is a genuine
   Wikimedia Commons and Openverse coverage limit for Bali's Marriott and Hilton properties, not a
   bug, and it is reported to the user rather than papered over — those cards say "No property
   photography available for this hotel" and never borrow the area's beach photograph. Fixing it
   properly means licensed photography: a commercial image provider, or the properties' own media
   kits, with the same verification bar.
2. **The travel zones are approximations and say so.** They are hulls around mapped content, not
   administrative boundaries and not official tourism areas. Uluwatu's zone stretches ~9 km
   because three of its hotels sit at Ungasan, and the zone has to contain them.
3. **Traffic is not modelled.** Every measured leg is a free-flow estimate, labelled as one.
4. **Multimodal journeys are described, not routed.** Ubud → Nusa Penida is curated data (drive to
   Sanur Harbour, fast boat); the boat crossing has no timetable and no live availability.
5. **Photography is not hand-curated.** Licences are restricted to commercial-use (CC BY, BY-SA,
   CC0, public domain) and every image carries its author, licence and source page, but some
   images are competent documentation rather than beautiful travel photography.
6. **The whole image set is ~32 MB**, which is heavy for a tunnel deployment. Each image is
   1100 px at quality 68 and lazy-loaded, so it is not on the critical path, but a CDN with
   responsive variants is the correct answer.
7. **Only Bali is deep.** The other nine destinations are selectable and architecturally complete;
   they are not yet good enough to plan a real trip from.
8. **No backend, no accounts.** Trips live in `localStorage` on one browser.

---

# Iteration 4 — Chinese first, real restaurant discovery, and a research pipeline

The brief for this pass had three goals and one constraint: make Simplified Chinese the
primary product language, make Bali restaurant and activity discovery actually useful, build a
structured pipeline for importing social-media travel guides — and do not redesign the
architecture again, do not expand to another destination, and do not build scrapers.

## 1. Localization, as architecture rather than strings

The requirement was explicit: "Create localization infrastructure rather than scattering Chinese
strings through components." So there is a catalogue, not a find-and-replace.

**`lib/i18n/messages.ts`** holds **546 keys**. `zhCN` is authored first and is the source of
truth; `en` is typed as `Record<keyof typeof zhCN, string>`, which makes a missing or misspelled
translation a **compile error** rather than a raw key rendered into the interface. That one
typing decision has caught more mistakes during this pass than any test.

**The Chinese is written, not translated.** This is the part that determines whether the product
reads as Chinese or as an English product wearing Chinese. Where the English says "Bali packs a
beach town, a surf coast, a cultural highland and a resort enclave into an island you can cross
in a day", the Chinese is two short clauses. The English long-form copy still exists for the `en`
locale; the Chinese is a different text for the same reader.

**Proper nouns are stored twice, and both are shown.** `nameZh` sits beside the canonical name on
places, hotels, areas and destinations. The rule is not "translate the name" — it is that a
traveller reads 乌鲁瓦图神庙 and then needs to type "Uluwatu Temple" into Grab. A card that showed
only one of the two would fail at one of those two jobs. So 乌鲁瓦图神庙 is followed by
Uluwatu Temple, and every restaurant keeps its Latin name because that is what the map apps know.

`nameZh` is omitted wherever a Chinese name is not genuinely in use. Only 29 of 145 places carry
one. `The Stones Hotel`, `Umana`, `Betelnut Café` and the rest keep their Latin names, because an
invented transliteration is a name the reader cannot search for — strictly worse than the English.

**Chinese copy lives in an overlay module**, `lib/data/zh/bali-zh.ts`, merged by the registry. The
hand-verified geography files — coordinates, sources, `coordNote`s — are never touched by a
translation pass, and the Chinese can be reviewed on its own without diffing thousands of lines of
English.

**Where the locale lives.** In the persisted UI store, not a cookie. The site is statically
exported to GitHub Pages, so no route may read a request header. Server and first client render
both use the default; a stored preference is applied after rehydration. Because rehydration
happens in an effect, the first client render matches the server exactly and there is no hydration
mismatch.

**One thing had to move.** `message()` was originally in the same module as the React bindings,
which is marked `'use client'` — and the root layout's `metadata` is a server component that needs
it. Every page 500'd. It now lives in the pure module and is re-exported for convenience.

## 2. The transport rationale had to stop being a sentence

`recommend.ts` used to build its explanation by concatenating around a number:

```ts
rationale: `About ${km.toFixed(1)} km by ${basis}. A ride-hailing car is cheapest…`
```

That cannot be rendered well in a second language — the clause order, the measure word, the way a
distance is expressed are all English. It now returns a **rule id and its numbers**
(`rationaleKey: 'short-hop'`, `rationaleParams: { km, measured }`) and the copy lives in the
catalogue. This is the difference between a localized product and an English product with
translated labels.

The English sentence is still emitted alongside for the `en` locale and for logging.

## 3. Restaurant and activity discovery

Bali went from **48 places to 145**.

| | count |
|---|---|
| Restaurants, cafés, bars and beach clubs | 46 |
| Bookable activities and operators | 51 |
| Original geography (temples, beaches, waterfalls, warungs) | 48 |

Restaurants span Seminyak (8), Canggu (8), Ubud (8), Uluwatu (7), Nusa Dua (5), Sanur (5) and
Jimbaran (5). Activities cover all 17 activity kinds, from surf schools and dive centres to
cooking classes, ATV operators, spa and yoga studios.

**The DO filter was rebuilt around a shared vocabulary.** `lib/data/place-taxonomy.ts` now holds
the category ids, cuisine ids, "recommended for" ids and activity kinds, and three consumers read
it: the DO chip row, the map's marker set, and the research matcher. Previously the category
matching was inferred from a single `markerLayer` enum, which lost most of the truth — a beach
club is genuinely a beach club *and* nightlife *and* a restaurant. Each place now declares
`discovery` ids explicitly.

The category row is Chinese and has eleven entries: 精选 · 美食 · 咖啡 · Beach Club · 海滩 · 自然 ·
文化 · 夜生活 · 水上活动 · Wellness · 购物. Underneath it is an **area row** listing only the areas
that actually hold something in the selected category, with counts. 美食 + 长谷 narrows 49
restaurants to 7. Both filters exist because they answer different questions: the category is
"what do I feel like", the area is "where am I willing to drive".

## 4. Coordinates come from a map, not from a model

The datasets were authored by **name**, not by latitude. Asking a language model for coordinates
produces plausible numbers that are wrong often enough to matter, and a pin 400 m off in Canggu
puts a traveller on the wrong side of a rice field.

So `scripts/geocode-pois.mjs` resolves them against **Nominatim**, and every resolved record keeps
the OpenStreetMap element it came from in its `coordNote`. Anything outside Bali, or more than
15 km from the area it claims to be in, is rejected and reported for a human.

**94 of 100 resolved.** Six did not — a surf school, a yoga studio, a water-sports operator, a
spa, a thalasso centre and a dive centre, none of which are in OSM. Those six keep
`confidence: 'demo'` and **are excluded from the map and cannot join an itinerary**. They still
appear in the DO list, where their card says 位置未核实. A pin at 0,0, or a route that measures
8,000 km to dinner, would be worse than an honest blank.

Two bugs in this script are worth recording, because both looked like success:

- The **query shape** mattered more than anything else. Nominatim returns nothing for
  `Goa Gajah (Elephant Cave)` and the temple for `Goa Gajah`; nothing for `Betelnut Café` and the
  café for `Betelnut Cafe`. The first version used one shape and resolved 23 of 100.
- The **write-back silently did nothing** for two runs. It recorded each entry's character offset
  in the original file and then did index surgery on a mutating string; the offsets and the output
  drifted apart, and it reported "11 resolved" while leaving every placeholder in place. It now
  patches by `id → placeholder`, which is idempotent and cannot drift.

## 5. The research pipeline

A **separate data layer**. Production POI data is curated and verifiable; a social guide is a
discovery signal — someone said something about somewhere. Merging them would let an unverified
mention become a published place, which is the exact failure this separation exists to prevent.

**What it is not: a scraper.** Xiaohongshu, Douyin, TikTok and Instagram prohibit automated
collection, and circumventing those controls is not something this product does. V1 takes a URL
for provenance and the text the researcher pastes. The interface says so in words:

> 我们不抓取这些平台的内容。请把你看到的有用文字粘过来，链接会作为来源保留。

**Extraction runs in three passes, in order of trust.** A dictionary pass scans the text for names
already in the dataset, longest-first so "Finns Beach Club" wins over "Finns". A pattern pass reads
the structures guides actually use (`店名：`, `📍`, `1.`, `「」`). A heuristic pass finds capitalised
Latin runs and Chinese runs next to a category keyword.

The first version of the heuristic pass produced **46 mentions for one sample guide, most of them
junk** — 早餐去了, 必点, 牛油果吐司. The Chinese rule matched any 2–10 character run, which is every
phrase in the language. It now only captures a run immediately followed by a venue noun
(咖啡, 餐厅, 海滩俱乐部…), and rejects runs containing function words. The same guide now yields
**11 mentions, all of them real venues.**

**Matching refuses to guess.** A name is reduced to two keys: a *loose* one with locational and
categorical noise stripped (`La Brisa Beach Club, Canggu` → `brisa`) and a *strict* one with only
punctuation and accents folded. The strict key exists because noise stripping is destructive on
names that legitimately contain a region word — reducing `Uluwatu Temple` to `temple` made it
unmatchable against its own canonical record.

Below a confidence of 0.55 the mention is marked 需要确认 and a human decides. The floor was set
against a real failure: "Old Man's" and "Old Man" score ~0.9, while "La Brisa" and "La Favela"
score ~0.34 — a lower floor started pairing them.

**Nothing is published automatically.** A mention moves 待处理 → 已匹配/待验证 → 已收录 by human
action, and the traveller only ever sees an aggregate over accepted mentions.

## 6. The honesty rule for social signals

The brief was specific: never claim 最热门 or 98% 推荐 unless a real methodology supports it.

So the card says **在 12 份已收录攻略中被提及** — a statement about our own corpus, which is
checkable — and carries the caveat 来自你收录的攻略，只作为参考，不代表全网热度. There is no
popularity ranking anywhere in the product.

`frequentlyMentioned` is a genuine frequency count, and the bar is two: one guide naming a dish is
an anecdote, two is a pattern. Themes are derived from recurring keywords, and only when they
recur.

A fresh install shows **no signals at all**, because there are no imported guides. That is the
correct behaviour: the alternative would be to seed fake research. The inbox offers a 载入示例文本
button whose sample is written for this product, and it is labelled as a demonstration rather than
a real guide.

## 7. Bugs this pass surfaced

Beyond the two in the geocoder and the three in localization:

1. **The review queue was always empty.** The 待处理 tab counted unprocessed *sources* while
   filtering that tab by mention status `pending` — a status extraction never assigns. The
   reviewer opened the inbox and saw nothing to do.
2. **Every card silently lost its 攻略参考 block.** Place cards read signals from the research
   store, and `skipHydration` means that store reads nothing until asked. It was hydrated on
   `/research` and nowhere else, so the destination page started empty every time.
3. **Three beach clubs were in the product twice.** The restaurant and activity datasets were
   authored independently and both included La Brisa, The Lawn and Sundays. The new canonical
   identity check caught all three — this is precisely the duplicate problem the brief describes,
   found by the rule written for it.
4. **`message()` in a `'use client'` module** 500'd every page, because the root layout's metadata
   is server-rendered.
5. **Area names were English under a Chinese interface.** The `areaNameById` map that every panel
   labels its rows with was built from the canonical name.
6. **The itinerary lost the Chinese name.** `itemFromPlace` stored only `name`, so a trip built
   from a Chinese card rendered in English. `ItineraryItem` now carries both, which also means an
   itinerary still reads correctly after the traveller switches language.

## 8. Verification

**98 of 98 checks pass** across 24 steps, with 0 console errors, 0 page errors and 0 failed
requests. Four steps are new this pass:

- **Chinese is the product language, and English names stay searchable** — asserts the four tabs
  read 探索/住宿/游玩/行程, that a place card carries a Latin proper noun alongside Chinese, and
  that switching to `en` and back actually relabels the chrome both ways.
- **DO filters by Chinese category and by area, together** — asserts eleven Chinese categories
  leading with 精选, that 美食 lists 20+ restaurants, that an area row is offered, and that
  美食 + 长谷 narrows it.
- **The research inbox imports a guide, extracts places and gates publication** — pastes a URL and
  text, asserts the platform is detected, that 8+ places are extracted, that known places match and
  unknown ones are flagged, and that 已收录 starts empty until a human accepts.
- **An accepted mention reaches the traveller as an aggregate signal** — asserts the count is
  phrased over our own corpus, that **no popularity claim appears**, and that the provenance
  caveat is present.

`npm run validate:data` gained four rule groups, all of which fired during this pass:

- **Canonical identity**: two places in one destination whose names normalise to the same thing.
  This is the La Brisa rule, and it caught the three duplicated beach clubs.
- **Unresolved coordinates must be marked `demo`**, so the map and the itinerary can exclude them,
  and a `0,0` record claiming `verified` is an error.
- **Taxonomy ids used by data must exist.** A typo here removes a place from a category silently —
  it simply never appears under 美食, and nothing errors.
- **The research pipeline must behave on the shipped sample**: no matched id that does not exist,
  no duplicate mentions, and a mention count low enough to prove the heuristic pass is not matching
  prose.

## Known limitations after this pass

1. **Six Bali places have no verified location.** They are readable but not plannable, and their
   cards say so. They need a human with local knowledge, or an operator website, not a better
   algorithm.
2. **The 97 new places have no photography yet.** The image pipeline covers the original 48 places
   and all 15 areas; restaurants and activity operators are not in it. Their cards render the
   honest no-photo state, and borrowing an area photo for a specific restaurant is exactly what
   this project refuses to do.
3. **Social signals are empty on a fresh install.** By design — they come only from guides the
   researcher imports. There is no seeded research, because seeding it would be inventing it.
4. **Extraction is deterministic and therefore literal.** It finds names against the catalogue and
   near category keywords. It does not resolve pronouns, follow an embedded map link, or read a
   screenshot of a Xiaohongshu post — which is the real format most of these guides arrive in.
5. **A guide's text is pasted by hand**, because fetching it automatically is not permitted. The URL
   is kept as provenance, but there is no way to verify that the pasted text matches it.
6. **The taxonomy is Bali-shaped.** Cuisines, activity kinds and "recommended for" ids were chosen
   for this island; another destination will need additions.
7. **Place names in Chinese are only as good as usage.** 29 of 145 carry one. The rest are Latin
   because that is what people actually write.
8. **Traffic is still not modelled**, photography is still Wikimedia-grade, and only Bali is deep —
   all unchanged from the previous pass.

---

# Iteration 5 — the origin becomes a first-class entity

The brief was narrow: stop assuming Singapore. Everything else about the product — the map, the
Bali EXPLORE/STAY/DO/PLAN experience, the research pipeline — was to stay exactly as it was. What
changed is the *other* end of the journey.

## 1. What "Singapore-centric" actually meant in the code

Finding every place the assumption lived was the first job, and it was more than a label:

| Where | The assumption |
|---|---|
| `lib/data/regions.ts` | `SINGAPORE_ORIGIN`, a single hard-coded city typed `id: 'singapore'` |
| `Airport.directFromSingapore` | Whether a route was non-stop — stored on the *destination* airport |
| `Airport.flightMinutes` / `airlines` / `flightNote` | Duration and carriers, also on the destination airport |
| `primaryRouteSummary()` | Read those fields and formatted "…non-stop" |
| `hasDirectFromSingapore()` | The 直飞 filter |
| `RegionMapView` | Viewport bounds built from `SINGAPORE_ORIGIN.coordinates`, origin marker pinned there |
| `RegionArcLayer` | The arc started at Singapore unconditionally |
| `EntityDetailCard` | Told a traveller from Guangzhou about Singapore's non-stop service |
| `map-markers.ts` | Airport sublabel read "non-stop from SIN" |
| `Trip` | No origin field at all |
| i18n | `从新加坡出发`, `Places within a four-hour flight of Singapore` |

Two of those are the interesting ones. **Direct service is a property of a PAIR, not of the
destination airport.** Singapore → Bali and Shanghai → Bali are different journeys to the same
island, and a boolean on Bali's airport record could only ever describe one of them. That is why
the fix is an `OriginDestinationConnection` and not a second boolean.

## 2. The origin dataset

Eleven cities in five groups, because that is what a selector needs:

| Group | Cities |
|---|---|
| 新加坡 | Singapore |
| 粤港澳大湾区 | Guangzhou, Shenzhen, Hong Kong |
| 长三角 | Shanghai, Hangzhou |
| 中国其他 | Beijing, Chengdu |
| 东南亚 | Bangkok, Kuala Lumpur, Jakarta |

**One city is not one airport.** Shanghai has PVG and SHA, Beijing PEK and PKX, Chengdu CTU and
TFU, Bangkok BKK and DMK, Singapore SIN and XSP. The model stores a list from the start, so adding
the second airport to a city is a data edit rather than a schema change — and the one-city-one-airport
version would have had to be rewritten the moment Hangzhou gained a second field.

All **16 airports** and every city centre were resolved against OpenStreetMap via Nominatim, and
each coordinate carries the aerodrome it marks. The verification script is kept
(`scripts/verify-origins.mjs`) so the dataset can be re-checked rather than trusted.

**China is an origin market, not a destination catalogue.** There are no Chinese destinations, and
none were added. Mixing the two would have turned a focused origin upgrade into an unbounded
content project.

## 3. Connections, and the rule that makes them honest

`directAvailable` is `boolean | null`. **`null` means unknown, and unknown is not "no".** A pair
with no record renders 航班信息待确认 and shows **no duration at all** — only a straight-line
distance, explicitly labelled as a straight line. Nothing is inferred from distance, from hub size,
or from the fact that some other origin has the route.

Three confidence levels, and the interface treats them differently:

- **verified (10)** — Singapore's connections, *harvested* from the destination data files where
  each carries a cited source: airline timetables, news reports, block times checked flight by
  flight. The harvest runs once at module load and is the only reader of the old fields, which are
  renamed `legacyRouteFromOrigin` and marked deprecated. The cited sources stay attached to the
  data they describe.
- **approximate (64)** — the long-standing, high-frequency routes that appear on any route map for
  those hubs. Marked 待确认 in the interface, with the curation date and a source string saying so.
- **unknown (36)** — no record. Jakarta has two of ten; that is the honest result, not a failure.

The alternative — marking all ten non-Singapore origins unknown — would have demonstrated the
architecture while telling a traveller nothing. Marking them all verified would have been a lie.

`weekendSuitability` is curated per pair rather than derived from distance. Singapore → Bali is
`not-ideal`; Bangkok → Siem Reap is `good`. The brief was explicit that this must not be computed
from a straight line, and it is not.

## 4. The selector

Compact, Chinese-first, map-first. A pill in the header reading `从 广州 CAN ▾` that opens a
320 px popover: a search field, cities grouped by region, nothing else. Choosing a city closes the
popover and reorients the map immediately — there is no confirm step, because the whole point of
the control is that changing origin is cheap.

Search resolves **广州, Guangzhou and CAN** — the three ways a person refers to the same place
depending on what is in front of them. It also matches airport *names*, which is why `SHA` returns
both 上海 and 杭州: Hongqiao's Chinese name is 虹桥 and Xiaoshan's is 萧山, and both contain the
letters. That is a feature; a traveller typing an airport code wants the airport.

It appears in three places: the homepage top bar, the mobile destination rail, and the destination
page. On mobile the top-bar copy is hidden and the rail copy takes over — changing where you are
leaving from is the first decision in the product, not a secondary action.

## 5. What reorients

Selecting 广州 changes, with no destination-specific code:

- the origin marker (★ 广州 / 出发地),
- the viewport, refit to contain the new origin *and* the destinations,
- the single route arc, now drawn from Guangzhou,
- every duration in the destination rail (Bali 2h45m → 5h25m),
- the 直飞 filter, which now means non-stop *from your city*,
- the preview card: 约 6 小时 25 分钟 从上海出发 · 直飞 · PVG · SHA → DPS · 待确认,
- the airport card inside a destination.

A destination with no connection record stays visible under 全部 but is excluded from 直飞 — we
cannot claim non-stop service we have no data for.

## 6. Persistence and migration

The origin lives in its own store (`lib/store/origin-store.ts`) with its own storage key. That is
deliberate: **"I live in Guangzhou" belongs to a person, not to a trip or a screen.** When accounts
arrive, this file is replaced by a profile read and nothing else changes. Hydration re-validates
the stored id, so a city removed from the dataset cannot leave the map without an origin.

`Trip.originCityId` is optional on the type, because trips saved before this iteration do not have
it. Two layers handle that:

1. a **zustand `version: 1` migration** that stamps `singapore` on any trip without an origin —
   which is where every such trip was in fact made from;
2. a **hydrate backstop** that catches anything the migration missed, because zustand only runs
   `migrate` when the stored version is *older*, and a payload written before versioning existed has
   no version field at all.

Nothing else about a trip changes: days, items and dates are untouched, and a trip that already has
an origin keeps it. An empty origin would otherwise render 出发地 — on an itinerary that is
perfectly readable.

## 7. Bugs this pass surfaced

1. **The selector panel opened off-screen.** The trigger sits at the top right, the panel was
   anchored `left-0`, and it ran past the viewport edge. It now hangs from the right.
2. **Two elements shared one test id.** The header and mobile-rail selectors are both in the DOM at
   every viewport (one is CSS-hidden), so `origin-trigger` resolved to two nodes and every strict
   locator failed. The mobile instance is now `origin-trigger-mobile`, with its inner elements
   prefixed to match.
3. **A `{count}` placeholder rendered literally.** The preview card called `t('region.placesMapped')`
   without params, so the card read "145 已收录 {count} 个地点". Added because the Chinese catalogue
   introduced placeholders the English call sites never passed.
4. **`bestForZh` arrays were the wrong length on seven destinations.** `pickList` falls back
   per index, so a mismatch degrades into a line that is half Chinese and half English — it reads as
   a bug and nothing catches it. The validator now checks alignment for `bestFor`, `weakFor` and
   `tags` across every destination, area and place.
5. **A circular import broke every page.** `connections.ts` imported `DESTINATIONS` from
   `lib/data/index.ts`, which re-exports `getConnection` from `connections.ts` — and the connection
   table was built at module scope. Every import failed with *"Cannot access 'DESTINATIONS' before
   initialization"*. The table is now built lazily on first use.

## 8. Verification

**104 of 104 checks pass** across 29 steps, with 0 console errors, 0 page errors and 0 failed
requests. Five steps are new:

- **The homepage opens on a default origin, with the origin as a control** — and asserts that the
  old fixed `出发地 SIN` static text is gone.
- **The selector searches by Chinese name, English name and airport code** — seven queries,
  including that an unmatched query says so rather than rendering an empty panel.
- **Every supported origin can be selected and the map reorients** — all eleven, plus a viewport
  assertion that the bounds actually contain the selected origin.
- **Destination metadata is origin-relative, and unknown stays unknown** — Singapore → Bali must be
  shorter than Guangzhou → Bali, and Jakarta's rail must contain 航班信息待确认 with **no duration
  beside it**.
- **The selected origin survives a refresh, and Bali still works** — all four tabs, hotels, places,
  the trip form, and the origin carried onto the destination page.

`npm run validate:data` gained four rule groups: origin and airport integrity (including that an
airport code cannot belong to two cities), connection provenance, that an `unknown` connection
carries no numbers, that a `verified` one carries a duration, and bilingual array alignment.

Dataset after this pass: **11 origin cities · 16 airports · 10 verified + 64 approximate + 36
unknown connections · 581 message keys.**

## Known limitations after this pass

1. **64 of 110 connections are curated, not verified.** They are marked 待确认 in the interface and
   carry a curation date, but they are route knowledge, not a schedule anyone checked. The
   architecture is built for a provider to replace them; nobody has yet.
2. **Nothing refreshes.** `verifiedAt` exists so a periodic review can be built, and it has not
   been. A curated route that stops operating will keep being shown as 直飞 until someone re-checks.
3. **No nearby-airport logic.** `nearbyOriginIds` is populated and displayed as a hint, and nothing
   acts on it. A traveller in Shenzhen is not yet told that Hong Kong might be cheaper.
4. **No multimodal origin journey.** 苏州 → PVG → Bali is the future flow the brief describes; the
   model does not prevent it (an origin has coordinates and airports, and a trip stores its origin)
   but nothing computes it.
5. **The origin is not on the itinerary.** A trip stores `originCityId`, and the PLAN timeline still
   starts at the destination airport rather than at the traveller's home city.
6. **Chinese origins have no visa or entry data.** Deliberately — the brief forbade building a visa
   engine, and inventing one would be worse than not having it.
7. **The 97 Bali restaurants and activities still have no photography**, and six places still have
   no verified location. Both carried over unchanged from the previous pass.

---

# Iteration 6 — turning a saved guide into places on a map

## 1. The core product moment, and what it rules out

The brief for this pass was one sentence: *paste a travel guide, and its places appear on my map.*
Almost every design decision below follows from taking that sentence literally.

It rules out the obvious implementation. A guide arrives as a link to Xiaohongshu, a TikTok or a
YouTube video, and the tempting move is to fetch it. That fails on three counts at once: the
platforms' terms prohibit automated collection, an anti-bot arms race breaks monthly, and the format
most of these guides actually take is a screenshot, which no amount of HTML parsing reads. Attempting
it would also have put the traveller's own account in the path of a ban.

So Meridian keeps the link as **provenance** and works on the text the traveller pastes. Because that
is a real limitation rather than an implementation detail, the interface says it in words, in the
place where a traveller would otherwise expect us to read the link:

> 暂时无法直接读取这个平台的内容。你可以复制攻略文字到这里，Meridian 会继续帮你整理。

That is §5 of the brief, and it is a piece of copy in the input panel rather than a line in a policy
page. A limitation the user can act on belongs where the action is.

## 2. The flow takes the panel, not the screen

The first build of this flow was a modal. It was wrong for a reason that is easy to state and was
easy to miss: the modal covered the map, so the places appeared behind a dialog. The product's whole
promise is spatial, and the moment it pays off was hidden.

The flow now takes over the destination's side panel — the same 400px rail the four tabs use on
desktop, the same bottom sheet on mobile — and the map stays visible throughout. On a phone the sheet
expands to full height when the flow opens, because reading a list of candidates is a full-height job
and at the half snap the traveller had to discover by dragging that there was more to see.

Candidates are plotted on the map as they are reviewed, and the camera moves to them when the review
opens. There is no separate "preview" step, because reviewing *is* previewing.

One bug here is worth recording, because it is the kind that passes every automated check that is not
a real browser looking at a real map: the import preview was scoped to the `do` tab. Open the flow
from EXPLORE — which is where the destination opens, so it is where most people are — and the review
appeared over a map with nothing on it. The fix was to make an import review outrank the tab: while a
review is open, the map draws that import's candidates and nothing else, because "the places from
this guide" and "every restaurant in Canggu" on the same canvas answer neither question.

## 3. Extraction behind an interface, so the key never ships

Extraction is entity recognition, classification and one-line summarisation over text. That is a task
a small cheap model does well, and it is also a task a deterministic rule-based extractor does
adequately. Which one runs should be a deployment decision, not an architectural one.

`lib/research/extractor.ts` defines one `GuideExtractor` interface and two implementations:

| Provider | Runs where | Key | Behaviour |
| --- | --- | --- | --- |
| `deterministic` (default) | the browser | none | Dictionary, explicit patterns, constrained heuristic |
| `llm` | server, via `/api/extract` | yes | Structured output from a chat model |

The client never holds a credential. It posts to `/api/extract`, and that route is the single place
`DEEPSEEK_API_KEY` or `OPENAI_API_KEY` is read — the same boundary `app/api/route/route.ts` already
uses for routing keys. It does not fetch the source URL, and the system prompt it sends to the model
says so explicitly.

The static GitHub Pages build is where this decision earns its keep. There is no server there, so the
route is not deployed at all; `NEXT_PUBLIC_GUIDE_EXTRACTOR` is unset, and the deterministic extractor
runs. The feature degrades rather than breaking, and a key in the deployment environment cannot
silently become a key in a JavaScript bundle — which was the one thing §29 said not to compromise on.

## 4. Two extraction bugs, and why the second one only appeared after fixing the first

**Every mention inherited its paragraph's themes.** `La Brisa` was classified as a temple and
credited with a sunset it never had. The cause was that a segment's context held the parent *line*,
so a three-sentence line gave every name in it the union of all three sentences' words. Segmenting to
the sentence fixed the attribution.

**And immediately broke extraction for a real venue.** `第二天在长谷吃了 Milk & Madu，早餐很好`
names a restaurant without using a restaurant word, and the heuristic pass gates on a category
keyword. The old, coarser context had accidentally contained one. A `VISIT_VERB` gate — 去了 / 吃了 /
住了 / visited / ate at — restored it, and a comment in the code records why, because the next person
to tighten that gate will hit the same wall.

**A venue glued to its sentence was never found at all.** `晚上去了蓝房子酒吧` was captured by the CJK
name pattern as one greedy run, 晚上去了蓝房子, which the function-word filter then discarded — and
because the scan resumed past the match, 蓝房子 was never tried. Writing "去了X酒吧" is the common
case, not the edge case, so the capture is now trimmed back to the last function word rather than
thrown away. This one was found by writing a test for alias learning and watching it fail for a
reason that had nothing to do with aliases.

## 5. Matching: one place, however many spellings

The brief called duplicate prevention critical, and it is: `La Brisa`, `La Brisa Bali` and
`La Brisa Canggu` must not become three canonical beach clubs. The architecture answers it in the
data shape rather than in a cleanup pass. A mention resolves to a **canonical place id**; saved places
store **references** to those ids, not copies; saving is idempotent. A guide mentioning the same place
twice is one place, and a place saved from three guides is one entry carrying three references.

Matching compares two keys, because one is not enough. The loose key strips locational and categorical
noise; the strict key only normalises punctuation and accents. Stripping noise is right for
`Warung Babi Guling Ibu Oka` and wrong for `Uluwatu Temple`, where 神庙 is part of the name. Two
candidates scoring within 0.05 of each other are treated as ambiguous and matched to neither.

Above the confidence floor the interface speaks in bands, never numbers: high preselects, medium asks,
low does not guess. And for a name nothing matches there is deliberately **no** "accept our best
guess" button. A wrong automatic answer is exactly how a duplicate canonical place is born, so the
traveller either points at the place Meridian already holds or creates one — and creating one is a
private submission, not a place.

## 6. What a created place actually is

`UserPlaceSubmission`, in `pending_verification`, owned by a local profile id, never written into the
canonical registry. It appears in 我的收藏 in its own visual register — dashed border, no photograph,
a 待核实 badge — because showing it with the card a dataset place gets would launder an unverified
submission into something that looks verified.

It cannot be added to an itinerary until the traveller has placed it on the map, and that is not a
limitation imposed for its own sake: there is no honest coordinate for a place nobody has located, and
a stop that sits nowhere is worse than a stop that is missing.

## 7. Honesty, restated for this feature

Four rules, each enforced in the type system or the validator rather than in a style guide:

1. **A guide's words are the guide's words.** Themes, dishes, warnings and times render under
   以下内容来自攻略，不是 Meridian 核实过的事实。
2. **No confidence number is ever rendered.** Bands are words. The raw score stays in the store for
   the internal view and a reviewer's audit.
3. **No popularity claim.** Signals are aggregates over the traveller's own corpus —
   在 N 份已收录攻略中被提及 — with the caveat 不代表全网热度.
4. **The two corpora are never summed.** 你的攻略 and 社区攻略 are different claims about different
   bodies of text, and adding them would produce a number that means nothing. Today 社区攻略 is
   legitimately zero; the field exists because the honest answer is a zero, not a merge.

Copyright follows the same logic. Meridian rehosts neither the guide's photographs nor its prose: it
keeps a link, the traveller's own paste, and at most one short sentence as the reason a name was
extracted. Deleting an import deletes the text it held.

## 8. A React bug that only a browser could find

我的收藏 was added to DO as an early `return` placed before the component's remaining hooks. Switching
scope changed the number of hooks between renders, and React refused to render the panel at all.

The type checker was clean. The 103-check behavioural suite was clean. It was the browser suite that
caught it, in the one place it could be caught: a real render that changed a real filter. This is the
argument for keeping an end-to-end suite that fails on console errors even when every DOM assertion
passes — the panel was still visibly broken while every assertion about its content was true.

## 9. Verification

- `npx tsc --noEmit` — clean.
- `npm run validate:data` — passes, including a check driven by the shipped sample guide: extraction
  must produce a known count of credible mentions, every matched id must exist, and no mention may be
  duplicated.
- `npm run test:social` — **103/103**. Chinese, English and mixed prose; duplicate and partial names;
  ambiguity; alias learning; unmatched names; saving; idempotence; creating a place; re-import
  caching; deletion; every failure code; hotel matching. All of it runs against controlled sample
  text, never a live platform, because the product does not fetch them.
- `npm run test:e2e` — **148/148**, and zero console errors, page errors or failed requests. New
  coverage: the entry point, the stated platform limitation, the review, the candidates being plotted
  on the map, saving, 我的收藏, the map still showing saved places, and the unreadable-link failure
  path. The two stale steps that referenced a deleted status vocabulary were rewritten rather than
  deleted, since what they were checking — that an internal decision does not become a public claim —
  still matters.
- `npm run build` and `npm run build:static` — both pass. The static export stashes `app/api`, so
  `/api/extract` is absent from the published bundle, which is the point.
- The message catalogue is **702 keys**, with `en` typed as `Record<keyof typeof zhCN, string>`, so a
  missing English translation is a compile error rather than a silent Chinese string in an English
  interface.

## Known limitations after this pass

1. **Guide import reads text, not images.** Most Xiaohongshu guides arrive as screenshots. Reading
   them needs OCR or a vision model, and it needs the same "the text came from you" contract. Until
   then the traveller retypes or copies the caption.
2. **No video transcripts.** A YouTube or TikTok link could yield a transcript the traveller is
   entitled to read, and that is a fetch with a declared purpose rather than a scrape. It is
   undelivered here because the legal and product framing needs designing, not because it is hard.
3. **The default extractor is rule-based.** It finds names it recognises and names sitting next to a
   category keyword. It does not resolve pronouns, follow a link inside the guide, or read a
   handwriting-style screenshot. An unrecognised name is reported as unmatched rather than guessed.
4. **No shared corpus.** Imported guides are private, so 社区攻略 counts are zero and every signal
   reads 你的攻略. A reviewed, aggregated contribution flow is the only honest path to a real
   community count, and the split already exists in the data shape for it.
5. **`/api/extract` is untested against a live provider.** The route compiles, is excluded from the
   static export, and reads its key server-side, but a real DeepSeek or OpenAI call has not been
   made from this repository, so the prompt's output shape is verified only against its own contract.
6. **Saved places are per-browser.** Like trips and the origin, they live in `localStorage`. Moving to
   accounts replaces the persist middleware, not the components.

---

# Iteration 7 — narrowing to Xiaohongshu, and reading pictures

## 1. Why narrowing was the hard part

The previous pass shipped an import flow that accepted nine platforms and read
only pasted text. It looked like breadth and it behaved like hedging. Every
surface — extraction, layout, copy, tests — carried conditionals for sources
nobody had tried, and the source most of this product's actual readers use sat
behind a generic paste box.

So this iteration deleted most of it. `SocialPlatform` is now a one-value union:

```ts
export type SocialPlatform = 'xiaohongshu';
```

That looks like a joke and is the point. A nine-value union is a standing
invitation for a second platform to arrive by accident — a selector here, a
branch there, and suddenly two half-tested workflows. A one-value union makes the
next platform a deliberate change to three files, and it makes the compiler
refuse to let anything slip through. Every generic platform selector is gone; a
TikTok link is now refused *by name*, with the reason on screen:

> 这看起来不是小红书链接。V1 只支持小红书，请换成小红书链接，或直接粘贴正文和图片。

## 2. Reading pictures without pretending to see

The product promise became: paste a Xiaohongshu post and Meridian finds its places
from **the text and the images**. Xiaohongshu guides are image-first — a name on a
shopfront is as good evidence as a name in a sentence — and treating the pictures
as decoration was the previous version's central mistake.

`lib/research/analyzer.ts` defines one `GuideAnalyzer` interface and two
implementations, because vision is the one part of this feature that costs money
per call, cannot run offline, and changes vendor annually:

| Analyzer | Where | Key | Behaviour |
| --- | --- | --- | --- |
| `heuristic` (default) | browser | none | Reads the text. Cannot see, and says so. |
| `multimodal` | server, `/api/analyze-guide` | yes | Text plus downscaled images, four per call |

Both return the same shape, so the merge step, the resolution pipeline and the
review screen cannot tell which produced their input. A deployment that gains a
key gains image understanding and nothing else changes.

The heuristic analyzer is deliberately **not** a cheap imitation of vision. It
does not guess at a place from a filename or a caption the traveller never wrote;
it marks images `unsupported` and hands the traveller the assignment UI. An
analyzer that manufactured findings would be worse than one that admits it is
blind, because the traveller cannot tell the difference from the card.

## 3. The one thing a model may never do

**No coordinates. Ever.**

This is stated three times — in the system prompt, in the pipeline's types, and in
the review UI — because it is the only failure of this feature that is actively
dangerous. A model that has never seen Bali cannot know where a beach club is. A
plausible-looking latitude would silently corrupt an itinerary with a stop that
does not exist, and nothing downstream would catch it: the router would route to
it, the timeline would show it, and the traveller would discover it in a car.

So position comes from exactly four places, in order:

1. **Meridian's dataset** — curated, photographed, verified.
2. **The alias table** — a name this profile already confirmed.
3. **An external place search** — somebody else's map knows it.
4. **The traveller** — they point at the map.

Each step is strictly more expensive and less trustworthy than the one before,
which is what makes the order correct rather than merely convenient. A step-3 hit
is offered as a question, never written onto the map, and confirming it saves a
place the traveller *owns*, pending review — because "a map search found something
with this name" and "Meridian knows this place" are different claims and the
record has to say which one it is.

The prompt's other two absolute rules exist for the same reason. No invented
attributes (ratings, prices, hours): the guide's words are the guide's words, and
the card says so underneath them. And every finding must declare where it came
from — `detectedFromText`, `detectedFromImageIds` — because
"识别来源：正文 + 图片 3、4" is what lets a traveller judge a card instead of
trusting it.

## 4. The unassigned tray, and why imperfect analysis is survivable

§26 and §17 are the parts of this brief that make the rest honest. Image
understanding is probabilistic, so the design has to assume it will be wrong
sometimes.

Two mechanisms:

**A low-confidence image-only finding is not a place candidate.** Below
`IMAGE_PROPOSAL_FLOOR` it is dropped from the review list entirely and the image
lands in the **未分配图片** tray instead. A card that says "La Brisa" with a map
pin on it reads as an answer; a picture in a tray reads as a question, which is
what a blurry sign actually is.

**Every assignment is a record with an author.** `ImagePlaceAssignment.source` is
either `suggested` or `user`, and a re-analysis only ever replaces `suggested`
rows. §33's worked example — "AI: image 4 is Finns. User: image 4 is La Brisa." —
is therefore enforced by the data shape rather than by a convention, and the
admin view has an audit section that shows which rows came from which.

Reassigning an image *moves* it rather than copying it. One image belongs to one
place, because "this photo also belongs to that other place" is almost never what
somebody means, and a photo attached to two cards makes the unassigned count lie.

## 5. The copyright boundary, in one function

An imported image and Meridian's canonical photography are different things and
must never be confused. One is licensed, attributed, and provably of its subject.
The other is a creator's work that a traveller happens to have in their own guide,
and it was never offered to us.

So there is exactly one function that can create an image record, and it has no
parameter for visibility:

```ts
export function buildImageRecord(input: {...}): ImportImage {
  return {
    ...
    visibility: 'private_import',
  };
}
```

`user_contributed` exists in the type for §21's future flow — a traveller offering
their *own* photograph with explicit consent — and is unreachable from every
import path. There is no code path that promotes an imported picture to public
photography, which is a stronger guarantee than a policy statement.

The boundary is also visible in the product. A saved place shows its canonical
photography, and underneath it, under its own heading:

> **来自你的攻略（只有你能看到）**

Two headings, two meanings. Merging them into one gallery would be the single
easiest way to lose the distinction that matters most.

## 6. Cost control, on both axes

Imported images are expensive in every dimension at once: browser memory,
IndexedDB quota, phone upload time, and tokens. None of that is visible to someone
who drags in thirty screenshots, so it is bounded in `lib/research/image-rules.ts`
— a file of pure functions, which is the only reason the arithmetic is testable
without a browser.

**Storage.** Three sizes for three jobs, and they are not copies: a 320px
thumbnail for lists (a review screen with twenty cards must not decode twenty
1600px bitmaps), a 1600px version to look at, and a 1024px payload generated on
demand and never persisted, sized so signage still survives the downscale.
Everything is re-encoded client-side before it is stored. A SHA-256 of the stored
bytes dedupes a screenshot uploaded twice.

**Calls.** Images are batched four at a time, which cuts request overhead *and*
lets the model relate a sequence — storefront, then the plate, then the sign —
which is exactly the signal §6 wants and a per-image call cannot see. Analyses are
cached by content hash and version. And the external place search runs only after
Meridian fails, only when a provider is configured, with a field mask that can
never request a rating, a photo, an opening hour, a review or a price level.

`NEXT_PUBLIC_PLACE_PROVIDER` and `NEXT_PUBLIC_XHS_RETRIEVAL` both default to
**off**. That is a cost decision and a noise decision at once: on a static
deployment the routes are not there, so making the call produces a 404 logged in
every visitor's console and teaches nobody anything. The honest fallback shows
immediately instead.

## 7. Bugs this pass surfaced

**A draft import was rejected as empty.** Adding pictures to an empty form called
`checkImportInput({ url: undefined, text: undefined })`, which returned
`empty_input` — so the first screenshot silently failed to upload and the form
said "请粘贴链接或攻略文字" to somebody who had just uploaded six screenshots.
§28's manual mode is explicitly "链接 optional, 正文 optional, 上传图片", and the
validator did not know that. It now takes an `imageCount`.

**Reassigning an image left it attached to two places.** `assignImage` filtered
the assignment rows but only ever *added* to the new candidate's
`assignedImageIds`; the old candidate kept the image in its own list, so the
picture rendered under both cards and the unassigned count was wrong. Found by a
test asserting exactly the §33 behaviour it was written for.

**A model prompt is not a mask.** The Places route's field mask is a literal
string, and the test that guarded cost control was asserting on the *comment*
listing the fields it does not request. Rewriting it to read the actual mask is
the difference between testing the behaviour and testing the prose around it. The
same mistake appeared twice more — a "no scraping" assertion that failed on the
comment saying "we do not solve CAPTCHAs", and a "no secrets in the client"
assertion that failed on a comment naming the key it does not read.

## 8. Verification

- `npx tsc --noEmit` — clean.
- `npm run validate:data` — passes.
- `npm run test:social` — **224/224**. Chinese, English and mixed prose;
  ambiguity; alias learning; the four-step resolution pipeline and its cache; the
  external-provider branch, both gated and primed; text/image merging including
  §24's worked example; image assignment, reassignment and the tray; saving with
  image references; pinning; creating from an image; the private-by-default
  boundary; the cost arithmetic; every failure code; the retrieval route's refusal
  semantics. Plus an assertion that no browser-side module reads a secret, made by
  reading the sources.
- `npm run test:e2e` — **172/172**, zero console errors, page errors or failed
  requests. New coverage: the platform refusal, three real generated PNGs uploaded
  and deduplicated, the review, the candidates plotted on the map, the unassigned
  tray, the image question, manual assignment changing both the card and the tray,
  saving, and the saved place keeping the traveller's pictures under their own
  heading.
- `scripts/shots-xhs.mjs` — the same flow at 1440×900 and 390×844, inspected by
  hand rather than asserted. Zero console errors.
- `npm run build` and `npm run build:static` — both pass; `app/api` is stashed for
  the static export and restored, so no route handler ships to GitHub Pages.

## Known limitations after this pass

1. **Nothing is actually retrieved from Xiaohongshu.** The route, its refusal
   semantics and the image relay all exist and are tested, but no approved
   retrieval provider is configured anywhere. The product's answer to a pasted
   link is the honest fallback, which is a complete workflow rather than a
   degraded one — and it is not what §2 asks for at full strength.
2. **Multimodal analysis has never run against a real key.** The prompt, the
   batching, the response contract and the merge are implemented and tested
   against their own contract; nobody has sent a real Balinese menu photograph to
   a real vision model and looked at what came back.
3. **`IMAGE_PROPOSAL_FLOOR` is a judgement, not a measurement.** 0.6 is where a
   wrong attribution stops being useful. It has not been calibrated against a
   labelled set, which is what the unassigned tray is for.
4. **One source, permanently for now.** TikTok, Instagram, Douyin, YouTube and
   blog import are out of scope. A second platform is a deliberate change to the
   union, a provider and a copy pass.
5. **Images are per-browser.** Like trips and the origin, they live in IndexedDB on
   one device. Moving to accounts replaces the persistence layer, not the
   components, but it has not been done.
6. **Twenty images is a ceiling, not a target.** A very long image-heavy guide has
   to be imported in parts, and the UI says so rather than silently truncating.

---

# Iteration 8 — ten destinations, five loyalty programmes

## 1. The audit that changed the plan

The brief was to bring the other nine destinations up to Bali's depth and add IHG,
Hyatt and GHA. Before authoring anything, I wrote a coordinate verifier to check the
existing data — and it found that **73 of Bali's 145 places shared a coordinate with
at least one other record**, most of them labelled `confidence: "verified"` with a
`coordNote` that named a *different* venue. `La Lucciola`, `Merah Putih` and
`Betelnut Café` all sat on Merah Putih's point; nine places sat on The Lawn Canggu's.

That is worse than a gap. A card claiming a source, sitting on somebody else's
doorway, is a lie the traveller cannot detect — and on the map it drew nine markers
stacked on one spot. So the order of work changed: fix Bali first, then expand, with
the verifier as the gate for everything new.

## 2. What the verifier does

`scripts/verify-coords.mts` runs three passes, cheapest first:

1. **Bounds** — is the point inside its destination's own box? Offline, instant, and
   it catches the most common real failure, a copy-paste from the wrong country.
2. **Collisions** — two records of the same kind within 30 m. Cross-kind coincidence
   is normal and only warns: an area's centre is very often the beach inside it.
3. **Reverse geocode** — optional, rate-limited, asks Nominatim what is actually there.

It also now mirrors `isLocatable()`: a record marked `demo` is *unlocatable*, not
misplaced, and bounds-checking one reports the Gulf of Guinea as a data error. The
first run produced 28 phantom collisions from eight such records.

`validate-data.ts` gained the same rule permanently, so two records sharing a point
can never ship again.

## 3. Correcting 73 coordinates, and refusing a false match

The corrections were researched by six independent workers using
`scripts/lookup-place.mjs`, which I rewrote twice during the run:

- **Nominatim IP-blocked the machine.** Six workers at one request per second each
  exceeded a limit that applies to the machine, and the endpoint answered 429 to
  everyone. Fixed two ways: a **cross-process rate limiter** (a shared lock file, so
  concurrent authors serialize rather than stampede), and two more sources serving
  the same OpenStreetMap objects — **Photon** first, **Overpass** second, Nominatim
  last.
- **The tool produced false matches.** Searching "Courtyard Hanoi" matched the OSM
  *city node* "Hanoi", because the name gate accepted a candidate whose name was a
  substring of the query; six hotels were nearly pinned on the city centroid. It now
  rejects generic place types (city, suburb, district, island…) and requires the
  candidate to be the whole query or a substantial part of it. A locality gate was
  added for the same reason: "Anomali Coffee Sanur" resolved to a different branch
  10 km away.
- **The cache corrupted itself.** Concurrent read-modify-write produced a file that
  was valid JSON for 156 KB followed by a truncated fragment, and every later run
  died before starting. Writes now go through a temporary file and a rename.

Every correction is a reviewable entry in
`lib/data/destinations/extra/coord-corrections.ts`, and it records *why*. The
hand-made decisions live in a separate file, `coord-corrections-manual.ts`, because
the first time the generated half was re-merged it silently deleted them.

The honest outcomes were not all corrections. Eight records are now **unlocatable**:
a generic "surf school" label, a "water sports" label, two transfer products that
depart from a harbour that already has its own record, and four venues no source
could place. They disappear from the map and the itinerary, which is better than a
marker on somebody else's doorstep. Bali's `approximate` count went from a claimed
zero to 28 — the same data, described accurately.

## 4. Five loyalty programmes

`HotelGroupId` went from two values to five, and the type system found every place
that needed updating — which is exactly why it is a union rather than a string:

- **Marriott Bonvoy, Hilton Honors, IHG One Rewards, World of Hyatt, GHA DISCOVERY**
- 73 brands in the registry, including Hyatt's soft brands (Unbound Collection,
  Destination, JdV), IHG's Vignette Collection, and 15 GHA members.

**GHA is modelled as an alliance, not a hotel company.** Its members are
independently owned brands sharing one loyalty scheme, and the filter row says 联盟
next to it rather than implying a parent group that does not exist — the same
honesty the dataset already applied to SLH as a Hilton partner.

**Five programmes, five shapes.** Colour was never the only differentiator in this
marker system, and five programmes on one island is exactly where that rule earns
its keep: M is a rounded square, H a circle, I a hexagon, Y a diamond, G a shield,
each with its own letter. At 26 px, in greyscale, or for a colour-blind reader, five
coloured circles would be indistinguishable.

Two bugs here were only findable in a browser. The **STAY filter narrowed the list
but not the map**, because the filter was component state the map could not read —
so the panel said "Hyatt" while every programme stayed drawn. And
`buildMapMarkers` assigned the marker layer with
`hotelGroup === 'marriott' ? 'marriott' : 'hilton'`, so every IHG, Hyatt and GHA
property was drawn as a **Hilton** marker: invisible when its own layer was on, and
mislabelled as the wrong programme. The counts had been fixed; the half that decides
what a marker looks like had not.

## 5. Where a target was wrong

Ten authors were given targets and told, repeatedly, that **a target is not a
quota**. Several reported shortfalls instead of filling them, and each was right:

- **Phu Quoc has 7 loyalty hotels, not 12.** Conrad, Hilton and DoubleTree Phu Quoc
  are signed-but-unbuilt APEC-2027 projects; Park Hyatt opens in 2027; GHA's Vietnam
  members are all on the mainland.
- **Cebu has 5.** Crowne Plaza and InterContinental Cebu do not exist; the former
  Hilton is now a Mövenpick; Courtyard Cebu never opened.
- **Boracay has 1 and Palawan has 1.** Both were verified against the brands' own
  location lists. Boracay's upscale inventory is domestic — Shangri-La, Crimson,
  Henann, Discovery Shores — and maps to no programme.
- **Phnom Penh has 5.** Hilton has exactly one hotel in the whole of Cambodia, in
  Siem Reap.

Those are findings, and they are in the report rather than papered over with invented
properties. Completing the set also required three small registry additions that real
authors were blocked on: Hyatt's soft brands, IHG's Vignette Collection, and
`sunway` — the only GHA member in Phnom Penh.

## 6. Other real bugs the expansion surfaced

- **Extra areas were registered globally but never attached to their destination.**
  A newly authored area had coordinates and hotels referencing it, while the EXPLORE
  panel, the area list and the validator's own referenced-area check all read
  `destination.areas` and could not see it. The symptom was `areaId "vung-bau" does
  not exist` for an area sitting right there in the dataset.
- **`El Kabron` was two records** — once as a restaurant, once as a cliff club —
  putting two markers on one cliff. Deduplicated, with the surviving record carrying
  the union of the categories.
- **`Pro Surf School Bali` was filed under Uluwatu**, 20 km from the Kuta street it
  is on.
- **The validator's duplicate-property rule compared coordinates only**, which
  worked for forty hotels and flagged neighbours at five: the Holiday Inn and Hilton
  Garden Inn on Nusa Dua's Jalan Pratama are 120 m apart, and three Saigon and Hanoi
  pairs are 96–134 m apart. A duplicate shares an *identity* — same name or same
  brand — not just a street.
- **The STAY subtitle said "Bali's Marriott and Hilton hotels" on the Hanoi page.**
  Stale copy from a two-programme, one-destination dataset.
- **The STAY panel's `hotelGroup must be marriott or hilton` check** in the
  validator would have rejected every new hotel for being correct.

## 7. After this pass

**10 destinations · 117 areas · 97 loyalty hotels · 446 places** — from 48 areas,
42 hotels and 216 places, all ten now at reference tier.

- `npx tsc --noEmit` — clean.
- `npm run validate:data` — passes, 0 errors.
- `npm run verify:coords` — **652 coordinates, 0 failures**, 29 expected cross-kind
  coincidences, 8 deliberately unlocatable.
- `npm run test:social` — 224/224.
- `npm run test:e2e` — 174/174, zero console errors.
- `npm run build` and `npm run build:static` — both pass.

## Known limitations after this pass

1. **Five programmes, not all of them.** Accor Live Limitless and Wyndham Rewards are
   absent rather than half-populated. A programme earns its place by having enough
   real inventory in Southeast Asia to change where somebody stays.
2. **Some destinations are genuinely thin.** Boracay has one loyalty hotel, Palawan
   one. That is the verified ceiling, not an omission, and the coverage table says so.
3. **28 of Bali's 145 places are not confidently located** — 20 approximate and 8
   unlocatable. Nearly all the approximate ones are small restaurants that no OSM
   mapper has surveyed, positioned at street level from their published address.
4. **Photography did not grow with the data.** The new hotels and places have no
   licensed imagery, so their cards fall back to the no-photograph state rather than
   borrowing a neighbour's picture.
5. **Overpass was unreachable for part of the run**, so some address lookups fell back
   to Nominatim street geocoding or were dropped.


---

# Iteration 9 — Malaysia, and the Chinese that was missing

## 1. The gap nobody had noticed

The product is Chinese-first, and Bali's records were written that way. The other nine
destinations were not: their ORIGINAL records predate that decision, so 河内's hotel cards
rendered in English beside 巴厘岛's Chinese ones. The expansion pass had authored Chinese
inline for everything it added, which made the gap harder to see — the new records looked
right, and the old ones were the ones reading wrong.

Measured, it was concrete: 4 areas, 4 hotels and 8–10 places missing per destination.

Written as an overlay (`lib/data/zh/starter-zh.ts` plus one module per author) rather than
edited into nine geography files, for the same reason the coordinate corrections are: the
data files are coordinate-verified and reviewed as such, and presentation copy has a
different lifetime. Three authors worked in parallel without touching a shared file.

`nameZh` is omitted wherever a name has no form in real Chinese use — most Western-named
beach clubs and restaurants, and hotel brands that do not publish a Chinese property name.
That is why the audit still reports ~115 Bali places without `nameZh`: they are Latin-named
venues, and inventing a transliteration would be worse than leaving the canonical name.

Two real mistakes surfaced while wiring it:

- **The `??` fallback never fired.** `BALI_HOTEL_ZH[id] ?? STARTER_HOTEL_ZH[id]` looks
  right until you notice the Bali entry *exists* and simply omits `nameZh` — so the overlay
  value was unreachable. The lookup is now per FIELD, which is what it always meant.
- **Two modules exported the same interface names** (`AreaZh`, `PlaceZh`), and the collision
  silently resolved to the narrower one. The overlay's types are namespaced now.

One agent also corrected a premise I had given it: I described an 18-hotel job in Bali; the
real gap was 3 records, one of which is a property Hilton *does* publish in Chinese — and the
existing comment in `bali-zh.ts` claiming otherwise was wrong. The comment is fixed.

## 2. Malaysia

A new country in the dataset: `RegionId` already had `malaysia`, so the work was geography,
hotels and connections rather than architecture.

- **Penang** — 14 areas, 10 hotels, 40 places. Marriott 3, Hilton 2, IHG 1, GHA 4, **Hyatt 0**.
- **Kuala Lumpur** — 14 areas, 14 hotels, 40 places across all five programmes.

Both reported their honest ceilings rather than filling targets, and both exclusions are the
kind that matter: **Conrad KL** is an OSM object tagged `(U/C)` — a construction site;
**Canopy by Hilton KL** is tagged `Former Canopy by Hilton KL`; **Holiday Inn Resort Penang**
closed and was sold with vacant possession in 2021, and **Four Points by Sheraton Penang** is
now a Mercure, which is why a lookup for it returns the wrong brand.

The consequences are visible in the product rather than hidden: Penang's STAY filter offers
four programmes, not five, because the filter row is generated from real inventory.

One flagged issue was fixed on integration: `butterworth-mainland` was a day-trip zone hosting
three bookable hotels, which contradicts its own label — it is a stay base now.

## 3. Wiring bug this pass introduced and caught

My first Malaysian integration registered each destination's areas **twice** — once through
the destination seed and once through the extra-geography aggregator — and the validator
reported 28 duplicate Penang area ids. The fix empties the seed's own `areas` array, because
these destinations' geography lives entirely in the extra modules. It is the same class of
mistake as adding a record to two registries "to be safe".

## 4. After this pass

**12 destinations · 145 areas · 121 loyalty hotels · 526 places.**

- `npx tsc --noEmit` — clean.
- `npm run validate:data` — passes, 0 errors.
- `npm run verify:coords` — **784 coordinates, 0 failures**.
- `npm run test:social` — 224/224.
- `npm run test:e2e` — 185/185, zero console errors.
- Both Malaysian destinations inspected at 1440×900 and 390×844.

## Known limitations after this pass

1. **Penang has no open Hyatt**, so its filter row offers four programmes. Its JdV by Hyatt is
   signed and not trading.
2. **Photography did not grow with the data.** The new Malaysian records — and the expanded
   ones from the previous pass — have no licensed imagery and show the no-photograph state
   rather than borrowing a neighbour's picture.
3. **Chinese `nameZh` is deliberately absent** for Latin-only venues. That is a judgement per
   record, not a gap, and it is recorded in each report.
4. **Accor and Wyndham are still absent**, which matters more in Malaysia than anywhere else in
   the dataset: a large share of Penang and KL's upscale inventory is Accor.


---

# Iteration 10 — accommodation as stays, not as rows on days

## 1. The mistake the old model made

A hotel is not a place you visit, it is where you *are*. The itinerary stored it
as an ordinary item, so a three-night stay meant adding W Bali to the timeline on
the 10th, the 11th and again on the 12th — and nothing in the data said those three
rows were one booking. Every question the itinerary actually needs answered was
therefore unanswerable: where does today start, where does it end, is today a
moving day, is any night unbooked.

The fix is a `TripStay` — a hotel and a date range — and a rule that makes
everything else follow:

> **start = the stay covering last night. end = the stay covering tonight.**

On the 12th of a 10–12 / 12–14 trip, that gives the hotel you woke up in and the
hotel you sleep in. Different hotels, and the day is a hotel-change day **without
anyone declaring it**. Arrival and departure days fall out too: the first day has
no previous night, the last day's night is not spent.

Nothing derived is stored. Anchors are recomputed from the stays on every render,
so editing a stay, changing a trip date or deleting a booking moves every affected
day with no invalidation step and no way for the timeline to disagree with the
accommodation list.

## 2. Five concepts, kept apart

The brief asked for this explicitly, and it is the reason the model is this small:

| Concept | What it is | Stored? |
| --- | --- | --- |
| `Place` / `Hotel` | a canonical geographic entity | yes, curated |
| `TripStay` | a booking: hotel + date range | yes, by the traveller |
| `ItineraryItem` | a visit, at a time | yes, by the traveller |
| `DayAnchor` | where a day starts and ends | **no — derived** |
| `TransportLeg` | a route between two stops | no — computed |

A hotel the traveller returns to later is one canonical record and two stays. No
copy of a hotel is ever made.

## 3. Routing the change day for real

`buildLegsForStops` takes an arbitrary ordered stop list, so a day is routed as
`startAnchor → items → endAnchor` through the **existing** provider. On a change
day that means the hotel-to-hotel drive is measured like any other leg rather than
asserted as metadata — the browser check showed `09:00 W Bali → 09:24 Holiday Inn
Benoa`, a real 24-minute drive between two real properties.

The seam for arrival and departure is the same one: an airport item already on the
day becomes the start anchor on an arrival day and the end anchor on a departure
day. `origin → airport → first hotel` needs no second model, and no flight
integration was built.

## 4. Two clock rules that were previously impossible

**The start time is a setting.** It was the literal `9 * 60` inside the timeline,
so "we leave at 08:30" could not be expressed. There is now a trip default and a
per-day override, and — importantly — the override is `undefined` rather than a
copied value, so changing the trip default still moves every day the traveller
never touched.

Changing a day's start time recomputes the **clock** without re-requesting the
routes: the legs hook keys on geometry only, so a time edit leaves the measured
legs in place. That distinction is the whole reason the schedule is a pure
function in `lib/schedule.ts`.

**A fixed time is a booking, not a preference.** Arriving early produces usable
waiting time; arriving late produces a conflict with the number of minutes. The
time is never moved, because it is the one thing in the day the traveller cannot
change.

## 5. Migration that refuses to guess

A hotel row on a single day could mean "I slept here" or "I went to look at this
hotel", and no amount of logic distinguishes them. So the migration converts **only**
a run of the same hotel on two or more consecutive days — which cannot mean
anything else — and leaves every ambiguous row exactly where it was, still
rendering as an itinerary item. It is idempotent, and it runs on every hydrate
because it is safe to.

The result is that no existing trip is rewritten on a guess, and a trip that
predates the model keeps working with its hotel rows intact.

## 6. What the interface says

- **住宿** sits above the day tabs, because it decides the shape of every day
  below it: hotel, check-in, check-out, and a warning per unbooked night.
- An overlapping stay is **refused** on add — a trip cannot be in two hotels on
  one night — but an *edit* that creates an overlap is stored and marked, because
  discarding a traveller's input mid-edit is worse than showing them the conflict
  they are fixing.
- Anchors render as flat tinted rows, quieter than the photograph cards for
  attractions, labelled 今天从这里出发 and 今晚住这里.
- 换酒店 appears on the day, not in a settings screen.

## 7. Verification

- `npx tsc --noEmit` — clean.
- `npm run test:trip` — **111/111**, covering every case the brief listed: one,
  two and three hotels; the change day; a normal day; arrival and departure days;
  an unbooked night; overlapping stays; editing dates; changing hotel; deleting a
  stay; `Hotel A → POIs → Hotel B` routing with an injected provider; the default
  and per-day start times; a fixed-time item; a fixed-time conflict; and legacy
  compatibility including the ambiguous single row.
- `npm run validate:data` — passes.
- `npm run verify:coords` — 0 failures.
- `npm run test:social` — 224/224.
- `npm run test:e2e` — 185/185, zero console errors.
- The stay editor and a real hotel-change day were driven in a browser at
  1440×900 and 390×844: two stays added, day 3 showing W Bali → 24 min → the next
  hotel with the 换酒店 badge, zero console errors.

One bug this pass introduced and caught: the derived-anchors `useMemo` was placed
after the `if (!trip)` early return, so creating a trip changed the number of hooks
between renders. Moving it above the returns fixed it — the same class of bug as
the DO panel's two iterations earlier, with the same symptom.

## Known limitations after this pass

1. **A stay is not a booking.** No confirmation number, no room type, no rate, and
   no check-in time.
2. **A hotel-change day is routed, not optimised.** The brief was explicit that the
   data model should support "put the southern sights on the day you move south";
   it does — the day knows both anchors and every stop's position — and the
   optimiser is not built.
3. **Arrival and departure anchors are expressible, not computed.** They use the
   airport item already on the day. The origin-to-airport leg and any live flight
   data are out of scope.
4. **Unbooked nights warn and nothing else.** The model never invents a hotel, and
   it does not suggest one either.

---

# Iteration 11 — 我的行程 as a real entity

The previous pass gave a trip an accommodation model. It still did not have a
home. A trip existed because the planner had put you into planning mode, and the
only way back to one was to remember which destination you had started from and
whether the store had rehydrated. This pass makes the trip the object and the
planner one of its doors.

## 1. The list is not a menu

My Trips answers a question the product previously could not: *what have I
actually got?* Each card carries the destination, the dates and night count, the
travellers, the accommodation in hotel order, and the number of planned places —
deliberately excluding hotel and airport rows, because a stay is not a place you
visit. Trips are grouped active → upcoming → past, and within a group the soonest
comes first. A past trip sorts below every future one; an in-progress trip sorts
above both.

## 2. A trip page, not planning mode

The detail page is `/trips/detail/?id=…`, not `/trips/[id]`. Trip ids are created
at runtime, so a static export cannot prerender a route segment for them; a query
parameter inside a Suspense boundary is the honest shape for a page whose only
argument is a client-side id. The 行程单 and 地图 tabs read the same trip, and the
map only requests routes when a single day is selected — a whole-trip map would
otherwise ask the routing provider for every leg at once to draw lines nobody
asked for.

## 3. Everything visible is editable

Dates and travellers, a stay's hotel and check-in/check-out, a day's start time,
and a stop's position, day and fixed time are all edited in place on the page that
shows them. Editing the end date re-derives the days **by date**, not by index, so
stays, per-day start times and the stops on them survive. That mapping bug is easy
to write and was caught by a test that changes the end date after setting both.

## 4. Deleting asks first, duplicating does not lie

Delete is two-step and stays two-step. Duplicate copies the days, the stops and
the stays — with **new stay ids**, because sharing them would make the copy and the
original the same accommodation object, and editing one would silently edit the
other.

## 5. Migration that refuses to guess, again

Old trips have no `stays` and no start times. Migration converts a hotel repeated
on consecutive days into a stay and preserves anything ambiguous as it was; the
default start time is 09:00 only where nothing was ever set. Both migrations are
idempotent, so rehydrating twice is the same as rehydrating once.

## 6. The Social Import link is prepared, not built

An import can record which trip it fed, and the places it saved can be read back
for a trip. That is the whole relationship for now: no automatic itinerary merge,
no implicit place creation.

## 7. Verification

- `npx tsc --noEmit` — clean.
- `npm run validate:data` — passes.
- `npm run verify:coords` — 0 failures, 37 expected warnings.
- `npm run test:social` — 224/224.
- `npm run test:trip` — 128/128, now including trip phase, ordering, summaries
  and the planned-place count.
- `npm run test:trips` — 42/42, the 15-step acceptance scenario driven through the
  real UI at 1440×900, zero console errors.
- `npm run test:e2e` — 196/196, zero console errors: the full V1 workflow plus a
  condensed My Trips pass.
- `npm run build:static` — `/trips` and `/trips/detail` both export.

## 8. Addendum — the PLAN panel gave the day its space back

The accommodation editor was permanently expanded: a hotel select and two date
inputs per stay. On a 1440×900 laptop that was **415px** of the right rail, and the
day itself — the thing PLAN is opened to work on — began ~340px lower and showed
**one** stop above the fold. Editing accommodation is a once-per-trip act; reading
the day is the continuous one. The panel had them the wrong way round.

It is now a summary of one or two lines with the editor behind 修改:

```
住宿 · 3 晚                                 修改
🏨 巴厘岛丽思卡尔顿酒店 1 晚 → 巴厘岛 W 度假酒店 1 晚 → 巴厘岛瑞吉度假酒店 1 晚
```

- One hotel reads as one line — `🏨 巴厘岛丽思卡尔顿酒店 · 1 晚` — and three
  hotels as `3 家住宿 · 2 次换酒店` plus the chain. The chain clamps to two lines,
  so nine hotels do not cost more height than three.
- **Nothing that must not be hidden is hidden.** A night with no bed and a stay
  whose dates cannot be true both stay visible when collapsed; only the per-night
  detail and the inputs move behind 修改.
- 修改 opens the existing editor unchanged, with 完成 to close it. Switching day
  closes it again — the day the traveller just switched to gets the space — while
  the choice is remembered in `sessionStorage` across tab switches and reloads.
- The day's start time is one row with the day's summary instead of its own row,
  and it is stated once: the native input already renders `上午 09:00`, so the
  separate `09:00` printed beside it said the same thing twice. When the day
  inherits the trip default the row now says 默认 instead.

Measured on the three-hotel / four-day trip, 1440×900:

| | editor | fully visible stops | itinerary window |
| --- | --- | --- | --- |
| before (always open) | 415px | 1 | ~90px |
| after (collapsed) | **71px** | **2** | **443px** |

The third stop is at 909px, nine pixels below the fold. Three stops *and* their
measured transport rows are ~760px of content; no collapsed summary makes that fit a
900px screen, and the transport rows are not being trimmed to pretend otherwise.
What the fix guarantees is that the window belongs to the itinerary.

`scripts/check-plan-space.mjs` builds exactly that trip through the UI and asserts
all of the above — 28 checks, including that no hotel select or date input exists
outside edit mode, that the summary survives a day switch, and that the session
remembers the editor was left open.

## Known limitations after this pass

1. **One browser, no account.** Trips are `localStorage`; clearing site data
   deletes them.
2. **No export or share.** No PDF, no calendar, no link.
3. **The import link is one-directional.** A trip can name its import; nothing
   proposes an itinerary from one.
4. **A duplicate is a full copy.** There is no "reuse this trip's hotels for new
   dates" operation beyond duplicating and editing the copy's dates.
5. **The list does not show cost, flights or bookings** — none of which exist.

---

# Iteration 12 — Phu Quoc gets photography, and the pipeline becomes a pipeline

Photography existed for exactly one destination, and the code said so out loud:
`lib/images/index.ts` imported `BALI_IMAGES` and returned nothing for everything
else. Phu Quoc — 13 areas, 7 hotels, 37 places — had no image anywhere in the
product. This pass gave it 54, and in doing so turned a Bali script into a
destination-agnostic pipeline that the other ten destinations can now use.

## 1. The pipeline, not a second script

`scripts/fetch-bali-images.mjs` became `scripts/fetch-destination-images.mjs`, and
the Bali subject lists were extracted verbatim into
`scripts/images/subjects/bali.mjs`. "Verbatim" is checked, not asserted: the
extraction was diffed list by list against the original file, so
`--destination bali` reproduces the manifest that shipped. `--destination
phu-quoc` uses `scripts/images/subjects/phu-quoc.mjs`, which is data — 7 hotels,
13 areas, 37 places with queries, name gates and locality rules.

`npm run images:resolve -- --destination phu-quoc` and its `--download` twin are
the whole interface. `lib/data/images/index.ts` merges one manifest per
destination, and the provider resolves from the merge.

## 2. Three real bugs the second destination exposed

**Hyphenated titles were silently dropped.** Commons writes "Bãi Khem" as
`Bai-Khem` and the JW Marriott as `Jw-marriott-phu-quoc-bai-kem.jpg`. The matcher
compared a human search phrase against the raw title, so the JW Marriott's own
photograph — already in the results — failed the "does this file name the
property" test. Titles are now normalised (`-`/`_` → space) before every gate.
This affected Bali too; it just had enough photographs that nobody noticed.

**Commons candidates had no destination gate.** Openverse candidates were checked
against a Bali locality regex; Commons candidates were not. For Bali that was
survivable — "Uluwatu" is not a word in anyone else's address. For Vietnamese
names it was not: "Bãi Thơm" resolved to a commune office in Thái Bình, "Vũng
Bầu" to three 1946 government documents and a coal mine in Poland, "Ông Lăng" to
a temple statue in the Mekong Delta. Every gate now sits behind a per-destination
`REQUIRE_LOCALITY`, which the pipeline already had the concept of.

**A name is not a location.** The fixes above still let through subjects whose
own name collides: a night market, a pearl farm, a fish-sauce works, a temple.
Every Phu Quoc subject now carries a `mustMatch` naming the subject itself, and a
beach or a nature subject additionally rejects titles naming a resort — a
photograph of the JW Marriott at Bãi Khem is real, licensed, and not a photograph
of Bãi Khem.

One exemption was tried and withdrawn. "Cổng miếu Gia Long" is the only Commons
file matching Gia Long's temple, and its name is specific enough that an
`ownName` exemption seemed reasonable. The Commons **category** check then showed
it is in `Dong Thap` — a thousand kilometres away, in the Delta. The exemption is
gone, the subject has no image, and the card says so. That is the pipeline
working: a wrong photograph is worse than no photograph.

## 3. What Phu Quoc has now

| | subjects with photography | images |
| --- | --- | --- |
| Areas | 7 / 13 | 21 |
| Hotels | 2 / 7 | 2 |
| Places | 18 / 37 | 31 |
| **Total** | **27** | **54** |

- **Areas**: Dương Đông, Long Beach / Bãi Trường, An Thoi & the south, Dương Tơ,
  Bãi Sao, Hàm Ninh, the An Thoi archipelago.
- **Places**: Sào and Khem beaches, the national park's Suối Tranh, VinWonders,
  Vinpearl Safari, Grand World, Aquatopia, the Hòn Thơm cable car, Phu Quoc
  Prison, Dinh Cậu, Sùng Hưng and Trúc Lâm Hộ Quốc pagodas, both markets, the
  night market, Bãi Vòng and An Thoi ports, and one restaurant.
- **Hotels**: JW Marriott Emerald Bay and InterContinental Long Beach. The other
  five have nothing verifiable, and their cards say 暂无该酒店实拍照片 rather than
  showing a beach that is not the hotel.
- Refused for cause: Gia Long's temple (the file is in Đồng Tháp), Khải Hoàn fish
  sauce (only generic fish-sauce-factory photographs), Ngọc Hà pepper farm (a
  pepper farm in Vietnam, not that farm), Ngọc Hiền pearl farm (a pearl shop in
  Phu Quoc, not the farm), and the restaurants and bars nobody has photographed.

## 4. What was NOT verified

**Nobody has looked at these photographs.** The model that ran this pass has no
image input, so verification is filename-based plus the Commons category check
described above, and `DEPICTS_OVERRIDE` — the per-file list of "this is actually
a pool, not the signage" corrections that the Bali pass filled in by eye — is
empty. The two hotel photographs are the weakest entries: the InterContinental
one is title-verified only, and the JW Marriott file carries no Commons category
at all. A human pass over `public/images/phu-quoc/` is the outstanding work, and
`DEPICTS_OVERRIDE` is where its findings go.

## 5. Verification

- `npm run test:images` — **110/110**. Every manifest entry has a file on disk
  over 4KB, a licence, an author and an alt text; exactly one hero per subject and
  it is first; no orphan files; no non-commercial licence; every subject is a real
  entity in the dataset; and no `kind:id` key is claimed by two destinations —
  which would be one destination's card showing another's photograph.
- `npm run check:images-ui -- --destination phu-quoc` — **10/10** in a real
  browser: 4 area images on the stay scope, 3 on the day-trip scope, 2 property
  photographs with 5 of 7 hotels stating their absence, 16 place images, zero
  foreign images, zero 404s, zero console errors.
- The same UI check against **bali** — 10/10, unchanged after the merge.
- `tsc --noEmit`, `validate:data`, `verify:coords`, `test:social`, `test:trip`,
  `test:trips`, `test:e2e` all pass unchanged.

## Known limitations after this pass

1. **Ten destinations still have no photography.** The pipeline supports them;
   each needs a subject file, and each will need the same gates authored.
2. **A title is not a photograph.** Everything above is title-and-category
   verification. No human eye has confirmed a single Phu Quoc image.
3. **Coverage is thin exactly where the market is.** Five of seven hotels have
   nothing, because Vietnamese resort photography is not on the open web.
4. **The `depicts` field is nearly all `general`.** It drives gallery variety, and
   with no one looking at the images the classifier only had titles to work from.

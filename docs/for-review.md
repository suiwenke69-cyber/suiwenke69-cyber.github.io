# Meridian — review brief

*Self-contained brief for an automated reviewer. Everything below is stated as it
actually is: what is real data, what is curated, what is modelled, and what is
deliberately absent. If a claim is not backed by something in this repository, it is
not claimed.*

---

## 1. What the product is

**Meridian** is a map-first travel planner for travellers departing from **Singapore**.
The map is the product; the itinerary is built around the map.

The core loop:

```
Southeast Asia map  →  select a destination  →  destination map
  →  explore hotels / activities / areas  →  add places to a trip
  →  organise by day  →  see the route  →  reorder it
```

Stack: Next.js 15 (App Router) · React 19 · TypeScript · Tailwind · **MapLibre GL JS**
over keyless vector tiles · Zustand + `localStorage`.

Runtime dependencies: `maplibre-gl, next, react, react-dom, zustand` — **five**.

---

## 2. Scope, stated plainly

| | |
|---|---|
| Destinations | **10** (Singapore origin + 9 destinations across Indonesia, Vietnam, Cambodia, the Philippines) |
| Reference destination | **Bali** — the only one built to full depth |
| Bali data | **15 travel areas · 20 loyalty hotels (15 Marriott Bonvoy, 5 Hilton Honors) · 48 places** |
| Ground transport | Real road geometry and drive times from **OSRM** |
| Live prices | **None.** No fares, no nightly rates, no availability. Deliberate. |
| Accounts / backend | **None.** Trips live in `localStorage` on one browser. |
| Traffic modelling | **None.** Every measured leg is labelled a free-flow estimate. |

The other nine destinations are architecturally complete and honestly thin: 2–4 areas,
1–4 loyalty hotels, 7–8 places each. They are selectable and they render; they are not
yet good enough to plan a real trip from, and the repository says so.

---

## 3. The honesty rules this project is built on

These are enforced in code and in the data validator, not just in prose.

### 3.1 No invented numbers

`RouteResult` carries `status: 'ok' | 'unavailable'`, and its distance and duration
fields are **nullable**. A leg with no routing answer cannot carry a fabricated
duration — the type makes it impossible. The UI renders "No route data" and marks the
whole day's clock `approx`.

Distance, duration and the transport *recommendation* are three separate abstractions.
They are never conflated, and the estimator that exists (a geodesic fallback) is fenced
off to efficiency heuristics and labelled a heuristic.

### 3.2 Hotel photography must be provably of that property

A hotel image is used **only** if it is verifiable:

- the file title names the property **and** the correct part of Bali, or
- the file sits in the property's own Wikimedia Commons category.

The rule was written because a looser one shipped the following:

| Candidate file | What it actually is |
|---|---|
| `Four Points by Sheraton Taipei Bali 01.jpg` | A hotel in **New Taipei City**, in a district called Bali |
| `Blanco Renaissance Museum Ubud Bali.jpg` | A **museum** in Ubud, matched on the word "Renaissance" |
| `Le Meridien Nirwana Bali Pool garden.jpg` | A **different resort**, in Tabanan, 30 km from Jimbaran |
| `The Pond with Stepping Stones, Karangasem Palaces.jpg` | A palace pond, matched on "stones" |

Photographs of a hotel's **name on a wall** are excluded: correctly licensed and provably
of the property, but not property photography. A traveller learns nothing from a dark
close-up of the word "WESTIN".

**Result: 4 of 20 Bali hotels have property photography.** The other sixteen have no
entry in the manifest at all and render:

> No property photography available for this hotel

They never borrow their area's beach photograph. Every image carries its author, licence
and source page, and licences are restricted to commercial use — CC BY, CC BY-SA, CC0,
public domain. No NC, no ND.

**Places fare better: 41 of 48 Bali places and all 15 areas have photography.** The seven
without are three beach clubs and four logistical waypoints (a road strip, a bus station,
two meeting points) where any photo would be a stand-in for "somewhere in Kuta".

### 3.3 Travel areas are approximations, and are drawn as such

Canggu, Seminyak and "Nusa Dua" are planning areas, not administrative units. Canggu is
three villages; Uluwatu is a temple, a surf break and a resort strip on one cliff;
"Nusa Dua" in traveller usage includes Tanjung Benoa.

So a zone is **a convex hull of that area's own mapped hotels and places**, offset
outward and smoothed. A hull keeps the shape of the content where a radius circle said
nothing. The boundary is drawn as a **fine dashed edge** — dashes being the conventional
cartographic signal for "approximate" — and the UI says in words that it is an
approximate travel extent, not an official boundary. The zones are inserted *below* the
basemap's water layer, so they clip to the coastline instead of floating out to sea.

### 3.4 No prices

Hotel cost is a **brand-positioning tier** (`$$`–`$$$$`) with its basis printed on the
card: *"Ultra-luxury positioning (brand tier plus location) — derived from brand
positioning, not a live rate."* A validator rule fails the build if a price-like string
appears in any hotel description.

---

## 4. What was built, verbatim from the brief

### 4.1 The Southeast Asia homepage

- MapLibre with a **style authored inside the repository** (`components/map/basemap-style.ts`)
  over keyless vector tiles (CARTO primary, OpenFreeMap fallback). No vendor style JSON is
  loaded at runtime, so no vendor can start watermarking the map.
- Label set cut to country → region → city → town. No villages, suburbs, POIs, road names,
  house numbers or waterway names. English preferred everywhere.
- Ten destinations drawn as **native map layers**, not floating pins, so they collide and
  fade like the cartography around them.
- Singapore is a bespoke `Home / Origin` marker; exactly one route arc is drawn, and only
  for the selected destination.

### 4.2 The destination experience — EXPLORE → STAY → DO → PLAN

The four tabs are the traveller's verbs, not the system's nouns, and each one re-frames
the map for its own question. The map shows **one thing at a time**: areas, or hotels, or
one place category, or the active day's route.

- **EXPLORE** draws nine stay-base zones *or* six day-trip zones — never both. The panel's
  segmented control and the map share that state through the UI store, so the list and the
  map cannot disagree.
- **STAY** lists hotels photographed-first, with Marriott/Hilton filtering and a removable
  area-filter chip.
- **DO** shows one category at a time; a place with no photo gets a compact text-led card
  rather than an empty box the size of the photograph it replaces.
- **PLAN** is the itinerary, with transport legs between every consecutive pair of stops.

### 4.3 Transport legs in PLAN

```
09:00  The Westin Resort Nusa Dua, Bali
         ↓
       Car / Grab · ≈25 min · 21 km
       About 21 km by road distance. Ride-hailing works, though drivers
       sometimes decline long pickups at peak times — a private car is
       the safer fallback.
       OSRM road route · free-flow estimate
       or a private car and driver / a taxi
09:25  Uluwatu Temple (Pura Luhur Uluwatu)
```

Mode, measured duration and measured distance on one line; the rationale and **the data
source** underneath; alternatives named. The badge names the routing engine or says
"No route data".

### 4.4 The basemap

The first version had no island in it: land and sea differed by about twenty colour
values and nothing drew a coastline. Fixed with a deeper sea, **a coastline stroked from
the water polygons** (neither tile host ships a coastline layer), and road widths that
are actually visible at island scale.

---

## 5. Data integrity — the checks that exist because something broke

`npm run validate:data` exits non-zero on any error. Each rule exists because the
corresponding failure actually happened:

1. **Duplicate hotel id** — caused a React key collision.
2. **The same property plotted twice under two ids.** `four-points-by-sheraton-bali-ungasan`
   and `four-points-bali-ungasan` were one hotel at identical coordinates with two Marriott
   codes (`DPSFG` and `DPSFP`). Only `DPSFG` is real. The STAY list simply showed it twice.
   The validator now compares every hotel against every other in the same destination
   within 150 m.
3. **An entity outside its destination's own map bounds** — the map fits those bounds on
   arrival, so such an entity can never be seen without panning into empty space. It is
   also the signature of a transposed lat/lng.
4. **Image manifest keys that match nothing.** The image generator keeps its own search
   list, and it had drifted from the dataset (`jatiluwih-rice-terraces` vs
   `jatiluwih-rice-terrace`, `amed-beach` vs `jemeluk-beach`, and nine more). A manifest
   key that matches nothing looks exactly like an entity with no photography, so
   **seventeen places and two areas showed "no photo yet" while their photographs sat
   unused on disk.** The validator now fails if any manifest key is not a real entity, and
   fails again if the generator's own id list names anything the dataset lacks.
5. **A NaN area radius** — crashed the mapping library and blanked the planner.
6. **A brand id with no registry entry** — silently lost its positioning label.
7. **A price-like string in a hotel description** — forbidden outright.

Additional guards: every hotel and place carries a coordinate `confidence`
(`verified` / `approximate`) plus a note on **what the point marks**. Four Bali places are
`approximate` and are drawn with a visible badge — a rice-terrace area or a several-
kilometre road strip genuinely has no single point, and the note says so.

---

## 6. Verification status

| Check | Result |
|---|---|
| `npx tsc --noEmit` | clean |
| `npm run validate:data` | passes — 10 destinations · 48 areas · 42 loyalty hotels · 119 places |
| `npm run test:e2e` (custom Playwright suite) | **73/73 checks pass**, 0 console errors, 0 page errors, 0 failed requests |
| `npm run build` | succeeds — 17 static pages, all 10 destinations pre-rendered |
| Production smoke test in a real browser | 4 tabs, 20 hotel cards, 21 place cards, 0 broken images, 0 errors |

The e2e suite drives a real browser through the whole workflow and fails on console
errors, page errors and failed requests. Three of its assertions had to be rewritten
because they were testing the wrong thing:

- **`querySourceFeatures('route')` returned zero while the route was plainly painted on
  screen.** That API reads the tiles that happen to be loaded. The check now asserts on
  *rendered* features **and cross-checks the line against the panel**: a solid line must be
  backed by a named routing engine, and a dashed one must be a day where no leg had route
  data. If the map and the legs ever disagree, the test fails.
- **A step that blew its timeout did not stop** — its remaining assertions landed minutes
  later under the next step's heading, so a timed-out step could be reported as passing.
- **A single-frame sample of an animating map** turned a working route into a failing test.

---

## 7. Known limitations, in order of severity

1. **16 of 20 Bali hotels have no property photography.** The STAY tab is effectively a
   text list for 80% of properties. This is a genuine Wikimedia Commons / Openverse
   coverage limit, not a bug. Fixing it needs licensed photography — a commercial provider
   or the properties' own media kits.
2. **Existing photography is documentation-grade, not travel-editorial.** W Bali's images
   date from 2011; St. Regis has one usable image. The brief's target of two useful images
   per important hotel is met for 4 of 20.
3. **Traffic is not modelled, and in Bali traffic is the planning constraint.** A leg shown
   as ≈35 min can be 90 min at 17:00. This is the most likely reason a real user would
   still check Google Maps.
4. **No opening hours, closure days or ticket availability.** The efficiency engine cannot
   know that Uluwatu's Kecak dance sells out or that a temple closes at 17:00.
5. **Multimodal journeys are curated data for Bali only, with no timetables.** Ubud → Nusa
   Penida correctly renders as drive-to-Sanur-Harbour + fast boat, but the boat has no
   departure times and no live availability.
6. **Travel zones remain approximations.** Uluwatu's zone necessarily stretches ~9 km
   because three of its hotels sit at Ungasan. Honest, but coarse.
7. **Content depth.** 48 mapped places for a whole island; DO's Highlights category has 21.
   A user browsing Canggu restaurants reaches the end of the list quickly.
8. **31 MB of imagery with no responsive variants or CDN.** Lazy-loaded, so not on the
   critical path, but the correct answer is a real image pipeline.
9. **Only Bali is deep.** The other nine destinations are selectable and architecturally
   complete, not trustworthy.

---

## 8. Screenshots

All at `docs/screenshots/` on this same host:

| File | Shows |
|---|---|
| `20-explore.jpg` | EXPLORE — content-derived travel zones, whole-island framing |
| `21-area-detail.jpg` | An area in depth, with photography and credit |
| `22-stay.jpg` | STAY — hotels photographed-first, individual Marriott/Hilton markers |
| `23b-hotel-detail.jpg` | Hotel detail — every photo labelled BEACH / POOL / ROOM |
| `23-do.jpg` | DO — one category at a time |
| `24-plan-transport.jpg` | PLAN — transport legs with mode, duration, distance, source |
| `25-route.jpg` | Real road geometry between numbered stops |
| `26-mobile-explore.jpg` | Mobile EXPLORE |
| `27-mobile-stay.jpg` | Mobile STAY |

## 9. Further documents

| Path | Contents |
|---|---|
| `docs/PROCESS.md` | Full engineering build log, three iterations, every bug and its cause |
| `docs/ROADMAP.md` | What shipped, what is planned, and what is explicitly **not** planned |
| `docs/project.json` | Machine-readable project summary |
| `docs/README.md` | Architecture, data sources, verification method |

**Explicitly not planned:** fake live data, generated prices, invented scarcity signals,
a conventional OTA booking funnel, content marketing.

# Meridian — Map-first Southeast Asia Travel Planner

A working V1 of an interactive geographic decision-making tool for travellers departing from
**Singapore**. The map is the product: you pick where to go, understand where the hotels and
attractions actually are, and arrange them into a trip that makes geographic sense.

It is deliberately **not** a travel blog, an OTA, or a list-first booking site. There is no hero
banner, no price grid, and — importantly — **no invented pricing anywhere**.

```
Southeast Asia map  →  select a destination  →  destination map
        →  explore hotels / activities / areas  →  add to a day
        →  see the route  →  get told where the plan is inefficient
```

---

## 1. What is in this repository

| Area | What it does |
| --- | --- |
| `app/` | Next.js App Router pages: the region overview and the destination planner |
| `components/map/` | The map engine layer — the only code that knows MapLibre exists |
| `components/destination/cards/` | Area, hotel and place cards — the visual vocabulary |
| `lib/images/` | Image provider layer: resolution, disclosure and credit |
| `lib/transport/` | TransportLeg building, mode recommendation, water-crossing data |
| `components/region/` | Homepage: Southeast Asia overview, destination rail, preview card |
| `components/destination/` | Planner shell, map toolbar, detail card, panel sections |
| `components/planner/` | Trip builder: setup form, day tabs, itinerary rows, drag & drop |
| `lib/data/` | The whole dataset, typed and registry-driven |
| `lib/routing/` | Routing provider abstraction (OSRM + geodesic estimator) |
| `lib/providers/flights/` | Flight data provider abstraction (static + inert Amadeus/Skyscanner adapters) |
| `lib/efficiency.ts` | The V1 route-efficiency engine |
| `lib/store/` | Trip and UI state, persisted to `localStorage` |
| `lib/research/` | Guide analysis, place resolution, image storage, saved places, the social signal corpus |
| `components/social/` | The traveller's 导入小红书攻略 flow and 我的收藏 |
| `app/api/import/xiaohongshu/` | The retrieval boundary: refuses to scrape, and relays images for a permitted read |
| `app/api/analyze-guide/route.ts` | Multimodal analysis — the only place a vision key is read |
| `app/api/resolve-place/route.ts` | External place search — the only place a Places key is read, field-masked |
| `scripts/e2e.mjs` | Browser end-to-end test covering the whole shipped workflow |
| `scripts/test-social-import.mts` | Behaviour checks for the import pipeline, on controlled sample text |
| `scripts/shots-xhs.mjs` | The visual walkthrough of the import flow, desktop and mobile |

---

## 2. Running it

```bash
npm install
npm run dev          # http://localhost:3000
```

That is the entire setup. **No API keys are required** — see §5.

Other scripts:

```bash
npm run build        # production build
npm run start        # serve the production build
npm run typecheck    # tsc --noEmit
npm run test:e2e     # drive a real Chromium through the V1 workflow (dev server must be running)
npm run validate:data
```

### If `npm install` fails with `EPERM` on `~/.npm`

Some machines have a root-owned `~/.npm`. The repo ships an `.npmrc` that redirects the cache
into the project, but an environment variable can override it. If you hit this:

```bash
npm install --cache=./.npm-cache
```

---

## 3. Screens and flows

**`/` — Southeast Asia overview.** A full-bleed map with Singapore marked `SIN · Home`, the other
destinations as flag pins, and dashed arcs showing that every route is measured from one origin.
Selecting a marker opens a compact preview **without leaving the map**: route, block time, non-stop
status, recommended trip length, best-for tags, and how many Marriott Bonvoy / Hilton Honors
properties we hold for that destination. `[Explore Bali]` is the only way out of the map.

**`/destination/[id]` — the destination experience.** Four steps in the order a traveller actually
takes them, not a set of tools:

| Tab | Question it answers | What the map shows |
| --- | --- | --- |
| **EXPLORE** | Which part of Bali suits me? | Areas, labelled with their tagline (`ULUWATU / Cliffs · Sunsets`) |
| **STAY** | Which property? | Loyalty hotels, individual markers, M/H distinguishable |
| **DO** | What should I actually do? | Only the selected category — never every marker at once |
| **PLAN** | How does the trip fit together? | The active day's route, following real roads |

Desktop keeps the map dominant with a contextual right rail; mobile converts the rail into a
three-snap bottom sheet so the map is never pushed off screen.

Two behaviours matter more than the rest:

- **Each tab re-frames the map.** Switching to STAY moves the camera to where the hotels are;
  switching to DO frames the places in the selected category. Without this the camera stayed at
  whole-island scale where every hotel collapses into a cluster and the map says nothing.
- **The destination opens on EXPLORE, not on a form.** The previous build asked for dates before
  the traveller understood the island. Planning is now something you choose.

**导入小红书攻略 — paste a Xiaohongshu guide, get places on the map.** Reachable from the header of
every tab and from PLAN. It takes over the side panel rather than opening a modal, because the
promise is that the places appear *on the map* and a dialog would cover the answer.

**One source, on purpose.** V1 reads Xiaohongshu and nothing else. A link to another platform is
refused *by name* rather than silently accepted, and the platform union has a single value so a
second source cannot arrive by accident.

1. **Paste a link, the text, some screenshots — or all three.** The URL is kept as provenance. If
   the deployment has an approved retrieval provider it is used; otherwise the limitation is stated
   in words (`暂时无法直接读取这篇小红书…`) and the two things that still work are offered
   underneath. An images-only import is a legitimate import.
2. **Watch four honest stages** — 读取内容 / 识别地点 / 解析地图坐标 / 等你确认. No percentages, because
   there is no meaningful denominator.
3. **Confirm what it found.** Each candidate card shows the name *exactly as the post wrote it*, what
   type of place it is, whether it has a map location and how confident that is in words, and —
   crucially — **识别来源：正文 + 图片 3、4**. The guide's own sentence, dishes, keywords, warnings and
   times are labelled as the guide's words rather than as facts. An unmatched name is never guessed
   at: the traveller searches Meridian, or pins it on the map themselves.
4. **Assign the pictures.** Every image can be attached to a place by hand, from the card or from
   the **未分配图片** tray at the bottom. An image belongs to one place; moving it moves it. Tapping
   an unattributed image asks 这张图是什么地方？ and offers attach, create, or leave for later.
5. **Save, and see them.** Kept places land in **我的收藏**, a scope inside DO, and stay plotted on
   the map. The pictures the traveller attached travel with the saved place under their own heading
   (来自你的攻略) — never merged into the place's canonical photography. From there they add to an
   itinerary through the existing trip builder, and the existing routing handles the rest.

**`/research` — the internal view.** The same pipeline, with the reviewer's controls: every mention
with its match band in words, the alias table, saved places and pending submissions. It is labelled
as internal and is not reachable from the traveller's four tabs.

---

## 4. Architecture

### 4.1 Data-driven destinations

Adding a destination is a data change, not a UI change:

1. Add `lib/data/destinations/<id>.ts` exporting a `Destination`, an `Area[]`, a `Hotel[]` and a
   `Place[]`.
2. Register it in `lib/data/index.ts` (`DESTINATIONS`).

Everything else — the region marker, the preview card, the planner, the layers, the efficiency
engine — reads from the registry.

### 4.2 Map engine logic is separate from product logic

`MapCanvas` is the **only** component that touches MapLibre's lifecycle. Overlays
(`MarkersLayer`, `DestinationLayer`, `RouteLayer`, `AreaLayer`, `RegionArcLayer`,
`MapFocusController`) are separate components that consume a map instance through context and are
re-registered automatically if the style is ever replaced.

Everything map-related is reached through two `next/dynamic({ ssr: false })` entry points
(`RegionMapView`, `DestinationMapView`), so the engine never enters the server bundle.

`lib/map-markers.ts` converts domain data + the user's plan into a flat marker list. It contains no
React and no map library (its import is a type-only reference), so the rule *"what is on the map
right now"* is testable in isolation. That separation is why swapping the entire map engine could be
done without touching the data layer at all.

`lib/map-markers.ts` converts domain data + the user's plan into a flat marker list. It contains no
React and no Leaflet (its Leaflet reference is a type-only import), so the rule *"what is on the map
right now"* is testable in isolation.

### 4.3 State

Two Zustand stores, both persisted to `localStorage` with manual hydration so server and first
client render agree:

- `lib/store/trip-store.ts` — trips, days, itinerary items. The public surface (`list / create /
  update / delete` + active selection) is deliberately the same shape a server-backed store would
  expose, so moving to a real backend means replacing the `persist` middleware rather than
  rewriting components.
- `lib/store/ui-store.ts` — layer visibility, selection, hover, camera requests. Only preferences
  (`visibleLayers`, `showAreas`, `showRoute`, basemap) are persisted; selection is session state.
- `lib/store/import-ui.ts` — where the import flow is and what the map should draw while it is
  there. Deliberately **not** persisted: a half-finished import that reappeared after a refresh
  would read as data loss rather than as an unsubmitted form.

`lib/research/store.ts` is a third persisted store (`meridian.social.v1`) holding imports, mentions,
saved places, submissions and the learned alias table. Saved places are **references** to canonical
ids, not copies, so a card and its marker can never disagree; a place the traveller creates is a
`UserPlaceSubmission` in `pending_verification` and never enters the canonical dataset.

### 4.4 Map ↔ itinerary synchronisation

The two views never talk to each other directly; they share one store.

| Action | Consequence |
| --- | --- |
| Select a day | Active day's stops get numbered markers; everything else dims; the route redraws |
| Hover an itinerary row | The matching marker scales up (`emphasised`) |
| Click a marker | Detail card opens **and** the matching itinerary item is selected, if it is already planned |
| Click an itinerary row | The map flies to that stop |
| Apply an efficiency suggestion | The item moves between days and both views re-render |

### 4.5 Routing: distance and travel time are separate abstractions

`lib/types.ts` defines `RoutingProvider`. Two implementations ship:

- **OSRM** (default) — real road geometry, road distance and drive duration.
- **Geodesic estimator** — great-circle distance plus a documented speed model, used automatically
  when the routing endpoint is unreachable or unset.

`lib/use-day-route.ts` always returns something renderable: while loading, and on failure, it falls
back to a **dashed straight-line corridor** and the UI says so. The route cache is keyed by the
point sequence, so panning and zooming never re-hit the network.

Adding Google Routes or Mapbox Directions later means implementing one interface.

### 4.6 Route efficiency (V1)

`lib/efficiency.ts` is geometry-only — no AI, no learned priors, and no invented "minutes saved".
Per day it derives legs, bounding-box spread and cross-area hops, then emits suggestions:

- a single transfer longer than ~28 km (road estimate),
- a day spanning ~42 km or more ("opposite parts of Bali"),
- backtracking, detected with a detour ratio,
- a hotel that sits far from the day's activity centroid,
- a day whose activities plus travel exceed ~11 hours,
- an actionable *"consider moving X to Day N"*, where N is the day whose planned geography already
  fits that stop best.

Every message quotes the number it is based on. When a real routing provider supplies legs for the
active day, the same analysis upgrades to road distances with no change to its contract.

### 4.7 Flight data

`lib/providers/flights/` defines a `FlightDataProvider` interface with a working static provider and
two **inert** adapters (Amadeus, Skyscanner-compatible). They report `isConfigured() === false`
without credentials, and the UI degrades to *"Curated route data"* rather than failing.

**No fares are shown anywhere.** V1 has no pricing source and will not invent one.

---

## 5. Environment variables and map setup

Copy `.env.example` to `.env.local`. **Everything is optional** — the app runs with zero
configuration.

### Basemap

The default is **OpenStreetMap raster tiles**, chosen deliberately:

- CARTO's keyless basemaps now return a watermarked `API KEY REQUIRED` tile.
- Stadia, MapTiler and Mapbox all reject keyless requests.

OpenStreetMap renders correctly without a key. A light CSS filter
(`.leaflet-tile-pane.tiles-muted`) desaturates it just enough to keep markers legible.

Set a key for any provider below and it is promoted automatically. A provider chain
(CARTO → CARTO Voyager → OpenStreetMap) means a blocked CDN degrades instead of showing a grey
rectangle.

| Variable | Provider |
| --- | --- |
| `NEXT_PUBLIC_CARTO_KEY` | CARTO Positron / Voyager |
| `NEXT_PUBLIC_MAPTILER_KEY` | MapTiler Streets |
| `NEXT_PUBLIC_STADIA_KEY` | Stadia Alidade Smooth |
| `NEXT_PUBLIC_MAPBOX_TOKEN` | Mapbox Light |
| `NEXT_PUBLIC_MAP_PROVIDER` | Force one: `osm`, `carto-positron`, `carto-voyager`, `maptiler`, `stadia`, `mapbox` |

> OpenStreetMap's tile usage policy applies. For production traffic, set a key for one of the
> commercial providers.

### Routing

| Variable | Default |
| --- | --- |
| `NEXT_PUBLIC_ROUTING_PROVIDER` | `osrm` |
| `NEXT_PUBLIC_OSRM_BASE_URL` | `https://router.project-osrm.org` |

The public OSRM demo server is fine for development; point this at your own instance for production.
Set `NEXT_PUBLIC_ROUTING_PROVIDER=geodesic` to disable road routing entirely.

### Guide extraction

| Variable | Default | Notes |
| --- | --- | --- |
| `NEXT_PUBLIC_GUIDE_ANALYZER` | `heuristic` | `multimodal` to read images as well as text |
| `VISION_API_KEY` | — | **Server only.** Read in `app/api/analyze-guide/route.ts` only |
| `VISION_API_BASE_URL` | — | An OpenAI-compatible endpoint (DashScope/Qwen-VL, and others) |
| `VISION_MODEL` | `qwen-vl-max` / `gpt-4o-mini` / Claude Haiku | Optional model override |
| `ANTHROPIC_API_KEY` | — | **Server only.** Used when no other vision key is present |
| `NEXT_PUBLIC_XHS_RETRIEVAL` | `0` | Whether to *attempt* reading a Xiaohongshu link |
| `XIAOHONGSHU_RETRIEVAL_ENDPOINT` | — | **Server only.** A retrieval provider you have an agreement with |
| `NEXT_PUBLIC_PLACE_PROVIDER` | `none` | `google` to enable external place resolution |
| `GOOGLE_PLACES_API_KEY` | — | **Server only**, and field-masked |

Every one of these defaults to OFF. A static deployment cannot reach any of the routes, and a request
guaranteed to 404 is noise rather than information — so the traveller is told what to do instead. The
`NEXT_PUBLIC_*` names are public-safe by design: they select a provider, they carry no secret. No
browser-side module reads an API key, and a test asserts that by reading the sources.

Never commit `.env.local`. `.gitignore` already excludes it.

---

## 6. Data structure

```ts
Destination {
  id, name, country, countryCode, flag, region, status,
  coordinates, mapView, mapBounds, recommendedDays,
  tags, bestFor, currency, timezone, language, visaNote, originNotes,
  airports: Airport[], areas: Area[], provenance
}

Airport { id, code, name, city, coordinates, role,
          directFromSingapore, flightMinutes, airlines, transfers[] }

Area    { id, destinationId, name, coordinates, isStayBase, zoneType, radiusMeters,
          bestFor[], weakFor[], scores{beach,nightlife,food,luxury,nature,accessibility},
          vibe, summary, idealFor[], priceTier }

Hotel   { id, name, destinationId, areaId, hotelGroup, brand, brandId, coordinates,
          priceTier, priceTierBasis, propertyType, tags[], beachAccess, beachAccessNote,
          airportTransfer, description, loyaltyProgramme, officialUrl?, loyaltyMeta? }

Place   { id, name, destinationId, areaId, category, subcategory, coordinates,
          recommendedDurationMin, bestTime, tags[], description, notes?,
          entryFee?, openingHours?, markerLayer }

Trip    { id, name, destinationId, arrivalDate, departureDate, travellers,
          styles[], budget?, loyalty[], days: TripDay[] }

TripDay { id, index, date, items: ItineraryItem[], note? }

ItineraryItem { id, refId, kind, name, areaId?, lat, lng, durationMin?, note?, confidence }
```

`loyaltyMeta` exists on `Hotel` and is intentionally **empty**. Elite benefits change constantly and
must never be hard-coded; the field is there so a benefits comparison can be added later without a
migration. The same applies to `Trip.loyalty`: the tier is stored as data, and V1 deliberately
claims nothing about what it gets you.

### Data confidence

Every coordinate carries a `DataConfidence`: `verified`, `approximate` or `demo`, plus a free-text
`coordNote` describing what the point actually marks. The UI renders a badge for it on every detail
card. A planner that quietly blends surveyed and estimated positions is worse than no planner.

### Coverage today

| Destination | Status | Areas | Marriott | Hilton | Places |
| --- | --- | --- | --- | --- | --- |
| Bali | reference | 15 | 16 | 5 | 48 |
| Phu Quoc | starter | 4 | 2 | 1 | 8 |
| Da Nang / Hoi An | starter | 4 | 2 | 2 | 8 |
| Ho Chi Minh City | starter | 4 | 3 | 1 | 8 |
| Hanoi | starter | 4 | 2 | 2 | 8 |
| Siem Reap | starter | 3 | 1 | 1 | 8 |
| Phnom Penh | starter | 3 | 1 | 0 | 8 |
| Cebu | starter | 4 | 2 | 0 | 8 |
| Boracay | starter | 3 | 1 | 0 | 7 |
| Palawan | starter | 4 | 1 | 0 | 8 |

**48 areas · 43 loyalty hotels · 119 places · 5 of 119 places marked approximate.**
A zero in the loyalty column is a real finding, not a gap: neither Marriott Bonvoy nor Hilton
Honors has a property we could verify in Phnom Penh, Cebu, Boracay or Palawan. The UI says so rather
than padding the list.

### Adding a destination

```ts
// lib/data/destinations/phuket.ts
export const phuketAreas: Area[] = [ /* … */ ];
export const phuketHotels: Hotel[] = [ /* … */ ];
export const phuketPlaces: Place[] = [ /* … */ ];
export const phuket: Destination = { /* … */ };
```

```ts
// lib/data/index.ts
import { phuket, phuketAreas, phuketHotels, phuketPlaces } from './destinations/phuket';
export const DESTINATIONS = [bali, phuket, ...starterDestinations];
```

The region map, preview card, planner, layer filters, hotel-group filters and efficiency engine all
pick it up automatically. Nothing else changes.

### Adding a hotel

Append to that destination's `Hotel[]`. Set `brandId` to an id from `lib/data/hotel-brands.ts`;
`priceTierBasis` must explain where the tier came from, and it must never be a nightly rate.

### Adding an activity

Append to that destination's `Place[]`. `markerLayer` decides which layer it belongs to
(`activity`, `nature`, `beach`, `food`, `nightlife`, `transport`), which in turn decides its icon,
its shape, its filter chip and its legend entry.

### Adding a whole new marker layer

Add an entry to `LAYERS` in `lib/layers.ts`. The legend, filter toolbar, marker palette and
accessibility labels all read from that list.

---

## 7. Data: what is real and what is not

| Dataset | Status |
| --- | --- |
| **Bali geography** (9 stay areas + 6 excursion zones, airport, 48 places) | Coordinates verified against a bulk Wikidata bounding-box query, the Wikipedia coordinates API and OpenStreetMap element ids. **44 of 48 places resolve to a specific point**; Tegallalang, Jatiluwih, Jemeluk beach and Sunset Road are marked `approximate` with a note saying what the point marks instead. |
| **Bali hotels** (20 properties: 15 Marriott, 5 Hilton) | Every coordinate resolves to the named property in OpenStreetMap, cross-checked against brand-published map links and sourced infobox coordinates. |
| **Other 9 destinations** | Same verification method, but a starter-scope dataset: 2–4 areas, 1–4 loyalty hotels and 7–8 places each. |
| **Airport transfer times (Bali)** | Curated road-time ranges that absorb traffic variance. Not live traffic. |
| **Airport transfer times (other destinations)** | Computed at runtime by the routing provider, so a real road network produces them rather than a number baked into a data file. |
| **Flight durations and carriers** | **Curated sample data**, labelled as such in the UI. Not live schedules. |
| **Price tiers** | Derived from brand positioning and market segment. **Never a nightly rate.** |
| **Room counts** | Only where a citable figure exists; omitted otherwise rather than invented. |
| **Entry fees / opening hours** | Indicative unless a source is named. Every fee string says so. |
| **Bar and beach-club operation** | Not guaranteed. Those notes say *"verify current operation"*, because venues in Bali change hands frequently. |
| **Everything else** | There is no live data anywhere in V1. |

### Corrections the verification pass caught

These are the reason the data layer went through a checking step rather than being written from
memory:

- **Bali has only five Hilton-branded hotels.** There is no DoubleTree, Curio, Tapestry, Canopy or
  Waldorf Astoria trading in Bali (Waldorf Astoria Nusa Dua is announced for 2027). An early draft
  of this dataset contained a DoubleTree that does not exist. It was removed.
- **Siem Reap's airport code is `SAI`, not `REP`** — REP was the old in-town airport, closed in
  October 2023.
- **Phnom Penh moved to Techo International (`KTI`)** in September 2025; the old `PNH` field no
  longer serves commercial traffic.
- **El Nido's IATA code is `ENI`**; "LIO" is only the local airstrip name.
- **Le Méridien Angkor is closed.** Marriott's marketing pages are still live, which is why it is
  widely listed as open. It is excluded here.
- **The former "Cebu City Marriott Hotel" closed in 2018.** Excluded.
- **Four Points by Sheraton Palawan is in Sabang**, about two hours from Puerto Princesa city.

---

## 8. Testing

`npm run test:e2e` drives a real Chromium through the full V1 workflow and asserts on the DOM. It
fails on console errors, uncaught page errors and failed requests, and writes screenshots plus
`test-artifacts/report.json`.

It requires a browser. The script defaults to a Playwright Chromium cache path and can be
overridden with `CHROME_PATH`. Install one with `npx playwright install chromium` if needed.

Coverage: homepage → region map → origin selection → destination selection → preview card →
planner → layer toggles → Marriott/Hilton filtering → price tiers → marker → detail card → trip date
generation → add to itinerary → reorder → move between days → day/map emphasis sync → route line →
efficiency panel → transport panel → refresh persistence → mobile layout → Chinese UI → the internal
research view → **importing a Xiaohongshu guide with real screenshots, seeing its places plotted on
the map, assigning pictures to them by hand, and keeping them** → the unreadable-link fallback →
**all nine other destinations load without crashing** → zero console errors.

`npm run test:social` is the behavioural suite for guide import: 224 checks over Chinese, English and
mixed prose, duplicate and partial names, alias learning, the resolution pipeline and its cache, the
external-provider branch, text/image merging, image assignment and reassignment, the unassigned tray,
saving with image references, pinning, creating from an image, the private-by-default boundary, the
cost-control arithmetic, deletion, rate limits, every failure code, and the retrieval route's refusal
semantics. It also asserts that no browser-side module reads a secret, by reading the sources. It runs
entirely against controlled sample text — never a live platform, because the product does not fetch
them.

`scripts/shots-xhs.mjs` drives the same flow in a real browser at desktop and mobile widths and
writes screenshots to `test-artifacts/xhs/`.

`npm run validate:data` runs the data-integrity checks (schema, coordinate bounds, duplicate ids,
NaN radii, brand-registry coverage, unknown area references, and a "no price in a description"
assertion) and prints a coverage table. Every check in it exists because the corresponding failure
actually happened during development — a `NaN` area radius crashed Leaflet and blanked the planner.

Notes on the harness: a warm-up phase compiles both routes before asserting, because `next dev`
compiles on first request and can otherwise take longer than any assertion should. Each step has a
hard timeout so one hung action cannot swallow the run.

---

## 9. Known limitations

1. **No prices, at all.** No fares, no nightly rates, no availability. This is a deliberate product
   decision, not an oversight.
2. **No backend and no accounts.** Trips live in `localStorage` on one browser.
3. **Only Bali is deep.** The other nine destinations have enough data to be selectable and to
   demonstrate the architecture; they are not yet good enough to plan a real trip from.
4. **Traffic is not modelled.** Routing gives a normal-traffic drive time.
5. **Efficiency analysis is geometric.** It cannot know that a temple closes at 17:00, or that the
   road to Kintamani washes out.
6. **Area shapes are approximations and are drawn as such.** Each travel zone is a convex hull of
   that area's own mapped hotels and places, padded and smoothed, with a fine dashed edge. A hull
   keeps the shape of the content — Canggu is a coastal strip, Uluwatu is a cliff line — where a
   radius circle said nothing. These are still not official boundaries, and the UI says so.
7. **No clustering of the *trip* itself** — a 21-day trip with 60 stops is legal and will render.
8. **Single user, single trip at a time** in the planner's active view (multiple trips are stored
   and switchable, but there is no comparison view).
9. **Photography is thin, and honest about it.** 141 images across 63 subjects, restricted to
   commercial-use licences and fully attributed. A hotel photo is used only if it is provably of
   that property; **4 of 20 Bali hotels clear that bar** and the other sixteen say "No property
   photography available" rather than borrowing their area's beach photograph. This is a genuine
   Commons and Openverse coverage limit — fixing it needs licensed photography.
10. **The public OSRM demo server** is rate-limited and not for production.
11. **Guide import reads text, not images.** The extractor handles prose the traveller pastes. It
    cannot read a screenshot, which is how most Xiaohongshu guides actually arrive, and it does not
    fetch a video transcript. It is also a rule-based extractor by default: it finds names it
    recognises and names next to a category keyword, and it does not resolve pronouns or follow a
    link inside the guide. A name it has never seen is reported as unmatched rather than guessed at,
    and the traveller resolves it by searching or creating the place.
12. **Imported guides are private.** There is no shared corpus yet, so 社区攻略 counts are
    legitimately zero and every signal reads 你的攻略. Imported places are per-browser, like trips.
13. **Xiaohongshu retrieval is not enabled anywhere.** The route, the refusal semantics and the image
    relay all exist and are tested; what is missing is a provider Meridian is permitted to read
    through. Until one exists, the product's answer to a pasted link is the honest fallback: paste
    the text or upload the screenshots. Nothing is scraped, ever — no login, no CAPTCHA solving, no
    anti-bot evasion, no headless browser, and a 403 ends the attempt.
14. **Image analysis needs a server.** On GitHub Pages there is no route to call, so the built-in
    analyzer reads text and the traveller assigns the pictures by hand. That is a complete workflow,
    and it is not §5's multimodal reading. The prompt and the batching are implemented; they need a
    deployment with a key.
15. **One source.** TikTok, Instagram, Douyin, YouTube and generic blog import are out of scope. A
    link to another platform is refused by name rather than silently accepted.
16. **Image analysis quality is unmeasured.** Nobody has benchmarked a given vision model against a
    Xiaohongshu menu photograph, and `IMAGE_PROPOSAL_FLOOR` is a judgement rather than a measurement.
    The unassigned tray exists precisely because that number is not trustworthy on its own.

---

## Hosted demo

The product is live, statically, on GitHub Pages:

**https://suiwenke69-cyber.github.io/**

`npm run deploy:pages` builds a fully static export and publishes it. Two things
about that build are deliberate:

- **The build is a script, not a config flag.** `output: 'export'` cannot emit a Route
  Handler, and this app has one: `app/api/route/route.ts`, the server-side proxy that
  keeps keyed routing credentials out of the browser. `scripts/build-static.mjs` moves
  it aside for the build and restores it in a `finally`, so an interrupted build cannot
  lose it. The deployed site is unaffected — with no key configured that proxy answers
  501 and the client falls back to keyless OSRM, which is what the static build uses.
- **It is published to the account's user site, not a project page.** A project page is
  served from `/<repo>/`, which would force a `basePath` prefix onto every absolute
  asset path — the MapLibre worker, the photography. The user site is served from the
  domain root, so nothing has to be rewritten and there is no class of bug where a path
  works locally and 404s in production.

`.nojekyll` is written at the root of the published site and is not optional: GitHub
Pages runs Jekyll by default, and Jekyll silently drops directories beginning with an
underscore — which is where Next.js puts every chunk and stylesheet. Without it the
site serves a blank page.

For an AI reader, note that `*.trycloudflare.com` quick tunnels return **403 to
GPTBot** — Cloudflare injects its own robots.txt on that domain. The GitHub Pages
domain has no such restriction.


---

## Where you are leaving from

Meridian is **not a Singapore product**. The origin is a first-class entity, and Singapore is
one of eleven supported departure cities.

| Group | Cities |
|---|---|
| 新加坡 | Singapore |
| 粤港澳大湾区 | Guangzhou, Shenzhen, Hong Kong |
| 长三角 | Shanghai, Hangzhou |
| 中国其他 | Beijing, Chengdu |
| 东南亚 | Bangkok, Kuala Lumpur, Jakarta |

**One city is not one airport.** Shanghai has PVG and SHA, Beijing PEK and PKX, Chengdu CTU and
TFU, Bangkok BKK and DMK, Singapore SIN and XSP. The model stores a list from the start, and every
airport coordinate was resolved against OpenStreetMap.

Selecting an origin reorients the product with no destination-specific code: the map marker, the
viewport, the single route arc, every duration in the destination rail, the 直飞 filter, and the
destination preview card.

### Unknown is not "no"

`OriginDestinationConnection.directAvailable` is `boolean | null`, and **`null` means we have no
data**. Those pairs render 航班信息待确认 and show **no duration at all** — only a straight-line
distance, explicitly labelled as a straight line. Nothing is inferred from distance, from hub size,
or from the fact that another origin has the route.

Three confidence levels, treated differently in the interface:

- **verified (10)** — Singapore's connections, harvested from the destination data files where each
  carries a cited source.
- **approximate (64)** — long-standing routes for those hubs, marked 待确认 with a curation date.
- **unknown (36)** — no record. Jakarta has two of ten; that is the honest result.

China is an **origin market only**. There are no Chinese destinations, deliberately.


---

## Language

**Simplified Chinese is the primary product language.** English is available and the
product is fully usable in it.

The localization is an architecture, not a find-and-replace. `lib/i18n/messages.ts` holds
793 keys; `zhCN` is authored first and `en` is typed as `Record<keyof typeof zhCN, string>`,
which makes a missing or misspelled translation a **compile error** rather than a raw key
in the interface.

Three decisions worth knowing:

- **The Chinese is written, not translated.** It is a different text serving the same
  reader: shorter, direct, no marketing register. The English long-form copy is still
  there for the `en` locale.
- **Proper nouns are stored twice and both are shown.** A traveller reads 乌鲁瓦图神庙 and
  then needs to type "Uluwatu Temple" into Grab. Cards show both, and every restaurant
  keeps its Latin name because that is what the map apps recognise. `nameZh` is omitted
  where no Chinese name is genuinely in use — 29 of 145 Bali places carry one.
- **Chinese copy lives in an overlay** (`lib/data/zh/bali-zh.ts`), merged by the registry.
  The hand-verified geography files — coordinates, sources, notes — are never touched by a
  translation pass.

The locale lives in the persisted UI store rather than a cookie, because the site is
statically exported and must not read request state.

## Restaurants, activities and the research pipeline

Bali went from **48 places to 145**: 46 restaurants, cafés, bars and beach clubs across
Seminyak, Canggu, Ubud, Uluwatu, Nusa Dua, Sanur and Jimbaran, and 51 bookable activities
covering all 17 activity kinds.

DO is a filter rather than a list: eleven Chinese categories over an area row that only
offers areas holding something in the chosen category. 美食 + 长谷 narrows 49 restaurants
to 7.

`/research` is an internal route for turning social travel guides into verifiable places.
**It is not a scraper.** Xiaohongshu, Douyin, TikTok and Instagram prohibit automated
collection, so the researcher supplies the URL for provenance and pastes the text.
Extraction runs in three passes — dictionary, explicit patterns, then a constrained
heuristic — and matching refuses to guess: below 0.55 confidence a mention is marked
需要确认 and a human decides.

Nothing reaches a traveller until it has been accepted, and what reaches them is an
aggregate over accepted mentions — 在 N 份已收录攻略中被提及 — never a popularity claim.
A fresh install shows no signals at all, because there is no imported research to show.

### How analysis is wired

`lib/research/analyzer.ts` defines one `GuideAnalyzer` interface and two implementations:

| Analyzer | Runs where | Needs a key | Behaviour |
| --- | --- | --- | --- |
| `heuristic` (default) | The browser | No | Reads the text with the rule-based extractor; cannot see pictures, and says so |
| `multimodal` | Server, via `/api/analyze-guide` | Yes | Reads text AND downscaled images in batches of four, returning findings with provenance |

Both return the same `GuideAnalyzerResult`, so the merge step, the resolution pipeline and the review
screen cannot tell which produced their input: a deployment that gains a key gains image
understanding without any other code changing.

```bash
# Analysis
NEXT_PUBLIC_GUIDE_ANALYZER=multimodal
VISION_API_KEY=...            # OpenAI-compatible endpoint, OpenAI, or Anthropic
VISION_API_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
VISION_MODEL=qwen-vl-max
```

### How place resolution is wired

`lib/research/place-resolver.ts` runs the pipeline in a fixed and non-negotiable order:

1. **Meridian's canonical dataset** — a place we curated, photographed and verified.
2. **The alias table** — a name this profile already confirmed.
3. **An external place search** — somebody else's map knows it.
4. **The traveller** — they point at the map, or create it.

Each step is strictly more expensive and less trustworthy than the one before, which is what makes
the order correct rather than merely convenient. A hit at step 3 is offered as a QUESTION and never
written onto the map: confirming it saves a place the traveller owns, pending review, because "a map
search found something with this name" and "Meridian knows this place" are different claims.

```bash
NEXT_PUBLIC_PLACE_PROVIDER=google   # OFF by default: every call costs money
GOOGLE_PLACES_API_KEY=...           # server-only
```

The field mask is a literal in the route — `places.id, places.displayName,
places.formattedAddress, places.location, places.primaryType` — so the expensive SKUs (ratings,
photos, opening hours, reviews, price level) can never be requested, by a client or by accident. A
confirmed hit is cached for 90 days, and confirming one records an alias, so the steady state is
zero external calls.

### Where nothing is modelled

**A language model never supplies a coordinate.** The system prompt says so explicitly, and the
pipeline has no parameter through which one could arrive. A model that has never seen Bali cannot
know where a beach club is, and a plausible-looking wrong latitude would silently corrupt an
itinerary with a stop that does not exist. Positions come from Meridian's data, an external place
provider, or the traveller's finger.

On the static GitHub Pages build none of these routes are deployed and neither switch is set, so the
feature runs the heuristic analyzer, resolves against Meridian's own data, and relies on manual
image assignment. That is a complete workflow rather than a degraded one — and a key in the
environment never silently becomes a key in a JavaScript bundle.

Matching compares a **loose** key (noise words stripped) and a **strict** key (punctuation and
accents only), because stripping noise destroys names like `Uluwatu Temple`. Two candidates within
0.05 of each other are treated as ambiguous and not matched at all.


---

## 10. Recommended next five improvements

1. **Take Bali to full depth on the other nine destinations** — or trim the destination list to the
   four or five that can be done properly. A shallow destination undermines the tool's promise more
   than a missing one does.
2. **Live pricing behind the existing provider interfaces.** The `FlightDataProvider` and pricing
   seams already exist. Ship it as an explicit, attributed *live* layer with a visible timestamp,
   never mixed into the curated data.
3. **Real area polygons** from OpenStreetMap administrative boundaries, so "where should I stay?"
   is answered with actual geography rather than a radius.
4. **Shareable, read-only itineraries.** A trip is already a self-contained JSON object; a signed
   URL or a tiny serverless store would make it collaborative, which is what actually happens when
   people plan a trip together.
5. **Weather and seasonality overlays.** The map already answers *where*; the next most useful
   question for a Singapore-based traveller is *when*. Surf season, monsoon timing and the
   Ramadan/high-season calendar all change the same decisions this product already models.

See `ROADMAP.md` for the full V1 / V1.1 / V2 breakdown.

# Roadmap

The north star: an interactive geographic decision-making tool that helps Singapore-based
travellers decide **where to go, where to stay, what to do, and how to arrange those places into an
efficient trip**. The map is the product; the itinerary is built around the map.

Anything on this roadmap is judged against one question: *does this help the traveller understand
where things are?*

---

## V1 — shipped

**The full loop works end to end, with no live data and no invented numbers.**

### Map foundation
- **MapLibre GL + vector tiles, no API key and no cost.** The basemap style is authored in
  `components/map/basemap-style.ts` over CARTO's keyless vector tiles, with OpenFreeMap as an
  automatic fallback. Raster tiles bake their labels into pixels; vector tiles let us cut the label
  set down to country → region → city → town, in English, in our own palette.
- Map engine isolated from product logic: one component owns MapLibre, overlays are independent, and
  the engine never reaches the server bundle. Swapping engines did not touch the data layer.
- Dependency-free grid clustering in projected pixel space, stable across panning.
- Distinct marker system: shape **and** glyph **and** label, never colour alone. Marriott = rounded
  square with an `M`, Hilton = circle with an `H`, airport = ringed plane, plus one silhouette per
  place category.
- Homepage destinations drawn as **native map layers** (dot + label, three states) rather than
  floating pins, so they collide and fade like the cartography around them.

### Southeast Asia overview
- Singapore as a first-class `Home / Origin` with a bespoke accent star. It is the only DOM marker
  on the page.
- 10 destinations across Indonesia, Vietnam, Cambodia and the Philippines, framed by bounds derived
  from the data. No arc is drawn until a destination is chosen, and then exactly one.
- Destination preview in place: flight time, non-stop status, ideal stay, what it is good for, and
  the loyalty inventory. The map is never left behind.
- Destination rail that reads like a travel guide rather than a database: *Bali / Indonesia ·
  4–7 days / ≈2h 45m / Direct*.

### Destination experience (redesigned)
- **EXPLORE / STAY / DO / PLAN**, in the order a traveller actually takes them. The destination
  opens on EXPLORE, not on a trip form.
- Each tab re-frames the map for its own question, so the camera is part of the navigation.
- Six headline regions with authored taglines, photography and structured best-for / less-ideal-for
  metadata, labelled directly on the map.
- Marriott and Hilton hotels as visual cards; place cards with photography and one category at a
  time so the map never becomes marker soup.
- Desktop map-first with a contextual right rail; mobile converts the rail into a three-snap sheet.

### Photography
- Provider-based image architecture (`lib/images/`) with licence, author and a `subject` field that
  records what a photo actually depicts.
- 80 images resolved from Wikimedia Commons with credit lines; three hotels have genuine property
  photos and the rest borrow their area image, labelled as such.
- Designed fallbacks for no image, a failed image and a representative image.

### Transport
- `TransportLeg` as a first-class itinerary item with mode, rationale, alternatives, distance,
  duration, geometry, source and confidence.
- `RoutingProvider` abstraction: OSRM by default, with server-only OpenRouteService / Mapbox /
  Google adapters behind `/api/route`.
- Recommendation kept separate from routing; water crossings modelled as data for future
  multimodal support.
- Nothing fabricated: an unanswered route renders as "Route unavailable", never as an estimate.
- Layers: Marriott, Hilton, Activities, Nature, Beaches, Food, Nightlife, Airport, Transport —
  each with a live count.
- Filters: hotel group, price tier, travel style, place category, free-text search. The legend and
  the marker set are produced by the same function, so they cannot disagree.
- Where-to-stay panel with per-area scores, best-for / weak-for, loyalty inventory and price tier.
- Area shapes drawn as dashed "approximate extent" circles, never invented boundaries.

### Trip builder
- Dates generate days; travellers, travel styles, budget tier and loyalty programmes are captured.
- Add hotels, activities, nature, beaches, restaurants and transport points to a specific day.
- Reorder within a day (drag **and** accessible move up/down) and move between days.
- Changing dates preserves as much of the plan as possible and warns when something no longer fits.
- Trips persist across refreshes in `localStorage`.

### Map ↔ itinerary synchronisation
- Selecting a day renumbers the map and redraws the route.
- Hovering a row emphasises its marker; clicking a marker selects its row.
- Waterfall: airport → hotel → activity → activity → restaurant → hotel.

### Routing and efficiency
- Routing provider abstraction with **real OSRM road geometry** as the default and a documented
  great-circle estimator as the fallback. Distance and travel time are separate abstractions and are
  never conflated.
- Efficiency engine that flags long transfers, spread-out days, backtracking, hotel/activity
  mismatch and over-packed days, and proposes concrete relocations with a one-click apply.

### Honesty layer
- Every coordinate carries `verified` / `approximate` / `demo` plus a note about what it marks.
- No prices anywhere. Hotel cost is a brand-positioning tier with its basis shown.
- Flight data is labelled sample data. Provider adapters for live data exist but are inert.
- Empty states that explain themselves (including "neither loyalty programme has a property here").

---

## V1.1 — depth over breadth

The goal is to make the reference destination genuinely trustworthy and the others honest.

- **Deep data for the remaining destinations, or a shorter list.** Take Phu Quoc, Da Nang / Hoi An
  and Siem Reap to Bali's depth; drop or clearly badge anything that cannot be.
- **Real area boundary polygons** from OpenStreetMap administrative data, replacing the dashed
  radius circles.
- **Curated photography**, licensed and attributed, used behind the map rather than instead of it.
- **Opening hours and closure days on the timeline.** A day that ends at Uluwatu after the Kecak
  dance has sold out is an inefficient plan too.
- **Weather and seasonality overlays** — surf season, monsoon timing, and the holiday calendar that
  drives Balinese traffic.
- **Trip comparison.** Two destinations, two area choices, or two hotel bases side by side on
  distance and travel time.
- **Shareable read-only itinerary** via a signed URL. A trip is already a self-contained JSON object.
- **Multi-trip management UI** (list, rename, duplicate, archive) instead of only switching.
- **Automated data validation in CI**: schema, coordinate bounds, duplicate ids, brand-registry
  coverage, and a "no price-like field" assertion.
- **Accessibility audit** with a screen reader and keyboard-only pass; a written audit log.
- **PWA shell** with offline map tiles for the downloaded destination.

---

## V1.2 — social guide import

**Paste a travel guide and its places appear on the map.**

- **The traveller pastes; Meridian never scrapes.** The link is kept as provenance and the text is
  what the traveller supplies. No login, no CAPTCHA solving, no anti-bot evasion, and the interface
  says so where the traveller would otherwise expect a fetch. A screenshot is still the format most
  of these guides arrive in, and reading one is not something this iteration attempts.
- **One place, however many spellings.** `La Brisa`, `La Brisa Bali` and `La Brisa Canggu` resolve
  to one canonical record, so the guide never becomes a second place database. Confirming a name
  teaches an alias, and the next import matches it without asking.
- **A confidence band, never a number.** High preselects, medium asks, low does not guess. The
  traveller is told what we think a name means, not how sure a model claims to be.
- **The guide's words are labelled as the guide's.** Themes, dishes, warnings and times are shown as
  source-derived, never as verified attributes of the place.
- **Creating a place writes a private submission**, pending review, and never touches the canonical
  dataset.
- **The review is on the map.** Candidates are plotted as they are reviewed, and what is kept stays
  plotted. The map is the answer, not a backdrop.
- **Extraction sits behind a provider interface.** A deterministic rule-based extractor runs in the
  browser by default; an LLM extractor calls `/api/extract`, which is the only place an API key
  exists. A static deployment degrades to the deterministic extractor rather than breaking.

Still open from this iteration:

- **Screenshot and image import.** Most Xiaohongshu guides arrive as images. Reading them needs OCR
  or a vision model, and it needs the same "the text came from you" contract.
- **Video transcripts.** A YouTube or TikTok link could yield a transcript the traveller is entitled
  to read; that is a fetch with a declared purpose, not a scrape, and it needs designing.
- **Community corpus.** Imported guides are private. A reviewed, aggregated contribution flow is the
  only honest path to "N guides mention this", and the split between 你的攻略 and 社区攻略 is already
  in the data shape for it.
- **Model-assisted matching for unmatched names** with the traveller confirming, rather than the
  current exact/fuzzy matcher.

---

## V1.3 — Xiaohongshu import, properly

**The product narrowed to one source so it could get one workflow right.**

The previous pass accepted nine platforms and read only pasted text. That breadth
made everything worse: extraction, layout and copy all hedged for sources nobody
had tested, and the source most of this product's users actually use stayed a
paste box. V1 now reads **Xiaohongshu only** — link, text and images — and the
one-value `SocialPlatform` union means adding a second source is a deliberate
change rather than something a union already permitted.

- **Text AND pictures.** A place named in the caption and visible on a shopfront
  is one place with two pieces of evidence, not two places (§24). All fourteen
  travel place types are detected, from 酒店 to 交通节点.
- **Real place resolution, in order.** Meridian's own dataset, then the alias
  table this profile has confirmed, then an external place search, then the
  traveller pointing at the map. A model never supplies a coordinate — a
  plausible-looking wrong latitude is the most damaging thing this feature could
  emit, and the prompt forbids it in words.
- **Image → place assignment is a first-class interaction.** AI suggestions are
  suggestions: the traveller ticks, moves, or clears them, and a human decision
  is never overwritten by a later analysis run. An **unassigned image tray**
  makes imperfect analysis survivable instead of hiding it.
- **Imported images are private by construction.** They live in IndexedDB, are
  written `private_import` by the only function that can create them, and are
  shown in a saved place under their own heading — never merged into the place's
  canonical photography. Deleting an import deletes the bytes.
- **Cost control on both axes.** Client-side downscaling to three sizes for three
  jobs, SHA-256 dedupe, a 20-image / 12MB-per-file budget, cached analyses, and a
  field-masked external lookup that runs only after Meridian fails and only when a
  provider is configured.
- **No retrieval means no request.** With no approved provider, the honest
  fallback is offered immediately rather than a doomed fetch: paste the text or
  upload screenshots, and the flow continues.

Still open from this iteration:

- **A retrieval agreement.** The route, the refusal semantics and the image relay
  all exist; what is missing is a provider Meridian is permitted to read through.
  Until then the fallback is the product.
- **Vision by default.** The multimodal analyzer is implemented and the prompt is
  written, but it needs a deployment with a key. On GitHub Pages the built-in
  analyzer reads text and the traveller assigns pictures by hand — a complete
  workflow, not a degraded one, but not the whole of §5.
- **Verified image analysis quality.** Nobody has measured how well a given model
  reads a Xiaohongshu menu photograph, and `IMAGE_PROPOSAL_FLOOR` is a judgement
  rather than a measurement.

---

## V1.4 — ten destinations, five loyalty programmes

**Every destination is now reference tier, and the loyalty model covers the
programmes that actually trade in Southeast Asia.**

- **A coordinate audit came first, and it changed the plan.** 73 of Bali's 145
  places shared a coordinate with another record, most labelled `verified` with a
  source note naming a *different* venue. Fixed as 73 reviewable corrections, and
  the rule is now permanent in `validate:data`.
- **Five programmes**: Marriott Bonvoy, Hilton Honors, IHG One Rewards, World of
  Hyatt, GHA DISCOVERY — 73 brands. GHA is modelled as an alliance, because that
  is what it is.
- **Five shapes, five letters** on the map, because five programmes on one island
  is exactly where colour-only differentiation fails.
- **10 destinations · 117 areas · 97 loyalty hotels · 446 places**, from 48 / 42 /
  216. Targets were treated as targets: Phu Quoc has 7 loyalty hotels, Cebu 5,
  Boracay 1, and those are verified ceilings rather than omissions.

Still open:

- **Photography has not grown with the data.** The new hotels and places fall back
  to a no-photograph state rather than borrowing a neighbour's image.
- **Accor and Wyndham are absent.** A programme earns its place by having enough
  real inventory to change where somebody stays.
- **28 of Bali's 145 places are approximate or unlocatable.** Nearly all are small
  restaurants no OSM mapper has surveyed.

---

## V1.5 — Malaysia, and Chinese everywhere

- **Two Malaysian destinations**: 槟城 (Penang) and 吉隆坡 (Kuala Lumpur), 14 areas and 40
  places each, with 10 and 14 loyalty hotels across five programmes. A new country in the
  dataset, not a reskin of an existing one.
- **Chinese coverage completed for the nine non-Bali destinations.** The original records
  predated the Chinese-first decision, so 河内's hotel cards read in English while 巴厘岛's
  were Chinese. Now authored in one reviewable overlay, with `nameZh` omitted only where a
  name has no form in real Chinese use.
- **Penang's honest ceiling is 10 hotels and no open Hyatt**, so its STAY filter offers four
  programmes rather than five — the filter row is built from real inventory, not a fixed list.

---

## V1.6 — accommodation as stays

- **A stay is a hotel and a date range**, not a hotel row repeated on each day. Day
  start and end anchors are DERIVED from the stays, so a hotel-change day falls out
  of the data instead of being declared.
- **The change day is routed for real**: `Hotel A → stops → Hotel B` through the
  existing provider, so the hotel-to-hotel drive is measured.
- **The 09:00 assumption is gone.** A trip default with per-day overrides, and a
  time change recomputes the clock without re-requesting the routes.
- **Fixed times are bookings**: early arrival shows usable waiting time, late
  arrival shows a conflict, and the time is never moved.
- **Migration refuses to guess.** Only a hotel repeated on consecutive days is
  converted; an ambiguous single row is preserved exactly as it was.

Still open:

- **No optimiser on the change day.** The model supports placing sensible stops on
  a moving day; nothing reorders them yet.
- **No live flights.** `origin → airport → first hotel` is expressible through the
  arrival anchor and is not computed.

---

## V2 — live data, accounts, and planning intelligence

### Live data behind the existing interfaces
- **Live flight prices** via Amadeus Self-Service or a Skyscanner-compatible partner API, rendered
  as an explicitly attributed live layer with a fetch timestamp.
- **Live hotel pricing and availability** for Marriott Bonvoy and Hilton Honors, if acceptable
  partner terms can be obtained. Until then, the tier model stays and no numbers are shown.
- **Award and points valuation.** `Hotel.loyaltyMeta` exists and is empty for exactly this reason:
  the shape is ready, and nothing is claimed before there is a maintained source.
- **Elite benefit comparison**, driven entirely by versioned data so it can be updated without a
  release.
- **Visa and entry requirements** with a review date on every claim.
- **Restaurant reservations and activity booking** via affiliate or partner deep links, clearly
  marked as external and clearly marked as paid where applicable.

### Planning intelligence
- **AI itinerary generation**, constrained by the efficiency engine rather than replacing it: the
  model proposes, the geometry disposes. Generated plans must be explainable stop by stop.
- **A real traffic model** in the routing layer, so "≈40 minutes" becomes "40–75 minutes at this
  hour on this day".
- **Cost calculator** once live pricing exists, in the traveller's currency, with points-vs-cash.

### Product and platform
- **Accounts and cloud-synced trips**, replacing `localStorage` without changing the store's public
  surface.
- **Collaborative trips.** Multiple planners, comments, and voting on candidate stops.
- **Native mobile apps or a mature PWA** — the map-first interface is already the right shape for a
  phone.
- **A public data contribution flow** with review, so destination depth scales beyond what one team
  can verify by hand.

---

## Explicitly not planned

- **Fake live data.** No generated prices, no invented "12 people are looking at this", no
  unsourced claims about elite benefits.
- **A conventional OTA booking funnel.** Meridian decides *where* and *in what order*; booking is a
  hand-off, not the product.
- **Content marketing.** No listicles, no SEO articles, no hero banners. If it does not help someone
  understand where things are, it does not ship.
- **Scraping Xiaohongshu — or any platform.** No headless browsers, no logged-in sessions, no
  CAPTCHA or signature solving, no rotation of user agents or proxies, no private or followers-only
  content. The platform forbids it, it breaks monthly, and it would put a traveller's own account at
  risk. A 403 is the end of the attempt, not the start of a workaround.
- **A second source platform.** TikTok, Instagram, Douyin and YouTube are out of scope until the
  Xiaohongshu workflow is genuinely excellent. The union has one value on purpose.
- **A social network.** No feed, no following, no comments, no messaging, no creator profiles, no
  monetisation of other people's guides. Meridian reads the text a traveller hands it and gives back
  a map.
- **Rehosting creator imagery or text.** A guide's photographs and prose belong to whoever made
  them. Meridian keeps a link and the traveller's own paste, and reproduces neither.

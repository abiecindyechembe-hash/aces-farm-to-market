# ACE's — Farm-to-Market Digital Cooperative

A responsive marketplace interface where rural cooperatives list produce (tomatoes,
cassava, plantains, maize) and urban buyers see live, verified prices — built for
the AntCode Hub 48-Hour Sprint.

**Live prototype:** open `home.html` directly in any browser, or serve it with
`npx serve .` — it is a single static file with no build step and no dependencies,
which is itself a deliberate choice (see Cameroonian Context below).

---

## 1. The problem this UI is solving

Four failure points drove the design: hidden prices that let middlemen underpay
farmers, transport that never lines up with the produce, price swings nobody can
see coming, and crops that spoil before a buyer is found. The interface is built
around four matching mechanisms, not four separate screens:

| Failure | Mechanism in the UI |
|---|---|
| Farmers underpaid, buyers overcharged | Live reference price shown *while typing* a listing, plus a "Fair price / Below market / Above market" tag on every card |
| Transport fragmented | A "Transport" button on every listing runs a nearest-available-vehicle match against a mock fleet(fake data used to make the demo work) and returns an Ectimated Time of Arrival. |
| Prices swing with no visibility | A scrolling ticker in the status bar re-prices every crop every 4 seconds, simulating a live feed |
| Produce spoils in transit | Every listing carries a spoilage-risk bar computed from crop-specific shelf life and hours since the time of harvest, and buyers can sort "sells soon first" |

**An added feature:** nothing in the original request spelled out a spoilage system in detail — it followed naturally from the goal of protecting farmers, since produce spoiling in transit is one of the ways they
lose money. The spoilage-risk bar and "sells soon first" sort exist for that reason: a simple rule-based model (shelf life vs. hours since harvest) that
runs entirely client-side, kept deliberately separate from the rest of the UI code so its logic can be swapped out later without touching how it's displayed
(see the architecture below).

---

## 2. System architecture

The app is one HTML file, deliberately split into five modules that only
communicate through a single store — so any module can be changed without
touching the others.

```
┌─────────────┐     ┌───────────────┐     ┌────────────────────┐
│    DATA     │────▶│     STORE     │────▶│      ENGINES        │
│ crop table, │     │ single source │     │ spoilageRisk()       │
│ fleet, seed │     │ of truth +    │     │ fairPriceTag()       │
│ listings    │     │ subscribe()   │     │ matchVehicle()       │
└─────────────┘     └──────┬────────┘     │ (pure functions,     │
                            │              │  no DOM access)      │
                            │              └──────────┬───────────┘
                            │                          │
                            ▼                          ▼
                     ┌─────────────────────────────────────┐
                     │               RENDER                 │
                     │ ticker · status bar · coop grid ·     │
                     │ buyer grid · transport card           │
                     └──────────────┬────────────────────────┘
                                    │
                                    ▼
                              <the screen>
                                    ▲
                                    │ user input
                     ┌──────────────┴────────────────────────┐
                     │            EVENTS / NET                │
                     │ form submit, filters, sort, offline     │
                     │ detection, simulated live-price tick    │
                     └─────────────────────────────────────────┘
```

**Why this split matters:** `ENGINES` never touches the DOM (it never reads or
writes anything on screen — see `spoilageRisk()`, `fairPriceTag()`, and
`matchVehicle()` in the code) and `RENDER` never contains business logic — it
only turns whatever `ENGINES` calculated into HTML. That separation means the
spoilage-risk model or the fair-price threshold can be rewritten on its own
without rewriting a single line of the UI, because the two are joined only
through plain return values, never through shared DOM state.

### State-management diagram (a price change, end to end)

What happens from the moment a price changes to the pixel updating on screen:

```
[every 4s: simulated market tick]
        │
        ▼
store.mutate() writes new prices into store.state.prices
        │
        ▼
store.notify() calls every subscriber  ── (currently one: renderAll)
        │
        ├──▶ renderTicker()      → new numbers scroll across the status bar
        ├──▶ renderCoopGrid()    → fair-price tag recalculated per listing
        └──▶ renderBuyerGrid()   → fair-price tag + sort order recalculated
```

A user action (submitting a listing, requesting transport) is the same loop
run from the other end: `EVENTS → store.mutate() → store.notify() → RENDER`.
There is exactly one way data reaches the screen, which is what keeps the UI
from drifting out of sync with the store as features get added.

---

## 3. Performance: sub-3-second load on a low-end Android phone over 3G

- **Zero external requests.** No CDN fonts, no icon library, no images — crop
  icons are emoji, all styling is inline CSS. On a real deploy this becomes a
  budget of one small font subset and SVG icons under ~15KB total, but the
  prototype ships at 0 KB of assets on purpose, to make the ceiling visible.
- **No framework runtime.** Vanilla JS avoids shipping and parsing a
  React/Vue runtime before anything is interactive — on a low-end Android CPU,
  JS *parse and execute* time dominates over network time once you're this
  small.
- **Render is a single innerHTML pass per section**, not per-row DOM
  diffing, which is cheap enough to run on every 4-second price tick without
  jank on low-end hardware.
- **Data loaded once, not paginated over the network** — five mock listings
  ship inline; a real backend would page listings (20 at a time) and lazy-load
  card images only when a card scrolls into view, which is the next
  optimization once real photos are added.

## 4. Offline / network-drop behaviour

The status bar is a first-class part of the UI, not a toast that disappears:
- Going offline switches the bar to a clay-red state and the copy changes from
  "Live · connected" to "Offline · showing last known prices" — prices freeze
  at their last known value instead of silently going stale-looking.
- If a cooperative submits a listing while offline, it is **not blocked or
  lost**: it's added to the screen immediately (optimistic update) and queued
  in `pendingSync`. When the connection returns, a toast confirms exactly how
  many listings just synced.
- A demo timer flips the connection every 18 seconds so this path is visible
  without opening devtools; in production this block is replaced by listening
  to the browser's native `online`/`offline` events (also wired in, see the
  bottom of the `<script>`).

---

## 5. Cameroonian context adaptation

Every technical choice above traces back to a real constraint for this market,
not a generic mobile-first checklist. Zero external assets and no framework
runtime exist specifically because a cooperative member in Foumbot is likely
opening this on a shared, older Android device on 3G where every kilobyte
costs money and every render-blocking script costs seconds — the app has to
be usable, not just responsive. The offline-first listing queue exists because
rural network coverage genuinely drops mid-session rather than failing
gracefully, so the form had to assume the connection will disappear rather
than treat it as an edge case. The fair-price tag on every listing is the
direct UI answer to the middleman problem: it's shown to the cooperative
*before* they publish, not after a buyer has already seen it, because the
leverage a farmer needs is knowing the market price before negotiating, not
after. Finally, the transport-matching step assumes what's true in
reality — trucks and motos that exist but aren't reliably visible to the
person who needs them — so it's modeled as a matching problem (nearest
available vehicle with enough capacity) rather than a static listing of
transport companies to call.

---

## 6. AI prompt ledger

This is a template ledger structure for the submission — fill in my actual
iteration history here before  submission, including places where i corrected what the AI produced.
i used clade ai as help for the bild up.
| # | Prompt (paraphrased) | What I asked for | What I changed / rejected afterward |
|---|---|---|---|
| 1 | "Build a responsive digital marketplace where rural cooperatives list produce (tomatoes, cassava, plantains, etc.), buyers see real-time price matches, transport is managed, cheating is prevented, and spoilage is solved with a smart system" | The full first version of the interface listing form, live price ticker, fair-price flagging, transport matching, and a spoilage-risk engine, plus the architecture and state-diagram documentation | Kept the four-mechanism structure as generated; this became the base I iterated on rather than rewrote |
| 2 | Explained that the spoilage/perishability problem hadn't been spelled out explicitly, and asked for it to be added and documented as an intentional addition | A spoilage-risk system covering the produce-decay problem, built on top of the core interface | i kept the rule-based shelf-life model as a first version, built so its logic stays separate from the display code |
| 3 | Renamed the project from the placeholder name to "ACE's," reworded field labels ("Origin hub" → "Location", "Your asking price" → "selling price"), and renamed the fleet vehicle types | Cosmetic/wording changes to make the interface feel like my own project rather than a generic template | Kept all renamed labels and fleet names; verified afterward that nothing downstream still referenced the old label wording |
| 4 | Reported that the "Active listings" section had stopped displaying after the rename, with screenshots of the code and the blank section | A diagnosis of why the listings grid was empty | The actual bug: the status-bar `<span>` id had been renamed to `statusmessage` but the script still looked up `#statusLabel`, so `renderStatus()` threw and stopped the rest of the render chain before the grids ever painted. Fixed by matching the id back to `statusLabel` rather than rewriting the render logic |
| 5 | Asked for a "more catchy" footer line | A short set of footer taglines to choose from, in the same plain voice as the rest of the copy | Picked from the options offered rather than the original placeholder footer text |

---

## 7. What's simulated vs. what's real in this prototype

- Reference prices, the vehicle fleet, and the five seed listings are fake
  data defined at the top of `home.html` — swap `CROPS`, `FLEET`, and the
  seed listings for real data to go live.
- The "live" price tick and the offline toggle are `setInterval` timers
  standing in for a live data feed and the browser's connectivity events —
  both are marked in the code where the swap happens.
- Payment and order fulfillment are out of scope for this interface — the
  "Order" button shows the intended hand-off point, where a real payment flow
  would take over.
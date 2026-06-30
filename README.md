# TCG Snipe

A Chrome (Manifest V3) extension that snipes underpriced Pokémon cards — it scans
marketplace listings and shows the **live market value** inline on each one.
Site: [tcgsnipe.co.uk](https://tcgsnipe.co.uk)

## 🧪 For testers — install in 2 minutes

1. **Download the zip:** [latest release →](https://github.com/mrilanduk/tcgsnipe/releases/latest)
   (grab `tcgsnipe.zip`).
2. **Unzip** it to a folder (the folder must contain `manifest.json` at its top level).
3. Open **`chrome://extensions`** → turn on **Developer mode** (top-right) →
   **Load unpacked** → select the unzipped folder.
4. Click the crosshair icon → **enter your access code** → Unlock.
   (No code? Request access at tcgsnipe.co.uk.)
5. Browse **eBay / Vinted / Facebook Marketplace** — price badges appear as you scroll.

Notes: keep the unzipped folder where it is (Chrome loads it live). To update to a
newer release, download the new zip, replace the folder contents, and hit **↻ Reload**
on the extension card.

---


It does **not** call PokePulse directly. It talks to your own
[`pokemon-price-checker`](../pokemon-price-checker) service, which holds the
partner API keys server-side, caches prices for 6h, and resolves set/number
shorthand. The extension only uses that service's public (no-auth) endpoints:

| Endpoint | Used for |
| --- | --- |
| `GET /api/search?name=&num=` | find the matching card from a listing title (cheap — public search) |
| `GET /api/lookup?code=setId-num` | get priced variants (partner API, **~1000 lookups/day cap**) |
| `GET /api/usage` | show today's call counters in the popup |

## Supported sites

- **eBay** (`.co.uk` / `.com`) — search grids + individual item pages
- **Vinted** (`.co.uk` / `.com`) — catalog grids + item pages
- **Facebook Marketplace** — listing grids
- **Any page** — select a card name/number → right-click → **“Pulse: price …”** → floating result

## Install (unpacked)

1. Open `chrome://extensions`, enable **Developer mode** (top-right).
2. **Load unpacked** → select this `pulse-price-scanner` folder.
3. Click the extension icon and **enter your access code** to unlock it (codes are
   issued by Pulse Collective from the `/access` admin page on the backend).
4. Browse eBay/Vinted/FB Marketplace. Badges appear on listings **as you scroll**.

Without a valid code the extension stays locked and prices nothing. Revoking a
code (in `/access`) locks that user out on their next lookup.

The backend URL is **hardcoded** (`SERVICE_URL` in `background.js`) and is not
shown or editable in the UI — the popup only toggles the extension on/off per
site. Note: a browser extension is client-side, so the URL can still be found by
unpacking the extension or watching the network tab; hardcoding keeps it out of
the UI and settings, which is as private as a client extension allows.

## How pricing stays within the rate cap

`/api/lookup` is capped at ~1000 cold lookups/day upstream. To respect that:

- Listings are priced **lazily** — only when they scroll into view
  (`IntersectionObserver`, 300px margin), not all at once on page load.
- Results are **cached client-side** for 6h (matching the server cache); "no
  match" is cached for 30m. Repeats cost nothing.
- Background concurrency is capped at **4** in-flight lookups, with identical
  queries deduped.
- The popup shows live `catalogue` / `market` call counts so you can watch usage.

Use **Clear price cache** in the popup to force fresh lookups.

## How a listing becomes a price

1. The listing title (free text like `Charizard ex 199/165 Pokemon 151 PSA 10`)
   is parsed into `{ name, num, grade, condition }` in `background.js#parseTitle`.
   On eBay item pages the structured **Condition field** ("Near Mint", "Lightly
   Played", …) overrides the title-derived condition.
2. `/api/search` finds candidate cards; the one whose `card_number` matches the
   parsed number wins (else the top result).
3. `/api/lookup` returns every variant; the badge shows **NM ungraded** market
   value by default, or the **graded** value if the title mentions PSA/CGC/BGS +
   a grade. Hover the badge for the full condition + graded breakdown.

## Files

| File | Role |
| --- | --- |
| `manifest.json` | MV3 manifest, permissions, content-script matches |
| `background.js` | service worker — API client, parsing, cache, concurrency, context menu, selection overlay |
| `content.js` | site adapters + lazy scan engine + badge injection |
| `badge.css` | inline badge styling |
| `popup.html` / `popup.js` | settings (service URL, per-site toggles), live usage, rescan, clear cache |

## Caveats

- **Marketplace DOM changes often.** The CSS selectors in `content.js`'s
  `adapters` (especially Vinted and Facebook, which use obfuscated class names)
  are the most likely thing to need updating. Each adapter is isolated and
  wrapped in try/catch, so a broken selector degrades to "no badges" rather than
  errors.
- **Title parsing is best-effort.** Odd titles may mis-parse; the server's fuzzy
  search absorbs most of it, but expect occasional "no match".
- No build step — plain JS/CSS/HTML, load unpacked as-is.

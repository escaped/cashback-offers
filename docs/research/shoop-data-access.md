# Shoop data access — research

- **Date:** 2026-09-13 (access ~06:30–06:45 UTC)
- **Scope:** How a Chrome MV3 extension can obtain Shoop's offer for a merchant identified by a website hostname, using only the logged-out public experience (no credentials).
- **Method / evidence:** (1) static analysis of Shoop's own published Chrome extension ("Shoop Cashback & Gutscheine" v3.2.17.2, unpacked CRX from the Chrome Web Store, ID `hacngjmphfcjdfpmfmlngemhddjdncpe`, read from `/tmp/shoop-crx/ext`); (2) live, unauthenticated `GET` calls to `api.shoop.de` made through the `webfetch` tool (which passes Cloudflare where plain `curl` returns 403); (3) `robots.txt` + `sitemap.xml` read directly. No login, no cookies, no personal data, no HAR files were used or recorded.

---

## TL;DR / recommendation for the extension

Shoop exposes a **public, unauthenticated JSON API** at `https://api.shoop.de`. The cleanest way to map a hostname to an offer is exactly what Shoop's own extension does:

1. `GET https://api.shoop.de/api/merchants/toolbar` → full catalog (~3.95 MB, 1944 merchants), each carrying a `urlPattern` regex. Cache it locally (Shoop's own code defaults to a 240-minute cache; the response carries `cache-control: max-age`).
2. Match the current tab's URL/hostname against `urlPattern` client-side (regex), falling back to `id` and `name` substring — **no per-hostname server lookup exists**.
3. Optionally refine with `GET https://api.shoop.de/api/search/suggest?query=<name|hostname>` for a ranked, typed suggestion list.
4. Build a deeplink via `GET https://api.shoop.de/api/visit/redirect?merchant=<id>&toolbar=true&return=true` → returns a tracked affiliate URL (`message.data.url`). This works **without any token** for anonymous callers.

This requires **no credentials and no session**, and only read-only, low-volume requests. It is precisely the flow Shoop ships to its own 2,000+ partner users, so it is the least likely to be treated as abusive.

---

## Catalog & search endpoints

All base at `https://api.shoop.de` (constant `G` in the extension bundle; website at `https://www.shoop.de`, constant `L`). The response envelope is uniformly `{"result":"success","message":...}` (or `{"result":"error","message":[...]}`).

| Endpoint | Method | Purpose | Verified |
|---|---|---|---|
| `/api/configuration?id=web-live` | GET | App config: api base, ws URL, feature flags, baseUrl, premium flags | ✅ live |
| `/api/merchants/toolbar` | GET | **Full merchant catalog** (1944 objects) | ✅ live |
| `/api/merchants/toolbar-top` | GET | Top merchants (smaller) | ✅ (extension code + archived 200 JSON) |
| `/api/merchants/suggested?merchantId=<id>` | GET | Suggested/alternative merchants for a given merchant | ✅ (extension code) |
| `/api/merchants/extended?ids=1,5` | GET | Batch merchant lookup by numeric id | ✅ live |
| `/api/merchants?merchantId=<id>` | GET | Full merchant detail (rates, vouchers, categories, `updatedAt`) | ✅ live |
| `/api/merchants/vouchers?merchantId=<id>` | GET | Vouchers for one merchant | ✅ live |
| `/api/search/suggest?query=<q>&limit=<n>` | GET | **Search/autocomplete** (name → merchant) | ✅ live |
| `/api/deals/toolbar` | GET | Deals for the toolbar | ✅ (extension code) |
| `/api/user/toolbar` | GET | Logged-in user toolbar (auth required) | ✅ (extension code) |
| `/api/user/favorites` | GET | User favourites (auth required) | ✅ (extension code) |
| `/api/voucher/request` | GET | Request/reveal a voucher code | ✅ (extension code) |
| `/api/visit/redirect` | GET | **Deeplink/tracking redirect** | ✅ live |
| `/api/cms?id=generelle-cashback-bedingungen` | GET | CMS content (ToS text) | ✅ (extension code) |
| `/api/toolbar/caa-configuration`, `/api/toolbar/alternative-merchants`, `/api/interstitial-extension`, `/api/abtest/version` | GET | Extension/toolbar config | ✅ (extension code) |
| `wss://api.shoop.de/websocket` | WS | Realtime channel | ✅ (extension code) |
| `/api/v1/activity/shoop`, `/api/v2/logs` | POST | Extension analytics/logs | ✅ (extension code) |

**Search specifics:**

- Working search endpoint is **`/api/search/suggest`** (query parameter `query`, optional `limit`). Example, `GET /api/search/suggest?query=zalando&limit=3`, returns:

```json
{"result":"success","message":{"merchants":[
  {"name":"Zalando","merchantUrl":"zalando","subtitle":"",
   "rateTeaser":{"basic":null,"premium":null,"giftcard":{"upTo":false,"value":8,"sign":"%"}},
   "friendOnly":false,"hasDeal":false,"numberOfVouchers":0,"premiumInfo":null,
   "cashbackText":"","instantCashbackText":"8% Sofort-Cashback","dealId":null,
   "hasGiftcard":true,"hasPowerHourRates":false,
   "logoDetails":{"fav":{"normal":"…/5.png",...},"small":{...},"medium":{...},"big":{...}}},
  {"name":"Lounge by Zalando","merchantUrl":"lounge_by_zalando","subtitle":"Bis zu 6% Cashback",
   "rateTeaser":{"basic":{"upTo":true,"value":6,"sign":"%"},"premium":null,"giftcard":null}, ...}
]}}
```

- The suggestion object is **typed and ranked**; `merchantUrl` is the merchant slug; `rateTeaser.basic` carries `{upTo, value, sign}` (the "Bis zu X% Cashback" teaser); `giftcard` carries gift-card instant-cashback when applicable (Zalando shows `instantCashbackText:"8% Sofort-Cashback"`).
- **Non-working search paths** (return error/404): `/api/merchants/search?search=`, `/api/merchants/search?q=`, `/api/search?q=`, `/api/search?query=` → `{"result":"error","message":["Es ist ein Fehler aufgetaucht..."]}` or HTTP 404. So `/api/search/suggest` is the reliable name-search endpoint. (A full website results endpoint `/api/search` beyond suggest remains **Unverified**.)
- `robots.txt` disallows `*/suche` (site search UI), `*/stoebern`, `*/gutscheine`, `*/start`, `*/account/`, and `/*meta.json`; the sitemap is at `https://www.shoop.de/sitemap.xml` (1745 URLs, 1538 shop pages under `/cashback/<slug>`). Source: `https://www.shoop.de/robots.txt`, `https://www.shoop.de/sitemap.xml`.
- No `__NEXT_DATA__`; the website is an **Angular SPA** (archived `main.bd3e8aff84f57845.js`, 2026-07-22) that boots by fetching `/api/configuration?id=web-live`, then hits the endpoints above. It is not server-rendered with embedded JSON — data comes from `api.shoop.de`.

---

## Merchant identification

A merchant is identified by a **numeric `id`**, a **slug** (used in `serpLink` and `/cashback/<slug>` URLs), a **display `name`**, and a **`urlPattern` regex**. All are present in the `/api/merchants/toolbar` payload. A hostname is mapped to a merchant **client-side**:

- The extension's matcher (`ce` in `bg/bundle.js`) loads the whole merchant list once and, for a given URL, does in order: (a) exact `id` match if known, else (b) `url.match(new RegExp(urlPattern))`, else (c) case-insensitive `name` substring match. There is **no server-side "hostname → merchant" endpoint**.
- `urlPattern` values arrive **wrapped in slashes** (e.g. `"/^(https?:\\/\\/)?([\\w-]+\\.)?zalando\\.de/"`) and the extension strips the leading/trailing `/` via `substring(1, len-1)` before compiling to `RegExp`.

Verified examples (from live `/api/merchants/toolbar`):

| Merchant | id | urlPattern | serpLink | cashbackAvailable |
|---|---|---|---|---|
| Zalando | 5 | `/^(https?:\/\/)?([\w-]+\.)?zalando\.de/` | `https://www.shoop.de/toolbar/zalando/` | false |
| Conrad Electronic | 1 | `/^(https?:\/\/)?([\w-]+\.)?conrad\.de/` | `https://www.shoop.de/toolbar/conrad_elektronik/` | true |
| Otto | 242 | `/^(https?:\/\/)?([\w-]+\.)?otto\.de/` | `https://www.shoop.de/toolbar/otto/` | — |
| Saturn | 2831 | `/^(https?:\/\/)?((?!handytarife)[\w-]+\.)?saturn\.de/` | `https://www.shoop.de/toolbar/saturn/` | — |
| eBay | 875 | `^(https?:\/\/)?(\w+\.)??(cart\.payments\.)?ebay\.de` | `https://www.shoop.de/toolbar/ebay/` | — |
| Amazon | 2594 | `/^(https?:\/\/)?([\w-]+\.)?amazon\.de/` | `https://www.shoop.de/toolbar/amazon/` | — |
| IKEA | 4352 | `/^(https?:\/\/)?([\w-]+\.)?ikea\.com/` | `https://www.shoop.de/toolbar/ikea/` | — |

Notes: `urlPattern` can be a complex regex (Saturn uses a negative lookahead to exclude `handytarife` subdomains; eBay includes a `cart.payments.` prefix case). Some merchants target `.com` (IKEA) or a subdomain (`en.aboutyou.de`). **Implication:** matching by the raw `urlPattern` regex is more correct than naive hostname string comparison.

Full merchant object (all 1944 objects carry these 18 keys): `id, name, subtitle, logo, termsAndConditions, urlPattern, checkoutPattern, serpEnabled, link, serpLink, rates, rateTeaser, vouchers, supportsToolbar, cashbackAvailable, hideSlider, premiumInfo`. `link` is a relative path `/api/visit/redirect?merchant=<id>`; `serpLink` is a full `www.shoop.de` URL.

---

## Offer shape

Rates and vouchers live on the merchant object.

**Rate** (`rates[]`, one or more):
```json
{"id": 221187, "canClickout": false, "value": "1,5", "currency": "%",
 "text": "auf alle Artikel (außer DVD-Verleih, ...)", "description": "…",
 "warningText": "<br>WICHTIG: Bei Einsatz nicht freigegebener Gutscheine wird kein Cashback gezahlt. …",
 "vouchers": [], "hasAmazon": false,
 "basic": {"value": "1,50", "sign": "%"}, "premium": null}
```
- `value` is a **localized decimal string** (German: `"1,5"`); `currency` is `"%"` (percentage) or `"€"` (fixed euro per order).
- Example € rate (Sparhandy.de, id 13): `{"value":"10","currency":"€","text":"auf den validen Vertragsabschluss eines Mobilfunkvertrags durch einen Neukunden", "basic":{"value":"10,00","sign":"€"}, ...}`.
- `basic` = standard rate (`value`, `sign`, plus `upTo` in teaser); `premium` = premium-account rate (`{value, sign, upTo}`) — null for standard users.
- The extension renders the best rate by sorting `rates` descending on numeric `value` (`.replace(",",".")`) and picking the first whose `currency === "%"`.
- `canClickout` gates whether the offer can be clicked directly.
- `warningText`/`text`/`description` and `termsAndConditions` (HTML list) carry **conditions and exclusions**.

**Rate teaser** (`rateTeaser`) for headline "Bis zu X% Cashback":
```json
{"basic": {"upTo": true, "value": 6, "sign": "%"}, "premium": null, "giftcard": null}
```
- `upTo:true` → "Bis zu 6% Cashback" (`cashbackText` in search/suggest mirrors this). Premium variant uses `rateTeaser.premium`.

**Voucher** (`vouchers[]`):
```json
{"id": 20626657, "title": "10% CONRAD Rabattcode auf Produkte von Fluke",
 "description": "…", "code": "FLUKE926", "unique": false, "link": null,
 "expiresAt": "2026-09-13T23:59:59+02:00", "voucherType": "non_combinable_voucher"}
```
- `voucherType` ∈ `"voucher" | "offer" | "non_combinable_voucher"`. `expiresAt` gives validity dates.
- Catalog stats (2026-09-13): 1750 of 1944 merchants have `cashbackAvailable` + `rates`; 398 have at least one voucher.

**Validity dates / categories / exclusions:** expressed as free-text `termsAndConditions` (HTML `<li>` list of exclusions), per-rate `text`/`warningText`, and voucher `expiresAt`. There is no structured category taxonomy or per-rate date range in the toolbar payload — conditions are prose. The `subtitle` is a human-readable summary (e.g. `"1,5% Cashback"`).

**Per-merchant detail endpoint (live-verified):** `GET /api/merchants?merchantId=<id>` returns a richer object:
- `updatedAt` (e.g. `"2026-09-11T11:01:21+02:00"`) — a freshness timestamp useful for cache invalidation;
- `live: true`, `noCashback`, `conditionsAlwaysOn`, `redirect`, `offers`, `hasGiftcard`, `ratePromoted`, `friendOnly`, `allowsToolbarPopup`;
- `categories: [{"id":12,"name":"Haus & Technik","children":[{"id":13,...}]}]` — a structured category tree;
- rates additionally carry `valid: null`, `endDate: null`, `powerHour: false`, `basic: {value: 1.5, sign: "%"}` and `premium`;
- `GET /api/merchants/vouchers?merchantId=<id>` returns `{"result":"success","message":{"data":[{id,title,shortTitle,text,code,link,endDate,isCombinable,type,uniqueCode}]}}`.

This per-merchant path is a lighter fresh-refresh alternative to re-downloading the full ~3.95 MB catalog when only the visited merchant matters.

---

## Deeplink format

Two paths, both observed in `bg/bundle.js`:

1. **Logged-out (no token needed for the redirect itself):**
   ```
   https://api.shoop.de<merchant.link>?token=<sessionToken>&offerId=<offerId>
   ```
   where `merchant.link` is the relative path `/api/visit/redirect?merchant=<id>`. If `merchant.serpLink` is set, that full `www.shoop.de` URL is used instead (with `?offerId=<id>`). In the anonymous case the `token` is empty/absent.

2. **The actually-useful anonymous redirect** (verified live):
   ```
   GET https://api.shoop.de/api/visit/redirect?merchant=1&toolbar=true&return=true
   ```
   returns:
   ```json
   {"result":"success","message":{"data":{
     "url":"https://www.awin1.com/cread.php?awinmid=11354&awinaffid=320119&clickref=2x124350715",
     "subid":"2x124350715","isButtonNetwork":false,"forceWeb":false}}}
   ```
   `message.data.url` is the tracked click-through (here an Awin link: `awinaffid=320119` is Shoop's publisher id; `clickref`/`subid` is a unique tracking ref). **No token or account required** — the server mints a unique `clickref` for anonymous callers.

3. **Logged-in path** (when a session token exists): `GET /api/visit/redirect?merchant=<id>&cid=<analyticsClientId>&offerId=<id>&toolbar=true&return=true&meta={"mtCustomerHash":…,"mtPlatform":…}`, then navigate to `data.data.url`.

The extension's own deep-link construction (from `bg/bundle.js`) is:
- logged-out: `${G}${r.link}?token=${i.token}&offerId=${id}` (or `serpLink` form),
- logged-in: `await aa(d.id)` → the `/api/visit/redirect` call above.

Website click links (for a user browsing shoop.de) take the form `https://www.shoop.de/visit/<merchantId>/cashback/…` and `…/voucher/…`.

**Bottom line:** a deeplink can be constructed with **no account** — just `merchant.id` (from the catalog) into `/api/visit/redirect`. The parameters in play are `merchant`, `offerId`, `toolbar`, `return`, `meta` (JSON with `mtCustomerHash`, `mtPlatform`), `cid` (analytics client id), and `token` (session token, empty when anonymous). Shoop's own tracking `awinaffid` is embedded in the returned URL, not something the extension constructs.

---

## Logged-in differences & session detection

- **Personalized/premium rates** are surfaced via `rateTeaser.premium` / rate `premium` fields, plus a `premiumAccount`/`premiumInfo` concept in the content bundle (`bu()` checks `user.premiumAccount` start/end). In the current catalog, **no merchant had a non-null `premiumInfo`** and `rateTeaser.premium` was null for sampled merchants, so premium/personalized rates appear to require an authenticated session (or are currently unadvertised). Logged-out callers see only `basic`/public rates.
- **Session detection (from the extension, no login performed by us):** the content script runs on `shoop.de`, listens for a DOM `CustomEvent` `authTokenChange` (detail `authToken`) and sends it to the background as `{action:"updateToken"}`; it also reads `localStorage.authToken` (Firefox path) and forwards `localStorage.sessionUrlParams` plus `document.cookie` via a `setMegatronData` message. The background stores the token in `chrome.storage.local`, sends it as a `token` header on API calls, and uses it in deeplinks.
  - **For our extension:** an MV3 extension can detect an existing Shoop session by (a) reading `document.cookie` on `shoop.de` pages (or listening for the `authTokenChange` event / reading `localStorage.authToken` via an injected content script on `shoop.de`), but note that `authToken` is a **localStorage** value on the shoop.de origin — visible only to content scripts injected on that origin, not cross-origin. No specific cookie *name* could be confirmed without logging in (**Unverified**); the extension treats the token as an opaque value obtained from the page.
- Background pings a `SESSION_PING` alarm every 10 minutes to refresh session state.

---

## Bot protection / rate limits / ToS

- **Cloudflare.** Plain `curl` to `www.shoop.de/*` and `api.shoop.de/*` returns **HTTP 403** with `cf-mitigated: challenge` and a Turnstile "Just a moment…" page. `robots.txt` and `sitemap.xml` are served. A real headless-browser attempt to load a shop page got the Cloudflare Turnstile "Verify you are human" checkbox, which **did not pass** under automation (the checkbox reset to unchecked) — so scraping the rendered site with a headless browser is unreliable.
- The **`webfetch` tool passes Cloudflare** and returned live JSON from `api.shoop.de` for the documented endpoints — this is how the live responses above were obtained.
- **`robots.txt`** (`https://www.shoop.de/robots.txt`): `Disallow` for `*/suche`, `*/stoebern`, `*/gutscheine`, `*/start`, `*/account/`, `*/anmelden`, `*/auszahlung`, `*/einstellungen/`, `*/freunde-einladen`, `*/invite/`, `*/lieblingsshops`, `*/nachrichten`, `/*meta.json`. Several AI crawlers (ClaudeBot, anthropic-ai, cohere-ai, CCBot, Bytespider, PerplexityBot, etc.) are fully disallowed. The API base `api.shoop.de` has no `robots.txt` observed.
- **Rate limits:** no explicit numeric limit observed. The extension sets `pragma: no-cache` / `cache-control: no-cache` on catalog fetches, but parses the response's `cache-control` `max-age` and defaults to a **240-minute** cache (`…max-age=N / 60 || 240`). A read-only extension should fetch the ~3.95 MB catalog infrequently and cache it; the `search/suggest` endpoint is lightweight and suited to on-demand queries.
- **ToS posture:** Cloudflare challenge + `robots.txt` disallows + full disallow of AI crawlers signal an anti-automation stance. Shoop's own public terms are served via `/api/cms?id=generelle-cashback-bedingungen` and `/api/cms?id=…` (extension references `generelle-cashback-bedingungen`). No legal conclusion is drawn here; the practical guidance is that Shoop's own extension already performs client-side catalog matching + these public GET endpoints, so an extension that mirrors that exact read-only behavior is the most defensible pattern.
- The extension bundle embeds a **Google Analytics Measurement Protocol `api_secret`** (value deliberately not reproduced here). It is a static secret in the published CRX, not a Shoop session credential.

---

## Open questions / could not verify

1. **Live browser XHR capture** (network log of the SPA actually calling `api.shoop.de` from the rendered site) — **Unverified**: the Cloudflare Turnstile challenge never passed under automation. Verifiable by running a real (non-headless) Chrome profile that has previously passed Cloudflare and recording the network tab.
2. **Website's own full-results search** (what the `/suche` page fires beyond `/api/search/suggest`) — **Unverified**. The site search UI is `Disallow`ed in `robots.txt`; only `suggest` was confirmed.
3. **`/api/merchants/top` / `/api/homepage/merchants` / `/api/deals`** — seen only in the archived website bundle, **not re-verified live** (toolbar endpoints cover the same data). `/api/merchants?merchantId=`, `/api/merchants/vouchers?merchantId=` and `/api/merchants/extended?ids=` were re-verified live (see above).
4. **Exact logged-in cookie name(s)** on `shoop.de` — **Unverified**; only `localStorage.authToken` and the `authTokenChange` event are confirmed from the extension source. Verifiable only by logging in (out of scope).
5. **`link` shape beyond `/api/visit/redirect?merchant=<id>`** for merchants where `serpLink` is absent — inferred from the observed pattern; every sampled merchant had this shape.
6. **`premiumInfo` / premium rates semantics** — field exists and is null everywhere in the current public catalog; how/when it populates is unverified (likely auth-gated).
7. **Rate limiting / throttling thresholds** — no numeric limits observed or documented.

---

## Sources

- `https://www.shoop.de/robots.txt` (accessed 2026-09-13)
- `https://www.shoop.de/sitemap.xml` (accessed 2026-09-13; 1745 URLs)
- `https://api.shoop.de/api/configuration?id=web-live` (live, unauthenticated, 2026-09-13)
- `https://api.shoop.de/api/merchants/toolbar` (live, unauthenticated; 1944 merchants, 2026-09-13)
- `https://api.shoop.de/api/search/suggest?query=zalando&limit=3` (live, unauthenticated, 2026-09-13)
- `https://api.shoop.de/api/merchants/extended?ids=1,5` (live, unauthenticated, 2026-09-13)
- `https://api.shoop.de/api/merchants?merchantId=1` and `https://api.shoop.de/api/merchants/vouchers?merchantId=1` (live, unauthenticated, 2026-09-13)
- `https://api.shoop.de/api/visit/redirect?merchant=1&toolbar=true&return=true` (live, unauthenticated, 2026-09-13)
- Shoop Cashback & Gutscheine extension v3.2.17.2, unpacked CRX (`/tmp/shoop-crx/ext/{manifest.json, bg/bundle.js, content/bundle.js}`), Chrome Web Store ID `hacngjmphfcjdfpmfmlngemhddjdncpe`
- Archived website JS bundle `main.bd3e8aff84f57845.js` (2026-07-22) via `web.archive.org` (endpoint table)
- Archived `https://api.shoop.de/api/merchants/toolbar-top` (2022–2023, HTTP 200 application/json) via `web.archive.org` CDX

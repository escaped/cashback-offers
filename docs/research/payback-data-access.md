# Research: PAYBACK data access for a browser extension

Ticket: [#4](https://github.com/escaped/cashback-offers/issues/4) · Branch: `research/payback-data-access` · Date: 2026-09-13

Question: how can a Chrome extension obtain PAYBACK's offer for a merchant identified by a
website hostname, **without storing credentials**?

Method: inspected payback.de pages and Next.js JS bundles with `curl` (browser UA),
the public `digital-shops` JSON API, and the publicly distributed PIA extension (PAYBACK's
official browser extension, v3.82.0 CRX from the Chrome Web Store). Claims are marked
**[verified]** (observed myself), **[inferred]** (from first-party code/artifacts), or
**[unverified]**.

## TL;DR

- There is a **public, unauthenticated JSON catalog** of all online shops:
  `GET https://www.payback.de/onsflows/api/digital-shops` (706 shops, ~375 KB full,
  server-side `searchText`/pagination). It contains the domain mapping for 620 shops
  via `browserExtensionInfoListItem[].domainNameListItem` — this is exactly the field
  PAYBACK's own extension uses for hostname matching. **[verified]**
- The catalog exposes only a headline points rate (`incentivationHeadline`, e.g.
  `1 °P pro 2 €`) and occasional promotions. Conditions/coupons/validity are **not** in
  the public API; conditions are server-rendered on `https://www.payback.de/shop/<vanity>`.
- **Click-through tracked links cannot be created anonymously.** `POST /onsflows/api/jumptoshop`
  requires the user's `cardNumber` or a logged-in token (401 otherwise). The anonymous
  shop page itself asks the user to type their "Kundennummer" per click. So a
  credential-free extension can only deep-link (untracked) or hand the user to
  `payback.de/shop/<vanity>`; storing the card number is a product decision, not required
  by any public API.
- Coupons and personalised multipliers are **login-gated**: `/coupons` redirects anonymous
  visitors to `/login`.
- payback.de sits behind **Imperva Incapsula**; POST APIs return 403 to non-browser
  clients. PAYBACK login is protected by reCAPTCHA (it broke the main community
  auto-activator). Treat automation beyond plain GET catalog reads as grey-area.
- The official PIA extension uses a **private API** (`services-ext.payback.de`,
  `getdigitalshops`) with embedded keys and stores user credentials. Do **not** copy that.

## 1. Public online-shop catalog (the usable API)

### Endpoint

```
GET https://www.payback.de/onsflows/api/digital-shops
```

- No auth, no cookies required (tested from a clean curl). Content type `application/json`.
- Full list: `{"totalResultSize":706,"digitalShopListItem":[...]}` (~375 KB).
- Discovered in the page bundle of `https://www.payback.de/online-shopping/shops-a-z`
  (`DIGITAL_SHOPS_ROUTE = "/onsflows/api/digital-shops"`). **[verified]**

### Query parameters (all server-side; confirmed by test)

| Param | Example | Effect |
|---|---|---|
| `searchText` | `?searchText=mediamarkt` | Name-only fuzzy search (3 hits: mediamarkt, mmtarifwelt, praemienshop). **Does not match domains** — `searchText=mediamarkt.de` → 0 hits. **[verified]** |
| `pageSize`, `pageOffset` | `?pageSize=3&pageOffset=3` | Server-side pagination, envelope keeps `totalResultSize=706`. **[verified]** |
| `partnerLevel` | `?partnerLevel=1` | 1 → 55 shops, 2 → 651 shops (disjoint; sums to 706). Semantics not documented — see Unknowns. **[verified]** |
| `labelListItem` | `?labelListItem=TOP_SHOP` | 8 top shops (mediamarkt, bader, ebay, lotto, aboutyou, tchibo, etsy, lufthansa). **[verified]** |
| `categoryShortNameListItem` | from item categories | Category filter. **[inferred from bundle]** |
| `sortByShopScore` | `?sortByShopScore=true` | Sort. **[inferred from bundle]** |
| `platform` | `?platform=1` | Passed by the site (platform 1 = browser extension); setting 2 changed nothing observable. **[verified]** |

### Item shape (one item, MediaMarkt, trimmed)

```json
{
  "partner": { "partnerShortName": "lp628164", "partnerDisplayName": "MediaMarkt Online" },
  "partnerVanityName": "mediamarkt",
  "partnerLevel": 2,
  "partnerLogoIdentifier": "lp628164",
  "labelListItem": ["TOP_SHOP"],
  "shopScore": 9999,
  "incentivation": { "incentivationHeadline": "1 °P pro 2 €" },
  "keywords": "m,m,mediamarkt,media markt,computer,smartphone,gaming,...",
  "promotionListItem": [],
  "browserExtensionInfoListItem": [ { "platform": 1, "domainNameListItem": ["mediamarkt.de"] } ],
  "categoryShortNameListItem": ["1-4o2l9al", "1001", "1003", "1-9l6m6lv", "1124001"]
}
```

### Merchant identification

- **Canonical merchant ID**: `partner.partnerShortName` (`lp` + number, e.g. `lp628164`).
  The same `lp` IDs appear on stationary-partner pages. Used as `shortname` in API calls.
- **Slug**: `partnerVanityName` (`mediamarkt`) → detail page `https://www.payback.de/shop/mediamarkt`
  (`/online-shopping/<slug>` 301-redirects there). **[verified]**
- **Name**: `partner.partnerDisplayName`.
- **Domain index**: `browserExtensionInfoListItem[0].domainNameListItem` exists for **620/706**
  shops, **623 distinct domains**, no duplicates. Platform is always `1`. **[verified]**
- **86 shops have no domain mapping** (e.g. `dm`, `apple`, `eventim`, `moreandmore`,
  `fleurop`, `expedia`, `ticketmaster`). Some have the domain inside `keywords`
  (e.g. `eventim.de`), most don't. Name/slug fallback needed. **[verified]**
- PAYBACK's own extension ([`pia/background/background.js`]) converts the API item into its
  partner model with `browserExtensionInfoListItem[0].domainNameListItem` when
  `platform === BROWSER_EXTENSION`, and matches hostnames with exact host/path/params first,
  then subdomain matching (`isSubDomainOf`). So `www.mediamarkt.de` and `de.trotec.com`
  map to `mediamarkt.de` / `trotec.com`. **[inferred from first-party code]**

### Offer shape

- `incentivation.incentivationHeadline` — the points rate, free text. 47 distinct values in
  the catalog; most common `1 °P pro 2 €` (598 shops); also `500 °P pro Abschluss`,
  `1 °P pro Kauf`, `Bis zu 1.500 °P`, etc. Not a structured "points per €" number. **[verified]**
- 6 shops carry `promotionListItem` (`incentivationHeadline` + `incentivationSubline`),
  e.g. `Bis zu 7.600°P` / `750 °P pro hochgeladener Versicherung`. **[verified]**
- `labelListItem` currently only `TOP_SHOP` (8 shops); `shopScore` (used for sorting).
- **Conditions** are not in the API. The shop detail page renders them server-side
  (`data-testid="partner-conditions"`), e.g. MediaMarkt: credit a few days after purchase
  (locked), unlock after ~93 days, exclusions list that explicitly excludes **stationary
  MediaMarkt stores**, marketplace items, phone contracts, etc. **[verified]**
- **Validity dates** were not found in the public catalog or shop page; `goLiveDatetime`/
  `goDarkDatetime` in the page payload are CMS block fields, not offers. **[verified absence]**
- **Coupons / multipliers are login-gated**: `https://www.payback.de/coupons` 307s anonymous
  users to `https://www.payback.de/login?redirectUrl=...`. Coupon APIs
  (`/couponing/api/nutshell-coupons`, `/couponing/api/coupon-activation`,
  `/couponing/api/coupon-details`, `/couponing/api/preview-coupon-details-list`) are POSTs
  behind Imperva and a session (403 to curl even with session cookies). **[verified]**
- PayPal-style multipliers for online shops only appear as part of the headline/promotions
  above when public.

### Online vs offline partners

- The public catalog is the **online-shop** list ("über 700 Online-Shops"). **[verified]**
- Stationary partners are presented on `https://www.payback.de/filialen-prospekte`
  (CMS/Next.js page, same `lp` IDs, 48 partners rendered there: ARAL, dm, EDEKA, Netto,
  fressnapf, Sparkasse, …; each has a `partnerLink`). No public structured points-rate API
  for offline partners was found; rates/offers live in the PAYBACK app's store finder
  (`mobile.pww-rtc-prod.pbext.io`, referenced in the site CSP). **[verified page, unverified app API]**
- **Inference flag:** `partnerLevel=1` (55 entries) contains stationary-first/hybrid brands
  (dm, EMP, Apollo-Optik, GALERIA-like, CHECK24 products) while level 2 (651) are typical
  online shops; `dm` (level 1) has no domain mapping. Semantics not documented. **[inferred]**
- Online purchases credit via click-through; offline via card at the till. Conditions such
  as MediaMarkt's explicitly exclude in-store purchases from online earning. **[verified]**

## 2. Click-through / tracking link

- `POST https://www.payback.de/onsflows/api/jumptoshop`
  body: `{shortname, cardNumber, category, excid, incid, jumpOutLocation:"JTS"}`
  (from the site bundle). Response: `{clickOutUrl, scrid}`; the site opens `clickOutUrl`. **[verified via bundle]**
- **Requires the user's card number or a token**: no `cardNumber` → `401 {"error":"Missing token or cardNumber"}`;
  invalid 13-digit number → `500 {"error":"Failed to create context"}`. Tested with a dummy
  value; no real card data was used. **[verified]**
- The anonymous shop page (`https://www.payback.de/shop/<vanity>`) renders a "Kundennummer"
  input (`cardNumberPlaceHolder`, `cardNumberError`) for non-logged-in users, i.e. PAYBACK
  itself asks for the number per click. Logged-in users get `isUserLoggedIn:true` and the
  token path. **[verified]**
- **PIA** (official extension) does not call `jumptoshop` directly from the extension UI; it
  opens `https://www.payback.de/onsflows/api/pia-jts?url=<partner-url>` in a tab
  (`JTS_BASE_URL` in `pia/background/background.js`) or falls back to
  `https://www.payback.de/shop/<vanity>?category=<id>`. An anonymous request to `pia-jts`
  307-redirects to the payback.de homepage — it needs a PIA/login session. **[inferred from first-party code, verified anonymous behaviour]**
- The PIA privacy policy states PAYBACK passes the user's PAYBACK number to the partner shop
  so the purchase can be credited. **[verified, PIA privacy policy on addons.mozilla.org]**
- **Conclusion:** there is no credential-free way to mint a tracked click-out. Options for
  our extension: (a) ordinary deep link to the shop (no points), (b) open
  `payback.de/shop/<slug>` so the user clicks/enters their number in PAYBACK's own UI,
  (c) optionally keep the card number in extension storage at the user's explicit choice.
  Do not scrape or replay `jumptoshop` with user data.

## 3. Login-state detection

- Page props contain `"isUserLoggedIn":false/true` in the RSC payload (`self.__next_f`
  stream / props), and the header switches between `authLinks.signIn` and logout. Rendering
  `https://www.payback.de/shop/<vanity>` (or the homepage) and checking `"isUserLoggedIn":true`
  is a viable detection without reading cookies. **[verified for anonymous; logged-in value inferred from same prop]**
- `/coupons` redirect to `/login?redirectUrl=...` is another reliable anonymous signal. **[verified]**
- Cookies are `__Host-Session` and `__Host-Xsrf-Token` — both `HttpOnly`, `Secure`,
  `SameSite=strict` — plus Imperva cookies (`visid_incap_*`, `nlbi_*`, `incap_ses_*`).
  Extension JS cannot read the session cookie; `fetch(..., {credentials:"include"})` can
  still use it. **[verified from response headers]**
- POST endpoints require `X-Xsrf-Token`, read from `meta[name="csrf_token"]` and sent with
  `X-Page-Load-Rcid` (`meta[name="rcid"]`) — see `fetchWithOptions` in the site bundle. **[verified via bundle]**
- What extra data appears when logged in (point balance, personal coupons, card-number
  prefilling) was **not verified** — no account was used, and none is needed for the public
  catalog use case.

## 4. ToS / anti-bot status (grey areas flagged)

- **Imperva Incapsula** fronts www.payback.de (`X-CDN: Imperva`, `X-Iinfo`, incap cookies).
  GET page/catalog requests worked from curl, but POST APIs returned `403` to non-browser
  clients even with session cookies. A real extension fetch has a browser TLS/header
  fingerprint, but service-worker requests without an Origin/Referer may still be
  challenged. Treat automated POSTs as unreliable/grey. **[verified]**
- **reCAPTCHA on login** broke the primary community auto-activator
  (`stefanjenkner/payback-activate`, README and issue #28 "broken since Payback introduced
  reCaptcha"). Credential-based automation is therefore fragile and against the grain of the
  platform. **[verified]**
- `robots.txt` is permissive for normal crawlers: only `/test` is disallowed (plus named
  SEO bots); sitemaps are published at `/web/sitemap.xml` etc. The catalog API is not
  disallowed. **[verified]** This is not a licence — ToS still govern use.
- ToS/Bedingungen page URL was **not found** (`/agb`, `/info/nutzungsbedingungen` → 404);
  only the privacy pages were located (`/info/datenschutz`). The contract terms could sit
  behind the login or another path — **unverified**; do not assume automation is permitted.
- **Grey-area approaches — flagged, not recommended:**
  - **PIA's private API** `services-ext.payback.de` (`getdigitalshops`, `getloyaltypartners`,
    richer per-shop `conditions` via `getShopDetails`) with an **embedded API key** found in
    the published extension config. Using another party's key/client credentials is
    unauthorised and will break without warning. **[inferred from first-party code]**
  - **`pia-jts` / `/shop/` JTS redirects** are first-party PIA UX, not a documented API.
  - **Coupon auto-activation** userscripts (Nergico Tampermonkey, BobbyCephy Selenium,
    stefanjenkner WebdriverIO) drive the logged-in DOM and are exactly what reCAPTCHA was
    aimed at; several are abandoned/broken.
  - Sending a stored `cardNumber` to `jumptoshop` from a third-party extension is possible
    but puts a sensitive loyalty number in extension storage.

## 5. Community projects (state as of this research)

| Project | What it documents | State |
|---|---|---|
| [jgoldhammer/moneymoney-payback](https://github.com/jgoldhammer/moneymoney-payback) | MoneyMoney extension for the PAYBACK points account; requires e-mail+password login; explicitly does **not** support card number/birthday/PIN auth | Old but maintained |
| [stefanjenkner/payback-activate](https://github.com/stefanjenkner/payback-activate) | WebdriverIO login + coupon activation; README: **broken since PAYBACK introduced reCAPTCHA** (issue #28) | Broken |
| [BobbyCephy/payback-coupon-activator](https://github.com/BobbyCephy/payback-coupon-activator) | Selenium against the user's logged-in Chrome profile | Unmaintained |
| [Nergico/Payback-Auto-Activator-Skript-by-Nergico](https://github.com/Nergico/Payback-Auto-Activator-Skript-by-Nergico) | Tampermonkey/bookmarklet that clicks coupon buttons in the logged-in page | Userscript |
| PAYBACK Internet Assistent (PIA) | Official first-party extension, ~100k Chrome users, v3.80 in store / v3.82 CRX; stores credentials, shows point balance/coupons/shop list | Active, closed source |

No public project documenting the `/onsflows/api/digital-shops` endpoint was found; the
endpoint was found directly in payback.de's own JS bundle (`shops-a-z` page).

## 6. Recommended shape for our extension (research conclusion)

1. Fetch `GET /onsflows/api/digital-shops` once, cache it (375 KB; add a TTL, e.g. daily),
   fall back to `searchText=<name>` if matching by name.
2. Build a hostname→shop index from `browserExtensionInfoListItem[].domainNameListItem`,
   with exact-host then suffix/subdomain matching (mirroring PIA's algorithm).
3. Show `incentivationHeadline` + promotions; fetch `https://www.payback.de/shop/<vanity>`
   lazily only when the user wants conditions (parse `partner-conditions`, or just link out).
4. Do **not** store credentials or card numbers. For click-through, open
   `payback.de/shop/<slug>` (user completes the jump there) or a plain deep link.
5. Everything personalised (coupons, multipliers, balance) is out of scope until the user
   logs in inside their own browser session; do not automate login.

## Unknowns

- Meaning of `partnerLevel` 1 vs 2 (observed split 55/651; likely stationary/hybrid vs pure
  online — unverified).
- Whether extension-context POSTs pass Imperva; GET catalog requests worked from curl.
- Logged-in offer data (personal coupons, multipliers, validity/end dates) — not inspected.
- Any public structured API for stationary partners / store finder.
- PAYBACK ToS text and its stance on automated catalog reads; AGB URL not found.

## Reproduce (raw commands)

```bash
# catalog
curl -s 'https://www.payback.de/onsflows/api/digital-shops' -H 'Accept: application/json'
curl -s 'https://www.payback.de/onsflows/api/digital-shops?searchText=mediamarkt&pageSize=5'
# jump-to-shop (needs cardNumber/token)
curl -s -X POST 'https://www.payback.de/onsflows/api/jumptoshop' \
  -H 'Content-Type: application/json' -d '{"shortname":"lp628164","jumpOutLocation":"JTS"}'
# shop detail (conditions, server-rendered)
curl -sL 'https://www.payback.de/shop/mediamarkt'
# login gate
curl -sIL 'https://www.payback.de/coupons'
```

Files/bundles inspected: `/online-shopping/shops-a-z`, `/shop/mediamarkt`,
`/onsflows/_next/static/chunks/*.js`, `/web/sitemap.xml`, `/robots.txt`, Chrome Web Store
CRX of PIA (`pbfjbhoglggakhkngkbfehgghkaadeba`). No credentials used; no personal data
captured. This note is the only artifact committed to the branch.

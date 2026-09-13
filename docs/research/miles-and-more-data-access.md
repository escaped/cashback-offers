# Research: Miles & More (Online Shopping) data access

- **Ticket:** escaped/cashback-offers#3
- **Date:** 2026-09-13
- **Status:** Resolved (primary sources; logged-out only). Items not verified are marked `[UNVERIFIED]`.
- **Question:** How can the extension obtain Miles & More Online Shopping's offer for a merchant
  identified by a website hostname, without storing (user) credentials?

## TL;DR

- The Online Shopping portal is a public React SPA at `onlineshopping.miles-and-more.com`; its JSON API
  lives at `https://api2.prd.mmi.q-two-hosting.com` and is documented below.
- The catalog and all offers are readable **without any user login**: the SPA ships OAuth2
  `client_credentials` values in its public JS bundle and exchanges them for a 1-hour Bearer token.
  No user credential is involved in reading offers.
- 536 partners are returned by `GET /api/partner/list`; `GET /api/search/<term>` searches by name.
  Every record carries `unique_name` (slug, e.g. `rewe_de`), `name`, `partner_id` and
  `loyalty_points` (miles per €) plus campaign fields.
- **No shop domain/URL is present anywhere in the API or in partner pages.** Hostname → merchant
  matching must come from a separate mapping (affects decision ticket #10).
- Tracked deeplinks: `POST /api/partner/link/<partner-UUID>` returns an Awin affiliate URL
  (`https://www.awin1.com/cread.php?awinmid=<advertiser>&awinaffid=329243&clickref=<memberRef>`).
  It works anonymously (empty `clickref`), but the member/service-card number must go into
  `clickref` for miles attribution. The member number is not a login credential.
- Login state is exposed client-side (`mam:auth` cookie/localStorage + plaintext
  `localStorage["mam:scn"]` service-card number). Offers observed are identical logged-out vs
  logged-in. `[UNVERIFIED]` whether M&M personalizes any offers per member.

## Method

All findings are from live, logged-out requests on 2026-09-13 (plain `curl`, no account, no cookies
from a real user). The OAuth token used below was obtained through the portal's own public
`client_credentials` flow. No credentials, member numbers, cookies or HARs are reproduced in this
document; the OAuth client values are intentionally not copied — they are embedded in the public JS
bundle cited below and can be read there.

## 1. Portal and API surface

| Thing | Value |
|---|---|
| Portal (SPA) | `https://onlineshopping.miles-and-more.com/` |
| Required query params | `?noredirect=1&site=de&language=en` (`site` ∈ `de`, `at`, `it`, `be`, `pl`, …) |
| Without `noredirect=1` | 302 to `www.miles-and-more.com/de/<lang>/earn/shopping/shopping-platform.html?...` (Cloudflare-protected, 403s non-browser clients) |
| Entry bundle (at research time) | `https://onlineshopping.miles-and-more.com/assets/index-Hrp9u7PD.js` (content-hashed name) |
| API base (hardcoded in bundle) | `https://api2.prd.mmi.q-two-hosting.com` |
| Partner page (server-rendered, no auth) | `https://onlineshopping.miles-and-more.com/partner/<unique_name>?noredirect=1&site=de&language=en` |

Server headers seen on the portal: Caddy (`Via: 1.1 Caddy`, `X-Server: app-01`), no bot challenge.
Response sets cookies `session=<uuid>; Max-Age=3600` and `locale=en-DE`.

### Anonymous token (no user login)

The bundle contains an OAuth2 client (`client_id` + `client_secret`) and calls:

```http
POST https://api2.prd.mmi.q-two-hosting.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=<embedded in index-*.js>
&client_secret=<embedded in index-*.js>
&scopes=partner_read network_read cms_read
```

Response: `{"token_type":"Bearer","expires_in":3600,"access_token":"<JWT>"}` — verified 2026-09-13.
Note the parameter is spelled `scopes` (plural) by the SPA. A user `grant_type=password`
(e-mail/password, per the portal’s login form) also exists for actual member login; it is
**not needed** for the catalog.

All `/api/...` calls below require `Authorization: Bearer <token>`; without it the API returns
HTTP 401. `Accept-Language: en` / `de` selects localized strings.

## 2. Shop list / search endpoints

### List (full catalog)

```http
GET https://api2.prd.mmi.q-two-hosting.com/api/partner/list?limit=100&offset=0
    [&sort=loyaltyPoints|name&order=asc|desc]
    [&filter[<key>]=<value>]        # filter keys not enumerated; built dynamically by the SPA
Authorization: Bearer <anonymous token>
Accept-Language: en
```

Verified response (2026-09-13), trimmed:

```json
{
  "data": [{
    "id": "ff72de67-0225-4691-b63a-2c2eddcb7e1e",
    "unique_name": "rewe_de",
    "name": "REWE",
    "partner_id": "11652",
    "network": { "id": "01920003-422e-7979-8ad5-0635da6b6672", "name": "Awin" },
    "loyalty_points": 5,
    "min_refund": {
      "type": "campaign", "point_type": "factor", "point_value": 5.0,
      "start_date": "2026-08-31T22:00:00+00:00", "end_date": "2026-09-30T21:59:59+00:00",
      "show_dates": true, "total_min": 0
    },
    "has_free_shipping": false, "has_reduction": true, "has_campaign": true,
    "currency": "EUR", "status": "online", "visible": true,
    "hidden_shop": false, "app_partner": true, "new": false,
    "partner_logo": { "resolved_url": "https://api2.prd.mmi.q-two-hosting.com/asset/image/<uuid>" }
  }],
  "limit": 100, "offset": 0, "length": 100, "max_length": 536,
  "has_next": true, "next_offset": 100
}
```

- Pagination: `limit` + `offset`; total catalog `max_length = 536` (2026-09-13).
- `min_refund` is absent for partners without an active campaign.
- `status` is normally `online`; clients should filter `visible == true` and `hidden_shop == false`.

### Search by name

```http
GET https://api2.prd.mmi.q-two-hosting.com/api/search/<term>?type=partner&limit=15&offset=0
    [&sort=...&order=...][&filter[...]=...][&track=true]
```

- Results live under `results.partner[]`; same fields but the display name is `item_name`
  (plus `partner_logo` includes full attachment metadata).
- `<term>` is matched against **names only**:
  - `search/mediamarkt` → `mediamarktsaturntarifwelt_de`, `mediamarkt_de`
  - `search/mediamarkt.de`, `search/amazon.de`, `search/rewe.de` → `{"results":[],...}` (empty)
- Empty term (`/api/search?type=partner...`) returns the whole catalog.
- `track=true` adds an internal tracking flag used by the SPA `[UNVERIFIED]` exact effect.

Other catalog endpoints found in the bundle:

- `GET /api/category/list?limit=&offset=[&sort=&order=]` — category list.
- `GET /{locale}/translation` — UI strings, including offer labels
  (`mam.blocks.partner.head.miles_factor` = `"%s -M- for every 1 EUR"`).
- `GET /asset/image/<file_id>/<w>x<h>[?preview=]` — images.

## 3. Merchant identification

| Field | Example | Notes |
|---|---|---|
| `unique_name` (slug) | `rewe_de`, `amazon_de_en`, `mediamarktsaturntarifwelt_de` | Best stable merchant key; encodes brand + country (`_de`, `_at`, …), sometimes language |
| `name` / `item_name` | `REWE`, `Amazon` | Display name, not unique |
| `partner_id` | `11652` | Network-specific advertiser ID (matches Awin `awinmid`) |
| `id` (UUID) | `ff72de67-…` | Internal partner UUID; required for deeplink generation |
| `network` | `{"name":"Awin"}`, `{"name":"VerticalAds (retailads)"}` | Affiliate network per partner |
| `partner_logo.resolved_url` | API asset URL | Image only |

**No domain, hostname, or shop URL is returned in list, search, or partner-page payloads.**
Domains cannot be derived reliably from the slug alone (`amazon_de_en` vs `mediamarktsaturntarifwelt_de`).
Hostname → `unique_name` mapping needs an external source (e.g. cashback-optimizer.de or a curated
static map) — this is the main gap for decision ticket #10.

## 4. Offer shape

Per partner, the list API exposes the effective rate and campaign meta; the partner page adds
vouchers/conditions. Verified sample (REWE, 2026-09-13):

```json
{
  "loyalty_points": 5,
  "min_refund": { "type": "campaign", "point_type": "factor", "point_value": 5.0,
                  "start_date": "2026-08-31T22:00:00+00:00",
                  "end_date": "2026-09-30T21:59:59+00:00" },
  "currently_active_hints": [
    { "type": "reduction", "point_type": "factor", "icon": "90",
      "title": [{ "locale": "de", "content": "<p>15€ Gutschein für Neukunden!</p>" },
                { "locale": "en", "content": "<p>€15 gift card for new customers!</p>" }],
      "start_date": "2026-08-26T22:00:00+00:00", "end_date": "2026-09-30T21:59:00+00:00",
      "show_dates": true, "voucher_code": "SOMMER15", "total_min": 60 },
    { "type": "campaign", "point_type": "factor", "point_value": 5, "icon": "13",
      "start_date": "2026-08-31T22:00:00+00:00", "end_date": "2026-09-30T21:59:59+00:00" }
  ]
}
```

- `loyalty_points` is the headline rate in miles per € (translations: `"%s -M- for every 1 EUR"`;
  list sort label “Miles”). In a 100-partner sample, `loyalty_points == min_refund.point_value`
  whenever a campaign exists; `min_refund` is absent when no campaign runs (e.g. `babywalz_de` → 1).
- `min_refund` describes the currently guaranteed campaign (factor, validity).
- `currently_active_hints` (partner page only) carries promos/vouchers with localized title,
  optional `voucher_code`, `total_min` minimum spend, dates and full `terms` per locale.
- `currency` (EUR), `vat_type`/`vat_percentage`, `has_reduction`, `has_campaign`,
  `has_free_shipping`, `additional_wording` complete the shape.
- Partner page JSON is embedded in the HTML at
  `<script type="application/json" id="__STATIC_ROUTER_DATA__">` →
  `loaderData["0"].view.page.bonded_data` (public, no Bearer token needed for the SSR HTML).

## 5. Deeplink / tracked click-through

```http
POST https://api2.prd.mmi.q-two-hosting.com/api/partner/link/<partner UUID>
Authorization: Bearer <anonymous token>
Content-Type: application/json

{ "userRef": "<service-card number, optional>",
  "binding": { "type": "partner", "id": "<partner UUID>" } }
```

Verified responses (2026-09-13), keyed by the `currently_active_links[].id` from the partner page:

```json
{ "019518b2-52ae-7470-bcda-47b704d62260":
  "https://www.awin1.com/cread.php?awinmid=11652&awinaffid=329243&clickref=" }
```

with `"userRef":"<number>"` → same URL ending `&clickref=<number>`.

- The URL is returned **without login and without a member number**; `clickref` is then empty, i.e.
  the click is not attributed to a member (no miles). A member/service-card number in `userRef`
  is required for attribution.
- The service card number is **not** an auth credential; the portal collects it in a separate
  “exit” overlay and validates it via `GET /api/check/customer/<ref>` → `{"valid":true|false}`.
- Response keys are the partner’s `currently_active_links[].id` entries, not the requested UUID;
  the SPA maps each exit button (`currently_active_links[].id`) to its URL.
- Awin is the network for many partners; `awinmid` equals the partner’s `partner_id`, `awinaffid`
  is Miles & More’s publisher ID (same across partners on Awin).
- Where to get the member number in a browser: the portal keeps it in plaintext
  `localStorage["mam:scn"]` after validation (see §6). It can be read transiently; it is personal
  data and must not be persisted by the extension.
- Internal analytics: `GET /asset/click/<base64(URLSearchParams)>` with `X-User-Session`
  (value = `session` cookie) records UI clicks; not needed for deeplinks.

## 6. Session / login-state detection

- Login session: zustand-persisted store `mam:auth` containing `token` (OAuth access token),
  `authenticated`, `expires_at`. When `window.cookieStore` exists the SPA writes it as an
  AES-encrypted `document.cookie` named `mam:auth` (feature detection, not the Cookie Store API
  itself), otherwise it falls back to localStorage; the AES passphrase is derived from the store
  name in the bundle, i.e. obfuscation rather than security.
- Service-card number (member ref): plain `localStorage["mam:scn"]` (set after
  `/api/check/customer/<ref>` returns `valid:true`). `localStorage["mam:last-visited"]` tracks
  recently viewed partners.
- Server session: `session=<uuid>` cookie (Max-Age 3600), sent as `X-User-Session` on CMS/click
  requests — not an auth signal.
- Login state for an extension: presence of `mam:auth` (cookie) / `mam:scn` (localStorage), or the
  SPA’s own `authenticated` flag. The catalog endpoints do not vary by login.
- **Rates logged-out vs logged-in:** no difference observed in the anonymous catalog; the portal
  only adds `userRef` to the deeplink. `[UNVERIFIED]`: whether M&M shows member-specific campaigns
  after login (would need a real account; out of scope for this repo).

## 7. Bot protection, rate limits, ToS

- `api2.prd.mmi.q-two-hosting.com`: Caddy, **`Access-Control-Allow-Origin: *`**, no Cloudflare /
  DataDome / Akamai challenge observed; `/robots.txt` → 404. ~15 logging-light requests during
  research, no throttling seen.
- `onlineshopping.miles-and-more.com`: Caddy, no bot challenge observed. Its `robots.txt`:

```
User-agent: *
Disallow: /config
Disallow: /app
Disallow: /app_dev
Disallow: /api

Sitemap: https://api2.prd.mmi.q-two-hosting.com/page/sitemap.xml
```

- `www.miles-and-more.com`: Cloudflare; non-browser clients get HTTP 403 (the portal redirects
  there when `noredirect=1` is missing — keep `noredirect=1` to stay on the open Caddy host).
- **Rate limits:** none advertised or observed; exact thresholds `[UNVERIFIED]`.
- **ToS:** the robots disallow above plus M&M’s general T&Cs are the relevant signals; no explicit
  rule about automated catalog access was found. Whether bulk/hosted use is acceptable is a legal
  call — flag for the architecture/legal review, and prefer client-side, user-agent requests at
  human rates over a shared server-side poller.

## Could not verify

- Personalization: whether logged-in members see different offers/rates per partner.
- Exact rate limits / abuse thresholds of api2.
- `filter[...]` key vocabulary for `/api/partner/list` and `/api/search`.
- `track=true` semantics on search.
- Long-term stability of the embedded OAuth client values (they may rotate with the bundle).
- Whether `www.miles-and-more.com` pages (help/ToS) contain an explicit automation clause —
  Cloudflare blocked non-browser fetches; not pursued to avoid bypassing bot protection.

## Implications for downstream decisions

- **Matching (#10):** M&M cannot map a hostname to a shop by itself — the catalog only has
  `unique_name`/`name`. A separate mapping (curated or from cashback-optimizer.de) is required.
- **Caching (#11):** catalog is a 536-entry, low-churn JSON; daily per-partner caching is ample.
  Partner pages (promos/vouchers) are server-rendered and could be cached per day too.
- **Architecture (#13):** all reads work anonymously with an app-level token; no user session or
  stored credential is needed for offers. Only tracked deeplinks need the user’s service-card
  number (`localStorage["mam:scn"]`), which must be read transiently and never persisted.
- **Session reuse constraint:** the extension can detect login via `mam:auth`/`mam:scn` and read
  the member number from the portal’s localStorage rather than asking the user to re-enter it —
  a decision to confirm in the architecture ticket.

## Sources

All retrieved 2026-09-13, logged-out, plain HTTP clients:

- `https://onlineshopping.miles-and-more.com/?noredirect=1&site=de&language=en` (SPA entry, JS bundle)
- `https://onlineshopping.miles-and-more.com/assets/index-Hrp9u7PD.js` (API base, OAuth client,
  endpoint paths, storage keys) — bundle name is content-hashed and will change
- `https://onlineshopping.miles-and-more.com/partner/rewe_de?noredirect=1&site=de&language=en`
  (server-rendered partner data incl. promos)
- `https://api2.prd.mmi.q-two-hosting.com/token` (anonymous client_credentials grant)
- `https://api2.prd.mmi.q-two-hosting.com/api/partner/list?limit=100&offset=0`
- `https://api2.prd.mmi.q-two-hosting.com/api/search/<term>?type=partner&limit=3`
- `https://api2.prd.mmi.q-two-hosting.com/api/partner/link/<uuid>`
- `https://api2.prd.mmi.q-two-hosting.com/api/check/customer/<ref>`
- `https://api2.prd.mmi.q-two-hosting.com/en/translation`
- `https://onlineshopping.miles-and-more.com/robots.txt`
- `https://api2.prd.mmi.q-two-hosting.com/page/sitemap.xml` (1,404 portal URLs incl. `/partner/<slug>`)
- Secondary (context only, not relied on): web search results for the portal domain

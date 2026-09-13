# Research: Corporate Benefits data access for a browser extension

Repo: `escaped/cashback-offers` · Date: 2026-09-13 · Platform: Corporate Benefits Germany
(`mitarbeiterangebote.de`)

Question: how can a Chrome MV3 extension obtain Corporate Benefits' offer for a merchant
identified by a website hostname, while the user is logged in — portal URL patterns, offer
discovery, offer shape, session detection, rate limits/ToS — and does prior art exist?

Method: public sources only. `curl` with a browser UA against portal pages, login/registration
pages and headers; the first-party webpack bundle `app.js`; the public Wayback CDX index;
GitHub code/repo search via the authenticated `gh` CLI; and — the key find — the **source of two
existing extensions and one userscript** that already solve this. No login, no credentials, no
auth bypass, no personal data. Claims are marked **[verified]** (observed in a first-party source
or response), **[inferred]** (consistent evidence, not directly observed), **[claimed]**
(third-party statement), **[unverified]**. Cookie values below are redacted.

## TL;DR

- Corporate Benefits runs **one shared multi-tenant cbg3 web app** on
  `<employer-slug>.mitarbeiterangebote.de` (e.g. `lufthansa.`, `bosch.`, `api-gmbh.`,
  `jaeger-gruppe.`, `kaufpilot-privat.`, `demo.`, `ihre.`). Branding/categories/offers are
  per tenant, the code and session cookie (`CBG3FE`) are shared. `www.corporate-benefits.de` is
  only the employer-facing TYPO3 marketing site. **[verified]**
- **There is no public/unauthenticated offer catalog.** `/`, `/search?s=`, `/overview/<id>`,
  `/offer/<id>/cat/<id>` all redirect (302) to `/login` when not logged in. **[verified]**
- **Logged-in offer discovery is server-rendered HTML** behind three routes: category lists at
  `/overview/<categoryId>`, free-text search at `/search?s=<term>`, offer detail at
  `/offer/<id>/cat/<catId>`. Category ids are discovered from the home page nav, not hardcoded.
  No JSON API is used by the portal's own pages. **[verified in prior-art source + first-party
  bundle]**
- **Search is by brand/offer name, not by domain.** Matching a shop hostname to an offer means
  guessing a brand name from the host and matching it locally against a cached catalog, or
  against `/search?s=`. That is exactly what the existing CB extension does. **[verified in
  prior-art source]**
- **Prior art exists and the user's suspicion is right — it is close to a complete blueprint:**
  - `lucasfirl/benefits-chrome` → **CB Deal Finder** on the Chrome Web Store (published
    2026-08-13, v1.1.1, 2 users). Uses the logged-in session, weekly catalog download of
    `/overview/<id>` pages, local hostname→brand matching, no credentials stored. **[verified]**
  - `JustAPC/Discount_Check` (Firefox AMO + repo) → the same solution for the **Italian sister
    portal** `almaviva.convenzioniaziendali.it`, with a self-hosted API. Source is public and
    includes the full crawl/login/disclaimer logic. **[verified]**
  - `1110101/tampermonkey_personal_scripts` → **CorporateBenefits Toolkit** userscript:
    DOM-level sorting/filtering on `/overview/*`, `/search*`, `/offer/*`; confirms class names
    and the offer-page redirect overlay. **[verified]**
- ToS risk is explicit and high: §5 of the Nutzungsbedingungen declares offers/conditions/codes
  **strictly confidential trade secrets** that may not be shared with third parties, with
  contractual penalties (min. €500, repeat €2,000). A local-only, user-session design is the
  only defensible shape. **[verified]**

## 1. Portal URL patterns and tenancy

| Host | What it is | Status |
|---|---|---|
| `mitarbeiterangebote.de` | apex; 302 → `https://www.corporate-benefits.de` **[verified]** | corporate marketing |
| `<slug>.mitarbeiterangebote.de` | employer portal (same cbg3 app for all tenants) **[verified]** | the target |
| `ihre.mitarbeiterangebote.de` | live tenant, title “Attraktive Angebote für corporate benefits GmbH - Firmenkundenplattform-Mitarbeiter” **[verified]** | generic/own tenant |
| `demo.mitarbeiterangebote.de` | live tenant, title “… - Demo Plattform DE-Mitarbeiter”; used as the default privacy link of the Android app **[verified]** | demo tenant |
| `corporatebenefits.mitarbeiterangebote.de` | 404 today; Wayback has 2016–2021 snapshots with `/generic-link` URLs **[verified]** | retired tenant |
| `www.corporate-benefits.de` | employer-facing TYPO3 site (robots.txt, sitemap.xml, “Für Arbeitgeber/Anbieter”) **[verified]** | marketing |
| `app.` / `mobile.` / `assets.mitarbeiterangebote.de` | serve the same app (404 on `/`, same `CBG3FE` cookie) **[verified]** | app aliases |
| `static.mitarbeiterangebote.de` | 302 → corporate site **[verified]** | redirect |
| `api.mitarbeiterangebote.de` | 403 “Request forbidden by administrative rules” on `/`, `/health`, `/v1`, `/api`, `/graphql` **[verified]** | internal/unknown |
| `cdn.mitarbeiterangebote.de` | Myra CDN (`server: myracloud`, `x-mpex: yes`); public assets under `/v1/<section>/<tenant>/display/<hash>.<ext>` (e.g. `cbg-frontend`) **[verified]** | asset CDN |
| `fsc.mitarbeiterangebote.de` | “coming soon …” placeholder; `fsc.*` hosts appear in CSP for sister brands **[verified]** | fulfillment |
| `text.mitarbeiterangebote.de` | appears in tracking/adblock lists (email/pixel assets) **[claimed, multiple filter lists]** | email assets |

**One shared backend, per-tenant config.** Evidence: identical login page structure and
`CBG3FE` session cookie on every tenant; the same webpack bundle (`/js/app.js`) and CSS; the same
CDN tenant key `cbg-frontend` in asset paths across tenants; cross-tenant `generic-link` URLs
carry `ppU=<tenant>.mitarbeiterangebote.de`; and the tenant portal CSP whitelists the whole
international family in one policy. **[verified]**

Group brands sharing the platform (from the `lufthansa.` portal CSP header; all **verified**):
`*.mitarbeiterangebote.de`/`.at`, `fsc.mitarbeiterangebote.de`, `text.mitarbeiterangebote.de`,
`fsc.convenzioniaziendali.it`, `fsc.personelindirimi.com`, `fsc.rahmenvereinbarungen.at/.de`,
`fsc.benefitsatwork.{be,ch,club,com.tr,es,eu,lu,pl,pt}`. So the German portal is one tenant
family of a single European platform.

**No public tenant directory.** The corporate site only points to a contact form for the login
link (“Sie sind auf der Suche nach dem Log-in zu Ihrer Plattform?”). The user must know the
employer slug — CB Deal Finder makes the user enter and probes it before saving. **[verified]**

## 2. Offer discovery

### 2.1 Routes (all login-gated)

| Route | Purpose | Evidence |
|---|---|---|
| `/` | logged-in home; nav contains category links `/overview/<id>` | CB Deal Finder `syncCatalog()`; userscript `@match` **[verified in source]** |
| `/overview/<categoryId>` | full category offer list (server-rendered, **not paginated**) | CB Deal Finder README + code; Discount Check crawls the same **[verified in source]** |
| `/search?s=<term>` | free-text search over offers/brands | `buildSearchUrl()` in CB Deal Finder `common.js`; userscript `@match /search*` **[verified in source]** |
| `/offer/<id>/cat/<catId>` | offer detail page | Discount Check regex `/offer/(\d+)/cat/(\d+)/`; userscript `@match /offer/*` **[verified in source]** |
| `/generic-link?...` | signed outbound/tracking redirect (to shop or gift-card shop); fields `link`, `cbtt`, `tToken`, `cbta`, `ppU`, `ppS`, `ppI`, `ppC`, `sig`, `rdt`, `app=1` | Wayback CDX examples; a tampered call returns 200 “Ooops, something went wrong!” (signature rejected) **[verified]** |
| `/registration`, `/login`, `/meta/{imprint,privacy,termsofuse,accessibility}` | public | fetched 200 without login **[verified]** |

Unauthenticated probes on 2026-09-13 (`lufthansa.`): `/search?s=bosch`, `/overview/123`,
`/offer/123/cat/456` all return `302 Location: /login`; `/` also 302s to `/login`. **[verified]**

### 2.2 Offer list DOM (cbg3 classes)

From CB Deal Finder's production parser (`src/offscreen.js`) and the userscript:

- Item: `.cbg3-list-item[data-id]`
- Offer link: `a[href^="/offer/"]` (category pages nest `<a><h3>`, search results nest
  `<h3><a>` — read the `h3` text, not `h3 a`)
- Title: `h3` (campaign suffix after `" - "`, e.g. “Bosch Siemens Hausgeräte - August-Special”;
  brand = part before the dash)
- Description: `.cbg3-list-item--copy` (search) / `.cbg3-offer-list-item--content p`
- Discount: `.cbg3-list-item--discount`
- Category breadcrumb: `.cbg3-list-item--breadcrumb`
- Logo: `.cbg3-offerlistitem--supplierlogo img` (`src`/`data-original`)
- Category containers (userscript): `.cbg3-category`, `.cbg3-category--content`,
  `.cbg3-search--offers`

**[verified in prior-art source; class names cross-confirmed by two independent projects]**

### 2.3 Offer detail page

- Shop button: `.cbg3-icon--shop button[data-href]` — direct shop URL in `data-href`, sometimes
  wrapped as `/generic-link?link=<real URL>`; the button opens an interstitial overlay
  (`data-overlay-type`, `data-overlay-id`, `.cbg3-overlay--open`), which the userscript strips.
- Code request: `.cbg3-code-request` (userscript deliberately excludes it from the
  “open directly” rewrite; the handler exists in the first-party bundle as
  `cbg3-code-request`).
- Disclaimer overlay: `#cbg3-overlay--disclaimer`, form `#cbg-user-disclaimer--form`;
  `window.disclaimerConfirmed` is set in the page. **[verified in first-party bundle and prior art]**

### 2.4 Is there unauthenticated/public data?

- **No offer catalog, no sitemap, no public JSON.** Tenant `robots.txt` allows everything except
  `/meta/privacy`; there is no sitemap. Every content route redirects to `/login`. **[verified]**
- The portal's own bundle exposes only user-scoped `/api/*` endpoints (`bookmarks`,
  `user/location`, `postcode`, `form/contact-get`, `registration-code-resend`,
  `survey-handler`, `hint-app/generate-token`, `app/*`); `/api/locations` itself 302s to `/login`
  anonymously. **[verified in first-party bundle + probe]**
- Publicly fetchable without login: login/registration/meta pages and the per-tenant logo
  carousel on the login page (Myra CDN). Nothing offer-specific. **[verified]**
- Old platform remnants exist in the bundle (`/listings/category/`), but the live app is cbg3.
  **[inferred]**

## 3. Offer shape

What prior art extracts from a list item (**verified in source**, tests and production code):

```
{ id, title, brand, description, discount, category, logo, url }   // CB Deal Finder
{ c: categoryId, t: title, d: discount, h: shopHost, k: shop|giftcard|affiliate } // Discount Check
```

- **Discount text examples** (userscript parser): `15% Rabatt`, `< 7% Rabatt`, `> 25% Rabatt`.
  The offer-page variant class `cbg3-discount-and-location--uppercase` can also hold a distance
  (`ca. 15.4 KM`) — parsers must keep only the `%` value. Discount Check's matcher additionally
  accepts `€` and “gratis”, so fixed-amount and freebie formats occur at least on the Italian
  sister platform. **[verified in source / inferred for DE]**
- **Merchant domain**: only the offer detail page exposes it (`data-href`, possibly via
  `/generic-link?link=`). Discount Check unwraps `link=`, skips internal hosts and `/bookmark`
  links, and classifies gift-card vs affiliate vs normal shop. CB Deal Finder does **not** fetch
  detail pages — it matches by brand name only (cheaper). **[verified]**
- **Voucher codes**: a code-request control exists (`.cbg3-code-request`); neither extension
  extracts codes. Whether a code is shown inline or after a click is **[unverified]**.
- **Conditions / validity**: not present on list pages; presumably in the offer detail body.
  Neither prior-art project parses them, and no public write-up documents them. **[unverified]**
- **Loyalty/redirect layer**: shop clicks go through the portal (overlay and/or signed
  `/generic-link`), so a deep link straight to the shop may skip tracking. **[verified structure,
  inferred consequence]**

## 4. Session / login-state detection

- Login is a server-rendered form: `POST /login` with fields `loginData[email]`,
  `loginData[password]`, submit `cbg3-submit` (label localized: “Jetzt einloggen”). **[verified
  in fetched login HTML; identical construction in Discount Check’s Italian code]**
- Session cookie: `CBG3FE` — `HttpOnly; Secure; SameSite` default, `Max-Age=28800` (8 h), set
  already on the anonymous login page; the value rotates per session. **[verified from response
  headers; value redacted]** Because it is HttpOnly, extension JS cannot read it — but
  `fetch(url, { credentials: "include" })` from the extension’s service worker with host
  permission attaches it at the network layer, exactly like a normal tab (this is CB Deal
  Finder’s documented design). **[verified in prior-art source/docs]**
- Logged-out detection (CB Deal Finder, production-tested): a fetched page is “logged out” if
  the response was redirected to a different origin, to `/login|/signin|/anmelden|/logout`, or
  silently to `/`. Sanity check: presence of `/logout` in the HTML. **[verified in source]**
- Disclaimer gate: on first visit per session the portal shows `#cbg3-overlay--disclaimer` /
  `#cbg-user-disclaimer--form`; until accepted, offer pages can return the home page instead of
  the offer, silently emptying a crawl (documented for the Italian tenant). Discount Check
  accepts it with `POST /` and body `disclaimerAccept=1&cbg3-submit=<localized label>`;
  the userscript clicks `#cbg3-overlay--disclaimer #cbg3-submit`. **[verified in sources;
  German payload label unverified]**
- The bundle also exposes `window.disclaimerConfirmed` and `window.loginNowLabel`,
  usable as page-state hints. **[verified in fetched HTML]**
- No SSO/OAuth observed; access is gated by employer email domain or a registration code.
  **[verified on registration page]**

## 5. Prior art (the direct answer to “someone already solved this”)

### 5.1 CB Deal Finder — `lucasfirl/benefits-chrome` (DE, exact match)

- Chrome Web Store: <https://chromewebstore.google.com/detail/cb-deal-finder/kmijkgcnhgjbkjlfcijccgnfhailkdoj>
  (Offered by “Lucas F.”, v1.1.1, updated 2026-08-22, **2 users**, 0 ratings, declares no data
  collection). **[verified]**
- Source (no license declared): <https://github.com/lucasfirl/benefits-chrome> — created
  2026-08-13. **[verified]**
- How it gets data, from its own source/README:
  - User enters their portal origin (e.g. `firma.mitarbeiterangebote.de`); it is probed before
    saving; host permission is requested for that origin only. **[verified]**
  - **No credentials, no `cookies` permission.** Requests use `credentials: "include"` and the
    user’s existing session. **[verified]**
  - Catalogue sync: fetch `/`, extract category ids from `a[href*="/overview/"]`, then fetch each
    `/overview/<id>` with **1.5 s spacing**; store `{id,title,brand,description,discount,
    category,logo,url}` in `chrome.storage.local`; refresh weekly (alarm every 12 h, catalog max
    age 7 days). Back-offs: 30 min after a failed sync, 5 min after a detected logout. **[verified]**
  - Matching is **local**: brand candidates from the hostname (`philips.de` → “philips”) plus
    optional `document.title`/`og:site_name` (exact-only); normalized comparison with
    conservative prefix rules (short brands like “On” only match exactly). **[verified]**
  - If the catalog is missing/stale, it falls back to live `/search?s=<term>` per candidate
    (documented as ~2.4 requests/page view; explicitly rejected as the primary design). **[verified]**
  - It does **not** handle the disclaimer gate or accept `/generic-link` — it only links to the
    portal offer page. **[verified absence in source; a potential gap on German tenants, where
    the userscript proves the disclaimer exists]**
- Privacy policy: <https://github.com/lucasfirl/benefits-chrome/blob/main/PRIVACY.md> — local
  storage only, no developer servers, no analytics, title/`og:site_name` only. **[verified]**

### 5.2 Discount Check — `JustAPC/Discount_Check` (Italian sister brand, same platform)

- Repo: <https://github.com/JustAPC/Discount_Check> · Firefox AMO: `discount-check`, v1.0.10,
  2026-08-19, host permissions include `*.convenzioniaziendali.it` and
  `sconti-api.andreapontillo.it`; data collection declared: `authenticationInfo`. **[verified]**
- Same cbg3 app on `PORTAL = https://almaviva.convenzioniaziendali.it`. Its `background.js` is
  the most explicit public documentation of the logged-in flow:
  - `POST /login` with `loginData[email]`/`loginData[password]`, success = response contains
    `/logout`. **[verified in source]**
  - Disclaimer: `POST /` with `disclaimerAccept=1`; marker constant
    `cbg-user-disclaimer--form`. **[verified in source]**
  - Crawl: home → category ids from `/overview/(\d+)` → `/offer/\d+/cat/\d+` paths → parse each
    offer page: title `h1`, discount from `cbg3-discount-and-location--uppercase` (keep `%`,
    not “km”), shop host from `data-href` (unwrap `generic-link?link=`, skip internal hosts and
    `/bookmark`), classify `shop|giftcard|affiliate`. ~900 offers. **[verified in source]**
  - It **stores portal credentials** in `chrome.storage.local` (the owner’s trade-off; README
    says the credentials never enter the repo/package). Our design constraint is stricter
    (user-session only). **[verified]**
  - It shows a popup at checkout based on `location.hostname`, with its own domain/name/alias
    index (`etld1`, brand labels). **[verified]**
  - It also pushes Revolut/Klarna data through a self-hosted API — unrelated to CB.
- AMO listing: <https://addons.mozilla.org/en-US/firefox/addon/discount-check/>. A Chrome/Edge
  Store submission was pending per the README (Aug 2026). **[claimed]**

### 5.3 CorporateBenefits Toolkit — `1110101/tampermonkey_personal_scripts` (DOM only)

- Script: `CorporateBenefits Toolkit.user.js` (`@match https://*.mitarbeiterangebote.de/`,
  `/overview/*`, `/search*`, `/offer/*`). **[verified]**
- What it does: sort/filter offers by parsed discount, hide `.cbg3-ad`/`.cbg3-banner`, turn
  `button[data-href]` into a real link to bypass the redirect overlay, auto-accept the
  disclaimer (`#cbg3-overlay--disclaimer #cbg3-submit`). **[verified]**
- No data extraction or hostname matching; it confirms routes, classes and the redirect overlay
  described above. **[verified]**

### 5.4 What was **not** found

- No Greasyfork/OpenUserJS script for CB (searched; 0 relevant). **[verified]**
- No Home Assistant/py integration, no scraper repo named for CB beyond the filter-list noise
  (GitHub repo search `mitarbeiterangebote` → 6 results, none applicable). **[verified]**
- No dedicated AMO/Edge add-on for CB; AMO search “mitarbeiterangebote” = 0, “corporate
  benefits” returns only unrelated add-ons (`Discount Check` is the Italian one). **[verified]**
- No official Corporate Benefits browser extension; the official product is the mobile app
  (Play/iOS). **[verified absence after store searches]**
- No blog/reverse-engineering write-up found; the only community material is the mydealz ToS
  discussion (<https://www.mydealz.de/diskussion/corporate-benefits-neue-nutzungsbedingungen-2424876>)
  and general mydealz/Reddit threads about CB being good or bad value. **[verified]**

### 5.5 Adjacent (not CB, but same product pattern)

- `BenefitHub` (CWS, member-only employee benefits), `myWorld` plug-in, `Shoop`, `GutscheinDeal`,
  `Foraum Deals`, `mydealz Shopping`, `Rabattcorner`, etc. — different programs, no CB access.
  Useful only as UX precedent. **[verified listings]**

## 6. Mobile app

- Play Store: `de.corporatebenefits.app`, by “corporate benefits IT solutions GmbH”, 1M+
  installs, 3.5★, updated 2025-12-17; iOS versions exist per country (e.g.
  `corporate-benefits-benelux`). The app requires linking to the employer’s profile. **[verified]**
- The Play listing’s privacy-policy link points to
  `https://demo.mitarbeiterangebote.de/meta/privacy?os=android&lang=de` — i.e. the app’s privacy
  text is served by a normal tenant portal with an `os=android` parameter. **[verified]**
- Strong inference that the app is a WebView/skin over the same cbg3 platform and not a separate
  API: the first-party bundle contains `/api/app/*` (`notification-permissions`,
  `terms-of-use-confirmation`, `delete-app-voucher`), `/api/hint-app/generate-token`, and the
  signed `generic-link` accepts `app=1`. No public app API was found; an APK download attempt
  from APKPure failed (not an APK). **[inferred / unverified]**

## 7. Rate limits, ToS, and legal footing

- **No published rate limits and no anti-bot wall observed.** Tenant portals are plain nginx
  behind Myra CDN (`server: myracloud`, `x-mpex: yes`); no Imperva/Cloudflare challenge, no
  `Retry-After`. `robots.txt` disallows only `/meta/privacy` and there is no sitemap. **[verified]**
- Prior art deliberately self-limits to stay unobtrusive: one catalog download per week, 1.5 s
  between category fetches, 30-min retry back-off, live search only as fallback. CB Deal Finder’s
  README motivates this with scale (“at 1,000 users, ~2,000 requests/day instead of ~600,000”).
  **[verified]**
- **ToS (Nutzungsbedingungen, `/meta/termsofuse`, fetched anonymously):**
  - §4.1–4.3: access only for employees of enrolled companies; proof via company email domain or
    registration code; passing codes on publicly is expressly called out. **[verified]**
  - §4.4: contractual penalty of at least **€500**, at CB’s discretion, **€2,000 minimum for
    repeat** violations of the access/commercial-use clauses; further damages reserved. **[verified]**
  - §5.1–5.2: offers and conditions are **strictly confidential**, derived from corporate
    conditions, may only be made accessible to the entitled user circle, and “may not be
    communicated to third parties”; they are a **business secret** (Geschäftsgeheimnis), and
    violations may be prosecuted under the GeschGehG. **[verified]**
  - This is the central risk for any extension that caches or redistributes offer data. A design
    that (a) runs only with the user’s own session, (b) stores offers locally on the user’s
    device, (c) never sends them to the developer or third parties, and (d) never exposes codes
    outside the user’s own portal session, is the only shape consistent with §5. **Even then the
    legal analysis is the user’s/repo owner’s call — this doc is not legal advice. [inference]**
- The 2024 mydealz thread shows the confidentiality clauses are the subject of community
  attention: <https://www.mydealz.de/diskussion/corporate-benefits-neue-nutzungsbedingungen-2424876>
  **[claimed secondary]**

## 8. Concrete answer to the extension question

The minimal design that matches both prior art and the ToS:

1. User configures their tenant origin once (`<slug>.mitarbeiterangebote.de`); request host
   permission for exactly that origin (and optionally all sites only for automatic scanning).
2. Sync the catalog weekly with the user’s session (`credentials: "include"`): home → category
   ids from `/overview/<id>` → each category page → store offer `{id,title,brand,description,
   discount,category,logo,offerUrl}` locally. Detect logged-out via redirect to `/login`;
   handle the disclaimer gate if present.
3. On a merchant page, derive brand candidates from the hostname (+ optionally title /
   `og:site_name`), match locally against the cached catalog with conservative normalization.
4. On click, open the matching **portal offer page** in a tab (or deep-link the shop via the
   offer page’s own button), so the user stays in their authenticated portal context.
5. Optional live fallback: `GET /search?s=<brand>` for uncached brands/on-demand search.
6. Do not store credentials; do not extract/redistribute voucher codes or conditions; keep
   everything local.

CB Deal Finder is effectively this design already, so the pragmatic move is to study (and
possibly fork, licensing permitting — it has **no license file**, so default copyright applies)
rather than reinvent.

## 9. Still requires a logged-in capture to confirm

None of this was tested with an account (per the rules). To finish the picture:

1. **Exact German search/list HTML** — confirm `.cbg3-list-item[data-id]` and the `h3` nesting on
   a German tenant; confirm whether `/search` supports anything besides name substrings (domains,
   aliases, SKUs).
2. **Merchant-domain exposure in lists** — does any list attribute (not just the offer page)
   carry the shop domain? If yes, domain matching gets exact.
3. **Offer detail fields** — voucher-code UX (`.cbg3-code-request`), conditions text, validity
   dates, fixed-amount (€) and “gratis” formats, exact `data-href`/`generic-link` shape.
4. **Disclaimer gate on German tenants** — exact overlay text and POST payload
   (`disclaimerAccept=1` + which `cbg3-submit` label); whether CB Deal Finder’s missing handling
   actually breaks a crawl.
5. **Catalog size/categories per tenant** — number of `/overview/<id>` categories and offers
   (Italian tenant: ~900) to size the weekly sync; whether category ids are stable over time.
6. **Rate-limit behavior** under modest load, and whether the server ever challenges (403/429)
   non-tab fetches from an extension.
7. **Session lifetime/rotation** — after login, does `CBG3FE` rotate and what is the real idle
   timeout?
8. **Mobile app** — whether `api.mitarbeiterangebote.de` (403 at the edge) is the app API and
   whether the app is a WebView; requires an APK/device capture.

## References

- Corporate Benefits marketing site: <https://www.corporate-benefits.de/> (robots:
  <https://www.corporate-benefits.de/robots.txt>)
- Tenant portals: <https://lufthansa.mitarbeiterangebote.de/login> (also `bosch.`, `api-gmbh.`,
  `jaeger-gruppe.`, `kaufpilot-privat.`, `demo.`, `ihre.` variants), `/meta/termsofuse`,
  `/meta/imprint`, `/meta/privacy`, `/registration`, `/js/app.js`
- Chrome Web Store — CB Deal Finder:
  <https://chromewebstore.google.com/detail/cb-deal-finder/kmijkgcnhgjbkjlfcijccgnfhailkdoj>
- GitHub — CB Deal Finder source: <https://github.com/lucasfirl/benefits-chrome>
  (`src/common.js`, `src/background.js`, `src/offscreen.js`, `PRIVACY.md`, `README.md`)
- GitHub — Discount Check: <https://github.com/JustAPC/Discount_Check> (`background.js`,
  `server/README.md`)
- Firefox AMO — Discount Check: <https://addons.mozilla.org/en-US/firefox/addon/discount-check/>
- GitHub — CorporateBenefits / MitarbeiterVorteile userscripts:
  <https://github.com/1110101/tampermonkey_personal_scripts>
- Google Play — corporate benefits app: <https://play.google.com/store/apps/details?id=de.corporatebenefits.app>
- Apple App Store — corporate benefits (Benelux example):
  <https://apps.apple.com/lu/app/corporate-benefits-benelux/id1402508907>
- mydealz — Corporate Benefits ToS discussion:
  <https://www.mydealz.de/diskussion/corporate-benefits-neue-nutzungsbedingungen-2424876>
- Wayback CDX — retired tenant with `generic-link` examples:
  `http://web.archive.org/cdx/search/cdx?url=corporatebenefits.mitarbeiterangebote.de*`

## Logged-in capture (2026-09-13, sanitized)

Session made with a remote browser on the agent host (user signed in). Raw captures stay local only.

- Offers require accepting the disclaimer overlay first: `#cbg3-overlay--disclaimer`, hidden `disclaimerAccept=1`, newsletter checkbox `platformData[disclaimerNewsletter]` (left off), submit `cbg3-submit` ("Akzeptieren"). Acceptance is per account/session.
- Search is by **brand name**, not domain: `GET /search?s=<name>`. Results in `.cbg3-list-item[data-id]` with `h3` title and `.cbg3-list-item--discount` badge (e.g. `17% RABATT`); item link `/offer/<id>/cat/<catid>`.
- Offer detail `/offer/<id>/cat/<catid>` shows title, discount badge, description, FAQ/conditions prose, and a confidentiality note.
- Outbound link lives in `#saleoptions`: a button whose `data-href` is an **opaque tracked deeplink** (observed pointing at a voucher-fulfilment partner), opened in an iframe overlay (`data-overlay-type="external-link-i-frame"`). The merchant domain appears only in prose — no structured domain field. Hostname matching therefore needs a curated name→domain map.
- Home feed cards carry a discount chip (`10% Rabatt`) and a `ZUM ANGEBOT` link.
- Not observed: rate limits, explicit validity dates, fixed-amount offers.

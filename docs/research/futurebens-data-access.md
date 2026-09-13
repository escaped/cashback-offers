# Research: FutureBens data access for a browser extension

Ticket: [#8](https://github.com/escaped/cashback-offers/issues/8) · Date: 2026-09-13

Question: how can a Chrome extension obtain FutureBens' offer for a merchant identified by a
website hostname, while the user is logged in?

Method: unauthenticated public pages inspected with `curl` (browser UA) — futurebens.co
Webflow pages, brand/company/rabattcode pages, sitemaps, robots.txt, terms, app privacy
policy; DNS via DNS-over-HTTPS + crt.sh; store searches on Chrome Web Store, Edge Add-ons,
Firefox AMO and chrome-stats; GitHub code/repo/issue search via `gh`; public MemberStack
docs. No account, no credentials, no auth bypass. Claims are marked **[verified]**
(observed directly in a first-party page/response), **[inferred]** (deduced from
artifacts), **[claimed]** (third-party statement).

## TL;DR

- futurebens.co is a **Webflow CMS site with MemberStack auth**, not a classic SPA/API
  app. Offer content (brand pages, per-company portals, even live discount codes) is
  **served in public HTML**; MemberStack only cosmetically shows/hides UI. **[verified]**
- Public catalog: **166 brand pages** `/brands/<slug>` containing structured hidden
  inputs — Brand Name, Discount Code, Brand Website URL, cover image — plus free-text
  conditions and the discount headline (`-50%`, `75€ Bonus`, `Voucher`). **[verified]**
- **897 company portals** `/companies/<slug>` expose the same ~167-offer list
  (paginated 100 + 67 via `?<hash>_page=2`). A member's `company-url` custom field routes
  them to their company portal. **[verified]**
- **167 `/rabattcode/<slug>` SEO pages** show discount, "Gültig bis" validity, redemption
  steps and embed the live code in inline JS; the "enter your business email" step is a
  client-side reveal. **[verified]**
- Merchant→offer mapping comes from the brand page's `Brand Website URL` (direct merchant
  URL with UTM, or affiliate links: Awin, Impact `*.pxf.io`, `eclick`). There is **no
  hostname field on offer cards**. **[verified]**
- Login: MemberStack (work email + password, or private email + registration code),
  Cloudflare Turnstile + reCAPTCHA; **no SSO observed**. Mobile app `co.futurebens` uses
  email OTP against a locked PHP API on Hostinger (`api/app/auto.futurebens.co`; all
  routes 401/405). **[verified website; mobile backend inferred]**
- **Prior art: nothing for FutureBens anywhere public.** Chrome Web Store, Edge Add-ons,
  Firefox AMO, chrome-stats and GitHub code search return no extension/userscript/scraper.
  Closest analogs are official extensions for *other* benefits platforms (Reward Gateway
  SmartSpending, ID.me Shop, Blue Light Card). **[verified]**
- ToS forbids giving third parties access, reproducing/redistributing the service,
  reverse-engineering or deriving the API, and archiving data not belonging to you; brand
  pages declare codes confidential. A logged-in read of the user's own offer is defensible;
  anonymous bulk crawl + republish is not. **[verified]**

## 1. What is behind futurebens.co

- FutureBens GmbH (Berlin) — closed-loop employee-discount platform, 1,600+ employers,
  200+ brands, employee conditions "up to -60%". **[verified:**
  <https://www.futurebens.co/>, <https://en.futurebens.co/>, Play listing**]**
- **Stack: Webflow (CMS + hosting, Cloudflare CDN) + MemberStack (auth + client-side
  visibility) + Weglot (i18n de/en) + GTM/Apollo.** Every page loads
  `https://api.memberstack.io/static/memberstack.js?webflow` and forms use
  `data-ms-form="login"` / `data-ms-member`. **[verified]**
- Hosts from crt.sh + DNS-over-HTTPS:
  - `www`, `en` → Webflow/Cloudflare. `members.` and `wf.` CNAME to
    `proxy-ssl.webflow.com` (standard Webflow proxy hosts). **[verified]**
  - `api.`, `app.`, `auto.` → `82.198.228.146` (Hostinger, LiteSpeed,
    `x-powered-by: PHP/8.2.33` resp. `8.3.33`) with JSON error envelopes. `api.` returns
    `401 {"error":"Unauthorized"}` on **every** path; `app.` returns
    `404 {"error":"Route Not Found","error_code":"route_not_found"}`; `auto.` returns
    `405 {"error":"Method Not Allowed"}` to GET. **[verified]**
- **Data source [inferred]:** brand pages carry a hidden `Memberstack ID` whose value has
  Airtable record-ID shape (`rec…`); offer fields look Airtable-managed, synced into
  Webflow CMS. No public Airtable/JSON endpoint found.
- Mobile app: package `co.futurebens` (Google Play, updated 2026-09-07; "coming in
  September") and App Store `id6797576756`. Register with work email **or access code**;
  login via **email one-time code (OTP)**; session token in OS secure storage; Firebase
  FCM. **[verified: Play listing + `/datenschutz-app`; the app's base URL is inferred]**

## 2. Offer data (all public, no login)

### Brand pages — the primary data source

`/brands/<slug>`: **166 URLs** in the sitemap. Example
<https://www.futurebens.co/brands/gymondo> contains **[verified]**:

- discount headline in `discount-wrap` markup (`-50%`, `75€ Bonus`, `50€ Bonus`,
  `Voucher`, `Test` variants)
- hidden inputs (exact names):

  | field | content |
  |---|---|
  | `Discount Brand ID` | internal offer ID |
  | `Discount Brand Name` | brand display name |
  | `Discount Code` | live voucher code (when the offer has one) |
  | `Memberstack ID` | Airtable-shaped `rec…` ID |
  | `Brand Website URL` | merchant URL with UTM **or** affiliate tracking link |
  | `Brand Cover Image URL` | asset URL |

- the code also appears in `data-clipboard-text` and in inline JS
  (`const code = 'FUTURE…'`) for some brands. Live values are redacted here.
- free-text conditions, e.g. *"Der Rabattcode kann nur online unter
  https://www.gymondo.com/de/packages/futurebens eingelöst werden. Das Angebot gilt nur
  für Neukunden, ist nur einmalig verwendbar und kann nicht mit anderen Aktionen
  kombiniert werden."*
- category, pricing level, origin, brand description, feedback widget.

Offers with codes coexist with "Discount anfordern" flows: general code copied
client-side, or a **personal code requested and emailed** (e.g. "Der Rabattcode wurde an
die angegebene E-Mail-Adresse gesendet"). **[verified]**

### Company portals

- `/companies/<slug>`: **897 URLs** in the sitemap. Title pattern
  "Deals for `<Company>` employees". The full offer-card list is in the initial HTML
  (Webflow CMS, Finsweet filter/load). Demo portal
  <https://www.futurebens.co/companies/futurebens-friends>: 100 cards on page 1, 67 on
  page 2 (`?75ce9b81_page=2`), no overlap → **167 offers**. **[verified]**
- Cards link to `/brands/<slug>` and show only name, category, image, discount; no
  merchant domain. Company-specific CMS variants (`w-condition-invisible`) exist but the
  first 100 cards fetched anonymously for two different companies were identical.
  **[verified]**
- `/dashboard` is also public Webflow HTML with the same card data. Its inline script
  reads the MemberStack member and rewrites links from the member's `company-url` custom
  field, defaulting to `/companies/futurebens-friends`. **[verified]**
- Company directory: `/find-signup` renders all companies client-side (Finsweet filter);
  each has `/signup/<slug>`, `/log-in/<slug>`, `/hr-flyers/<slug>`. **[verified]**

### Rabattcode pages

- `/rabattcode/<slug>`: **167 URLs**; SEO landing pages with discount (`-18%`), validity
  ("Gültig bis September 2026"), FAQ JSON-LD, redemption steps, conditions, and the live
  code embedded in inline JS (`Get_Unique_Value`, `Handle_General`). Entering a business
  email triggers a **client-side reveal**; the page already contains the code.
  **[verified]**
- A client-side cookie (`unique_requests` / `requests_date`) disables the reveal form
  after **5 codes per day per browser**. **[verified]**

### Offer shape summary

| dimension | observed form |
|---|---|
| discount | percentage (`-N%`), fixed (`N€ Bonus`), free-text (`Voucher`, `50€ Bonus`) |
| code | plain string (hidden input / clipboard / inline JS); some offers have none |
| conditions | free text: channel, minimum, new customers, one-time, no stacking, exclusions |
| validity | `Gültig bis <month year>` on rabattcode pages; not always on brand pages |
| extras | category, pricing segment, origin, cover image, FAQ, redemption instructions |

## 3. Merchant / hostname mapping

- Offer cards have **no domain field**; mapping lives on brand pages via
  `Brand Website URL`. Observed variants **[verified]**:
  - direct merchant URL with UTM, e.g. `achat-hotels.com/?utm_source=partner…`,
    `ostrom.de/?utm_source=futurebens…`
  - merchant landing subpath, e.g. `gymondo.com/de/packages/futurebens`
  - Awin: `awin1.com/cread.php?awinmid=…&awinaffid=…&campaign=B2B-FUTUREBENS-…&ued=<merchant>`
  - Impact: `airalo.pxf.io/…`
  - eclick tracking: `partnerships.armedangels.com/trck/eclick/…`
- So an index build must unwrap affiliate links (resolve redirects; parse Awin `ued=`)
  and reduce to the registrable host. **[inferred approach]**
- There is **no single affiliate/benefits feed**: FutureBens negotiates direct deals and
  routes outbound clicks through several networks plus direct UTM links. **[verified]**
- Buildable index: sitemap → 166 brand pages → `Brand Website URL` (unwrap) → hostname →
  `{brand, slug, discount, code, conditions}`. ~166 requests; XML `lastmod` available for
  incremental refresh. **[inferred]**

## 4. Login flow, session shape

- **Web auth = MemberStack.** Signup supports "Unternehmens-E-Mail" (company email) or
  "Private E-Mail + Registrierungs-Code"; login is email + password. Every form carries a
  Cloudflare Turnstile sitekey; the login page also loads reCAPTCHA; an anti-spam delay
  guards submissions. **No SSO (Google/Microsoft) button observed.** **[verified]**
- Member custom fields drive the product: `company-url` (portal routing),
  `company-location` (logged into signup forms). **[verified in page source]**
- **Session shape (generic, no values):** MemberStack issues a JWT; MemberStack's own docs
  show backend verification reading it from the `_ms-mid` cookie. The site sets session
  duration 720 hours (~30 days) via `ms_settings` in the page source. Site code uses the
  legacy `MemberStack.onReady(member => member["company-url"])`. **[verified / documented]**
- **Mobile:** email OTP only; token stored in OS secure storage (Keychain/Keystore);
  device/app metadata + FCM token collected. **[verified: `/datenschutz-app`]**
- `api.futurebens.co` requires auth for every route; no usable public JSON API exists.
  **[verified]**

## 5. Login-state detection (generic)

- Preferred: in page context on `www.futurebens.co`, call the MemberStack DOM API —
  `window.$memberstackDom.getCurrentMember()` returns the member or `null`; custom fields
  via `member.customFields` (e.g. `company-url`). Documented by MemberStack:
  <https://docs.memberstack.com/hc/en-us/articles/8517501248539-Check-if-a-Member-is-Logged-in-or-Logged-out>
  and <https://docs.memberstack.com/hc/en-us/articles/17242974908699-Member-Management-with-the-Memberstack-DOM-Package>.
- Fallbacks that need no API: the logged-out nav renders a "Login" link (mirrored by
  `data-ms-content="!members"`), `#memberstack-js` is present on every page, and
  `MemberStack.onReady` exists once the SDK boots. **[verified]**
- Do **not** read, store or transmit tokens/cookies — no need; member ID/company slug
  suffice for routing, and even that can stay in-page.

## 6. Rate limits / anti-bot

- Cloudflare fronts `www` (`cf-ray`, `_cfuvid`); Webflow CDN caches pages
  (`cf-cache-status`, `surrogate-control`). No published rate limits; treat as a normal
  website and cache aggressively. **[verified]**
- `/rabattcode` reveal form: client-side 5/day cookie (trivially reset); server-side
  limit unknown. **[verified]**
- `api.futurebens.co` is fully locked (401 on all paths) — consistent with a global
  app-key/token middleware; using it would require extracting embedded app credentials,
  which the ToS explicitly forbids ("attempt to obtain or derive the API"). **[verified +
  ToS]**

## 7. ToS / legal notes

- `/nutzungsbedingungen` (§4) prohibits: renting/leasing/sub-licensing/selling or
  **giving third parties access** to the service; reproducing/modifying it; **reverse
  engineering, decompiling or attempting to obtain/derive the source code or the API**;
  **combining the service with other software/data**; **storing data that does not belong
  to you in an archive database**. **[verified]**
- Platform copy repeatedly states employee conditions and discount codes are
  **confidential and must not be shared with third parties outside the platform**.
  **[verified]**
- `robots.txt`: only `/for-companies` disallowed; sitemap published. Not a licence — the
  ToS govern use. **[verified]**
- Practical line: showing the logged-in user their own FutureBens offer (read in their
  browser) is defensible; anonymously crawling the whole catalog and republishing codes
  conflicts with the ToS/confidentiality notices. Personal-code automation is out.

## 8. Prior art

- **No FutureBens extension/userscript/scraper exists publicly.**
  - chrome-stats search "futurebens" → only the iOS app + publisher, 0 extensions
    (<https://chrome-stats.com/search?q=futurebens>). **[verified]**
  - Chrome Web Store and Edge Add-ons searches → no FutureBens item. **[verified]**
  - Firefox AMO `q=futurebens` → 0 results. **[verified]**
  - `gh search code futurebens | "futurebens.co" | "co.futurebens"` → only job ads,
    résumés and an AI-citation corpus mentioning the platform; `gh search repos
    futurebens` → 0. No tooling. **[verified]**
- **Analogous prior art (other providers, same problem):**
  - **Reward Gateway "SmartSpending"** — official extension (Chrome 20k users; Firefox
    AMO v1.3.0): "Sign in using the same details you use to sign in to your Discounts
    website… if an online discount is available by the retailer, we'll show you a
    notification." The closest validated pattern to this project.
    <https://chromewebstore.google.com/detail/smartspending/akjlnngigpbekgfjcbpohdfnfjfgflpb>,
    <https://addons.mozilla.org/en-US/firefox/addon/smartspending/> **[verified]**
  - **ID.me Shop** — surfaces community/employee discounts by hostname.
    <https://chromewebstore.google.com/detail/idme-shop-discover-commun/iifmmpcbkkjplbamhfohikljoogdbadp>
    **[verified]**
  - **Blue Light Card** — employer/affiliation discounts while browsing. **[verified]**
- Third-party context: an AI-citation corpus samples an indexed FutureBens code page
  (`en.futurebens.co/rabattcode/kapten-son`), confirming these pages are public and
  crawlable. **[claimed]** A Medium design-bootcamp case study describes the product.
  **[claimed]**

## 9. Viable approaches for the extension

- **A — Anonymous catalog index (cheap, no auth):** crawl `/brands/<slug>` (166 pages),
  build `hostname → {brand, discount, code, conditions}`, match at runtime, deep-link to
  the FutureBens brand page. Fully public; but caching/republishing codes conflicts with
  the confidentiality notice and ToS §4.
- **B — Logged-in session read (preferred for compliance):** content script on
  `www.futurebens.co` detects the member via `$memberstackDom.getCurrentMember()`, reads
  `company-url`, fetches the same-origin company + brand pages in the user's browser, and
  surfaces the user's own offer on merchant sites. No credentials or tokens leave the
  page; content is already server-rendered HTML so no private API is needed. **[inferred
  from verified page structure + MemberStack docs]**
- **C — Mobile app API (no):** locked 401, would need embedded client credentials /
  API derivation — explicitly prohibited by ToS §4.
- **D — Personal code requests (no):** if a code is generated/emailed per user, deep-link
  the user to the brand page instead of automating the request.

## 10. Unknowns — need a logged-in capture to confirm (no values recorded)

1. Whether the logged-in brand page/card shows company-specific or personal codes or
   discounts that differ from the public HTML (only generic variants seen anonymously).
2. Whether `/companies/<slug>` content actually changes after login (two companies were
   identical anonymously; gating may happen entirely client-side).
3. Which of the 167 offers each of the 897 companies actually sees, and whether any are
   hidden by plan/company rules.
4. Attributes of the MemberStack session cookie (`_ms-mid`: HttpOnly/SameSite) and
   whether `getCurrentMember()` works from an extension content script's MAIN world.
5. Whether `api.futurebens.co` uses a static embedded app key or per-session bearer
   tokens, and its real rate limits.
6. Existence/terms of a private API for the "personal discount code" request flows.

## Sources

- <https://futurebens.co/> · <https://en.futurebens.co/> · <https://www.futurebens.co/login> ·
  <https://www.futurebens.co/find-signup> · <https://www.futurebens.co/dashboard> ·
  <https://www.futurebens.co/companies/futurebens-friends> ·
  <https://www.futurebens.co/companies/futurebens-friends?75ce9b81_page=2> ·
  <https://www.futurebens.co/brands/gymondo> · `.../brands/{adidas,airalo,achat-hotels,armedangels,ostrom,arket}` ·
  <https://www.futurebens.co/rabattcode/achat-hotels> ·
  <https://www.futurebens.co/sitemap.xml> · <https://www.futurebens.co/robots.txt> ·
  <https://www.futurebens.co/nutzungsbedingungen> · <https://www.futurebens.co/datenschutz-app> ·
  <https://www.futurebens.co/app>
- <https://play.google.com/store/apps/details?id=co.futurebens> ·
  <https://apps.apple.com/ua/app/futurebens-mitarbeiterrabatte/id6797576756>
- chrome-stats: <https://chrome-stats.com/search?q=futurebens> ·
  <https://chrome-stats.com/d/id6797576756>
- MemberStack docs: `getCurrentMember`, member management, `_ms-mid` token verification,
  logged-in check (links in §5)
- SmartSpending: <https://chromewebstore.google.com/detail/smartspending/akjlnngigpbekgfjcbpohdfnfjfgflpb> ·
  <https://addons.mozilla.org/en-US/firefox/addon/smartspending/> ·
  ID.me Shop: <https://chromewebstore.google.com/detail/idme-shop-discover-commun/iifmmpcbkkjplbamhfohikljoogdbadp>
- GitHub searches via `gh search code|repos|issues futurebens` (results: job ads, résumés,
  citation corpus; no tooling)

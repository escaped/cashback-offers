# Research: "Benefits Buddies" / "Benefit Buddies" data access for a browser extension

Project: cashback-offers (Chrome MV3, hostname → offer lookup) · Date: 2026-09-13
Method: public web only. DNS/RDAP lookups, direct HTTP inspection (curl + real Chrome via agent-browser, no login),
public app-store listings, EUIPO/trademark records, GitHub code search (`gh search code`), Chrome Web Store /
Firefox AMO search, and third-party deal communities (mydealz). No credentials used, no auth bypassed.

Claims are marked **[verified]** (observed directly in a response/DOM/registry), **[inferred]** (from a first-party
artifact such as the app's own JS/privacy text), **[third-party]** (claimed by someone else, not independently
confirmed), or **[unverified]**.

## TL;DR

- **No German platform named "Benefits Buddies" exists.** `benefitsbuddies.de`, `benefitbuddies.de` and
  `benefits-buddies.de` do not resolve (NXDOMAIN, checked 2026-09-13). **[verified]**
- The name collisions are foreign/unrelated: `benefit-buddies.com` is a US "employee benefits advocacy" (healthcare
  navigation) GoDaddy website-builder site marked "Launching soon"; `benefitbuddies.com` is a parked NameBright domain
  registered 2025-07-31. Neither sells offers/discounts. **[verified]** The UK "Benefits Buddy" (HWWA Consulting,
  `benefitsbuddy.co.uk`, `BENEFITSBUDDY LTD`) is a pensions/insurance benefits hub, not a cashback/discount portal.
  **[third-party]** A UK "BENEFIT BUDDIES CIC" (Doncaster charity) also exists. **[third-party]**
- The **only German platform matching the name is cadooz's `BenefitBuddy`** (singular), operated by **cadooz GmbH,
  Hamburg** (HRB 102734; part of Euronet/epay). It is a **consumer gift-card/voucher/deal app** — cadooz's own FAQ
  calls its users "BenefitBuddies" ("von BenefitBuddies für BenefitBuddies"), the most plausible source of the user's
  phrase. **[verified]** EUIPO word mark "Benefit Buddy" No. 018901343, owner cadooz GmbH. **[third-party]**
- BenefitBuddy is **app-only**. `app.benefitbuddy.de` is a landing page that says "Exklusive Deals – Nur verfügbar in
  der App" (even emulated as an iPhone). There is no web shop and no web login. **[verified]**
- Its vouchers are branded **"BestChoice BenefitBuddy"** and are redeemed on the **public, login-free** cadooz catalog
  `bestchoice-ace-catalog.cadooz.com`. That catalog lists acceptance-partner names/logos only — **no discount rates,
  no prices, no domains**. **[verified]**
- **A browser extension cannot obtain BenefitBuddy's per-merchant discount rates from public web data.** The rates live
  in the app; the public catalog only says *where* a voucher can be redeemed. There is no public domain→brand mapping.
  Practical options are limited to deep-linking, a curated name→hostname map, or (grey) the mobile app API.
- It is **not employer-gated**. If the user's access really is employer-gated, BenefitBuddy does not match and the
  platform is probably a different (possibly cadooz-powered) portal — see §6 "What a logged-in capture must confirm".

## 1. Disambiguation: which "Buddies" platform?

| Name / domain | What it is | Evidence |
|---|---|---|
| `benefitsbuddies.de`, `benefitbuddies.de`, `benefits-buddies.de`, `benefitsbuddies.eu`, `benefitbuddies.co.uk` | **Do not exist** (no DNS) | `getent hosts` NXDOMAIN, 2026-09-13 **[verified]** |
| `benefitbuddies.com` | Parked domain; NameBright/TurnCommerce registrar; registered 2025-07-31; serves a `/lander` redirect | RDAP `rdap.verisign.com/com/v1/domain/benefitbuddies.com`; HTTP response **[verified]** |
| `benefit-buddies.com` | US **employee benefits advocacy** service (help navigating health/benefits; "Launching soon"); GoDaddy Website Builder 8.0; `og:description` "Launching soon!" | https://benefit-buddies.com/ + RDAP (GoDaddy) **[verified]** |
| `benefitsbuddy.co.uk` / Google Play `com.hwwaconsulting.benefitsbuddy` | **UK** "Benefits Buddy" by HWWA Consulting / BENEFITSBUDDY LTD, London — benefits *document hub* (pensions, insurance, wellbeing), no merchant offers | https://benefitsbuddy.co.uk/ ; https://play.google.com/store/apps/details?id=com.hwwaconsulting.benefitsbuddy **[third-party]** |
| `benefit-buddies` (CIC), "Money Buddies / Benefit Buddies" (Leeds) | UK charities/advice services, unrelated | https://okredo.co.uk/en-gb/company/benefit-buddies-cic-17398878 ; https://moneybuddies.org.uk/services/benefit-buddies/ **[third-party]** |
| `benefitbuddy.de` / `app.benefitbuddy.de` | **Germany — cadooz BenefitBuddy**, the voucher/deal app. cadooz GmbH, Osterbekstraße 90b, 22083 Hamburg, HRB 102734, part of Euronet/epay | https://www.benefitbuddy.de/impressum/ ; https://www.benefitbuddy.de/ **[verified]** |
| `benefit-buddy.de` | Separate host (IONOS IP `2001:8d8:100f:f000::200`), did not respond to HTTP(S) probes | DNS + curl **[verified absence]** |
| `bestchoice-ace-catalog.cadooz.com` | Public redemption catalog for the "BestChoice BenefitBuddy" voucher | Browser session (no login) **[verified]** |

**Assessment (inference):** the German employee almost certainly means **cadooz BenefitBuddy**, or a cadooz-powered
portal. There is no distinct German product called "Benefits Buddies". Confidence: **medium** for BenefitBuddy;
**low** for any other reading. The premise "employer-gated" conflicts with BenefitBuddy's consumer model — see §6.

## 2. BenefitBuddy: product, domains, login shape

- **Operator**: cadooz GmbH (Hamburg), an incentives/benefits group (25+ years, part of Euronet Worldwide, sister of
  epay). BenefitBuddy app listed on Apple App Store (`id6737117696`) and Google Play (`de.benefitbuddy.app`).
  **[verified]** `https://www.benefitbuddy.de/impressum/`; app-store listing pages.
- **Domain layout** (all checked 2026-09-13):
  - `www.benefitbuddy.de` — WordPress marketing site (theme author "W3 digital brands GmbH"; content by cadooz).
    FAQ, app-store links, support `support.benefitbuddy@cadooz.de`. No login, no offers. **[verified]**
  - `app.benefitbuddy.de` — Nuxt 3 SPA (`window.__NUXT__.config.public.appUrl = "https://app.benefitbuddy.de"`).
    Routes `/` and `/login` both render only `WebLandingPage`: heading "Exklusive Deals - Nur verfügbar in der App"
    plus App Store / Play download links (tested desktop and iPhone-12 emulation). **[verified]**
  - `api.benefitbuddy.de` — HTTP redirect to `https://app.benefitbuddy.de/`. **[verified]**
  - All `/api/*` paths on `app.benefitbuddy.de` return the SPA shell (catch-all), i.e. no Nitro API is exposed there.
    **[verified]**
- **Login/session shape**: there is **no web login**. Deals are only in the mobile app; registration/account creation
  happens in-app ("App runterladen und Konto erstellen"). Payment methods per FAQ: credit card (Mastercard/Visa),
  Apple Pay, Google Pay. **[verified]** The app privacy policy names **Adjust** (mobile attribution), **Google
  Firebase** (push), **Cookiebot** (consent) and payment processors **Transact / Worldline Saferpay** as processors.
  **[inferred from first-party privacy text bundled in the web app]**
- **Backend**: no API host is exposed in any public web asset; the mobile app talks to a cadooz backend that is not
  discoverable without app-binary inspection (APK mirrors blocked this research: APKPure returned 403; Aptoide has no
  listing). **Unknown by design.**

## 3. Offer discovery and offer shape

### 3.1 Public cadooz catalog (the only public web surface found)

- **URL pattern**: `https://bestchoice-ace-catalog.cadooz.com/frontend/cat/view.do`
  with query params `view=custom_view`, `lt=default`, `sortBy=alpha`, `ptg=vou` (vouchers; `ptg=all`, `ptg=phy` for
  physical rewards), optional `shpMthd=`, `categoryId=`. It is a Java web app (`JSESSIONID`, `ROUTEID`; noindex).
  **[verified]**
- **Access**: publicly browsable in a real browser with **no login and no voucher code** (asked for nothing). Plain
  curl gets an Imperva Incapsula challenge — the host is bot-protected. **[verified]**
- **Content**: an alphabetical list of acceptance partners, rendered as `<div class="shopitem">` → `<img>` +
  `<h5 class="shopitemname">`. Examples: 43einhalb.com, adidas DE, Douglas DE, H&M DE, IKEA DE, Zalando DE, Zalando,
  zooplus DE, Tchibo, Thalia DE … (full BenefitBuddy partner list mirrored by mydealz). **No price, no discount %,
  no validity, no domain, no link per partner** in the public listing. **[verified]**
- The catalog's own GTC page (`/frontend/view.do?path=%2Fshop%2Fagb`, "as at February 2013") describes the
  *redemption* flow: enter BestChoice voucher code → choose partner vouchers → checkout. No clause about automated
  access/scraping was found in the rendered text. **[verified]**
- Same software is used for sibling catalogs (`ace-catalog`, `premium-catalog`, `plus-catalog`, `bc-aktion-catalog`,
  and category catalogs `style-beauty-`, `fit-healthy-`, …), which the prior-art userscript lists. **[third-party]**
- There is **no public search API by brand or domain**. The catalog's search/view state is driven by `.do` query
  params; partner matching is by display name only. **[verified]**

### 3.2 What the offers actually are (app side)

BenefitBuddy sells **discounted gift cards / vouchers**, not percentage-off promo codes on a merchant's own site:

- Product families: BestChoice vouchers in fixed denominations (branded "BestChoice BenefitBuddy"), brand gift cards
  (Zalando, IKEA …), experiences (MovieChoice), and some physical rewards. **[third-party, mydealz; consistent with
  catalog `ptg=vou`/`ptg=phy`]**
- Typical economics reported by the deal community:
  - 10–12% off BestChoice vouchers (e.g. €50 for €45, "50 € + 5 € bonus" promos). **[third-party]**
  - Brand-level examples: Zalando gift cards ~10–11%; Quirion 12% vs 8% normal; Douglas 12%; zooplus 12%; Globetrotter
    12%; MovieChoice 2 tickets €12–14 vs €16. **[third-party, https://www.mydealz.de/deals/benefitbuddy-app-by-cadooz-mehr-rabatte-als-sonst-zb-2x-moviechoice-fur-12eur-statt-16eur-2500914]**
  - ALDI Nord 5% BestChoice Aktion (redeemable to ALDI credit). **[third-party]**
- Conditions/validity are on the **brand voucher**, not the platform: e.g. Zalando max €200 voucher redemption per
  cart, 5-year validity, "Fashion" assortment only; ALDI credit requires printing, 3-year validity; cadooz GTC: the
  accepting partner is issuer/debtor, no returns on voucher redemptions. **[third-party; cadooz GTC verified]**
- The app also shows "Exclusive product deals" (partner sale offers) and has a "Deal Tracker". **[verified from
  app-store description]**

### 3.3 The practical blocker for a hostname-driven extension

- **No public domain→merchant mapping exists.** The catalog and app use **display names** ("Zalando DE", "IKEA DE"),
  occasionally domain-like names ("43einhalb.com", "Herrenausstatter.de", "myMuesli"). Same problem as any
  name-based portal; the prior-art userscript solves it with a hand-curated normalize()+override map (§5).
- **No public per-merchant discount data.** The public catalog lists redemption partners only; the discount rate for
  buying a voucher is displayed in the app after registration/login. **[verified for the catalog; app behaviour
  inferred from third-party reports]**
- Possible extension strategies, in order of legality/robustness:
  1. **Deep link / education**: detect a brand on the partner list and open the app store page or `benefitbuddy.de`
     for the user to check the app. No offer data. (Honest, zero-risk.)
  2. **Curate rates from public deal feeds** (e.g. mydealz) — no official API; ToS and accuracy caveats.
  3. **Mobile app API** — reverse-engineering the app and replaying a logged-in session is credential-dependent,
     likely against the platform's ToS, and outside this research's rules. Not recommended.

## 4. Session/login-state detection (generic, no values captured)

- **BenefitBuddy web**: nothing to detect. `benefitbuddy.de` is static marketing; `app.benefitbuddy.de` serves the
  same landing page to every visitor and sets no user session. The only cookies observed on cadooz hosts are Imperva
  (`visid_incap_*`, `nlpi_*`, `incap_ses_*`) and — on the catalog — `JSESSIONID` + `ROUTEID`. **[verified]**
- **cadooz catalog**: no user login; the "credential" is the **BestChoice voucher code** entered manually. The
  `JSESSIONID` is a server session for the cart/redemption flow, not an identity. **[verified to the extent that the
  listing loaded anonymously; the redemption POST flow was not exercised]**
- If a future logged-in capture shows a web session (cookie/localStorage token), generic detection is: read a
  first-party marker only via `chrome.cookies`/`fetch(..., {credentials:'include'})`, never export secrets, and treat
  absence-of-login as the default state. (No such marker found for this platform in this research.)

## 5. Prior art

### Browser extensions

| Project | Platform | Notes |
|---|---|---|
| [BenefitHub: Your Wallet Will Thank You](https://chromewebstore.google.com/detail/benefithub-your-wallet-wi/gpinpagnbmllflbnipojcnnkdhmemcio) | BenefitHub (US/global employee-benefits platform) | **Official member-portal extension, ~10k users.** Does exactly the target UX: alerts on visited shops for member discounts, cashback, gift-card savings; requires an existing membership. Also on [Firefox AMO](https://addons.mozilla.org/en-US/firefox/addon/benefithub-your-wallet-will/) and Edge. Closest UX precedent, different operator. |
| [ysamjo/cashback-optimizer-popup](https://github.com/ysamjo/cashback-optimizer-popup) (userscript v5.12, updated 2026-04) | Multi-portal (DE) | **Most relevant prior art.** A Tampermonkey script that pops up on any shop page, matches the hostname to a shop name (normalized comparison + `HOST_OVERRIDES` for otto.de, netto-online.de, g-star.com, …) and links to cashback portals. Its `directLinks` map contains `"Benefit Buddy": "https://www.benefitbuddy.de/"`; its `bcLinks` map contains `"BestChoice BenefitBuddy"` plus 18 sibling cadooz catalogs. It **link-outs**, it does not fetch per-merchant offers. `@connect cashback-optimizer.de`. |
| [cashback-optimizer.de](https://www.cashback-optimizer.de/) | Multi-portal (DE) | Companion comparison site by the same author; static HTML (1.6 MB), no public API (`/api`, `/shops.json` → 404). |
| CouponBuddy, ShopBuddy USA, general cashback extensions | various | Generic "discount/cashback popup" pattern; none related to BenefitBuddy. |

No extension for BenefitBuddy, cadooz or BestChoice was found on the Chrome Web Store, Firefox AMO or Edge store
(searches 2026-09-13). **[verified absence via websearch]**

### GitHub code search

- `benefitsbuddies`, `benefit-buddies`, `benefitbuddies` → no relevant code. **[verified]**
- [bonusly/cadooz](https://github.com/bonusly/cadooz) — Ruby wrapper for cadooz's **B2B SOAP**
  `BusinessOrderService` (`https://webservices.cadooz.com/services/businessorder/1.5.2/BusinessOrderService/BusinessOrder?wsdl`),
  username/password + program/payment credentials. The only public cadooz API reference; B2B, not the app. **[verified]**
- [invoice-collector](https://github.com/invoice-collector/invoice-collector/blob/master/src/collectors/sketch/incentive_mall_by_cadooz/incentive_mall_by_cadooz.ts)
  — collector for cadooz **Incentive Mall** (`incentivemall.cadooz.com/mall/history.do`, username/password login URL).
  Another B2B cadooz portal. **[verified]**
- JDownloader `CadoozVoucherCrawler` (mirrors) parses `ecard.cadooz.com/frontend/ecard.do?id=…` public e-card pages.
  **[verified]**
- `sma6871/dealbot` logs mydealz deal titles mentioning BenefitBuddy (a mydealz scraper, not a BenefitBuddy client).
  **[verified]**

### Press / community / registry

- EUIPO word mark **"Benefit Buddy"** 018901343 by cadooz GmbH, filed 2023-07-14, registered 2023-10-27.
  **[third-party, trademarkelite.com]**
- cadooz corporate: [cadooz.com](https://www.cadooz.com/), Employee Benefit Club, BestChoice product line; parent
  Euronet/epay. **[third-party/verified on cadooz.com]**
- mydealz threads documenting BenefitBuddy deals and the "BestChoice BenefitBuddy" vs "BestChoice Classic" partner-set
  difference (e.g. Rossmann available in Classic but not BenefitBuddy): e.g.
  https://www.mydealz.de/deals/benefitbuddy-app-bestchoice-classic-50eur5eur-gutschein-cadooz-ikea-conrad-hm-thalia-ca-usw-2595460
  **[third-party]**

## 6. ToS / rate limits / unknowns

- `robots.txt`: `www.cadooz.com` and `benefitbuddy.de` both allow all crawlers with `Crawl-delay: 10`
  (Yoast default). **[verified]** The catalog host's `robots.txt` is itself behind Imperva and not readable
  anonymously. Catalogue pages serve `NOINDEX, NOFOLLOW` under the bot challenge. **[verified]**
- The catalog is behind **Imperva Incapsula**; automated POSTs/parallel requests will be challenged. Treat any
  catalog use as low-volume, human-paced, and link-out rather than scrape. **[verified]**
- App ToS: the marketing site's "Nutzungsbedingungen" only govern the *website* (no contract formed by visiting);
  the app's contract terms and any API terms are only reachable after registration → **unverified**. No public
  clause about automated access was found.

## 7. What a logged-in capture must confirm

1. **Which platform it really is**: hostname + login page the user sees. If it is an employer portal (SSO, company
   email, voucher code from HR), it is not BenefitBuddy — likely Corporate Benefits (`mitarbeiterangebote.de`),
   benefits.me, Shopbuddies, or a cadooz white-label "Vorteilswelt". The user's exact wording/branding will settle
   this.
2. **App login mechanics** (if BenefitBuddy): email+password vs OTP/magic link, token storage, refresh.
3. **App offer feed**: the request(s) fired when browsing/searching a merchant in the app (likely a JSON catalog with
   brand names, discounted price and face value) — needed before any per-merchant offer can be shown.
4. **Whether any web variant exists for a logged-in account** (e.g. app.benefitbuddy.de showing more after auth, or a
   `web.` host) — currently every route renders the download landing page.
5. **In-app terms** for automated use and any documented rate limits.
6. **Brand→domain coverage**: which names in the app/catalog are matchable to hostnames, and what the "Deals"
   (product deals) look like versus the voucher catalog.

## Reproduce (commands, all public)

```bash
# Name/domain disambiguation
for d in benefitsbuddies.de benefitbuddies.de benefits-buddies.de benefitbuddies.com benefit-buddies.com \
         benefitsbuddy.co.uk benefitbuddy.de; do getent hosts "$d"; done
curl -s https://rdap.verisign.com/com/v1/domain/benefitbuddies.com
curl -sL https://benefit-buddies.com/ | head -c 2000

# BenefitBuddy surfaces (no login)
curl -sL https://www.benefitbuddy.de/ | head -c 1500          # WordPress marketing site
curl -sIL https://api.benefitbuddy.de/ | grep -i location      # -> app.benefitbuddy.de
# app.benefitbuddy.de: Nuxt SPA; every route renders the "download the app" landing page (verified in Chrome)

# Public cadooz catalog (needs a browser; curl gets Imperva)
# https://bestchoice-ace-catalog.cadooz.com/frontend/cat/view.do?view=custom_view&lt=default&sortBy=alpha&ptg=vou
#   DOM: .shopitem > img + h5.shopitemname (partner names only)
#   GTC: .../frontend/view.do?path=%2Fshop%2Fagb

# Prior art
gh search repos benefitbuddy
gh search code "benefitbuddy.de"
gh search code "cadooz.com"
curl -s https://raw.githubusercontent.com/ysamjo/cashback-optimizer-popup/main/cashback-optimizer.user.js

# robots
curl -s https://www.cadooz.com/robots.txt
curl -s https://benefitbuddy.de/robots.txt
```

No credentials were used and no personal data captured. This note is the only artifact of the research.

## Update (2026-09-13): app-only confirmed by user

The user's "Benefits Buddies" access is app-only — there is no web portal or logged-in web session to integrate. Per the map, the fallback for this platform is **cashback-optimizer.de** (discovery/fallback only, never as authority).

# Research: Shoop Cashback-Assistent — browser-extension UX

Ticket: [#5](https://github.com/escaped/cashback-offers/issues/5) · Branch: `research/shoop-extension-ux` · Date: 2026-09-13

Question: what exactly does Shoop's own extension do on a partner-shop page, so the offer-panel prototype can mirror the UX the user likes?

Artifact inspected: **Shoop Cashback & Gutscheine, Chrome MV3, v3.2.17.2** — CRX fetched 2026-09-13 from Google's public update endpoint (extension id `hacngjmphfcjdfpmfmlngemhddjdncpe`). The bundle is the shipped product, so manifest/CSS/UI-string evidence is primary for behavior, not a write-up.

## Sources

| Ref | Source | Notes |
|-----|--------|-------|
| S1 | Chrome Web Store listing — <https://chromewebstore.google.com/detail/shoop-cashback-gutscheine/hacngjmphfcjdfpmfmlngemhddjdncpe> | Store description; rating 3.95 (411 reviews); 100k users; v3.2.17.2 |
| S2 | Official product page — <https://www.shoop.de/cashback-assistent> | Fetched via `r.jina.ai` relay (Cloudflare blocks direct fetch); rendered snapshot dated 2025-09-25 |
| S3 | Official help article 127 — <https://www.shoop.de/hilfe-support/answer/127> | Fetched via `r.jina.ai` relay; icon-colour legend |
| S4 | Shipped CRX bundle v3.2.17.2 | `manifest.json`, `content/bundle.js`, `content/styles.css`, `bg/bundle.js`, `popup/bundle.js`, `popup/styles.css`, `images/*` |
| S5 | Opera add-ons listing — <https://addons.opera.com/de/extensions/details/shoopde-cashback-assistent/> | Same description, "blue Shoop logo top right", click to activate |
| S6 | Honey: help article 39 — <https://help.joinhoney.com/article/39-what-is-honey-and-how-do-i-get-it>; PayPal guide — <https://www.paypal.com/us/money-hub/article/guide-to-using-paypal-honey> | |
| S7 | iGraal: <https://fr.igraal.com/extension>; Chrome listing — <https://chromewebstore.google.com/detail/igraal-cashback-codes-pro/kmhkepipobnjllejbafajoemahjejdcm> | |
| S8 | TopCashback: US help — <https://www.topcashback.com/help/browser-extension-about/>; UK FAQ — <https://www.topcashback.co.uk/help/topcashback-browser-extension/>; UK landing — <https://www.topcashback.co.uk/browser-extension/> | |

## 1. Surfaces

Shoop uses **two surfaces in Chrome — a toolbar popup and an in-page top-right notification** — plus search-result annotations. There is **no system/desktop notification**: the manifest requests `tabs, storage, webRequest, unlimitedStorage, alarms` but **not** `notifications` [S4].

- **Toolbar icon (top-right of the browser toolbar).** Four static states, colour-coded, with a text badge [S3][S4]:
  - grey (`default`) — not a partner / no cashback;
  - blue (`merchant`) — cashback available;
  - green (`activated`) — cashback activated;
  - orange (`inactive`) — cashback inactive / only non-combinable vouchers (pixel-sampled from `images/default.png`, `merchant.png`, `activated.png`, `inactive.png`).
  - Badge text shows the rate teaser (e.g. `Bis zu 5%` trimmed to 3 chars, blue `#00509C`) while the merchant state is blue/green, and the number of vouchers on non-partner pages [S4].
  - Help says the icon "blinks" on a partner page [S2][S3]. The shipped bundles contain no animation code and 22 unused `images/animation/*.png` frames — **not verified** how/whether the blink still runs in v3.2.17.2; the colour change + badge is the verifiable signal.
- **Toolbar popup** — clicking the icon opens `popup.html`, set at runtime via `chrome.action.setPopup`; the panel is **360 × 600 px**, anchored under the icon (top-right) [S4]. Official mock: `https://files.shoop.de/toolbar-landing-page/cashback-aktivieren.png` [S2].
- **In-page notification** — a Vue app injected by a content script matched at `<all_urls>`, `document_end`; root class `.shoop-de-notification` is `position: fixed; top: 10px; right: 10px; z-index: 2147483646`, card width **310 px**, with a slide-from-top animation [S4]. It appears automatically on merchant pages (no click needed); the root component draws its slider on mount [S4].
- **In-page settings modal** (gear) — fixed centred overlay; toggles: SERP hints, partner-page popups, non-partner popups, anonymous analytics, interstitial page, plus mute lists [S4].
- **Search-result annotation** — a small "X% Cashback" tooltip injected next to Google results (`shoop-de-serp__tooltip`), toggleable in settings [S4].
- **Optional interstitial** — "Zwischenseite mit Informationen zu Cashback-Raten und Gutscheinen anzeigen" (default on): an intermediate Shoop page between click and merchant [S4].
- **No in-page floating button** on the merchant page (the TopCashback-style big pink button is not Shoop's pattern); the in-page element is the top-right card only [S4].

## 2. Trigger and lifecycle (partner page)

- Content script runs at `document_end`; once the merchant lookup resolves, the notification component picks **one** card by priority: PinTutorial → Alternative → SuggestedMerchants → RateUs → merchant notification [S4].
- The merchant card **auto-hides after 5 s** (`setHideTimer`, 5000 ms), and the timer is **paused while the pointer hovers** it and restarted on mouse-leave [S4].
- Closing with the **X** hides it and applies a **1-hour mute for that merchant** (`hide(..., shouldMute: true)` → `mute` with strategy `"hour"`) [S4].
- The **eye / "Nicht mehr anzeigen"** row opens a mute panel: **"24 Stunden deaktivieren"** or **"Dauerhaft deaktiveren"** [S4].
- Alternatives card on non-partner pages: close calls `muteAlternatives` for that page and the card is shown at most **3 times per merchant** (`countShown < 3`, counter persisted per session) [S4].
- Muted merchants live in storage as `{id, timestamp: "forever" | epoch}` — a per-site memory with expiry [S4].
- Logged-out users can still mute, but a warning card says **"Melde dich an, um deine Einstellungen zu speichern"** — settings don't persist without login [S4].

## 3. Content of the two surfaces

**In-page merchant notification (310 px, white, Inter font, top-right)** [S4]:
- Shoop logo, close X, shop logo + shop name;
- cashback line, e.g. "4% Cashback" / "Bis zu 10€ Cashback" (click = activate) and a flame icon if a deal is running;
- "+ N Gutscheine" row (click = voucher list);
- "Nicht mehr anzeigen" eye row.
- After activation the card turns **green `#82B201`**, full 310 px card with title and a warning/confirmation line ("Du hast Cashback erfolgreich aktiviert…"). Login required for activation; a separate **`merchant-unlogged`** variant shows the shop + cashback and routes activation to login.

**Toolbar popup (360 × 600)** [S4]:
- header: Shoop logo, **"Partner suchen"** search field, **balance** ("Kontostand 1.452,50 €");
- current merchant row: shop logo, name, **favourite heart**;
- rate card: e.g. "4% Cashback", primary button **"Cashback aktivieren"** (pink/red `#FD3259`; green `#82B201` when active), "Details anzeigen";
- voucher cards: code/description, copy button, combinable ("Kombinierbar mit Cashback") vs non-combinable (orange) badge, "Gutscheine ausprobieren" (auto-apply);
- bottom navigation tabs: **Partner · Aktionen · Lieblingsshops · Einstellungen** (home / flame / heart / gear);
- "Beliebteste Partner", "Aktionen deiner Lieblingsshops", conditions/terms, and error states.
- Logged out: the popup still opens and shows rates; favourites show **"Um deine Lieblingsshops sehen zu können, musst du dich zuerst einloggen."** with an **"Anmelden"** action [S4].

**Non-partner page** [S4]:
- in-page card: **"Hier gibt es leider kein Cashback, aber du kannst bei diesen Partnern sparen:"** with a short alternative-merchant list (48 px logos, name, cashback line, "Zum Partner" button), plus mute options **"Nur für diesen Partner deaktivieren" / "Alle Vorschläge ausschalten"**; toolbar badge shows a count for these alternatives.

## 4. Interaction model

- **Click-to-activate, never auto-activate.** Official copy: "Mit nur einem Klick aktivierst du dein Cashback" [S1][S2][S5]; activation is tracked as a click ("Lediglich wenn du auf 'Cashback verdienen' klickst, vermerken wir einen Klick" [S3]).
- **Coupons**: offered in both surfaces, with an **auto-applier** that tries codes and applies the best one ("Wir probieren gerade Gutscheine für dich aus…") [S2][S4].
- Activation requires login; logged-out users get login prompts and a non-persistent preference warning [S4].
- Dismiss memory is per merchant, with 1 h / 24 h / forever strategies [S4].

## 5. Logged-out and non-partner pages

| State | Toolbar icon | In-page | Popup |
|-------|--------------|---------|-------|
| Partner, logged in | blue → green when activated | merchant card, click to activate | full panel with balance/favourites |
| Partner, logged out | blue (merchant detected) | unlogged merchant card, activation → login | rates visible, login prompts for favourites/activation |
| Non-partner | grey (or orange if cashback inactive/NCV-only) | alternatives card + mute; max 3 shows | current merchant/vouchers still queryable |
| Not a shop / disabled | grey | nothing | normal panel |

## 6. How iGraal / TopCashback / Honey differ (brief)

- **Honey (PayPal)** — coupon-first, no cashback activation: the "h" icon top-right turns orange when a site is supported, green when coupons are found, with a red count badge (Safari); the coupon window **auto-pops at checkout** and "Apply Coupons" auto-tests/applies the best code. Rewards instead of cashback; Droplist price tracking and Amazon seller comparison. No click-to-activate step and no per-site rate panel [S6].
- **iGraal** — closest to Shoop: cashback **reminder to activate** on partner merchants ("L'extension vous rappelle d'activer le cashback…"), discount alerts while browsing, and **automatic coupon tests on the cart page**; up to 30% cashback. Store/help copy confirms the reminder + alert behaviour but does **not** describe the surface's exact placement (toolbar vs in-page) — mark exact placement **unverified** [S7].
- **TopCashback** — hummingbird icon **flashes** on eligible stores; clicking the logo opens its pop-up with an **"Activate Now"** button; on eligible visits a **"big pink get cash back" button** is triggered; a second prompt appears at checkout to apply coupon codes; Google results are annotated. **Must be logged in to activate** — logged out, "Activate Now" routes to the sign-in page. That is the same click-to-activate model as Shoop, but with an explicit "activate now" CTA both in the toolbar popup and as an in-page alert [S8].

## 7. Recommended pattern for the offer-panel prototype

Mirror Shoop's capture + consent model, adapted to a six-platform aggregator ("links only, no silent redirect"):

1. **Two surfaces, one anchor — top-right.** Primary surface = an in-page card, `position: fixed; top: 10px; right: 10px`, ≈310–360 px wide, slide-from-top, auto-dismiss ~5 s with hover-pause. Secondary = toolbar action badge/icon (rate teaser or offer count) whose click opens the full panel. No system notifications.
2. **Panel content, in priority order:** merchant name + logo → **best offer first (normalized € estimate + rate, raw value shown)** → one primary action button → other platforms as compact rows (platform, rate, "open" link) → details/conditions. This maps directly onto Shoop's "merchant row → rate card → vouchers list" hierarchy.
3. **Click-to-activate only.** One primary button opens the chosen platform's deeplink in a new tab; never auto-redirect. Show "activate on {platform}" state in the card after the click.
4. **Per-site memory + dismiss.** X = snooze that merchant (Shoop: 1 h); an explicit "don't show again" row with 24 h / forever (per merchant) and "disable alternatives for this site" (Shoop's "Nur für diesen Partner" / "Alle Vorschläge ausschalten"). Persist in `storage`, keyed by hostname.
5. **Non-partner pages:** show the alternatives card ("no cashback here, but these platforms have offers for similar shops" — in our case: this merchant is on platform X/Y). Cap repeat shows (Shoop: 3) and badge the icon with the offer count.
6. **Logged-out / gated platforms:** keep showing public rates; make the activation button route to login for gated platforms (Miles & More, PAYBACK account offers, employer portals) and label rows "log in to see"; never hide the panel.

Skipped for the prototype: coupon auto-applier, interstitial page, SERP annotations, PinTutorial/RateUs/analytics nudges — all Shoop engagement extras, not core to the aggregator.

## 8. Unverified / limitations

- Exact "blink" animation of the toolbar icon in the current version (official docs say it blinks; no animation code found in the bundle) — [S2][S3] vs [S4].
- Exact delay between page load and the in-page card (draws on mount; merchant lookup is async) and whether it re-shows on SPA navigation.
- iGraal's exact surface placement (toolbar vs in-page banner) — not described in the sources checked [S7].
- Reviews were not mined beyond the store rating; the store screenshots are third-party "related extension" promos, not Shoop UI — the official mock in §1 is the reliable visual.

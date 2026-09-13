# Cashback offers extension

A Chrome MV3 extension that matches the hostname of the shop the user is visiting against six offer platforms — Shoop, Miles & More, PAYBACK, Corporate Benefits, FutureBens, Benefits Buddies — and shows the available offers in a top-right panel.

## Language

**Merchant**:
The shop a user is visiting or that an offer is for (e.g. Zalando).
_Avoid_: shop, brand, partner

**Platform**:
One of the six offer programs the extension queries.
_Avoid_: portal, provider

**Offer**:
What a platform presents for a merchant — a cashback rate, discount, voucher, or points reward.
_Avoid_: deal, coupon, rebate

**Offer page**:
A platform's own page for one offer — Shoop merchant page, Miles & More partner page, PAYBACK shop page, Corporate Benefits offer page, FutureBens brand page.
_Avoid_: shop page, deeplink

**Tracked link**:
An affiliate or redirect URL that credits a platform when followed. The extension never mints or links to one; credit is triggered by the user on the offer page.
_Avoid_: affiliate link, deeplink

**Possible match**:
A merchant match made on normalized name alone, rendered muted and never counted as the platform's offer.
_Avoid_: fuzzy match, guess

**Platform session**:
The user's logged-in session on a gated platform (in v1, Corporate Benefits only), read in their own browser. Credentials are never stored; when the session is absent or expired, the panel says "Not logged in" and links the platform login.
_Avoid_: login, cookie

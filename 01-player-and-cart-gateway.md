---
layout: default
title: Player & Cart Gateway
nav_order: 2
---

# Loading the Player & the Cart Gateway

The **player** is the heart of Live Me Up: a full-screen video experience (live or replay) with chat and
shoppable products. Every other widget (Mini Player, Pop Me Up, Playlist, Live Shopping Page) is just a
different way to **launch** it.

This guide covers the two things every integration needs, regardless of which widget you build:

1. **Loading & starting the player** — pulling in the player script and calling its public JS API.
2. **The cart gateway** — telling the player how to add products to *your* cart.

> **Production URLs** (see the [overview](README.md) for the full list):
> Player ES module → `https://cdn.livemeup.io/player.esm.js`

---

## 1. Loading the player

The player ships as a standard ES module that exports a small public API. You do **not** add a `<script>`
tag up front; instead you dynamically `import()` it the moment the user actually wants to watch, so it
costs nothing on initial page load.

### 1.1 The public API

The module exports these functions:

```ts
// Launch the full player for a single live event or replay.
startPlayer(eventId
:
string, params ? : PlayerParams
):
void

// Launch the player in playlist mode over a media collection (see the Playlist guide).
  startPlayerCollection(mediaCollection
:
MediaCollection, params ? : MediaCollectionParams
):
void

// Close / tear down the player.
  stopPlayer()
:
void
```

`PlayerParams` (all fields optional):

```ts
type PlayerParams = {
  // Storefront language as an ISO code, UPPER-CASED (e.g. "EN", "FR").
  language?: string;
  // Storefront country as an ISO code (e.g. "US", "FR").
  country?: string;
  // Start playback at this offset, in MILLISECONDS (used to jump to a product highlight in a replay).
  t?: number;
  // End playback at this offset, in MILLISECONDS.
  e?: number;
  // Pre-fill the viewer's chat nickname.
  chatNickname?: string;
  // Show the close (X) button. Defaults to true.
  showCloseButton?: boolean;
  // Share button configuration. Defaults to { active: true }.
  share?: { active: boolean };
  // Called when the viewer closes the player.
  onClose?: () => void;
};
```

`MediaCollection` / `MediaCollectionParams` (used by `startPlayerCollection`, see the [Playlist
guide](04-playlist-widget.md)):

```ts
type MediaCollection = {
  mediaCollectionId: string;  // the collection's id
  startIndex: number;         // which item to open first (0-based)
};

type MediaCollectionParams = {
  language?: string;
  country?: string;
  chatNickname?: string;
  showCloseButton?: boolean;
  share?: { active: boolean };
  onClose?: () => void;
};
```

`eventId` **must be a string** — it is either a live event id or a replay id. Both are launched the same
way.

### 1.2 Launching it

```js
function openLiveMeUpPlayer(eventId, options = {}) {
  import('https://cdn.livemeup.io/player.esm.js')
    .then(({ startPlayer }) => {
      startPlayer(eventId, {
        language: 'EN', // storefront language ISO code, upper-cased
        country: 'US',  // storefront country ISO code
        ...options,     // e.g. { t: 42000 } to start 42 seconds (42000 ms) in
      });
    })
    .catch((err) => console.error('Failed to load Live Me Up player', err));
}

// e.g. on a button click:
document
  .getElementById('watch-live')
  .addEventListener('click', () => openLiveMeUpPlayer('THE_EVENT_OR_REPLAY_ID'));
```

The player injects its own full-screen overlay (inside a Shadow DOM, so it will not clash with your
site's CSS) and manages its own React tree. You only ever call `startPlayer` / `startPlayerCollection` /
`stopPlayer`.

**Important — `language` and `country`.** These drive currency and translations. Pass your storefront's
active language/country so prices and copy match what the shopper sees elsewhere. If you omit `country`,
prices fall back to the store's default currency.

Where to get the values: in the standard Shopify theme widgets these come straight from Shopify's
`localization` object — `language` is `localization.language.iso_code` **upper-cased** (e.g. `"EN"`,
`"FR"`) and `country` is `localization.country.iso_code` (e.g. `"US"`, `"FR"`). On a headless storefront,
use the **same market/locale the shopper has currently selected** — i.e. the exact `country` and
`language` values you already pass to Shopify's Storefront API `@inContext(country:, language:)` directive
when fetching prices. Reusing those guarantees the player's currency matches the rest of your store.

### 1.3 Where do event / replay ids come from?

You never hard-code ids. Each widget reads them from a data source:

- **Currently-live event:** the shop metafield `app--5813997--current_live` / key `event_infos` (see
  the [Mini Player guide](03-mini-player-widget.md)).
- **Upcoming events & published replays:** the Live Shopping Page data — either the metafield-backed theme
  block or the `GET https://api.livemeup.io/v2/landing-page` endpoint (see the [Live Shopping Page
  guide](02-live-shopping-page.md)).
- **Replay for a specific product:** the product metafield
  `app--5813997--last_replay_with_product_highlighted` (see the [Pop Me Up guide](05-pop-me-up-widget.md)).
- **A curated playlist:** the `media_collections` metafield (see the [Playlist guide](04-playlist-widget.md)).

---

## 2. The cart gateway

When the player runs inside a normal Shopify theme it talks to Shopify's AJAX cart (`/cart/add.js`, etc.)
automatically — **you don't need to do anything.** On a **headless / custom** storefront that endpoint is
not available, so you must give the player your own cart implementation.

### 2.1 How the player picks a gateway

On the **first** call to `startPlayer`, the player builds its internal container and chooses a cart
gateway by inspecting `window`. The very first thing it checks is:

```js
if (window.LiveMeUpCartGatewayV4) {
  // use this object as the cart gateway
}
```

So the integration contract is simply:

> **Assign an object implementing `CartGatewayV4` to `window.LiveMeUpCartGatewayV4` _before_ the first
> `startPlayer` call.**

This is read once, when the container is built. Setting it after the player has already started has no
effect, so set it as early as possible (e.g. at app bootstrap).

> **Set it on _every_ page where the player can be launched.** `window.LiveMeUpCartGatewayV4` lives on the
> `window` object, which is wiped on every full page navigation. The player can be opened from many places
> (a product page, the Pop Me Up widget, the Mini Player, a "watch live" button in the header, …), so the
> gateway must be registered on each of those pages **before** the shopper can trigger `startPlayer`.
>
> - **Multi-page / server-rendered site:** register the gateway in a script that runs on every page (or at
    > least every page that can open the player) — not only on the homepage.
> - **Single-page app (SPA):** the `window` global survives client-side route changes, so setting it once at
    > app startup is enough. Just make sure it runs before the first possible `startPlayer` call.

### 2.2 The `CartGatewayV4` interface

```ts
type CartGatewayV4 = {
  // Add one variant to the cart. Returns the created line id.
  addToCart(params: AddToCartParamsV4): Promise<AddToCartResponseV4>;

  // Return the current cart contents. May return undefined if there is no cart yet.
  getContent(): Promise<CartProductV4[]> | undefined;

  // Set absolute quantities for given lines (quantity 0 = remove the line).
  updateQuantities(products: UpdateQuantitiesParamsV4[]): Promise<void>;

  // Tag/attach the originating video id to a cart line (for sale attribution).
  flagLineForSaleTracking(line: FlagLineParamsV4): Promise<void>;

  // Send the shopper to checkout.
  validateCart(): Promise<void>;

  // Marked optional in the type for backward-compatibility, but you SHOULD implement it.
  // Apply discount/promo codes to the cart. Without it, FLASH-SALE DISCOUNTS WILL NOT APPLY
  // (the shopper is charged full price even when a live flash sale is running).
  addPromoCodes?(codes: string[]): Promise<void>;

  // Marked optional in the type for backward-compatibility, but you SHOULD implement it.
  // Attach a Live Me Up user id to the cart. Without it, SALE ATTRIBUTION IS BROKEN
  // (purchases can't be tied back to the viewer / live session for analytics).
  tagCartWithUserId?(userId: string): Promise<void>;
};

type AddToCartParamsV4 = {
  cmsProductId: string;  // Shopify product GID, e.g. "gid://shopify/Product/123"
  cmsVariantId: string;  // Shopify variant GID, e.g. "gid://shopify/ProductVariant/456"
  videoId: string;       // the live/replay video id the add originated from
  quantity: number;
};

type AddToCartResponseV4 = {
  lineId: string;        // your cart's identifier for the created line
};

type UpdateQuantitiesParamsV4 = {
  lineId: string;
  cmsProductId: string;
  cmsVariantId: string;
  quantity: number;      // absolute target quantity (0 = remove)
  videoId?: string;
};

type FlagLineParamsV4 = {
  lineId: string;
  cmsProductId: string;
  cmsVariantId: string;
  quantity: number;
  previousVideoId?: string;
  videoId: string;
};

type CartProductV4 = {
  cmsProductId: string;  // product GID
  cmsVariantId: string;  // variant GID
  videoId?: string;
  quantity: number;
  lineId: string;
};
```

**Which methods are required vs. optional?** `addPromoCodes` and `tagCartWithUserId` are marked optional
on the type purely for backward-compatibility with older integrations — **you should implement both.**
Skipping them does not crash the player, but it silently disables features:

| Method                                                        | If you skip it…                                                                         |
|---------------------------------------------------------------|-----------------------------------------------------------------------------------------|
| `addToCart`, `getContent`, `updateQuantities`, `validateCart` | Core cart will not work — these are mandatory.                                          |
| `flagLineForSaleTracking`                                     | Per-line video sale attribution is lost (see 2.4). Implement if possible.               |
| `addPromoCodes`                                               | **Flash-sale discounts never apply** — shoppers pay full price during live flash sales. |
| `tagCartWithUserId`                                           | **Sale attribution is broken** — purchases can't be linked to the viewer/live session.  |

### 2.3 ID format — important

The player always speaks in **Shopify GIDs**, not numeric ids:

- Product: `gid://shopify/Product/<numericId>`
- Variant: `gid://shopify/ProductVariant/<numericId>`

If your cart API needs numeric ids, strip the prefix; when returning data to the player, re-wrap them:

```js
const PRODUCT_GID = 'gid://shopify/Product/';
const VARIANT_GID = 'gid://shopify/ProductVariant/';

const toNumericVariant = (gid) => gid.replace(VARIANT_GID, '');
const toVariantGid = (id) => (String(id).startsWith(VARIANT_GID) ? id : VARIANT_GID + id);
const toProductGid = (id) => (String(id).startsWith(PRODUCT_GID) ? id : PRODUCT_GID + id);
```

### 2.4 Sale attribution (`videoId`)

`videoId` is how Live Me Up attributes a sale back to the live stream or replay it came from. The
reference Shopify gateway stores it as a **line item property** named:

```
_lmu_video_id
```

If your checkout/cart supports line-item properties (custom attributes), set this property to the
`videoId` on the line. If it does not, you can no-op `flagLineForSaleTracking` — add-to-cart will still
work, you just lose video-level sale attribution. (`tagCartWithUserId` likewise stores `_lmu_user_id` as a
cart-level attribute and is optional.)

### 2.5 Reference implementation (Shopify AJAX cart)

This is exactly what the built-in gateway does — adapt the `fetch` calls to your own cart backend.

```js
const SALE_TRACKER_PROP = '_lmu_video_id';
const USER_ID_PROP = '_lmu_user_id';
const ROOT = '/'; // your cart API root

window.LiveMeUpCartGatewayV4 = {
  async addToCart({ cmsVariantId, videoId, quantity }) {
    const res = await fetch(`${ROOT}cart/add.js`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        items: [{
          id: cmsVariantId.replace('gid://shopify/ProductVariant/', ''),
          quantity,
          properties: { [SALE_TRACKER_PROP]: videoId },
        }],
      }),
    });
    if (!res.ok) throw new Error(`addToCart failed: ${res.status}`);
    const cart = await res.json();
    return { lineId: cart.items[0].key };
  },

  async getContent() {
    const res = await fetch(`${ROOT}cart.js`);
    if (!res.ok) throw new Error(`getContent failed: ${res.status}`);
    const cart = await res.json();
    return cart.items
      .filter((i) => i.properties && SALE_TRACKER_PROP in i.properties)
      .map((i) => ({
        cmsProductId: 'gid://shopify/Product/' + i.product_id,
        cmsVariantId: 'gid://shopify/ProductVariant/' + i.id,
        quantity: i.quantity,
        lineId: i.key,
        videoId: i.properties[SALE_TRACKER_PROP],
      }));
  },

  async updateQuantities(products) {
    const updates = Object.fromEntries(products.map((p) => [p.lineId, p.quantity]));
    const res = await fetch(`${ROOT}cart/update.js`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ updates }),
    });
    if (!res.ok) throw new Error(`updateQuantities failed: ${res.status}`);
  },

  async flagLineForSaleTracking(line) {
    const res = await fetch(`${ROOT}cart/change.js`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ id: line.lineId, properties: { [SALE_TRACKER_PROP]: line.videoId } }),
    });
    if (!res.ok) throw new Error(`flagLineForSaleTracking failed: ${res.status}`);
  },

  async validateCart() {
    // navigate the shopper to checkout however your storefront does it
    window.location.href = `${ROOT}checkout`;
  },

  // Optional:
  async addPromoCodes(codes) {
    const res = await fetch(`${ROOT}cart/update.js`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ discount: codes.join(',') }),
    });
    if (!res.ok) throw new Error(`addPromoCodes failed: ${res.status}`);
  },

  async tagCartWithUserId(userId) {
    const res = await fetch(`${ROOT}cart/update.js`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ attributes: { [USER_ID_PROP]: userId } }),
    });
    if (!res.ok) throw new Error(`tagCartWithUserId failed: ${res.status}`);
  },
};
```

### 2.6 Bare-minimum gateway (degraded — not recommended)

The smallest gateway that lets the player *function* implements only the four mandatory methods. Use this
**only** as a starting scaffold — shipping it means **flash-sale discounts and sale attribution will not
work** (see the table in 2.2). Fill in `addPromoCodes` and `tagCartWithUserId` before going live.

```js
window.LiveMeUpCartGatewayV4 = {
  async addToCart({ cmsVariantId, quantity }) { /* your add logic */
    return { lineId: '...' };
  },
  getContent() {
    return undefined;
  },                 // "no cart yet"
  async updateQuantities() {
  },
  async validateCart() {
    window.location.href = '/checkout';
  },

  // ⚠️ Stubs below DISABLE features — replace with real implementations (see 2.4 and 2.5):
  async flagLineForSaleTracking() {
  },                 // no-op = no per-line video attribution
  async addPromoCodes() {
  },                           // no-op = flash-sale discounts won't apply
  async tagCartWithUserId() {
  },                       // no-op = sale attribution broken
};
```

---

## Checklist

- [ ] **Load the player on demand** via `import('https://cdn.livemeup.io/player.esm.js')` — never with a
  static `<script>` tag — and call `startPlayer(eventId, { language, country, ... })`.
- [ ] **`eventId` is a string.** A replay id is launched exactly like a live event id.
- [ ] **Pass `language` + `country`** (ISO codes; `language` upper-cased) so currency and translations
  match your storefront. Reuse the same market/locale you pass to the Storefront API.
- [ ] **Cart gateway, before any `startPlayer` call.** Assign `window.LiveMeUpCartGatewayV4` early, and on
  **every page** the player can be launched from (the `window` global resets on full page navigations).
  *(Standard Shopify themes get this for free — only headless/custom storefronts need it.)*
- [ ] **All product/variant ids exchanged with the player are Shopify GIDs** (`gid://shopify/...`).
- [ ] **Implement the "optional" gateway methods too.** `addPromoCodes` (flash-sale discounts) and
  `tagCartWithUserId` (sale attribution) are optional only for backward-compat — skip them and those
  features silently break.
- [ ] **Set the `_lmu_video_id` line-item property** (or your equivalent) from `videoId` for sale
  attribution.

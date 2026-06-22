---
layout: default
title: Live Shopping Page
nav_order: 3
---

# The Live Shopping Page

The **Live Shopping Page** is a dedicated page on your store that lists:

- **Upcoming events** — your scheduled live events, each with a countdown and a call-to-action (a
  *Notify me* button before it starts, a *Watch live* button once it's on air).
- **Published replays** — a scrollable row of your past events, available on demand. Clicking one opens
  the full player.

Unlike the other widgets, the Live Shopping Page ships as a **self-contained JS bundle**, so even on a
headless / custom storefront you have **two ways** to build it — pick one:

- **Solution 1 — drop in the script + a div.** You add one `<script>` and one `<div>`; the bundle fetches
  your events and renders and manages the whole page for you. It's plain JS on a CDN, so it works on any
  storefront — headless included. **This is the simplest path; start here unless you need full control over
  the markup.**
- **Solution 2 — reimplement it yourself** against the public `GET /v2/landing-page` endpoint. For when you
  want the markup, styling, and behaviour fully in your own components.

> **Production URLs** (see the [overview](README.md)): page script →
> `https://cdn.livemeup.io/landingpage.min.js`; API base → `https://api.livemeup.io/`.

---

## Solution 1 — Drop in the script + a div

The whole page is two tags: a **loader script** and a **mount div**. The script reads the div's `id` and
`data-*` attributes, fetches your events, and renders everything inside the div. Drop these into whatever
renders your page's HTML (a template, a component, a raw `dangerouslySetInnerHTML`, …).

```html
<!-- 1. The loader. data-livemeup-tenantname identifies your store. -->
<script
	defer
	id='lmu-loader'
	src='https://cdn.livemeup.io/landingpage.min.js'
	data-livemeup-tenantname='your-store.myshopify.com'
></script>

<!-- 2. The mount point. The script renders into this div. -->
<div
	id='livemeup-landingpage'
	data-livemeup-language='EN'
	data-livemeup-country='US'
></div>
```

What each attribute does:

| Attribute                   | On     | Meaning                                                                                                                                                                  |
|-----------------------------|--------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `id="lmu-loader"`           | script | **Required.** The script finds itself by this id to read `data-livemeup-tenantname`.                                                                                     |
| `data-livemeup-tenantname`  | script | **Required.** Your store identifier — the Shopify permanent domain (`*.myshopify.com`). Sent as the `x-livemeup-tenant-name` header when the script fetches your events. |
| `id="livemeup-landingpage"` | div    | **Required.** The mount point the script renders into.                                                                                                                   |
| `data-livemeup-language`    | div    | Storefront language ISO code, **upper-cased** (`EN`, `FR`). Passed to the player for currency/translations. Falls back to `window.Shopify?.locale`.                      |
| `data-livemeup-country`     | div    | Storefront country ISO code (`US`, `FR`). Falls back to `window.Shopify?.country`.                                                                                       |

> On a standard Shopify theme the same bundle is delivered as the **"Widget Live & Replay Page"** theme app
> extension block, which fills these attributes for you. On a headless storefront you supply them yourself —
> the snippet above is all you need.

That's the entire integration for Solution 1. Stop here unless you need full control over the markup.

---

## Solution 2 — Reimplement against `GET /v2/landing-page`

On a headless / custom storefront you fetch the same data the script uses and render it yourself.

### 2.1 The endpoint

```
GET https://api.livemeup.io/v2/landing-page
Header: x-livemeup-tenant-name: your-store.myshopify.com
```

- **No auth** beyond the tenant header — it's a public read endpoint.
- The `x-livemeup-tenant-name` header is **required**; it identifies your store (your Shopify permanent
  domain). Without it the request fails with *"Missing header x-livemeup-tenant-name"*.

```js
async function getLandingPageData(tenantName) {
  const res = await fetch('https://api.livemeup.io/v2/landing-page', {
    headers: { 'x-livemeup-tenant-name': tenantName },
  });
  if (!res.ok) throw new Error(`landing-page fetch failed: ${res.status}`);
  return res.json();
}
```

### 2.2 The response shape

```ts
type LandingPageResponse = {
  // Upcoming / highlighted events, already sorted by show time ascending.
  highlightedEvents: PublicEventDepiction[];
  // Published replays, on-demand.
  publishedReplayEvents: PublicEventDepiction[];
  // Optional heading to show above the replays section.
  replaySectionTitle: string;
  // Whether the "Notify me" CTA should collect an email / an SMS number
  // (depends on the store's plan & integrations — see 2.4).
  askEmail: boolean;
  askSMS: boolean;
};

type PublicEventDepiction = {
  id: string;            // the eventId — pass this to startPlayer()
  title: string;
  description: string;
  status: 'PLANNED' | 'ON_AIR' | 'PRIVATE_PREVIEW' | 'FINISHED' | 'REPLAY' | 'REPLAY_FAILED';
  showTime: string | null;   // ISO 8601 datetime, or null
  coverUrl: string;          // cover image path (resolve through the image CDN, see 2.5)
  preEventCoverUrl: string;  // the pre-event cover, always present
};
```

Notes:

- **`highlightedEvents`** are your upcoming/live events. Use `status` + `showTime` to decide what CTA to
  render (see 2.3).
- **`publishedReplayEvents`** are your replays.
- **`status`** values you'll actually branch on: `PLANNED` (scheduled, show a countdown + Notify me),
  `ON_AIR` (live now, show Watch live), `FINISHED` (just ended, replay coming soon). `REPLAY` items are
  the published replays.

### 2.3 Rendering the upcoming-events CTA

For each event in `highlightedEvents`, pick the CTA from its `status` (and `showTime`):

- **`ON_AIR`** (or `showTime` already passed) → a **Watch live** button that calls `startPlayer(event.id)`.
- **`PLANNED`** → a countdown to `showTime`, plus a **Notify me** button *if* the store collects reminders
  (`askEmail || askSMS`).
- **`FINISHED`** → a **Replay coming soon** label (no action yet).

> The **Notify me** flow (collecting an email/SMS reminder) is optional and plan-dependent. Skip it and just
> omit the CTA for `PLANNED` events. If you do want to build it:
>
> - **SMS reminders** — register the shopper's number through the Live Me Up **event-reminders API**
>   ([reference](https://cdn.livemeup.io/openapi.external.html#tag/event-reminders)).
> - **Email reminders** — Live Me Up sends these via **Klaviyo**. If you already use Klaviyo, subscribe the
>   shopper directly through your own Klaviyo integration.
>
> Use `askEmail` / `askSMS` from the response to decide which inputs to show — they reflect what the store's
> plan and integrations support.

### 2.4 Rendering the replays

Each entry in `publishedReplayEvents` is a thumbnail that calls `startPlayer(event.id)` on click (a replay
id launches like any event id). Build its cover from `coverUrl` (falling back to `preEventCoverUrl`) via the
image CDN — see 2.5.

### 2.5 Resolving cover images

`coverUrl` / `preEventCoverUrl` are image paths, not final URLs. Serve them through the Live Me Up image
CDN, sizing as you go:

```js
function buildImageCDNLink(imagePath, width, height) {
  return `https://chquwzbkea.cloudimg.io/${imagePath}?width=${width}&height=${height}&force_format=webp%2Cjpeg`;
}
```

The reference page uses portrait `250×432` for replay posters.

### 2.6 Don't forget the cart gateway

The page launches the **player**, so on a headless storefront you must register
`window.LiveMeUpCartGatewayV4` before the first click — see
the [Player & Cart Gateway guide](player-and-cart-gateway.md).

---

## Checklist

**If you used Solution 1 (script + div):**

- [ ] Added the loader `<script id="lmu-loader" src="https://cdn.livemeup.io/landingpage.min.js"
  data-livemeup-tenantname="your-store.myshopify.com">`.
- [ ] Added the `<div id="livemeup-landingpage">` mount point with `data-livemeup-language` (upper-cased)
  and `data-livemeup-country`.
- [ ] On a custom storefront, confirmed the cart gateway is registered (theme installs handle this
  automatically).

**If you used Solution 2 (reimplement):**

- [ ] `GET https://api.livemeup.io/v2/landing-page` with the **`x-livemeup-tenant-name`** header (your
  `*.myshopify.com` domain).
- [ ] Rendered `highlightedEvents`, branching on `status`/`showTime` → countdown / Notify me / Watch live.
- [ ] Rendered `publishedReplayEvents`; each thumbnail calls `startPlayer(event.id)` on click.
- [ ] Resolved cover images through
  `https://chquwzbkea.cloudimg.io/<coverUrl>?width=…&height=…&force_format=webp%2Cjpeg`.
- [ ] Passed `language` + `country` to `startPlayer`.
- [ ] Registered the **cart gateway** (`window.LiveMeUpCartGatewayV4`) before any player launch — see the
  [Player & Cart Gateway guide](player-and-cart-gateway.md).

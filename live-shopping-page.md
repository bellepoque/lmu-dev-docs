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
- **Solution 2 — reimplement it yourself** against the public `/v3/live-shopping-page` endpoints: fetch the
  events once, then poll their status. For when you want the markup, styling, and behaviour fully in your
  own components.

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

## Solution 2 — Reimplement against the `/v3/live-shopping-page` endpoints

On a headless / custom storefront you read the same data the bundle reads and render it yourself.

> **API reference:** both endpoints, with their full schemas and examples, are documented in the
> [Live Me Up OpenAPI reference](https://bellepoque.github.io/lmu-dev-docs/openapi.external.html#tag/live-shopping-page).

### 2.1 The flow

The page talks to two endpoints, in two steps:

1. **Fetch the page, once, on load** — `GET /v3/live-shopping-page` returns everything the page renders:
   the upcoming/live events, the replays, and the store settings. Render it all from this response.
2. **Poll for status changes** — while an upcoming event has not gone live yet, call
   `GET /v3/live-shopping-page/status` every 5 seconds. It returns only `{ id, showTime, status }` for each
   upcoming event; merge that onto the event you loaded in step 1 and re-render its CTA, so the *Notify me*
   countdown flips to *Watch live* the moment the show opens, without a page reload.

```
page load ──► GET /v3/live-shopping-page ──► render events + replays
                                               │
                    any upcoming event not yet ON_AIR / FINISHED?
                                               │ yes
                                               ▼
              every 5 s ──► GET /v3/live-shopping-page/status ──► merge status onto the loaded event,
                  ▲                                               re-render its CTA
                  └──────────── until no event is left to watch ◄─┘
```

### 2.2 The endpoints

```
GET https://api.livemeup.io/v3/live-shopping-page?tenant=your-store.myshopify.com
GET https://api.livemeup.io/v3/live-shopping-page/status?tenant=your-store.myshopify.com
```

- **Public, no auth.** Your store is named by the **`tenant` query parameter** — your Shopify permanent
  domain (`*.myshopify.com`). It's required and case-insensitive.
- **No custom header.** Don't send `x-livemeup-tenant-name` (that was the legacy `/v2/landing-page`
  route): a custom header makes the browser send a CORS preflight before every request. With the tenant in
  the query string the request stays "simple". Responses carry `Access-Control-Allow-Origin: *`.
- **Cached at the edge.** The page route is served with `Cache-Control: public, max-age=15, s-maxage=30`,
  so a change the merchant publishes can take up to ~45 s to show. The status route uses
  `public, max-age=0, s-maxage=5`, matching the 5-second poll.
- **Errors:**
  - `400` — `tenant` is missing or has an invalid length.
  - `404` — no store carries this name, or the store has no Live Shopping Page (the two cases are
    deliberately indistinguishable). Render nothing.

```js
const API_BASE = 'https://api.livemeup.io/';

function liveShoppingPageUrl(path, tenantName) {
  return `${API_BASE}${path}?tenant=${encodeURIComponent(tenantName)}`;
}

// Step 1 — once, on page load. Resolves to null when there is no page to render.
async function fetchLiveShoppingPage(tenantName) {
  const res = await fetch(liveShoppingPageUrl('v3/live-shopping-page', tenantName));
  if (!res.ok) return null; // 404: this store has no Live Shopping Page
  return res.json();
}

// Step 2 — on every poll tick. Resolves to null on any failure: a failed tick is not "no events".
async function fetchHighlightedEventStatuses(tenantName) {
  try {
    const res = await fetch(liveShoppingPageUrl('v3/live-shopping-page/status', tenantName), {
      // A request that never settles would block the poll for good (see 2.5).
      signal: AbortSignal.timeout(15_000),
    });
    if (!res.ok) return null;
    return (await res.json()).highlightedEvents;
  } catch {
    return null;
  }
}
```

### 2.3 Step 1 — Fetch the page

`GET /v3/live-shopping-page` returns:

```ts
type LiveShoppingPage = {
  tenantId: string;          // your store's Live Me Up id — sent along with reminder sign-ups
  shareUrl: string;          // the public URL of your Live Shopping Page
  replaySectionTitle: string;  // heading to show above the replays section
  // Whether the "Notify me" CTA should collect an email / an SMS number
  // (depends on the store's plan & integrations — see 2.4).
  askEmail: boolean;
  askSMS: boolean;
  // Upcoming / live events, sorted by showTime ascending (events without a showTime last).
  highlightedEvents: HighlightedEvent[];
  // Published replays, in the order the merchant arranged them.
  publishedReplayEvents: PublishedReplayEvent[];
};

type HighlightedEvent = {
  id: string;                // the eventId — pass this to startPlayer()
  title: string;
  description: string;
  status: 'PLANNED' | 'PRIVATE_PREVIEW' | 'ON_AIR' | 'FINISHED' | 'REPLAY' | 'REPLAY_FAILED';
  showTime: string | null;   // ISO 8601 datetime, or null
  coverUrl: string;          // absolute image URL (resize it through the image CDN, see 2.7)
};

// Narrower than a highlighted event: a replay tile needs no description or status.
type PublishedReplayEvent = {
  id: string;                // the eventId — pass this to startPlayer()
  title: string;
  showTime: string | null;
  coverUrl: string;
};
```

Notes:

- **`highlightedEvents`** are your upcoming/live events. The API doesn't filter on time: an event whose
  `showTime` has passed without it opening is still returned. Use `status` + `showTime` to decide what CTA
  to render (see 2.4).
- **`status`** values you'll actually branch on: `PLANNED` (scheduled, show a countdown + Notify me),
  `ON_AIR` (live now, show Watch live), `FINISHED` (just ended, replay coming soon).
- **Keep the response around**: step 2 merges each status update onto these events.

> **Migrating from `/v2/landing-page`?** The tenant moved from the `x-livemeup-tenant-name` header to the
> `tenant` query parameter; `preEventCoverUrl` is gone (read `coverUrl`, which is always set); replays no
> longer carry `description` or `status`; `tenantId` and `shareUrl` are new.

### 2.4 Rendering the upcoming-events CTA

For each event in `highlightedEvents`, pick the CTA from its `status` (and `showTime`). Write this as one
function: you call it for every event on load, and again for each event the poll updates (2.5).

- **`ON_AIR`** (or `showTime` already passed) → a **Watch live** button that calls `startPlayer(event.id)`.
- **`PLANNED`** → a countdown to `showTime`, plus a **Notify me** button *if* the store collects reminders
  (`askEmail || askSMS`).
- **`FINISHED`** → a **Replay coming soon** label (no action yet).

> The **Notify me** flow (collecting an email/SMS reminder) is optional and plan-dependent. Skip it and just
> omit the CTA for `PLANNED` events. If you do want to build it:
>
> - **SMS reminders** — register the shopper's number through the Live Me Up **event-reminders API**
>   ([reference](https://bellepoque.github.io/lmu-dev-docs/openapi.external.html#tag/event-reminders)).
> - **Email reminders** — Live Me Up sends these via **Klaviyo**. If you already use Klaviyo, subscribe the
>   shopper directly through your own Klaviyo integration.
>
> Use `askEmail` / `askSMS` from the response to decide which inputs to show — they reflect what the store's
> plan and integrations support.

### 2.5 Step 2 — Poll for status changes

Once the page is rendered, poll `GET /v3/live-shopping-page/status` to catch an event going live. It
returns the upcoming events only — replays never appear — and only the fields that change:

```ts
type LiveShoppingPageStatus = {
  highlightedEvents: { id: string; showTime: string | null; status: HighlightedEvent['status'] }[];
};
```

The rules the bundle follows, and that your loop should too:

- **What to watch** — the highlighted events that are not yet `ON_AIR` or `FINISHED`. If there are none,
  don't poll at all.
- **Every 5 seconds.** That's the edge cache's lifetime for this route; polling faster just reads the same
  cached answer.
- **Merge, never replace.** The status payload has no `title` or `description`: merge its `showTime` and
  `status` onto the event you hold from step 1 (matched by `id`), then re-render that event's CTA. Rendering
  the status payload directly loses everything else the CTA needs.
- **When to stop watching an event** — once it reaches `ON_AIR` or `FINISHED`, or when its `id` is missing
  from the response (it's no longer highlighted, so it will never flip). Stop the loop once nothing is left
  to watch.
- **A failed call is not an answer.** On a network error, a non-2xx or a timeout, skip the tick and keep
  watching; don't treat it as an empty list, or the loop stops on the wrong signal.
- **Poll only when it can matter:**
  - Skip ticks while the tab is hidden (`document.visibilityState === 'hidden'`) or the player is open — the
    player opens over the page without leaving it, so your timers keep running underneath it. Tick right
    away when the tab becomes visible again or the player closes (`onClose`, see the
    [Player & Cart Gateway guide](player-and-cart-gateway.md)), so the CTA is right when the shopper looks.
  - Don't call the endpoint until **15 minutes before** the earliest `showTime` you're watching.
  - Keep one request in flight at a time, and give each a timeout (the bundle aborts after 15 s), or a hung
    request blocks every later tick.

```js
const POLL_INTERVAL_MS = 5000;
const POLL_ARM_LEAD_MS = 15 * 60 * 1000;

const hasOpenedOrEnded = event => event.status === 'ON_AIR' || event.status === 'FINISHED';

// `renderCta(event)` is your 2.4 function; `isPlayerOpen()` tracks your startPlayer()/onClose calls.
// Returns `tick`, to call from the player's onClose.
function startStatusPolling(tenantName, highlightedEvents, { renderCta, isPlayerOpen }) {
  // Your own copy of each event still being watched, keyed by id: the poll merges onto it.
  const watched = new Map(
    highlightedEvents.filter(event => !hasOpenedOrEnded(event)).map(event => [event.id, event]),
  );
  if (watched.size === 0) return () => {};

  let inFlight = false;
  const intervalId = setInterval(tick, POLL_INTERVAL_MS);
  document.addEventListener('visibilitychange', tick);

  function stop() {
    clearInterval(intervalId);
    document.removeEventListener('visibilitychange', tick);
  }

  function firstShowIsFarOff() {
    const showTimes = [...watched.values()]
      .filter(event => event.showTime)
      .map(event => new Date(event.showTime).getTime())
      .filter(Number.isFinite);
    return showTimes.length > 0 && Date.now() < Math.min(...showTimes) - POLL_ARM_LEAD_MS;
  }

  async function tick() {
    if (watched.size === 0) return stop();
    if (document.visibilityState === 'hidden' || isPlayerOpen()) return;
    if (firstShowIsFarOff() || inFlight) return;

    inFlight = true;
    try {
      const statuses = await fetchHighlightedEventStatuses(tenantName);
      if (!statuses) return; // transient failure: keep watching

      for (const { id, showTime, status } of statuses) {
        const event = watched.get(id);
        if (!event) continue;
        const merged = { ...event, showTime, status }; // merge, never replace
        watched.set(id, merged);
        renderCta(merged);
        if (hasOpenedOrEnded(merged)) watched.delete(id);
      }

      // Missing from the response = no longer highlighted: stop following it.
      const returnedIds = new Set(statuses.map(({ id }) => id));
      for (const id of watched.keys()) {
        if (!returnedIds.has(id)) watched.delete(id);
      }
    } finally {
      inFlight = false;
    }
  }

  return tick;
}
```

Wire the two steps together on page load:

```js
const page = await fetchLiveShoppingPage(tenantName);
if (page) {
  renderPage(page); // highlighted events (2.4) + replays (2.6)
  // Call the returned tick from the player's onClose, so the CTA catches up as soon as it closes.
  const wakePoll = startStatusPolling(tenantName, page.highlightedEvents, { renderCta, isPlayerOpen });
}
```

### 2.6 Rendering the replays

Each entry in `publishedReplayEvents` is a thumbnail that calls `startPlayer(event.id)` on click (a replay
id launches like any event id). Build its cover from `coverUrl` via the image CDN — see 2.7.

### 2.7 Resolving cover images

`coverUrl` is the original image's URL. Serve it through the Live Me Up image CDN, sizing as you go:

```js
function buildImageCDNLink(coverUrl, width, height) {
  const url = new URL(`https://cdn-images.livemeup.io/${coverUrl}`);
  url.searchParams.set('width', String(width));
  url.searchParams.set('height', String(height));
  url.searchParams.set('func', 'bound');
  url.searchParams.set('force_format', 'webp,jpeg');
  return url.toString();
}
```

The bundle requests twice the display size (for high-density screens): `560×560` for a highlighted event's
cover, `600×600` for a replay poster.

### 2.8 Autoplay

The bundle can launch the player **automatically on page load**, in two cases:

- **From a share link.** *Share* buttons produce a URL with an `autoplay-live-shopping-be=<eventId>` query
  param (and an optional `t=<seconds>` start offset). When the page loads with that param, the bundle
  immediately calls `startPlayer(eventId, t)` so the shared event opens straight into the player.
- **A single live event.** If there's no autoplay param but exactly **one** `highlightedEvents` entry is
  `ON_AIR`, the bundle autoplays it — landing on the page during a live drops the shopper straight into the
  stream.

In both cases the bundle also **hides the site-wide [Live Now Widget](mini-player-widget.md)** (it injects a
`#lmuLiveWidgetContainer { display: none !important; }` style rule) so the floating widget and the
full-page player never run the same stream at once.

**Why you have to reimplement it:** none of this is server-side — it lives in the bundle's load sequence. On
Solution 2 the bundle isn't running, so on page load you must read the `autoplay-live-shopping-be` (and `t`)
query params yourself and, failing that, check for a lone `ON_AIR` event in the step-1 response, then call
`startPlayer(...)` and hide your own live-now widget — otherwise share links won't deep-link into the player
and going live won't auto-open it.

### 2.9 Don't forget the cart gateway

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

- [ ] On load, `GET https://api.livemeup.io/v3/live-shopping-page?tenant=your-store.myshopify.com` (your
  `*.myshopify.com` domain, in the query string — no custom header). Render nothing on a `404`.
- [ ] Rendered `highlightedEvents`, branching on `status`/`showTime` → countdown / Notify me / Watch live.
- [ ] Polled `GET https://api.livemeup.io/v3/live-shopping-page/status?tenant=…` every 5 s while an event is
  not yet `ON_AIR`/`FINISHED`: merged `showTime` + `status` onto the loaded event by `id` (never replaced),
  re-rendered its CTA, stopped following events that opened, ended or dropped out of the response, kept
  polling through failed calls, and skipped ticks while the tab is hidden or the player is open.
- [ ] Rendered `publishedReplayEvents`; each thumbnail calls `startPlayer(event.id)` on click.
- [ ] Resolved cover images through
  `https://cdn-images.livemeup.io/<coverUrl>?width=…&height=…&func=bound&force_format=webp,jpeg`.
- [ ] Passed `language` + `country` to `startPlayer`.
- [ ] Handled **autoplay**: on load, read `autoplay-live-shopping-be` (+ `t`) → `startPlayer`; else autoplay
  a lone `ON_AIR` event; hide your own live-now widget when autoplaying.
- [ ] Registered the **cart gateway** (`window.LiveMeUpCartGatewayV4`) before any player launch — see the
  [Player & Cart Gateway guide](player-and-cart-gateway.md).

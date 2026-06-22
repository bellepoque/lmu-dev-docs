---
layout: default
title: Mini Player Widget
nav_order: 4
---

# The Mini Player Widget

The **Mini Player** is a small floating thumbnail pinned to the corner of *every* storefront page **while a
Live Shopping event is currently live**. The thumbnail shows a muted, real-time preview of the live stream
(via Agora); clicking it launches the full Live Me Up player. When no event is live, nothing is shown.

On a headless storefront you rebuild it yourself. Everything you need lives in a single Shopify shop
metafield; the rest is an Agora live preview + a `startPlayer` call.

> **Production URLs** (see the [overview](README.md)): player ES module →
> `https://cdn.livemeup.io/player.esm.js`.

---

## The "current live" metafield

The widget is driven by one metafield set on the **shop** (not on individual products). It contains
everything needed to connect to the currently-running live event. **When no event is live, the metafield is
empty (`{}`)** — that absence is how you detect that nothing should be shown.

- **Namespace:** `app--5813997--current_live` (app-owned)
- **Key:** `event_infos`
- **Type:** `json`
- **Owner:** `Shop`
- **Storefront access:** Public read

Its JSON value:

```ts
{
  appId: string;         // Agora application id, to initialise the Agora RTC client
  channelName: string;   // Agora channel to join (equal to the event id)
  coverUrl: string;      // cover image — thumbnail placeholder & autoplay fallback
  eventId: string;       // the live event id — pass to startPlayer() to launch the full player
  eventTitle: string;    // human-readable event title
  streamProvider: 'agora';
  viewerToken: string;   // Agora RTC token, audience (subscriber) role, valid 7 days
}
```

A non-empty **`eventId`** and **`viewerToken`** is your signal that a live event is running and the widget
should be shown.

**Reading it in Liquid (theme):**

```liquid
{% assign eventInfos = shop.metafields.app--5813997--current_live.event_infos %}
{% if eventInfos.value.eventId != blank and eventInfos.value.viewerToken != blank %}
  ...
{% endif %}
```

**Reading it headless (Storefront GraphQL):** query the same shop metafield, namespace
`app--5813997--current_live`, key `event_infos`, and parse `value` as JSON.

---

## Building it

Two parts: (a) show the live preview, (b) open the full player on click.

### 1. Showing the live preview (Agora)

Unlike the Pop Me Up widget — which plays a short recorded clip — the Mini Player shows the actual live
stream in real time. Connect to the Agora channel as an **audience** member using `appId`, `channelName`
and `viewerToken` from the metafield, and render the host's video track into a muted `<video>` element.

```javascript
const eventInfos = /* the parsed JSON value of the metafield */;
const videoElement = document.getElementById('lmuLiveWidgetVideo'); // muted + playsinline

// 1. Load the Agora Web SDK
const script = document.createElement('script');
script.src = 'https://download.agora.io/sdk/release/AgoraRTC_N-4.24.0.js';
document.head.appendChild(script);

script.onload = async () => {
  const AgoraRTC = window.AgoraRTC;
  const client = AgoraRTC.createClient({ mode: 'live', codec: 'vp8' });

  // Stop the preview: pause the video and leave the Agora channel
  const stopStream = () => {
    videoElement.pause();
    client.leave();
    // optionally reveal a "continue watching" overlay inviting the user to open the full player
  };

  // 2. Render the host's video track once it is published
  client.on('user-published', async (user, mediaType) => {
    await client.subscribe(user, mediaType);
    if (mediaType === 'video') {
      user.videoTrack.play(videoElement);
      // reveal the widget now that we have a picture
      setTimeout(stopStream, 30000); // stop the preview after 30s to bound bandwidth
    }
  });

  // 3. Browsers may block autoplay — surface a manual "tap to play" button if so
  AgoraRTC.onAutoplayFailed = () => {
    // reveal a play overlay whose click handler calls videoElement.play()
  };

  // 4. Join the channel, then switch to a low-latency audience role
  await client.join(eventInfos.appId, eventInfos.channelName, eventInfos.viewerToken, null);
  client.setClientRole('audience', { level: 1 });
};
```

Notes:

- Join as an **audience** member with `setClientRole('audience', { level: 1 })` (level 1 = ultra-low
  latency). **Never publish a track** from the storefront — the `viewerToken` is a subscriber-only token
  scoped to this one channel; it cannot broadcast.
- The `<video>` element **must** be `muted` and `playsinline` so the browser allows autoplay. Always handle
  `AgoraRTC.onAutoplayFailed` by showing a play button that calls `videoElement.play()` on a user gesture.
- Show the `coverUrl` image as a placeholder until the first video frame arrives, and keep it as a fallback
  if the stream can't autoplay.
- Auto-pause the preview after a while (the reference widget uses 30s) and invite the viewer to open the
  full player, to keep the on-page preview's bandwidth bounded.

### 2. Opening the full player

The corner preview is just a teaser. When the user clicks the widget, launch the full Live Me Up player
(chat, shoppable products, …) with `startPlayer`, passing the **`eventId`** from the metafield:

```javascript
import('https://cdn.livemeup.io/player.esm.js').then(({ startPlayer }) => {
  startPlayer(eventInfos.eventId, {
    language: 'EN', // storefront language ISO code, upper-cased
    country: 'US',  // storefront country ISO code
  });
});
```

See the [Player & Cart Gateway guide](01-player-and-cart-gateway.md) for the full `startPlayer` contract and — on a headless
storefront — the **cart gateway** you must register so add-to-cart from the player works.

---

## Checklist

- [ ] Read the **shop** metafield `app--5813997--current_live` / `event_infos`.
- [ ] Treat an **empty value (`{}`)** — or a blank `eventId`/`viewerToken` — as "no live event": render
  nothing.
- [ ] Render the corner thumbnail using `coverUrl` as the placeholder.
- [ ] Load the **Agora Web SDK**, join with `appId` / `channelName` / `viewerToken` as **audience, level
  1**, and play the host's track into a **`muted` + `playsinline`** `<video>`.
- [ ] Handle **`onAutoplayFailed`** with a tap-to-play button; keep `coverUrl` as the fallback.
- [ ] **Bound bandwidth**: auto-pause the preview after ~30s and invite opening the full player.
- [ ] On click, `import('https://cdn.livemeup.io/player.esm.js')` and call `startPlayer(eventId, {
  language, country })`.
- [ ] On a headless storefront, register the **cart gateway** (`window.LiveMeUpCartGatewayV4`) — see the
  [Player & Cart Gateway guide](01-player-and-cart-gateway.md).

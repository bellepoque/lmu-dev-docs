---
layout: default
title: Pop Me Up Widget
nav_order: 6
---

# The Pop Me Up Widget

The **Pop Me Up** widget lives on a **product page**. When the product was featured in a Live Shopping
event, it shows a small bubble or thumbnail (optionally auto-playing a short muted preview clip), and
clicking it opens the replay **jumped to the exact moment that product was presented**. When the product
has never been featured, the widget renders nothing.

On a headless storefront you rebuild it yourself: read the product metafield and, when it's present, render
a thumbnail that opens the replay at the highlighted moment.

> **Production URLs** (see the [overview](README.md)): player ES module →
> `https://cdn.livemeup.io/player.esm.js`; HLS.js → `https://cdn.livemeup.io/hls.js`.

---

## The data source — a product metafield

Everything the widget needs comes from a single **product** metafield:

- **Namespace:** `app--5813997` (the app's default reserved namespace)
- **Key:** `last_replay_with_product_highlighted`
- **Type:** `json`
- **Owner:** Product
- **Storefront access:** Public read

Its JSON value:

```ts
{
  replayId: string;   // pass this to startPlayer
  timecode: number;   // MILLISECONDS offset where the product was presented
  videoUrl: string;   // HLS (.m3u8) URL for the optional preview clip
}
```

> **Unit note:** both `timecode` here and `startPlayer`'s `t` parameter are in **milliseconds**, so pass
> `timecode` straight through (`t: timecode`). (The `<video>` preview's `currentTime`, however, is in
> seconds, so divide by 1000 only there.)

**If the metafield is absent or empty for a product, render nothing** — that product has no replay.

**Reading it (Storefront GraphQL).** The metafield definition has **storefront public read** access, so
your storefront token can query it:

```graphql
query ProductReplay($handle: String!) {
  product(handle: $handle) {
    metafield(namespace: "app--5813997", key: "last_replay_with_product_highlighted") {
      value   # JSON string: { replayId, timecode, videoUrl }
    }
  }
}
```

Parse `value` as JSON to get `replayId` / `timecode` / `videoUrl`.

### The replay title & cover image — metaobject (optional)

For a nicer presentation you can also resolve the replay's title and cover image from the `lmu_replay`
**metaobject**, keyed by the `replayId` from the metafield:

- Metaobject type: `lmu_replay`, looked up by the `replayId` (it's the metaobject's handle)
- Fields used: `title` (string) and `image` (an image path)

Resolve it with a second Storefront query (`$handle` is the `replayId` from the metafield):

```graphql
query Replay($handle: String!) {
  metaobject(handle: { type: "lmu_replay", handle: $handle }) {
    title: field(key: "title") { value }
    image: field(key: "image") { value }   # an image path — build the CDN URL below
  }
}
```

The cover image is served through the image CDN. Build the URL as:

```
https://chquwzbkea.cloudimg.io/<replay.image>?width=<W>&height=<H>&force_format=webp%2Cjpeg
```

The original widget uses `250×250` for the round **bubble** style and `250×432` for the **thumbnail**
(portrait 9:16) style.

If you cannot or don't want to read the metaobject, fall back to a static title (e.g. "Seen live!") and
your own placeholder image — only `replayId` and `timecode` from the metafield are strictly required to
make the click-to-play work.

---

## Building it

Below is a self-contained, framework-agnostic reimplementation. Feed it the parsed metafield values
(`replayId`, `timecode`, `videoUrl`) plus a `title` and `coverUrl`. It renders a thumbnail and launches
the replay at the right moment on click.

```html
<div class='lmuPmu' id='lmuPmu' role='button' tabindex='0' aria-label='Watch this product live'>
  <!-- optional muted preview video; omit if you don't want a preview -->
  <video id='lmuPmuVideo' muted loop playsinline></video>
  <img id='lmuPmuCover' alt='' />
  <div class='lmuPmuTitle'></div>
</div>

<script type='module'>
  // --- inject these from the metafield / metaobject ---
  const replayId = 'REPLAY_ID';
  const timecodeMs = 42000;                 // milliseconds, from the metafield
  const videoUrl = 'https://.../stream.m3u8'; // optional
  const title = 'Seen live!';
  const coverUrl = 'https://chquwzbkea.cloudimg.io/.../cover?width=250&height=432&force_format=webp%2Cjpeg';
  const language = 'EN';
  const country = 'US';
  // -----------------------------------------------------

  const root = document.getElementById('lmuPmu');
  document.getElementById('lmuPmuCover').src = coverUrl;
  root.querySelector('.lmuPmuTitle').textContent = title;

  // Launch the replay at the highlighted moment (t is in milliseconds, same unit as the metafield).
  function open() {
    import('https://cdn.livemeup.io/player.esm.js').then(({ startPlayer }) => {
      startPlayer(replayId, { t: timecodeMs, language, country });
    });
  }

  root.addEventListener('click', open);
  root.addEventListener('keydown', (e) => {
    if (e.key === 'Enter' || e.key === ' ') open();
  });

  // --- OPTIONAL: muted HLS preview that plays only while on screen ---
  const videoEl = document.getElementById('lmuPmuVideo');
  if (videoUrl) {
    loadHls(() => {
      if (!window.Hls || !Hls.isSupported()) return;
      const hls = new Hls({ autoStartLoad: false, maxMaxBufferLength: 6, capLevelToPlayerSize: true });
      hls.loadSource(videoUrl);
      hls.attachMedia(videoEl);
      hls.on(Hls.Events.MANIFEST_PARSED, () => {
        // play only while the widget is visible, and loop a few seconds around the timecode
        const io = new IntersectionObserver((entries) => {
          if (entries[0].isIntersecting) {
            hls.startLoad();
            videoEl.currentTime = timecodeMs / 1000;
            videoEl.play();
          } else {
            hls.stopLoad();
            videoEl.pause();
          }
        }, { threshold: 0.5 });
        io.observe(videoEl);
      });
    });
  }

  function loadHls(cb) {
    if (window.Hls) return cb();
    const s = document.createElement('script');
    s.src = 'https://cdn.livemeup.io/hls.js';
    s.onload = cb;
    s.onerror = () => console.error('Failed to load HLS.js');
    document.head.appendChild(s);
  }
</script>
```

Notes on the preview clip:

- The `<video>` element **must** be `muted` and `playsinline` so browsers allow autoplay.
- Only load/play the stream while the widget is on screen (the `IntersectionObserver` above) to keep
  bandwidth bounded. The original widget loops a short segment by resetting `currentTime` back to the
  timecode every few seconds.
- The preview is purely decorative. If you skip it entirely, the bubble/thumbnail + click-to-open replay
  is the core experience.

### Behaviour summary

| Step | What happens |
|------|--------------|
| Product page loads | Read the `last_replay_with_product_highlighted` product metafield. |
| Metafield empty/absent | Render nothing. |
| Metafield present | Render bubble/thumbnail using the metaobject `title` + cover image (or your fallbacks). |
| (Optional) widget visible | Autoplay a muted HLS preview starting at `timecode` (ms). |
| Shopper clicks | `startPlayer(replayId, { t: timecode, language, country })` (timecode in ms). |

### Don't forget the cart gateway

Clicking the widget launches the **player**, so on a headless storefront register
`window.LiveMeUpCartGatewayV4` before the first click — see the [Player & Cart Gateway guide](player-and-cart-gateway.md).

---

## Checklist

- [ ] Read the **product** metafield namespace `app--5813997`, key
  `last_replay_with_product_highlighted` (JSON: `{ replayId, timecode, videoUrl }`).
- [ ] **Render nothing** if the metafield is absent or empty.
- [ ] *(Optional)* Resolve the **`lmu_replay`** metaobject by `replayId` for `title` + cover `image`; build
  the cover via the image CDN (`250×250` bubble, `250×432` thumbnail). Otherwise use your own
  title/placeholder.
- [ ] On click, `import('https://cdn.livemeup.io/player.esm.js')` and call `startPlayer(replayId, { t:
  timecode, language, country })` — **`t` is in milliseconds**, same as the metafield's `timecode`.
- [ ] *(Optional)* Muted, `playsinline`, on-screen-only **HLS preview** via `https://cdn.livemeup.io/hls.js`,
  seeking to `timecode / 1000` (seconds).
- [ ] Pass `language` + `country`; register the **cart gateway** on headless storefronts (see the [Player &
  Cart Gateway guide](player-and-cart-gateway.md)).

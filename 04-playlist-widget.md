---
layout: default
title: Playlist Widget
nav_order: 5
---

# The Playlist Widget (Carousel & Stories)

The **Playlist widget** shows a curated row of replay videos that open the player in **playlist mode** —
the viewer can swipe from one video to the next without leaving the player. You upload up to six videos to
a collection from the Live Me Up console, then place the widget on a product, collection, page, or the home
page.

It comes in **two visual styles**, both backed by the same data and the same `startPlayerCollection` call —
pick whichever fits your design:

- **Stories** — a row of round thumbnail **bubbles**.
- **Carousel** — a horizontally-scrolling row of portrait **posters**, each optionally playing a muted
  inline **video preview**.

On a headless storefront you rebuild it yourself: read the collection metafield, render the thumbnails, and
open the player in playlist mode on click.

> **Production URLs** (see the [overview](README.md)): player ES module →
> `https://cdn.livemeup.io/player.esm.js`.

---

## The media-collection metafield

The widget is driven by a **`media_collections`** metafield whose owner depends on where the widget is
placed:

| Page template | Metafield owner | Liquid access |
|---------------|-----------------|---------------|
| Home (`index`) | Shop | `shop.metafields.app--5813997--media_collections.media_collection` |
| Product | Product | `product.metafields.app--5813997--media_collections.media_collection` |
| Collection | Collection | `collection.metafields.app--5813997--media_collections.media_collection` |
| Page | Page | `page.metafields.app--5813997--media_collections.media_collection` |

- **Namespace:** `app--5813997--media_collections`
- **Key:** `media_collection`
- **Type:** metaobject reference (resolves to a *media collection* metaobject)
- **Storefront access:** Public read

If the metafield is **absent or empty**, render nothing — there's no playlist for that page.

The resolved collection metaobject gives you:

```ts
{
  system: { handle: string };   // the mediaCollectionId — pass to startPlayerCollection
  medias_infos: MediaInfo[];     // the ordered list of videos
}

type MediaInfo = {
  mediaId: string;               // id of the replay/event (also resolves the lmu_replay metaobject)
  startTimestamp: number | null; // ms offset to start the preview/playback at
  endTimestamp: number | null;   // ms offset to stop at
  videoUrl: string;              // HLS (.m3u8) URL for the inline preview (Carousel style)
  largeThumbnailUrl?: string;    // optional portrait poster image path
}
```

For each item's **title and cover image**, resolve the `lmu_replay` metaobject by `mediaId`:

- Metaobject type: `lmu_replay`, looked up by `mediaId`
- Fields used: `title` (string) and `image` (image path → resolve through the image CDN)

```liquid
{% assign media = shop.metaobjects.lmu_replay[media_infos.mediaId] %}
{% assign cover = 'https://chquwzbkea.cloudimg.io/' | append: media.image | append: '?width=250&height=432&force_format=webp%2Cjpeg' %}
```

Headless: read the `media_collections` metafield and the `lmu_replay` metaobjects via the Storefront
GraphQL API the same way.

---

## Building it

### 1. Render the thumbnails

Loop over `medias_infos`, resolving each `lmu_replay` for its `title` + `image`. Tag each clickable element
with its **0-based index** in the collection — that's the `startIndex` you'll open the player at.

```html
<div class="lmuPlaylist">
  <!-- one per media, in order; data-index = forloop index, starting at 0 -->
  <div class="lmuPlaylistItem" data-index="0">
    <img src="https://chquwzbkea.cloudimg.io/<image>?width=250&height=432&force_format=webp%2Cjpeg" />
    <p class="lmuPlaylistTitle">Episode title</p>
  </div>
  <!-- … -->
</div>
```

- **Stories style:** round bubbles (`border-radius: 50%`), wrapping on desktop, horizontal scroll on
  mobile.
- **Carousel style:** portrait `9:16` posters, horizontal scroll with snap. Optionally overlay a `muted
  loop playsinline` `<video lmuVideoUrl="<videoUrl>">` that plays the HLS preview (use HLS.js — see the
  [Pop Me Up guide](05-pop-me-up-widget.md) for the HLS loading pattern). Start the preview at `startTimestamp`.

### 2. Open the player in playlist mode

Clicking any item opens the **whole collection** at that item's index, so the viewer can swipe through:

```js
function startPlayerCollection(mediaCollectionId, startIndex) {
  import('https://cdn.livemeup.io/player.esm.js').then(({ startPlayerCollection }) => {
    startPlayerCollection(
      { mediaCollectionId, startIndex: Number(startIndex) },
      { language: 'EN', country: 'US' }, // storefront locale; language upper-cased
    );
  });
}

document.querySelectorAll('.lmuPlaylistItem').forEach((el) => {
  el.onclick = () => startPlayerCollection(COLLECTION_HANDLE, el.getAttribute('data-index'));
});
```

`mediaCollectionId` is the collection metaobject's `system.handle`. `startIndex` is the clicked item's
position. See the [Player & Cart Gateway guide](01-player-and-cart-gateway.md) for the full `startPlayerCollection` contract.

### 3. Autoplay deep-link (optional)

The widgets support opening the player automatically from a URL — used by "watch this event" links shared
from the console. Check two query params on load:

- `autoplay-live-shopping-be` — the event id to autoplay.
- `t` — optional start offset in **milliseconds**.

Logic: if that event id is **in the playlist**, open the collection at its index; otherwise open it
standalone.

```js
const params = new URLSearchParams(window.location.search);
const autoplayId = params.get('autoplay-live-shopping-be');
const t = params.get('t');

if (autoplayId) {
  const index = medias.findIndex((m) => m.mediaId === autoplayId);
  if (index !== -1) {
    startPlayerCollection(COLLECTION_HANDLE, index);          // in playlist → playlist mode
  } else {
    import('https://cdn.livemeup.io/player.esm.js').then(({ startPlayer }) => {
      startPlayer(autoplayId, t == null ? { language: 'EN', country: 'US' }
                                        : { t: Number(t), language: 'EN', country: 'US' });
    });
  }
}
```

### 4. Don't forget the cart gateway

Playlist mode launches the **player**, so on a headless storefront register
`window.LiveMeUpCartGatewayV4` before the first click — see the [Player & Cart Gateway guide](01-player-and-cart-gateway.md).

> **Native apps (Tapcart):** the reference blocks also branch on `window.Tapcart` to open the player inside
> a Tapcart hybrid screen instead of the web player. Ignore this unless you're integrating a Tapcart app.

---

## Checklist

- [ ] Read the **`media_collections`** metafield (`app--5813997--media_collections` / `media_collection`)
  from the **correct owner** for the page (shop / product / collection / page).
- [ ] Render nothing if the metafield is absent or empty.
- [ ] For each item in `medias_infos`, resolve its **`lmu_replay`** metaobject (by `mediaId`) for `title` +
  `image`; build the cover via the image CDN.
- [ ] Tag each clickable item with its **0-based index** in the collection.
- [ ] On click, `import('https://cdn.livemeup.io/player.esm.js')` and call
  `startPlayerCollection({ mediaCollectionId: <system.handle>, startIndex }, { language, country })`.
- [ ] *(Carousel style)* Optional muted, looping, `playsinline` HLS **video preview** per poster, starting
  at `startTimestamp`.
- [ ] *(Optional)* Handle the `autoplay-live-shopping-be` (+ `t` in ms) deep-link.
- [ ] Pass `language` + `country`; register the **cart gateway** on headless storefronts.

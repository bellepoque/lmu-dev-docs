---
layout: default
title: Overview
nav_order: 1
permalink: /
---

# Live Me Up — Integration Documentation

Live Me Up turns your storefront into a **live shopping** and **shoppable video** experience: you run
live events and publish replays from the Live Me Up console, and your shoppers can watch them, chat, and
add the featured products straight to their cart — without ever leaving your store.

This documentation set explains how to put Live Me Up on a storefront. It is written for two audiences:

- **Developers** reimplementing the widgets by hand (custom/headless storefronts, native apps, or any
  frontend that can't use the Shopify theme app extension).
- **Clients using AI builders** (Lovable, v0, Bolt, …). Hand these documents to your AI builder and ask it
  to build the components — each guide ends with a copy-pastable **checklist** the AI can follow.

---

## Who this is for

These guides are for **headless / custom storefronts** — any frontend that **can't** use the Live Me Up
Shopify theme app extension: a headless Shopify build, a Lovable (or other AI-builder) site, a custom
React/Vue/native frontend. You rebuild each widget yourself. The widgets are deliberately thin: each one
reads a piece of data (a Shopify metafield, or a Live Me Up API endpoint) and calls the player's public JS
API. Every guide gives you the data source, the exact contract, and a framework-agnostic reference
implementation.

> **Running a standard Shopify theme instead?** You don't need any of this — install the Live Me Up app and
> add its ready-made theme blocks in the theme editor. These docs are only for storefronts where those
> blocks aren't available.

**One exception — the Live Shopping Page.** It ships as a self-contained JS bundle, so even on a headless
storefront you can add it with a single `<script>` + `<div>` (no reimplementation needed). Its guide covers
both that drop-in and a full reimplementation. **Every other widget must be rebuilt by hand** — there is no
drop-in script for them.

---

## The building blocks

| Piece | What it is | Guide |
|-------|-----------|-------|
| **The Player** | The full-screen video player (live or replay) with chat and shoppable products. Everything else is just a way to *launch* it. You also wire a **cart gateway** so add-to-cart works with your store. | [Player & Cart Gateway](01-player-and-cart-gateway.md) |
| **Live Shopping Page** | A dedicated page listing your upcoming live events (with countdowns / notify-me) and your published replays. | [Live Shopping Page](02-live-shopping-page.md) |
| **Mini Player widget** | A small floating thumbnail shown site-wide **while an event is live**, playing a muted real-time preview. Clicking it opens the full player. | [Mini Player Widget](03-mini-player-widget.md) |
| **Playlist widget** (Carousel & Stories) | A row of replay thumbnails (round "stories" bubbles or portrait posters with video previews) that open the player in playlist mode. | [Playlist Widget](04-playlist-widget.md) |
| **Pop Me Up widget** | A per-product thumbnail on a product page that opens the replay **at the exact moment that product was presented**. | [Pop Me Up Widget](05-pop-me-up-widget.md) |

> **Start with the Player guide.** The player-loading and cart-gateway concepts it describes are shared by
> every other widget — read it first, then the widget you need.

---

## Production URLs

All assets are served from the Live Me Up CDN.

| Asset | URL |
|-------|-----|
| Player (ES module) | `https://cdn.livemeup.io/player.esm.js` |
| Player (classic / IIFE build) | `https://cdn.livemeup.io/player.min.js` |
| HLS.js (only for video previews) | `https://cdn.livemeup.io/hls.js` |
| Live Shopping Page script | `https://cdn.livemeup.io/landingpage.min.js` |
| Public API base | `https://api.livemeup.io/` |
| Image CDN | `https://chquwzbkea.cloudimg.io/<imagePath>?width=<W>&height=<H>&force_format=webp%2Cjpeg` |

---

## The Live Me Up app id

Live Me Up stores its data on your store in **app-owned Shopify metafields/metaobjects**. Their namespaces
are prefixed with the app's numeric id, in the form `app--<APP_ID>--<namespace>`. Throughout this
documentation the production app id is:

```
5813997
```

So, for example, the "current live event" metafield namespace is `app--5813997--current_live`. **If
your install was given a different app id, substitute it everywhere you see `5813997`.**

---

## Glossary

- **Event** — a live shopping broadcast. While it is on air it has a live stream; once finished it can be
  published as a **replay**.
- **Replay** — the recorded, on-demand version of a past event.
- **`eventId` / `replayId`** — the unique id of an event or replay. It is the single value you pass to the
  player to launch it. (Both are launched the same way — a replay id *is* an event id.)
- **`videoId`** — the id of the live/replay video a cart action originated from. Used for **sale
  attribution** (tying a purchase back to the stream that drove it).
- **GID** — a Shopify Global ID, e.g. `gid://shopify/ProductVariant/456`. The player always speaks in GIDs,
  never bare numeric ids.
- **Cart gateway** — the object you give the player so it can add products to *your* cart. See the Player
  guide.

Questions, or a different app id than `5813997`? Contact the Live Me Up team.

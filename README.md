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

---

## Who this is for

These guides are for **headless / custom storefronts** — any frontend that **can't** use the Live Me Up
Shopify theme app extension: a headless Shopify build, a Lovable (or other AI-builder) site, a mobile app builder.
You rebuild each widget yourself. The widgets are deliberately thin: each one
reads a piece of data (a Shopify metafield, or a Live Me Up API endpoint) and calls the player's public JS
API. Every guide gives you the data source, the exact contract, and a framework-agnostic reference
implementation.

> **Running a standard Shopify theme instead?** You don't need any of this — install the Live Me Up app and
> add its ready-made theme blocks in the theme editor. These docs are only for storefronts where those
> blocks aren't available.

---

## The building blocks

| Piece                                    | What it is                                                                                                                                                                                           | Guide                                               |
|------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| **The Player**                           | The full-screen video player (live or replay) with chat and shoppable products. Everything else is just a way to *launch* it. You also wire a **cart gateway** so add-to-cart works with your store. | [Player & Cart Gateway](player-and-cart-gateway.md) |
| **Live Shopping Page**                   | A dedicated page listing your upcoming live events (with countdowns / notify-me) and your published replays.                                                                                         | [Live Shopping Page](live-shopping-page.md)         |
| **Mini Player widget**                   | A small floating thumbnail shown site-wide **while an event is live**, playing a muted real-time preview. Clicking it opens the full player.                                                         | [Mini Player Widget](mini-player-widget.md)         |
| **Playlist widget** (Carousel & Stories) | A row of replay thumbnails (round "stories" bubbles or portrait posters with video previews) that open the player in playlist mode.                                                                  | [Playlist Widget](playlist-widget.md)               |
| **Pop Me Up widget**                     | A per-product thumbnail on a product page that opens the replay **at the exact moment that product was presented**.                                                                                  | [Pop Me Up Widget](pop-me-up-widget.md)             |

> **Start with the Player guide.** The player-loading and cart-gateway concepts it describes are shared by
> every other widget — read it first, then the widgets you need.

---

## Production URLs

All assets are served from the Live Me Up CDN.

| Asset                            | URL                                                                                        |
|----------------------------------|--------------------------------------------------------------------------------------------|
| Player (ES module)               | `https://cdn.livemeup.io/player.esm.js`                                                    |
| Player (classic / IIFE build)    | `https://cdn.livemeup.io/player.min.js`                                                    |
| HLS.js (only for video previews) | `https://cdn.livemeup.io/hls.js`                                                           |
| Live Shopping Page script        | `https://cdn.livemeup.io/landingpage.min.js`                                               |
| Public API base                  | `https://api.livemeup.io/`                                                                 |
| Image CDN                        | `https://chquwzbkea.cloudimg.io/<imagePath>?width=<W>&height=<H>&force_format=webp%2Cjpeg` |

---


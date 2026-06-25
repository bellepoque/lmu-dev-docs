---
layout: default
title: Metafields & Metaobjects Reference
nav_order: 7
---

# Metafields & Metaobjects Reference

This document inventories every Shopify **metafield definition** and **metaobject definition** that the LiveMeUp app
creates on client stores.

---

## Summary

| Kind       | Namespace / Type                  | Key                                    | Owner(s)                        | Data type              |
|------------|-----------------------------------|----------------------------------------|---------------------------------|------------------------|
| Metaobject | `lmu_replay`                      | —                                      | —                               | composite              |
| Metaobject | `app--5813997--media_collections` | —                                      | —                               | composite              |
| Metafield  | `app--5813997`                    | `last_replay_with_product_highlighted` | PRODUCT                         | `json`                 |
| Metafield  | `app--5813997--current_live`      | `event_infos`                          | SHOP                            | `json`                 |
| Metafield  | `app--5813997--media_collections` | `media_collection`                     | PRODUCT, COLLECTION, SHOP, PAGE | `metaobject_reference` |

---

## Metaobjects

### 1. Replay media — `lmu_replay`

**Purpose:** Stores one shoppable-video replay as a storefront-readable object (thumbnail, replay id, title, chapter
timecodes) so themes can render replays.

- **Type:** `lmu_replay`
- **Access scopes:** storefront `PUBLIC_READ`

**Fields**

| Name      | Key         | Type                     | Required | Content                                                                                 |
|-----------|-------------|--------------------------|----------|-----------------------------------------------------------------------------------------|
| Image     | `image`     | `single_line_text_field` | yes      | Thumbnail / poster image URL                                                            |
| Replay ID | `replay_id` | `single_line_text_field` | yes      | LiveMeUp replay identifier                                                              |
| Title     | `title`     | `single_line_text_field` | yes      | Display title of the replay                                                             |
| Timecodes | `timecodes` | `json`                   | yes      | Always empty (`[]` or `{}`) — kept for backward compatibility; no data is written to it |

---

### 2. Media collection — `app--5813997--media_collections`

**Purpose:** Stores a curated collection of replay medias (title + per-media playback info) that can be attached to
storefront pages/products/collections/shop.

- **Type:** `app--5813997--media_collections`
- **Access scopes:** admin `MERCHANT_READ`, storefront `PUBLIC_READ`

**Fields**

| Name        | Key            | Type                     | Required | Content                                                                             |
|-------------|----------------|--------------------------|----------|-------------------------------------------------------------------------------------|
| Title       | `title`        | `single_line_text_field` | yes      | Collection display title                                                            |
| MediasInfos | `medias_infos` | `json`                   | yes      | Array of `{ mediaId, videoUrl, largeThumbnailUrl, startTimestamp?, endTimestamp? }` |

---

## Metafields

### 1. Last replay with product highlighted (PRODUCT)

**Purpose:** On each product, records the most recent replay in which that product was highlighted, so the storefront
can surface "seen in replay" content on the product page.

- **Namespace / key:** `app--5813997` / `last_replay_with_product_highlighted`
- **Owner:** `PRODUCT`
- **Type:** `json`
- **Access path:** on a product, `metafield(namespace: "app--5813997", key: "last_replay_with_product_highlighted")`
- **Access scopes:** admin `MERCHANT_READ_WRITE`, storefront `PUBLIC_READ`

---

### 2. Current live event info (SHOP)

**Purpose:** Shop-level pointer to the currently live event (event id, stream/timing info). Lets the storefront know
whether a live is running and how to embed it.

- **Namespace / key:** `app--5813997--current_live` / `event_infos`
- **Owner:** `SHOP`
- **Type:** `json`
- **Access path:** on the shop, `metafield(namespace: "app--5813997--current_live", key: "event_infos")`
- **Access scopes:** admin `MERCHANT_READ`, storefront `PUBLIC_READ`
- **Value:** the event payload, or `{}` when no live is active

---

### 3. Published media collection reference (PRODUCT / COLLECTION / SHOP / PAGE)

**Purpose:** Links a storefront resource to a published media-collection metaobject, so the theme can render that media
collection on the product, collection, page, or shop home.

- **Namespace / key:** `app--5813997--media_collections` / `media_collection`
- **Owners:** `PRODUCT`, `COLLECTION`, `SHOP`, `PAGE` (one definition per owner type)
- **Type:** `metaobject_reference`
- **Access path:** on the owner resource,
  `metafield(namespace: "app--5813997--media_collections", key: "media_collection")` → resolves to the referenced
  `app--5813997--media_collections` metaobject
- **Validation:** pinned to the `app--5813997--media_collections` metaobject definition
- **Access scopes:** admin `MERCHANT_READ`, storefront `PUBLIC_READ`
- **Note:** when set on the **SHOP** owner, it signals that the referenced media collection should be displayed on the store's home page.

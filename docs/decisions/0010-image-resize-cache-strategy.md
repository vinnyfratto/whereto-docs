# ADR-0010: Resize and persistently cache destination photos instead of serving Storage originals

- **Status:** Accepted
- **Date:** 2026-08-11
- **Deciders:** Vinny Fratto (with Claude)

## Context

Supabase Storage Cached Egress hit 188% of the free-tier quota (9.4GB / 5GB) before the
org had even upgraded to Pro, driven almost entirely by one bucket:
`destination-stock-images`, 8,464 stock photos averaging 261KB (up to 1.98MB), served at
full resolution into UI contexts as small as a 38px badge.

Investigation found four compounding causes, not one:

1. No image resizing anywhere in the app. Every card, tile, and badge requested the
   original file.
2. React Native's core `Image` component has no persistent cross-session cache, so the
   same photo was re-fetched on every view, every session.
3. A `preferThumb` code path used on three screens (saved list, search results, Wander
   Together cards) had been silently broken since the ingest script wrote `thumb` as an
   exact copy of the full-size `url` — the "lighter thumbnail" was never actually
   lighter.
4. `ImageCarousel` mounted every slide in a destination's gallery (some run 20+ photos)
   as a live, fetching image on render, regardless of how many the user ever scrolled
   to. Opening one destination's hero could pull its entire gallery at full resolution.

## Decision

- Route every card/tile-context image through `images.weserv.nl` (already present in
  the codebase as a Wikimedia-403 workaround) with explicit `width`/`height` params, via
  `photoSource()` in `src/utils/imageSource.ts`.
- Switch all destination-photo render sites from RN core `Image` to `expo-image` with
  `cachePolicy="memory-disk"`, so a device never re-fetches a photo it has already
  displayed.
- Rewrite `ImageCarousel` to mount only the currently-visible slide (plus the outgoing
  one, for the duration of the crossfade), prefetching one slide ahead instead of the
  whole gallery.
- Run a one-time bulk pass re-encoding every object over 750KB in
  `destination-stock-images` through weserv at width=1800 / quality=75
  (`scripts/shrink-oversized-images.ts`), plus manual crop/clean on the dozen images
  that didn't clear the line automatically (mostly MS Photos re-embedding bloated
  EXIF/XMP/ICC metadata on save, `scripts/clean-review-folder.js`).
- Deliberately **not** adopt Supabase Storage's own Image Transformations feature
  (`/storage/v1/render/image/...`), despite now having Pro-tier access to it: it bills
  per distinct "origin image" transformed beyond a 100/month quota, and this bucket has
  8,000+ images, so it would trade the Cached Egress cost for a new, comparably-sized
  metered line item on the same bill rather than actually reducing spend.

## Consequences

- `images.weserv.nl` is now a load-bearing runtime dependency for every card/tile image
  in the app, not just an edge-case proxy. It is a free, no-SLA third-party service. See
  risk R-17 for the fallback plan.
- The 750KB cap only cleaned up what was already uploaded. Nothing in
  `scripts/ingest-stock-images.ts` enforces a size ceiling on new uploads, so the bucket
  can drift back into oversized-original territory over time. See risk R-17.
- Cached Egress should now scale roughly with unique popular content touched, not with
  concurrent viewers: weserv's resize cache is shared across every user, so only the
  first viewer of a given destination pays the origin-fetch cost, not each one. The
  per-device `expo-image` cache is a separate, smaller effect — it only cuts a single
  user's own repeat visits, not first-time load across the userbase.
- Verified against live Supabase Storage request logs during multi-device (iPhone + 2
  Android) testing, not just local reasoning: confirmed card contexts route through the
  resize proxy, the lazy carousel roughly halved per-device duplicate fetches on Android
  (the worst-hit platform, likely from disk-cache-write contention when 15-20+ large
  images mounted simultaneously), and the size-cleanup pass dropped the bucket from
  2.16GB to 1.98GB with zero objects left over 750KB.

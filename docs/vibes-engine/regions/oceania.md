# Vibe Engine — Oceania

_Last updated: 2026-08-06. See [../README.md](../README.md) for how the pipeline works._

1 of 2 sub-regions complete. 34 vibes built so far (22 complete + 12 in progress), 348
scored destination rows so far.

| Sub-Region | Vibes | Scored Rows | Countries | Gateways | Status |
| --- | --- | --- | --- | --- | --- |
| Australia & New Zealand | 12 of 25 | 120 | 2 | 44 | **In progress** |
| Pacific Islands | 22 | 228 | 18 countries/territories | 44 | v1.1, Hawaii removed |

## Notes

- **Australia & New Zealand is not finished.** 12 of the planned 25 vibes are written
  (the Big City, Coast & Water, and part of the Nature & Adventure groups); the
  remaining 13 (island escapes, coastal road trips, Great Walks, geothermal wonders,
  wildlife, whales, rainforests, Aboriginal & Maori culture, wine country, luxury lodges,
  hot springs, ski fields, scenic rail) are outstanding. Progress log:
  `Content/VibesEngine/Oceania/AustraliaNZ/_PROGRESS.md`.
- **Pacific Islands** covers French Polynesia, Fiji, Samoa, Tonga, Vanuatu, New Caledonia,
  the Cook Islands, and Micronesia, plus flagged cross-border additions at Easter Island,
  Papua New Guinea, and Guam.
- **Hawaii is no longer here.** It moved to US Pacific under North & Central America in
  2026-08, because the `destinations` table has always filed it as `region NA` /
  `subregion north_america` and the engine had to agree. All 22 Pacific Islands blogs were
  rewritten to remove it, and the `honolulu_waikiki` vibe was renamed `key_island_stops`
  since the merge left it holding seven island capitals and no Honolulu. See
  [../hawaii-subregion-anomaly.md](../hawaii-subregion-anomaly.md).

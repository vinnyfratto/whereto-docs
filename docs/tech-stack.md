# Tech Stack

_Living doc. Update when a dependency is added, removed, or majorly upgraded. Last
updated: 2026-07-05 (from `package.json`, `app.json`, `eas.json`)._

## At a glance

| Layer | Choice |
| --- | --- |
| Mobile framework | Expo SDK 56 (`~56.0.4`), React Native 0.85.3, React 19.2.3 |
| Routing | expo-router `~56.2.6` (file-based, typed routes) |
| Language | TypeScript `~6.0.3` |
| Compiler | React Compiler enabled (`experiments.reactCompiler`) |
| State | Zustand 5 |
| Styling | In-house Skins design-token system (`src/Skins/`), see [ADR-0006](decisions/0006-skins-design-tokens.md) |
| Animation / gesture | react-native-reanimated 4.3.1, react-native-gesture-handler, react-native-worklets |
| Icons | Solar via Iconify codegen (`@iconify-json/solar`), see [ADR-0008](decisions/0008-icon-codegen.md) |
| Fonts | DM Serif Display + Nunito Sans (Google Fonts) |
| Auth | expo-auth-session (Google OAuth), expo-secure-store, expo-crypto |
| Local storage | @react-native-async-storage/async-storage |
| Notifications | expo-notifications (+ Expo push) |
| Analytics | posthog-react-native 4.x |
| HTTP | axios, node-fetch |
| Backend | Supabase (Postgres, Auth, Storage, Edge Functions in Deno/TypeScript) |
| Flights | Duffel (server-side), behind `src/providers/flights/` |
| Hotels | LiteAPI, behind `src/providers/hotels/` (legacy Priceline provider retained) |
| Email | Resend (via Edge Functions) |
| Dev/build tooling | EAS Build + Submit, tsx, @expo/ngrok, Anthropic SDK, sharp, exceljs, xlsx |

## Build and release

- `eas.json`: `appVersionSource: local` (version and versionCode read from `app.json`).
- Build profiles: `development` (APK, dev client), `preview` (app-bundle, internal),
  `preview-apk`, `production` (app-bundle).
- Submit: `production` to Google Play **internal** track via service account.
- **Android only** today. No iOS build or submit profile yet.

## Dependency and license inventory (SBOM starter)

A full SBOM should be generated from the lockfile. Until that is automated, this is the
watch-list of licenses that carry obligations or risk:

| Dependency | License | Obligation / note |
| --- | --- | --- |
| Solar icon set (Iconify) | CC BY 4.0 | **Attribution required** in-app at go-live. Tracked as a launch task. |
| DM Serif Display, Nunito Sans | OFL 1.1 | Open Font License, redistribution allowed, keep license text. |
| Expo, React Native, most JS deps | MIT / Apache-2.0 | Permissive, no material obligation. |
| `LICENSE` file at repo root | MIT (stock Expo) | **Wrong.** See [GAP-REPORT.md](GAP-REPORT.md) G-02. Must be replaced with a proprietary notice. |

**Action:** automate SBOM generation. Recommended: run a license scanner in the weekly
job and diff the output. Command to add:

```bash
npx license-checker --production --json > docs/sbom.json
```

## How to update this doc

The weekly agent compares `package.json` week over week. When a dependency is added or
changed it updates the "At a glance" table and appends any new license obligation to the
inventory above. New permissive-licensed deps do not need a row; anything copyleft
(GPL/LGPL/AGPL), CC, or with an attribution clause **must** get one.

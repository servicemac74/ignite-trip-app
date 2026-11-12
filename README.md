# ignite-trip-app

A single-page trip planner/checklist app, originally built for Chris and Inez's November 2026 Microsoft Ignite trip, now also doubling as a conference-session planner. Live at **https://servicemac74.github.io/ignite-trip-app/**.

If you are an AI coding assistant picking this repo up with no memory of prior sessions, this file plus [`AGENTS.md`](AGENTS.md) are meant to be everything you need. Project history and forward planning that doesn't belong in the codebase live in the linked Obsidian vault (see **Where things live**, below) — read this file and `AGENTS.md` first regardless of whether you can reach the vault.

## Architecture

- **One file, no build step.** `index.html` is the entire app: HTML + CSS + vanilla JS inline. No `npm install`, no bundler, no framework. Open it in a browser and it runs.
- **State lives in one JS object, `STATE`**, defined/seeded by `defaultState()` near the top of the `<script>` block. Five top-level lists: `checklist`, `days`, `notes`, `places` (`notes` and `checklist` share a `parentId`-tree shape for unlimited nesting; `days` is a fixed array of trip dates with `entries`; `places` is a flat array — see **Data model: Places** below).
- **Persistence is two-layer:**
  - `localStorage` (key `ignite-trip-2026-v1`) — always on, works with no network, keeps the app usable offline.
  - **Firebase Realtime Database** (project `ignite-trip-app`, path `sessions/ignite-trip`, whole `STATE` stored as one JSON value at that path) — syncs across devices when online. Entirely optional at the code level: if the Firebase SDK fails to load or the network is down, the app still works from `localStorage` alone.
- **Schema versioning guards against data loss.** Every write tags the payload with `SCHEMA_VERSION` (currently `8`, declared near the top of the script). If a client ever sees a *higher* version already saved than it understands, it blocks all editing with a full-screen "out of date, reload" message instead of silently overwriting newer data with an older shape. See `AGENTS.md` for the rule this implies about changing the data model.
- **Hosting is GitHub Pages**, not Firebase Hosting. `firebase.json`/`.firebaserc`/`database.rules.json` configure only the Realtime Database (rules deploy), not a hosted site — see **Deployment**.

## Data model: Places (the newest, most actively-changing part)

Each entry in `STATE.places` is a flat object (see `migratePlaces()` in `index.html` for the authoritative shape and defaults):

| Field | Meaning |
|---|---|
| `id` | unique id |
| `label` | display name |
| `mapsUrl` | the Google Maps link, if one was pasted (kept as the "open in Maps" deep link even after geocoding) |
| `lat` / `lng` | coordinates, numeric or `null` if geocoding hasn't resolved the place yet |
| `date` | which trip day this belongs to (`""` = "No day" bucket) |
| `category` | `"personal"` or `"conference"` — a second, independent top-level filter so the same app can plan the family trip and a conference schedule at once |
| `tier` | `"love"` or `"pass"` — set via a dropdown on the card, or by dragging the card between the two tier sections |
| `time` | optional `"H:MM AM/PM"` string, same picker Trip Days uses |
| address | street address; editing retries geocoding and updates Maps link |
| notes | free text for highlights, duration, verified details and uncertainties |
| references | array of {label, url, creator}; HTTP(S) links; creator optional |
| `ptype` | one of `restaurant` / `site` / `touristTrap` / `logistics` / `conferenceSession` / `vineyard`, or `""` if untagged |

**Filters** (`_selectedPlacesDate`, `_selectedCategory`, `_selectedPtype`) are independent and all apply at once — see `placesForView()`. A newly-added place is filed under whichever day + category is currently selected (so there's no separate day/category picker in the quick-add form); type is the one field left untagged on add and set afterward from the card.

**All days:** includes dated and undated places in the list and map, respecting category/type filters. No day shows only unscheduled places. Quick-add in All days creates an undated place. Places without coordinates remain in the list.

**Details and references:** Edit details & links edits address, notes and multiple labeled links. Save persists; Cancel discards. URLs must use HTTP(S). Older places default to empty fields; numeric-keyed Firebase reference arrays normalize on read.

**Map/color:** a free Leaflet.js + OpenStreetMap mini-map (no API key, no cost) renders a pin per visible place, colored by whichever of two independent signals the `_colorMode` toggle currently selects — `"tier"` (bright pink = love, muted gray = pass) or `"time"` (gold/blue/coral/purple by time-of-day bucket, see `placeColor()`/`timeOfDayBucket()`). Whichever signal isn't driving color still shows as the pin's glyph.

**Geocoding:** pasting a full Google Maps link with embedded coordinates (`@lat,lng` or `q=lat,lng`) works instantly with no network call. Typing a plain address, or pasting a *shortened* Google "Share" link (`maps.app.goo.gl/...`, which never carries coordinates and can't be resolved client-side — a CORS limitation, not a bug in this app), falls back to free Nominatim (OpenStreetMap) geocoding with a progressive-retry fallback (`geocodeWithFallback`/`buildGeocodeAttempts`) for addresses its parser initially misses. **The fix for a dead-end shortened link is renaming the place** to its real name/address — renaming retries geocoding, same as adding a new place.

**Conference sessions and vineyards:** both are available in the type filter and each place's type dropdown, independently of the Personal/Conference category filter. Use the existing day, time, place name/address, and Love it/Or not controls to plan a session. The app's `time` field and time-of-day bucketing remain timezone-naive (plain "H:MM AM/PM" strings, no timezone attached). Enter Pacific start times as plain time values; automatic timezone handling is deferred.

## Running and validating locally

No install step. Two ways to run it:
1. Open `index.html` directly in a browser (`file://`) for a quick look. Firebase sync may be flaky over `file://` in some browsers' security models — fine for checking markup/layout/local-only logic, not for testing sync.
2. Serve it over plain HTTP for anything involving Firebase sync or multi-tab/multi-device testing — any static file server pointed at the repo root works (e.g. `npx serve .`, or `python -m http.server`). Claude Code sessions on this project use a `.claude/launch.json` Node one-liner in the Obsidian vault (see **Where things live**) — not required, just what this environment happens to use.

To validate a change before pushing:
- Open the browser console; confirm no JS errors on load and none while exercising the tab(s) you touched.
- Exercise the actual feature by hand (click through it) — this app has no automated test suite.
- If you changed the shape of anything in `STATE` (added/removed/renamed a field on an existing record type), **bump `SCHEMA_VERSION` and extend the matching `migrate*()` function** so older saved data (local or already in Firebase) upgrades instead of breaking. See `AGENTS.md`.
- If you touched sync, test with two tabs/devices open against the same Firebase project and confirm an edit on one appears on the other within a second or two, and that reloading doesn't lose data.

## Deployment

There is no build/deploy command. `git push` to `master` is the deploy:
- GitHub Pages is configured to deploy from `master`, folder `/ (root)` (Settings → Pages on the GitHub repo).
- A push triggers a Pages rebuild automatically, live at the Pages URL within roughly a minute.
- The in-app update-check banner (polls the page's own `Last-Modified`/`ETag`) exists specifically because phones that "Add to Home Screen" can keep a stale copy in memory for days after a push — see `AGENTS.md` before touching that logic.

Database rule changes (not app code) deploy separately, only if `database.rules.json` changes: `firebase deploy --only database --project ignite-trip-app` (requires the Firebase CLI logged in as the account that owns this project).

## Where things live (so you don't have to reconstruct this from chat history)

This repo is one app built from a reusable pattern documented in an Obsidian vault at `C:\Users\dad\Documents\ObsidianVault\alpharemote-mac-obsidian-01` (not part of this repo; may not be reachable from every environment this app is edited in):

- **`APP TEMPLATES/family-web-app/TEMPLATE_PROCESS.md`** — the reusable process for building *any* new app on this same single-file-HTML + Firebase + GitHub Pages pattern (one-time setup, per-app checklist, the standing rules every app on the pattern follows — schema versioning, array normalization, the echo-loop guard — and the registry of which app uses which Firebase project/path). This repo used to carry a committed copy of that file; it's been removed in favor of linking here, so the vault copy stays the single source of truth and this repo doesn't go stale against it. If you can't reach the vault, `AGENTS.md` in this repo restates the specific rules that matter for editing *this* app's code.
- **`ENT FAM EVENTS/MS IGNITE 2026/`** — this app's own project folder: trip logistics, and `disco/DISCO - 26.11.09 - ignite app feature updates.md`, a dated feature-by-feature changelog of everything built in this repo, kept current by pulling from `git log` each session.
- **This repo** (`README.md` + `AGENTS.md`) — the implementation and operating instructions: what's actually true about the current code, independent of any particular chat's memory.

## Authoritative checkout

GitHub (`servicemac74/ignite-trip-app`, branch `master`) is the source of truth. On the machine this app is normally edited from, the durable local clone is `C:\Users\dad\Documents\ignite-trip-app` — pull it (`git pull --ff-only`) before editing rather than assuming it's current, and don't rely on a path under a Claude Code session's own temp scratchpad directory as "the" checkout: those are session-scoped and get cleaned up.

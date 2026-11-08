# Family web app template — setup & process

How we build a quick, free, synced-across-devices web app for the family (to-do lists, planners, trackers — anything shaped like "a few people need to see and edit shared data from their phones"). First example: this repo (`ignite-trip-app`).

**Stack:** one self-contained HTML file (vanilla JS, no build step) + Firebase Realtime Database for live sync + GitHub Pages for free hosting. No accounts needed for anyone using the app — just the link.

---

## 0. One-time setup (already done, nothing to redo)

- GitHub Desktop installed, linked to the `servicemac74` GitHub account.
- Firebase project `mcdonald-dashboard-01` exists, with Realtime Database security rules already open for `sessions/`, `users/`, and `budget/` (Firebase Console → Realtime Database → Rules). Any path under `sessions/` is usable by a new app with no rules change.

---

## 1. Per-app checklist (repeatable, ~10–15 minutes)

### A. Local files

1. Create a new local folder, e.g. `C:\Users\dad\Documents\<app-name>\`.
2. Copy `index.html` from an existing app (e.g. this repo) as the starting skeleton. Keep as-is:
   - The CSS theme block (`:root` variables, layout).
   - `esc()`, `slug()`, `uid()` helper functions.
   - The entire **Firebase Sync Module** IIFE at the bottom (`var FB = (function(){ ... })();`) — don't rewrite this from scratch each time.
3. Replace, per app:
   - The data model (`STATE` shape) and `defaultState()` seed content.
   - The render functions and the HTML structure they build.
4. Change exactly **one line** in the Firebase module: `_ref = _db.ref('sessions/<app-name>');` — pick a short, unique slug that can't collide with another app (`mcd`, `redhelm`, and `ignite-trip` are already taken).

### B. GitHub repo

1. GitHub Desktop → File → Add Local Repository → point at the new folder.
2. If it's not yet a git repo, Desktop offers to create one — accept.
3. Commit the files (first commit).
4. **Publish repository** → name it → **uncheck "Keep this code private"** (GitHub Pages needs a public repo on the free plan) → Publish.

### C. GitHub Pages

1. On github.com, open the repo → **Settings → Pages**.
2. Build and deployment → Source: **Deploy from a branch**.
3. Branch: `main` (or `master`, whatever Desktop created) → Folder: **`/ (root)`** → Save.
4. Wait 1–2 minutes. Live at `https://servicemac74.github.io/<repo-name>/`.
5. Every later `git push` (mine or yours) auto-redeploys within about a minute — no separate deploy step.

### D. Firebase — reusing the existing project (default choice, what this app does)

- Nothing to configure in the Firebase console. The existing rules already open `sessions/` and everything under it.
- Just make sure the app's `_ref` path is `sessions/<unique-app-name>` so it can't collide with another app's data.
- Test by opening the Pages URL in two browser profiles (or two devices) and confirming an edit on one shows up on the other within a second or two.

### E. Firebase — a separate project instead (only when an app needs full isolation)

Use this if a future app shouldn't share billing/quota or security rules with `mcdonald-dashboard-01` — for example something outside the family, or with different access needs.

1. console.firebase.google.com → Add project → name it (Analytics optional, can decline).
2. Build → Realtime Database → Create Database → choose a location → start in locked mode.
3. Project settings (gear icon) → General → "Your apps" → Add app → Web (`</>`) → register (no Firebase Hosting needed, we use GitHub Pages).
4. Copy the generated `firebaseConfig` object into the new app's FB module, replacing the whole `_config` block.
5. Realtime Database → Rules tab → open the paths this app needs, same pattern as the existing rules:
   ```json
   { "rules": { "<path>": { ".read": true, ".write": true } } }
   ```
   → Publish.
6. This project now scales/bills independently of `mcdonald-dashboard-01`.

---

## 2. Giving family members access

- They just need the Pages URL. No GitHub account, no Firebase account, no login — anyone with the link can use the app and see live updates.
- This also means anyone with the link can **edit** the shared data. Fine for a family-trust-level tool; if a future app needs read-only viewers or locked-down editors, that requires Firebase Auth + UID-scoped rules — a real step up in complexity, not set up on any app yet. Flag it if a future app needs that.

---

## 3. Rules carried forward from the Dashboard app (don't relearn these the hard way)

- Firebase Realtime Database silently converts JS arrays to numeric-keyed objects on write. Always run incoming array fields through an `_fbArr()`-style normalizer before using them — it's already in the template's FB module; keep it when copying to a new app.
- The `_lastSavedAt` / payload `savedAt` timestamp guard stops a device from echoing its own write back to itself. Don't strip this out when simplifying.
- `scheduleWrite()`'s 300ms debounce batches rapid edits into one write. Keep it — don't call the internal write function directly from UI event handlers.
- Last-write-wins on conflict, no merge logic. Acceptable for a family-scale tool; call it out explicitly in a new app's own notes if true conflict resolution is ever needed.
- Firebase is additive, not required: a page should still render and be usable (locally, via `localStorage`) if Firebase is unreachable, same as this app does.

---

## 4. Known app slugs in use under `mcdonald-dashboard-01` → `sessions/`

| Slug | App |
|---|---|
| `mcd` | MCD Dashboard (weekly SharePoint tracker board) |
| `redhelm` | Redhelm Dashboard (same, Redhelm tracker) |
| `ignite-trip` | This app — Ignite 2026 trip checklist + day planner |

Add a row here whenever a new app is built from this template, so the slug is never reused by accident.

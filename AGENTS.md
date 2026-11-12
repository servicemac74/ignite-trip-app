# AGENTS.md — ignite-trip-app

Model-neutral editing instructions for this repo. Applies to any coding assistant (Claude, Codex, or otherwise) — there is no separate `CLAUDE.md` in this repo, and there shouldn't be one: this file is the one canonical set of editing rules. Read [`README.md`](README.md) first for architecture/data-model/run/deploy; this file is the rules for safely changing the code.

## Invariants — do not break these without understanding why they exist

All of this is implemented in `index.html`; search for the function name to find it.

1. **Every write goes through `persist()`.** It saves to `localStorage`, re-renders, and (if Firebase loaded) calls `FB.scheduleWrite()`. Don't mutate `STATE` and skip it — that silently desyncs the UI, local storage, and Firebase from each other.
2. **Never call the Firebase write function directly from a UI handler.** `scheduleWrite()` debounces (300ms) so rapid edits coalesce into one write. Calling the internal `_doWrite()` directly defeats that and can hammer the database.
3. **Firebase reads must go through `_fbArr()`** before being used as arrays. Firebase's Realtime Database silently turns JS arrays into numeric-keyed objects on write; reading one back without normalizing breaks `.map()`/`.filter()`/`.forEach()` on that data the moment another device has written to it.
4. **The `savedAt` timestamp guard (`_lastSavedAt`) is what stops a device from echoing its own write back to itself** as if it were a remote change. Don't remove it when touching the sync listener.
5. **Firebase is additive, never required.** The app must still render and be usable from `localStorage` alone if the Firebase SDK fails to load or the network is down. Don't add a code path that assumes `FB`/`firebase` exists.
6. **Any breaking change to the shape of `STATE.checklist`, `STATE.notes`, or `STATE.places`** (new field, renamed field, restructured nesting) requires, together, in the same change:
   - Bump `SCHEMA_VERSION` by 1.
   - Extend the matching `migrate*()` function (`migrateChecklist`/`migrateNotes`/`migratePlaces`) so older saved data — local or already in Firebase — gets the new field defaulted sanely instead of silently dropped or crashing a filter/render that now expects it.
   - Update the version-history comment directly above `var SCHEMA_VERSION` — it's the one place that explains what each version number means; keep it a living log, not a stale snapshot.

   This is the mechanism that makes `showStaleBlock()` work: a client that's behind the saved schema version gets blocked from editing (full-screen "out of date, reload" message) instead of silently overwriting newer data with an old-shaped payload. **Skipping the version bump is the one mistake in this codebase that causes real, hard-to-notice data loss** — it won't show up in your own testing, only on someone else's device running the old code days later.
7. **Pointer Events, not HTML5 Drag-and-Drop**, for the Places tier drag interaction (`tierDragStart`/`tierDragMove`/`tierDragEnd`) — iOS Safari doesn't support native HTML5 DnD reliably. If you touch this, keep `pointer-events:none` on the dragged element for the duration of the drag; removing it reintroduces a hit-testing bug where `elementFromPoint` detects the dragged element itself instead of the drop target underneath it.
8. **Reentrant save-on-blur:** `saveEditPlace`/`saveEditNote` guard against a reentrant call (a stray synchronous `blur` fired mid-DOM-rebuild, before the previous element is actually removed) with a module-level "currently saving this id" flag, checked *before* any DOM mutation. Don't replace that with a plain `if(!el) return` — the element can still exist at the instant of the reentrant call.
9. **No external API keys.** Geocoding is free Nominatim (OpenStreetMap), mapping is free Leaflet tiles. Don't introduce a paid/keyed mapping or geocoding service without Chris's explicit go-ahead — this was a deliberate cost decision, not an oversight.

## Workflow

- **Discuss → edit → test → push.** Chris's established pattern for this app: describe/confirm the change, make it, verify it (see README's "Running and validating locally"), then push. For a small, unambiguous fix this can compress to edit → test → push with a clear commit message describing what and why.
- **Commit messages** describe the change and, where it isn't obvious, why — this repo's own `git log` doubles as the feature changelog that gets pulled into the vault's `ENT FAM EVENTS/MS IGNITE 2026/disco/` page (see README's "Where things live"). A vague commit message degrades that changelog for whoever reads it next, human or AI.
- **Don't duplicate `README.md`/this file's content into a new per-session doc.** If something here is wrong or missing, fix it here.

## Access and publishing

These two are different and shouldn't be conflated:

- **Editing permission** — being able to read and write files in the authoritative local checkout (`C:\Users\dad\Documents\ignite-trip-app` on the machine this is normally edited from; see README's "Authoritative checkout"). This is an environment/filesystem-permission question specific to whatever machine or sandbox you're running in — this file can't grant it, only describe where the checkout is.
- **Publishing authorization** — `git push` to `origin master`, which is what actually makes a change live (GitHub Pages deploys from `master` on every push). Editing local files doesn't publish anything by itself. Within the working pattern above ("discuss → edit → test → push"), push once the change has been discussed and tested — not before, and not silently. A different assistant picking this up without that established relationship should get Chris's explicit go-ahead before the first push of a session, the same way any agent should confirm before an action that's visible to others or hard to reverse.
- **Checking git auth without exposing credentials:** `git ls-remote origin` confirms read access without pushing anything. `git push --dry-run origin master` (or `HEAD`) simulates a push — including the auth check — without actually sending anything, so it's a safe way to confirm push access is configured before relying on it. Never print, log, or paste an actual token/credential to confirm it "looks right" — the dry-run commands above answer that without ever displaying the secret.
- **Firebase CLI access** (only needed if `database.rules.json` changes) is separate from GitHub auth — it requires the Firebase CLI logged in as the account that owns the `ignite-trip-app` project. `firebase projects:list` confirms that login without deploying anything.

## What's out of scope here

- Building a *new* app on this same pattern (new repo, new Firebase project, GitHub Pages setup from scratch) is process knowledge that belongs in the vault's `APP TEMPLATES/family-web-app/TEMPLATE_PROCESS.md`, not here — this file only covers maintaining *this* app's existing code.
- Project history, trip logistics, and forward planning for this app live in the vault's `ENT FAM EVENTS/MS IGNITE 2026/` folder, not in this repo. If you can't reach that vault from your environment, that's a historical/planning context gap, not a blocker to safely editing the code — everything required to do that safely is above.

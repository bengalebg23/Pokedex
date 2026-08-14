# House Ledger — Handoff

A family household-chores PWA. Single-file `index.html` served from GitHub Pages, with localStorage + Firebase Realtime Database for persistence. Built around an existing paper-spreadsheet chore tracker (`Chores.pdf`) used by the Gale family at 17 Melody Drive, Sileby.

**Live app:** https://bengalebg23.github.io/Chores/
**Repo:** https://github.com/bengalebg23/Chores
**Current version:** v1.6 (2026-05-09)
**Branch:** `main` (no dev/staging split — push straight to live)

---

## Architecture

Single-file PWA. `index.html` contains everything:
- React 18 + Babel standalone (compile-in-browser, no build step)
- Tailwind via CDN
- Firebase compat SDK v10.7.1 (app + database modules)
- Inline service worker registration

Companion files:
- `sw.js` — service worker, network-first for navigation. **Bump `CACHE_VERSION` on every release** to force clients to fetch fresh code.
- `patches/` — every release patch archived here for audit/replay. Convention: `vX.Y_description.sh`.

**No build step, no node_modules, no bundler.** Everything ships as-is.

### Storage model

- `localStorage` key: `house-ledger-v1` — JSON `{ history, tasks }`. Read first on load (fast).
- Firebase Realtime Database `/history` node — source of truth, source of recovery.
- Theme preference: `localStorage["house-ledger-theme"]` = `"dark"` or `"light"`.
- On every state change, both stores are written. Firebase subscribes to `/history` so changes from any device propagate. Last write wins.
- If Firebase is empty on first load but localStorage has data, the local data is pushed up to seed the cloud.

### Firebase config

Project: `house-ledger-26622` (separate from meal planner project)
DB URL: `https://house-ledger-26622-default-rtdb.europe-west1.firebasedatabase.app/`
Rules: writes restricted to `/history` path; reads/writes open without auth (no Firebase Auth setup yet). Family-scale, not internet-public.

---

## Workflow

### Environment

- **Termux on Pixel**, with git authenticated to push to `bengalebg23/Chores`
- Repo at `~/Chores/`
- Shell aliases in `.bashrc`:

```bash
c() {                       # quick push, no tag
  cd ~/Chores && git add -A && git commit -m "${1:-update}" && git push
}

ct() {                      # tagged release push
  if [ -z "$1" ]; then echo "Usage: ct <version>"; return 1; fi
  cd ~/Chores && git add -A && git commit -m "v$1" && git tag "v$1" -m "v$1" && git push && git push --tags
}
```

### Patch style

Patches are **Python heredocs wrapped in bash**, applied to `~/Chores/index.html` in place. They `assert` against expected pre-state — failing loudly if the file isn't in the version they were authored for.

**NEVER deliver downloaded HTML files for Ben to `cp` into the repo.** That path has a long history of going wrong on this project (wrong files picked up from Downloads, Pages cache confusion). Patches only.

Standard delivery: I generate a `.sh` file, Ben downloads it once, runs:

```bash
cp ~/storage/downloads/<patch>.sh ~/Chores/patches/vX.Y_description.sh
cd ~/Chores
bash patches/vX.Y_description.sh
sed -i "s/const CACHE_VERSION = .*/const CACHE_VERSION = 'vX.Y';/" sw.js
ct X.Y
```

Pages takes 1–2 min to rebuild. Service worker auto-reloads clients once new SW takes control.

### Cache invalidation

Every patch **must** bump:
1. `VERSION` constant in `index.html` (e.g. `const VERSION = '1.7'`)
2. `VERSION_DATE` constant in `index.html`
3. `CACHE_VERSION` in `sw.js` (e.g. `'v1.7'`)

The SW change is what forces installed PWAs to reload. Skipping it means clients keep serving the old cached version forever.

---

## Version history

- **v1.0** — Initial release. Bed Change, Hoover, Dusting, Grooming, Misc groups. List + calendar views. localStorage only.
- **v1.1** — Bathrooms group added (Lower, Middle, Upper × toilet/sink/floors plus bath/shower). Version stamp in footer.
- **v1.2** — Split into `index.html` + `sw.js` (was single self-contained file). Proper cache invalidation via `CACHE_VERSION`.
- **v1.3** — *Tag exists but commit was botched* (pushed wrong file from cluttered Downloads folder). Skip.
- **v1.4** — Recovered correct file. Added "merge defaults with saved tasks" loading logic so new task groups appear for existing users without wiping their data.
- **v1.5** — Dark mode. Designed dark palette (deep ink-brown backgrounds, warm cream text, muted-but-distinct status colours), manual toggle (sun/moon icon top-right of masthead), persisted preference.
- **v1.6** — Firebase Realtime Database sync. localStorage stays as fast local buffer + offline write queue. `☁ synced` indicator in footer.

---

## Known gotchas

### Pages cache lag (real)
GitHub Pages has its own CDN cache layer that's slower than git. After a push, the raw GitHub URL might show fresh content while `bengalebg23.github.io/Chores/` still serves stale. Use `?bust=randomstring` query strings on the live URL to verify, or push an empty commit to nudge a rebuild:

```bash
git commit --allow-empty -m "trigger pages rebuild" && git push
```

### The "fresh artifact" problem (Claude chat-level, not the app)
When Ben asks for changes to a Claude artifact mid-chat, modifying the artifact creates a new instance with no data. This burned us in early versions. Mitigation now: patches are made to live code on the phone, not to artifacts.

### Downloads folder collisions
Ben's `~/storage/downloads/` has *many* `index.html` files from different Claude projects (meal planner, enemy sheets, others). Old patches used `cp ~/storage/downloads/index.html` which silently grabbed the wrong file. **This is why we moved to patch-style.** If we ever go back to file-download workflow, use distinctive filenames like `index_v1.x.html`.

### Pages might serve a different app's file
At one point the live URL was serving a totally different chore-tracker app (dark mode, DM Serif Display, generic "Vacuum floors / Do laundry" tasks). Source unknown — possibly an old artifact-export from a different chat that got copied in. The recovery was a real-file push + `git commit --allow-empty` to force Pages rebuild.

### Firebase rules are wide open
Currently no auth. Writes restricted to `/history` path. For real lockdown we'd need Firebase Auth setup — flagged for a future session.

### Data loss on Apr 25–May 9
The early localStorage-only versions lost data more than once. Likely cause: Chrome on Android evicts site storage under pressure when the site isn't installed as a PWA. v1.6 + Firebase makes this impossible to repeat. **Ben had a small number of pre-v1.6 entries that are gone; don't try to reconstruct.**

---

## Design philosophy

- **Tactile paper aesthetic, not tech-dashboard.** Cream paper background with subtle ruled-line texture, Fraunces serif for display, JetBrains Mono for technical metadata. Warm sepia accents. Inspired by bullet journals and old library ledgers — see masthead "No. 001 · Est. [year]" framing.
- **Dark mode is warm too** — deep ink-brown, not black. Candlelight on dark wood, not VSCode.
- **Status colours are the only saturated palette element.** Stat cards (Overdue/Due/Fresh) deliberately stay vivid in both modes — they're meant to grab the eye.
- **Bullets in interface, never in chat output.** Ben dislikes report-style structured chat responses. Keep mobile-friendly: brief paragraphs, lead with the answer.
- **Honest mistakes get owned.** Ben values diagnostic transparency over recovery theatre. When something breaks, dig and explain, don't paper over.

---

## Open threads

1. **Multi-user attribution** — currently single-user. Add `loggedBy` field to each Firebase entry (Ben/Emily/Reuben/Vivien), with a first-load name prompt (cf. meal planner's user system).
2. **Firebase Auth** — replace open rules with proper auth-based read/write rules.
3. **PWA install** — Ben wants this installed to home screen for protected-storage status. Install option hasn't been appearing reliably; might need engagement heuristic time or explicit "Add to home screen" via Chrome menu.
4. **Historical data import** — Ben has the original `Chores.pdf` with last-done dates for several tasks (mid-late April). Could be entered manually via the existing date-picker UI or batch-imported via a one-off script that writes directly to Firebase.

---

## To resume in a new chat

Open a fresh chat with a prompt like:

```
I'm continuing work on House Ledger, a household chores PWA.
Please fetch:
- https://raw.githubusercontent.com/bengalebg23/Chores/main/HANDOFF.md
- https://raw.githubusercontent.com/bengalebg23/Chores/main/index.html
- https://raw.githubusercontent.com/bengalebg23/Chores/main/patches/v1.6_firebase_sync.sh

Current version: v1.6
Workflow: patches via Python heredoc, applied in Termux, pushed via `ct X.Y` alias.
NEVER deliver downloaded HTML files — always patches.

[What I want to change today: ...]
```

That gets everything bootstrapped in two fetches.

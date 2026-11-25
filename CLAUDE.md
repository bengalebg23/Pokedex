# The Full Index (Pokédex)

National Pokédex tracker for Ben's son's card collection: species list with type/number/rarity filters, a caught tracker, a 5-star rarity system with catch celebrations, a Binder Guide mapped to the physical binder, and a view-only Friends tab.

Live: https://bengalebg23.github.io/Pokedex/

## How Ben works
- Ben works from his phone. He asks for changes in chat and Claude pushes them straight to the deploy branch. There's no local build and no pull-request review.
- Pushing to `main` deploys via GitHub Pages within a minute or two.
- The repo is **public**. Never commit secrets, API keys meant to stay private, family photos or personal data.

## Every release
- Bump `APP_VERSION` in `index.html`. There is no service worker in this repo, so there's no cache version to bump.
- Keep `HANDOFF.md` up to date (version, what changed, open threads) if the repo has one.
- After pushing, give Ben the live URL and the new version number so he can check it on his phone.

## Code rules
- Single-file app: keep HTML, CSS and JS in `index.html`. No build step, bundler or npm packages.
- Pin exact CDN versions (no `@latest` or floating majors). A floating Babel major broke an app before.
- Self-review before pushing: duplicate `const`/`let` in one scope, variable shadowing inside loops, unclosed template literals, undefined references.
- Defer re-renders triggered inside click handlers with `setTimeout(..., 0)` to avoid detached-node bugs from bubbled events.
- Don't swallow Firebase errors with a bare `console.warn`. Surface them so permission problems are visible.
- UK English in all user-facing text.
- Bulk Firebase writes: use direct awaited `set()` calls. Debounced save helpers only keep the last write in a tight loop.
- Never let a fresh or cleared device overwrite cloud data on first load. Read the cloud first, merge, then write.

## This app
- Binder layout: page 1 is single-sided with 9 slots (3×3). Every page after it is a double-page spread of 18 slots (9 left + 9 right).
- Firebase project `pokemonapp-19788` (europe-west1) with email/password sign-up for friends. The Friends tab is view-only browsing, never a ranking or race.
- Keep it fun and readable for a 6-year-old: big celebrations, simple words.

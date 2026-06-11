# Code review (2026-06-11): optimization opportunities

> Prepared as a GitHub Issue, but Issues are disabled on this repository.
> Once enabled, this file can be filed verbatim. A companion document covers
> bugs and correctness issues.

None of these are broken today; they reduce run time, runner load, Actions minutes, and failure surface. Ordered by expected payoff.

## 1. Skip the browser for API-only courses

Miami Shores is fetched via the Chronogolf club-widget JSON API (`fetch_chronogolf_club_teetimes`, plain `requests`), yet `check_course` unconditionally launches a Chromium instance for it that is never used by that path. Launch the browser lazily — only when the course actually needs a Playwright scraper. Saves one full Chromium boot per run.

## 2. Move blocking I/O off the event loop

All five courses run concurrently via `asyncio.gather`, but several calls inside the coroutines are synchronous and stall every other course while they run:

- `fetch_chronogolf_club_teetimes` — `requests.get` with a 30 s timeout
- `send_pushover` — `requests.post`, 10 s timeout
- `load_cache` / `save_cache` — file I/O (small, but on every date)

Wrap the network calls in `asyncio.to_thread(...)` (or use an async HTTP client). Worst case today, a slow Chronogolf API response freezes all Playwright scrapes for 30 s.

## 3. One browser, one context per course

Each course launches its own headless Chromium (`check_course` → `launch_browser`), so a full run holds up to 5 browser processes on a 2-core runner. Browser *contexts* already provide the per-course cookie/UA isolation the design wants — a single `browser` with `new_context()` per course keeps the "returning visitor" behavior at a fraction of the memory/CPU.

## 4. Fix the pip / Playwright cache keys

```yaml
key: ${{ runner.os }}-pip-playwright-requests-astral
key: ${{ runner.os }}-playwright-chromium
```

Both keys are static. `actions/cache` only saves on a cache **miss**, so after the first save these entries are frozen forever: the moment any dependency or the Playwright browser build changes, every run re-downloads (~150 MB Chromium) on top of restoring a stale cache, indefinitely. Key them on content instead:

```yaml
key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
key: ${{ runner.os }}-playwright-${{ hashFiles('requirements.txt') }}
```

(Best combined with pinning `requirements.txt` — see the bugs document.)

## 5. Prune stale dates from the per-course cache files

`save_cache` merges the new date key into the existing JSON and never deletes anything, so `cache_<course>.json` accumulates every date ever scraped. The files are re-uploaded to the Actions cache on every run (a fresh `run_id`-keyed entry each time), so the dead keys ride along forever. Drop keys older than today during `save_cache` — also makes the cache files human-readable when debugging.

## 6. Don't run pytest on generated-file commits

`tests.yml` triggers on every push to `main`/`production`, including syncs that only touch `index.html` / `data.json` / `version.json` — that's a pointless pytest run every ~10 minutes' worth of Actions minutes whenever generated commits arrive via PAT push. Add:

```yaml
push:
  branches: [main, production]
  paths-ignore: [index.html, data.json, version.json]
```

PR-triggered runs are unaffected. (While in there: drop the nonexistent `dev` branch from the list.)

## 7. Reuse the sunset computation in `generate_html`

Per course per day, `generate_html` computes `sun(MIAMI.observer, ...)` directly for `sunset_dt` *and* calls `get_sunset_cutoff` (which computes sunset again, behind its `lru_cache`). Derive the cutoff from `sunset_dt` (`sunset_dt - timedelta(hours=4, minutes=10)`) or cache the sunset itself. Trivial CPU, but it removes a duplicated-logic trap: the two call sites can drift (one already filters, the other classifies twilight with a different threshold).

## 8. Tidy the notification layer

`notify(subject, body, push_msg)` ignores `body` entirely and `send_email` is now dead code (nothing calls it). Either delete `send_email` + the email env plumbing, or restore the call behind an env flag. Today it's ~30 lines of code and three workflow secrets (`EMAIL_*`) that do nothing — and the docs still describe re-enabling a commented-out call that no longer exists.

## 9. Chronogolf notification links

`check_course` deliberately omits the booking URL from Pushover messages for `chronogolf` courses (`if course["type"] != "chronogolf"`), but `chronogolf_book_url(course, d)` already builds a correct per-date deep link (including the Miami Shores widget URL). Including it would make the push notification one-tap bookable — which is the whole point of the alert.

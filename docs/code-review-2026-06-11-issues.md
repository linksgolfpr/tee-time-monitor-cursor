# Code review (2026-06-11): bugs and correctness issues

> Prepared as a GitHub Issue, but Issues are disabled on this repository.
> Once enabled, this file can be filed verbatim. A companion document covers
> optimization opportunities.

Full review of the current branch (`tee_time_monitor.py`, workflows, docs, generated output). Items ordered roughly by impact.

## 1. Miami Lakes "Book" link embeds a raw Python datetime

`generate_html()` builds the CPS booking URL from `get_sunset_cutoff()`, which returns a `datetime`:

```python
t_max_book = get_sunset_cutoff(d, course["tee_time_max"])
book_url = f"{course['url']}?TeeOffTimeMin={course['tee_time_min']}&TeeOffTimeMax={t_max_book}"
```

The published `index.html` currently contains:

```
...search-teetime?TeeOffTimeMin=6&TeeOffTimeMax=2026-06-12 16:02:24.866244-04:00
```

That's an unencoded space and a value the site can't interpret as an hour. Fix: pass an integer hour (e.g. `t_max_book.hour` when it's a datetime, falling back to `course["tee_time_max"]`), or drop the param entirely.

## 2. `main()` silently swallows whole-course failures

```python
results = await asyncio.gather(*[check_course(...)], return_exceptions=True)
all_detected = [label for r in results if isinstance(r, list) for label in r]
```

If `check_course` raises outside the per-date try/except (e.g. `launch_browser` fails, or `browser.close()` errors), the exception object lands in `results` and is filtered out without ever being logged. A course could fail on every run for weeks with no trace in the Actions logs while the page silently serves stale cache data. Fix: iterate `results` and `logger.error(...)` any non-list entries (course name + traceback).

## 3. Monitor job has no `timeout-minutes`, and the concurrency group makes hangs expensive

`tee-time-monitor.yml` uses `concurrency: group: tee-time-monitor / cancel-in-progress: false` with a 10-minute external cron. GitHub keeps **at most one pending run per group**: while a run is executing, a newer dispatch *replaces* the queued one. That's fine in steady state, but a hung run (default job timeout is 6 hours) blocks the group, and every subsequent cron trigger is silently dropped — up to 6 hours of no scrapes. Add e.g. `timeout-minutes: 20` to the job. (Side note: the comment above the concurrency block says a waiting trigger means "no scrape is silently dropped" — pending-slot replacement means drops can happen; worth rewording.)

## 4. Miami Lakes scrape window can clip valid summer afternoon slots

`scrape_cpsgolf` requests `?TeeOffTimeMin=6&TeeOffTimeMax=15` (from course config), but the effective filter applied later is the sunset cutoff — currently **16:02 ET** in June (sunset − 4h10m). Slots between 15:00 and the cutoff may never be returned by the site because the scrape URL pre-filters them. Winter is unaffected (cutoff ≈ 13:30 < 15). Either raise the URL bound (e.g. 17) and let the sunset filter do the trimming — per the docs, `tee_time_max` is not supposed to be the final filter — or document the 3 PM bound as intentional.

## 5. `requirements.txt` is not pinned (docs say it is)

```
playwright
requests
astral
jinja2
```

CLAUDE.md/AGENTS.md state "Dependencies are pinned in `requirements.txt`". Unpinned `playwright` is the risky one: a major bump silently changes the bundled Chromium and API behavior between runs, and the static Actions cache keys make that worse (see optimization document). Pin all four (and dedupe `jinja2`, which appears in both requirements files).

## 6. Pushover failures are partially silent

`send_pushover` never checks the HTTP response — a revoked/typo'd token returns 4xx and is ignored. Add `resp.raise_for_status()` inside the existing try/except so it at least logs.

## 7. `manifest.json` is dead weight

The repo ships a PWA manifest, but the generated page has no `<link rel="manifest">` (or theme-color/apple-touch-icon), so it's never used. Either add the link tag in `HTML_TEMPLATE` or delete the file.

## 8. Documentation drift (CLAUDE.md / AGENTS.md / README.md)

These docs describe a previous incarnation of the project and will actively mislead future agents/contributors:

- `get_upcoming_weekend_dates()` no longer exists — it's `get_monitor_dates()` (driven by `DEFAULT_SCRAPE_WEEKDAYS`/`EXTRA_SCRAPE_WEEKDAYS`).
- "the `send_email` call is commented out — re-enable by uncommenting in `notify()`" — there is no commented call anymore; `notify()` only calls Pushover.
- "`<meta refresh content="300">` reloads the page every 5 min" — the meta refresh is gone; the page now polls `version.json` every 30 s and reloads when `ts` changes.
- `DEBUG_SCRAPE=1` is documented as a run mode but the variable is read nowhere in the code.
- "sunset − 4 hours" — code uses sunset − 4h **10m** (`get_sunset_cutoff`), and `_slot_time_class` uses a 4h30m twilight threshold.
- The `dev` branch and `tee-time-monitor-dev.yml` no longer exist (`tests.yml` still lists `dev` in its push branches; harmless but stale).
- The `archive/` directory described in both docs no longer exists.
- Repo URLs still point at `amapr24/tee-time-monitor`.
- `data.json`, `version.json`, `manifest.json` and the Miami Shores club-widget API path (no browser, plain `requests`) aren't mentioned in the architecture docs, and "State lives in two places" is now three.
- README says checks run "every 15 minutes"; cron is every 10 (the page itself says "10–15").
- Workflow step name "Commit and push index.html and version.json" also commits `data.json`.

None of these are runtime bugs, but for an agent-maintained repo the docs are part of the control loop — stale instructions cause wrong edits (the June 10 pull/commit ordering incident shows how cheap a wrong assumption is).

## Verification notes

- 32/32 tests pass on this branch.
- `main` and `production` are identical except generated output files.
- The June 10–11 incident: moving `git pull --rebase` before the commit (`c7508f1`) broke every run because the scraper leaves the tree dirty and git refuses to rebase with unstaged changes — automated commits stopped at 23:19 UTC and resumed only after `1c95fee` restored commit-before-pull. The current workflow order (add → commit → pull --rebase → push, serialized by the concurrency group) is correct.

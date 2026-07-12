# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` and updated the one call site in `routes/watchlist/watchlist.py` (`add_film`). This matches the `verb_to_noun` convention documented in `CONTRIBUTING.md` and already used by `add_to_collection()` / `remove_from_collection()` / `get_collection()` in `services/collection_service.py`.
**How I verified:** Grepped the whole repo for `save_to_watchlist` to confirm no call sites were missed, then ran `pytest tests/` to confirm the app still imports and runs cleanly.

## Comment 2 — Deduplication
**What I did:** Added an `AlreadyInWatchlistError` exception and a duplicate check in `add_to_watchlist()`, mirroring the exact pattern `add_to_collection()` uses with `AlreadyInCollectionError` in `services/collection_service.py`: query for an existing `(user_id, film_id)` pair before inserting, and raise instead of silently creating a second row. I also updated the `/watchlist/<user_id>/add` route to catch `AlreadyInWatchlistError` (409), matching the status code `routes/collection.py` uses for the same case.
**How I verified:** Added `test_add_to_watchlist_duplicate_raises` in `tests/test_watchlist.py` (see Comment 3) which adds a film twice and asserts the second call raises and that only one row exists afterward. Ran the full suite locally with `pytest tests/`.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py`, mirroring the fixtures and structure of `tests/test_collection.py`. Added `test_add_to_watchlist_nonexistent_film_raises`, which asserts `FilmNotFoundError` is raised for a `film_id` that doesn't exist, using the same fake-UUID pattern as `test_add_to_collection_nonexistent_film_raises`. I also added the happy-path and duplicate tests (`test_add_to_watchlist_creates_entry`, `test_add_to_watchlist_duplicate_raises`) in the same file, since `CONTRIBUTING.md` requires all three (happy path, conflict, nonexistent ID) for a new service function, and the duplicate test is what verifies Comment 2's fix.
**How I verified:** `pytest tests/test_watchlist.py -v` — all 3 new tests pass. Then `pytest tests/` — all 7 tests pass (4 pre-existing collection tests + 3 new watchlist tests).

## Comment 4 — Default visibility
**My position:** I changed the default from `public=True` to `public=False` — watchlists are private unless a user explicitly opts in (see the stretch feature below, which adds a `public` parameter to `add_to_watchlist()` / the `/watchlist/<user_id>/add` endpoint so a caller can set visibility explicitly at creation time instead of only getting the default).

**Reasoning:** CineLog already draws a real distinction between two kinds of film data: a `CollectionEntry` (a film the user has *already watched and rated*) and a `WatchlistEntry` (a film the user *intends* to watch). Notably, `CollectionEntry` has no `public`/`private` field at all — it's implicitly always visible, which fits the README's framing of CineLog as a "community film tracking app" where users "log films they've watched, rate them, and build collections." That's an intentional, existing product decision: what you've *done* is the social, shareable artifact.

A watchlist is a different kind of data — it's a statement of intent, not accomplishment, and it's easy to construct realistic cases where a user wouldn't want it shown by default: they might be tracking a film festival's catalog for personal use, saving something a friend recommended (spoiler-adjacent info about what they haven't seen yet), or building a private "to-watch" backlog they don't want to broadcast as if it were a curated public list. Because the watchlist model is new — this PR introduces the `public` column — there's no existing "public by default" precedent within the watchlist feature itself to preserve; the `True` default in the original code was just SQLAlchemy's `default=True` inherited from copying the `CollectionEntry`-style column, not a deliberate choice.

I want to be honest about the limits of this argument: I don't have user research or product data showing CineLog users actually want watchlist privacy — the "intent vs. accomplishment" distinction is a reasonable story, but it's a story, and someone could just as easily read the README's "community" framing as implying *all* user activity, watchlists included, should default to visible. Absent real evidence either way, I'm deliberately choosing the more conservative default: the cost of an unwanted default (a user's "want to watch" list being visible when they didn't think about it) is a privacy leak that can't be undone once seen, while the cost of the opposite mistake (a watchlist that's private when the user would have been fine sharing it) is just delayed visibility that a future opt-in UI can recover. Asymmetric, hard-to-undo downside is why I picked private, not because I can prove it's what users want.

**Why a code change and not just documentation:** the reviewer's comment technically only asked for a documented rationale before approving — not necessarily a different default. I considered writing up a justification for keeping `public=True` and leaving it at that, but that felt like retroactively rationalizing a value nobody actually chose (the original `default=True` was inherited from copying the column, not a deliberate decision — see above). Documenting an accident as if it were intentional isn't the same as being intentional, so I changed the default to the value I'd actually defend, and I'm treating this note as that defense.

**Tradeoff acknowledged:** The real cost here is discoverability: a "community" app benefits from social features like seeing what others want to watch (it drives the kind of engagement collections already provide), and a private-by-default watchlist means many watchlists will likely stay private simply because most users never revisit a setting they didn't choose at creation time — that's a genuine cost to the feature's social value, not a hypothetical one, and I'm accepting it rather than explaining it away. I mitigated it partially with the stretch feature — an explicit `public` parameter on `add_to_watchlist()` — so callers (e.g., a future "share my watchlist" UI flow) can set visibility intentionally per-entry instead of only inheriting a buried default. If product data later shows private-by-default is suppressing a feature users actually wanted, that's a signal to revisit this decision with real evidence, which is exactly the kind of decision this API-layer default shouldn't be presumed to make unilaterally forever.

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:** Ran `git fetch origin` then `git rebase origin/main` on `feature/watchlist`. Two conflicts came up:
1. `.gitignore` — an "add/add" conflict, since `main` had already merged its own `.gitignore` (from a separate `chore/add-gitignore` PR) while my branch added its own. The two files were nearly identical; main's version additionally ignored `.pytest_cache/`.
2. `models.py` — a real content conflict on the `refactor: migrate film IDs from integer to UUID` commit. `main` changed `Film.id` from `db.Column(db.Integer, ...)` to `db.Column(db.String(36), default=generate_uuid)` and updated `CollectionEntry.film_id` to match, but it has no knowledge of `WatchlistEntry` (that model only exists on `feature/watchlist`), so git couldn't auto-merge the two additions to the end of the file.

**How I resolved it:** For `.gitignore`, I kept the superset of both lists (added `.pytest_cache/` to mine) — after that my `.gitignore` commit became empty relative to main's, and git auto-dropped it during the rebase. For `models.py`, I kept the `WatchlistEntry` class from my branch and changed `film_id` from `db.Column(db.Integer, db.ForeignKey("film.id"), ...)` to `db.Column(db.String(36), db.ForeignKey("film.id"), ...)`, matching the same type main used for `CollectionEntry.film_id`. I also went through `services/watchlist_service.py` and `routes/watchlist/watchlist.py` and updated the docstrings/comments that still described `film_id` as an integer, and the endpoint's example request body, so nothing in the code contradicts the UUID schema.
**How I verified no conflict remains:** `git status` showed a clean rebase (`Successfully rebased and updated refs/heads/feature/watchlist`), `git log --graph` showed a fully linear history with no merge commits, and `pytest tests/` passed with the post-rebase UUID schema (the fake-nonexistent-film-id test already used a UUID-shaped string, so it didn't need changes).

## Stretch — remove_from_watchlist()
**What I did:**
**How I verified:**

## Stretch — Additional test
**What I chose:**
**Why:**

## Stretch — Visibility toggle
**What I did:**
**How I verified:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->

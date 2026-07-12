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
**What I did:**
**How I verified:**

## Comment 4 — Default visibility
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

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

# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end -->

## Comment 1 — Rename
**What I did:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`. Searched the codebase with `grep -rn "save_to_watchlist" . --include="*.py"` and found one call site in `routes/watchlist/watchlist.py`, in both the import statement and the function call inside `add_film()`. Updated both.

**How I verified:**
Re-ran the grep after the change and confirmed zero remaining references to `save_to_watchlist`. Ran the full test suite (`pytest tests/ -v`) to confirm nothing broke.

## Comment 2 — Deduplication
**What I did:**
Added a duplicate check to `add_to_watchlist()` that queries `WatchlistEntry` for an existing row matching the same `user_id` and `film_id`. If one exists, raises a new `AlreadyOnWatchlistError` exception instead of creating a second entry.

**How I verified:**
Modeled this directly on `add_to_collection()` in `services/collection_service.py`, which uses the identical pattern (query first, raise `AlreadyInCollectionError` on match) before creating the entry. Wrote `test_add_to_watchlist_duplicate_raises` to confirm adding the same film twice raises the error and that only one entry ends up in the database.

## Comment 3 — Missing test
**What I did:**
Created `tests/test_watchlist.py` with `test_add_to_watchlist_nonexistent_film_raises`, which attempts to add a film_id that doesn't exist in the database and asserts it raises `FilmNotFoundError`.

**How I verified:**
Modeled this on `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py`, using the same fixture structure (`app`, `sample_user`) and a fake UUID as the nonexistent film_id. Also added two supporting tests (happy path and duplicate handling) following the same fixture pattern used across `test_collection.py`. All tests pass.

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

## PR Description
<!-- Written at the end -->
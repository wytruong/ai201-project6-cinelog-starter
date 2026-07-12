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
I'm changing the default for `public` on `WatchlistEntry` from `True` to `False`.

**Reasoning:**
I searched the codebase for any social or sharing functionality (`grep -rn "friend"`, `"follow"`, `"social"` across the app) and found nothing: no follows, no friend feeds, no way for one user to view another user's watchlist or collection. The only endpoints that exist are `GET /watchlist/<user_id>` and `GET /collection/<user_id>`, both scoped to a single user's own data. Given that, defaulting `public` to `True` exposes data with no corresponding feature to make use of that exposure. There's currently no way for "public" to actually mean anything to another user. Defaulting to `False` is the more conservative, safer choice: it doesn't silently opt users into visibility they didn't ask for, and it follows the general privacy-by-default principle of opt-in over opt-out.

**Tradeoff acknowledged:**
The tradeoff is that if CineLog later adds social/discovery features (following users, a friend activity feed, etc.), a `False` default means every existing watchlist entry stays private unless the user explicitly changes it — which could slow adoption of that future feature, since most users won't go back and toggle old entries. A `True` default would have made that future feature "just work" without a data migration. But I think that's a reasonable tradeoff: it's easier and safer to prompt users to opt in to a new sharing feature when it launches than to have silently exposed their data the whole time before that feature existed.

## Comment 5 — Sort order
**My position:**
I agree with the maintainer — I'm changing `get_watchlist()` to sort by `date_added` descending (most recently added first), replacing the current alphabetical-by-title sort.

**Reasoning:**
A watchlist is fundamentally about current intent — it's the list of "what am I thinking about watching next," not a reference list I'm searching through. When I add something new to my watchlist, it's usually because I just heard about it or I'm currently excited about it, and that's exactly the thing I want to see first the next time I open the app. Alphabetical order buries that recency signal — a film I added yesterday could easily land at the bottom of the list if its title starts with a letter late in the alphabet, even though it's the most relevant thing to me right now.

**Engagement with reviewer's point:**
I think the maintainer's instinct here is correct, and it's also worth noting this brings the watchlist in line with how `get_collection()` already sorts (`date_added.desc()`), which makes the two features behave consistently from a user's perspective — if collection already surfaces "recently watched" first, watchlist surfacing "recently added" first is the parallel behavior a user would expect. The one case where alphabetical genuinely helps is if someone has a long watchlist and is specifically hunting for a title they remember adding a while back ("did I already add Dune?") — but I think that's a search/filter problem, not a sort-order problem, and solving it by defaulting to alphabetical would make the more common case (checking in on recent additions) worse for everyone to fix a less common one.


## Comment 6 — Rebase
**What conflicted:**
Rebasing onto `origin/main` surfaced a conflict in `models.py`. While `feature/watchlist` was open, `main` had a refactor that migrated `Film.id` from an integer primary key to a UUID string (`db.String(36)`), and `CollectionEntry.film_id` was updated to match. My branch's `WatchlistEntry` model was still defined with `film_id = db.Column(db.Integer, ...)` from before that refactor, so the merge couldn't automatically reconcile the two versions of `models.py`. There was also a smaller conflict in `.gitignore`, since both branches added one independently.

**How I resolved it:**
For `.gitignore`, I kept the union of both versions' entries. For `models.py`, I kept the `WatchlistEntry` model (which only exists on my branch) but updated `film_id` from `db.Column(db.Integer, db.ForeignKey("film.id"), ...)` to `db.Column(db.String(36), db.ForeignKey("film.id"), ...)` to match the now-UUID `Film.id`, following the same pattern already applied to `CollectionEntry.film_id` in the refactor.

**How I verified no conflict remains:**
After resolving, `git rebase --continue` completed with "Successfully rebased and updated refs/heads/feature/watchlist" and no further conflicts. I ran the full test suite (`pytest tests/ -v`) and all 7 tests passed, confirming the watchlist service code — which references `film_id` in queries but never hardcodes its type — worked correctly against the new UUID column without needing any changes itself. I also confirmed with `git log --oneline` that the branch history is linear with no merge commits.

## PR Description
<!-- Written at the end -->
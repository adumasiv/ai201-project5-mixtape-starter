# Project 5: Mixtape Bug Hunt — Submission

## Part 1: Codebase Map

Mixtape is a Flask + SQLAlchemy social music app. The code is organized in three
layers with a single direction of dependency:

```
HTTP request → routes/ (thin adapters) → services/ (business logic) → models.py (data) → SQLite
```

### Main files and what each one does

**`app.py`** — The Flask application factory (`create_app`). It instantiates the
shared `db = SQLAlchemy()` object, reads config (`DATABASE_URL`, defaulting to
`sqlite:///mixtape.db`), registers the four route blueprints under URL prefixes
(`/songs`, `/playlists`, `/users`, `/feed`), and calls `db.create_all()`. Nothing
else lives here — no logic, no models.

**`models.py`** — Defines the data model: **6 SQLAlchemy models** plus **3
association tables**.

Models:
- **`User`** — has `username`, `email`, and the streak fields `listening_streak`
  (int) and `last_listened_at` (datetime). Notably it has a *self-referential*
  many-to-many `friends` relationship: a user's friends are other users, joined
  through the `friendships` table.
- **`Song`** — `title`, `artist`, `album`, `genre`, plus `shared_by` (FK to the
  `User` who shared it) and `share_note`. Many-to-many with `Tag`.
- **`Tag`** — just a unique `name`. Attached to songs through `song_tags`.
- **`ListeningEvent`** — a timestamped `(user_id, song_id, listened_at)` record.
  This is the raw event stream that powers both listening streaks and the feed.
- **`Rating`** — `(user_id, song_id, score 1–5)` with a **unique constraint on
  `(user_id, song_id)`**, so a user re-rating a song *updates* their existing
  rating rather than creating a duplicate.
- **`Notification`** — `(user_id, notification_type, body, read)` for the
  recipient user.

The three association tables are the interesting part of the schema:
- **`friendships`** — `(user_id, friend_id)`. Symmetric friendships are stored
  as *two rows* (both directions), which the seed script does explicitly.
- **`song_tags`** — plain `(song_id, tag_id)` join.
- **`playlist_entries`** — the richest join table. Beyond `(playlist_id,
  song_id)` it carries **`position`** (explicit ordering — a song's place in a
  playlist is a stored integer, not insertion order), **`added_by`**, and
  **`added_at`**. Playlist song order is therefore driven by `position`, not by
  row order.

**`routes/`** — Four thin blueprint modules. Each route parses the HTTP request,
calls exactly one service function, and translates the result (or a raised
`ValueError`) into JSON with the right status code. No business logic here.
- `songs.py` — `/search`, `GET /<id>`, `POST /<id>/rate`, `POST /<id>/listen`
- `playlists.py` — create, `GET /<id>`, `GET /<id>/songs`, `POST /<id>/songs`
- `users.py` — user detail, `/streak`, `/notifications`, mark-notification-read
- `feed.py` — `/<user_id>/listening-now`, `/<user_id>/activity`

**`services/`** — Where all business logic lives (and, per the README, where all
five bugs live).
- `streak_service.py` — records listening events and maintains consecutive-day
  streaks (`record_listening_event`, `update_listening_streak`, `get_streak`).
- `feed_service.py` — "Friends Listening Now" (recent, filtered by a time
  threshold) and the general activity feed (most-recent-N, unfiltered).
- `search_service.py` — song search by title/artist, with tags joined in.
- `notification_service.py` — ratings, playlist-adds, and the notifications
  those actions generate (`rate_song`, `add_to_playlist`, `create_notification`,
  `get_notifications`, `mark_as_read`).
- `playlist_service.py` — playlist creation and ordered song retrieval.

**`seed_data.py`** — Drops and recreates the DB, then populates 5 users (with
friendships), 25 songs (deliberately including songs with 0, 1, and 3+ tags), 3
playlists, a mix of recent and old listening events, streak values, and one
sample notification. The data is intentionally shaped to expose the bugs.

**`tests/`** — pytest suites for streaks, search, and playlists.

---

### Data flow: adding a song to a playlist triggers a notification

This is the app's cleanest end-to-end "action produces a notification" flow.

1. **Route** — `POST /playlists/<playlist_id>/songs` hits `add_song()` in
   `routes/playlists.py`. It pulls `song_id` and `added_by` out of the JSON body,
   returns 400 if either is missing, then calls the service.
2. **Service** — `notification_service.add_to_playlist(playlist_id, song_id,
   added_by_user_id)`:
   - Loads the `Song`, the adding `User`, and the `Playlist`, raising `ValueError`
     (→ 400 at the route) if any is missing.
   - Appends the song to `playlist.songs` (unless it's already there) and commits.
   - **Then generates a notification** — but only if the adder is *not* the
     song's original sharer: it calls `create_notification()` targeting
     `song.shared_by`, with type `"song_added_to_playlist"` and a human-readable
     body naming the adder, the song, and the playlist.
3. **`create_notification`** writes a `Notification` row for that recipient and
   commits.

So the notification recipient is the **song's original sharer** (`song.shared_by`),
not the playlist owner — the feature is "someone added *your* song to a playlist."

A parallel flow exists for rating: `POST /songs/<id>/rate` → `routes/songs.py:
rate()` → `notification_service.rate_song()`. Note that `rate_song` lives in
`notification_service` (not a separate rating service) precisely because rating a
song is *meant* to notify the song's sharer — the same pattern as playlist-adds.

---

### Patterns I noticed

- **Routes are pure adapters.** Every route does three things and only three:
  parse input, call one service function, format the response. All logic —
  validation, DB queries, notification triggering — lives in `services/`. This
  makes the services independently testable (which is exactly what `tests/`
  does — it imports services directly, not routes).

- **`ValueError` is the error-handling contract between layers.** Services raise
  `ValueError` for any "not found" / bad-input case; every route wraps its
  service call in `try/except ValueError` and maps it to a 400 or 404. There's no
  custom exception hierarchy — the plain built-in is the whole convention.

- **Ordering is explicit, not implicit.** Playlist order comes from the stored
  `position` column in `playlist_entries`, and feeds order by `listened_at`. The
  app never relies on database row order.

- **Notifications are a side effect owned by the acting service.** Rather than a
  central event bus, the service that performs an action (adding to a playlist,
  rating) is also responsible for calling `create_notification`. There's a
  guard so users don't get notified about their own actions
  (`if song.shared_by != added_by_user_id`).

- **Timestamps default to UTC-aware `datetime.now(timezone.utc)`** at the model
  level, and the streak logic defensively re-attaches `timezone.utc` to any naive
  datetime it reads back — a sign the code is aware SQLite can return naive
  datetimes.

---

## Part 2: Bug Fixes

### Bug #1 — "My listening streak keeps resetting" (`streak_service.py`)

**Trace.** Starting from the symptom, I followed the call chain named in the
README: `POST /songs/<id>/listen` → `routes/songs.py: listen()` → calls
`streak_service.record_listening_event(user_id, song_id)`, which creates the
`ListeningEvent` and then delegates the actual streak math to
`update_listening_streak(user, now)`. I read that function line by line:

```python
today = now.date()
...
last_date = last_listened.date()
days_since_last = (today - last_date).days

if days_since_last == 0:
    return                                          # same day, no change
elif days_since_last == 1 and today.weekday() != 6:  # <-- suspicious
    user.listening_streak += 1
else:
    user.listening_streak = 1
```

The `today.weekday() != 6` clause immediately stood out: the docstring says the
rule is "if the user listened yesterday: streak increments by 1" — full stop, no
day-of-week exception. `weekday() == 6` is Sunday, so this reads as "increment on
a consecutive day, *unless that day is a Sunday*, in which case reset instead."
There's no listed product reason for a Sunday exception, which made it the prime
suspect for "keeps resetting."

**Verify.** Rather than trust the read, I called the function directly in a
Python shell with controlled inputs (a `FakeUser` with `last_listened_at` set to
a Saturday, then updating with a Sunday timestamp — genuinely one day apart):

```python
u.last_listened_at = datetime(2024, 6, 15, ...)  # Saturday
update_listening_streak(u, datetime(2024, 6, 16, ...))  # Sunday, weekday()==6
# streak went from 1 -> 1, not 1 -> 2
```

This confirmed the bug: a real consecutive-day listen (`days_since_last == 1`)
falls into the `else` branch and resets to 1 whenever the second day is a
Sunday. Since every ongoing streak eventually lands on a Sunday, this explains
why streaks "keep" resetting rather than resetting once.

**Root cause.** An erroneous extra condition (`today.weekday() != 6`) on the
increment branch causes Sunday to be treated like a skipped day instead of a
consecutive one.

**Fix.** Removed the weekday condition so the branch is purely
`days_since_last == 1` (`services/streak_service.py:73`) — the smallest change
that restores the documented rule. No other logic needed to change.

**Verification against regressions.** Ran `tests/test_streaks.py` (all 5 cases,
including the two boundary sides — same-day no-op and skip-a-day reset — plus
the new Sunday case) and confirmed manually with two additional REPL checks:
Sunday→Monday (consecutive, streak should still increment) and Friday→Sunday
(a skipped Saturday, streak should still reset). All passed. Confirmed
`update_listening_streak` has exactly one call site
(`record_listening_event`), so no other code depended on the old behavior.

**AI usage.** Used the AI to explain the difference between Python's
`weekday()` and `isoweekday()` to double check `weekday() == 6` really does mean
Sunday (it does — Monday is 0). Did not ask the AI to locate the bug; the
suspicious line was found by reading the function against its own docstring.

Commit: `fix: correct Sunday boundary condition in streak reset logic`

---

### Bug #3 — "The same song keeps showing up twice in search" (`search_service.py`)

**Trace.** Call chain: `GET /songs/search?q=...` → `routes/songs.py: search()` →
`search_service.search_songs(query)`. That function was short enough to read in
full immediately:

```python
results = (
    db.session.query(Song)
    .outerjoin(song_tags, Song.id == song_tags.c.song_id)
    .filter(db.or_(Song.title.ilike(...), Song.artist.ilike(...)))
    .all()
)
```

The `outerjoin(song_tags, ...)` stood out immediately: nothing in the `filter()`
or the return value touches a `song_tags` column — tags are attached later,
per-song, via the `Song.tags` relationship inside `to_dict()`. A join that
doesn't constrain or select anything is a classic setup for row fan-out: for a
song with 3 tags, the join produces 3 physical rows for that one song.

**Verify.** Rather than trust the read, I checked what the join actually
returns at the SQL level, bypassing the ORM with a raw connection execute
against a song with 3 tags (`"Crown Heights Anthem"` from the seed data):

```
SELECT song.id FROM song LEFT OUTER JOIN song_tags ON ... WHERE title LIKE '%Crown%'
-> 3 rows, same song.id repeated 3 times
```

This confirmed the fan-out exists at the database level. But calling
`search_songs("Crown")` directly returned only **1** result, and the existing
`tests/test_search.py::test_search_no_duplicates_multi_tag_song` already
passed. I didn't stop there — a passing test with a demonstrably fanned-out
join underneath meant something in between was hiding it. I compared the exact
same join expressed two ways: the legacy `db.session.query(Song)...all()` API
used in the file (1 result) vs. the modern `select(Song)...` /
`db.session.execute()` API (3 results, i.e. the raw duplicates). The legacy
Query API auto-deduplicates full-entity results in this SQLAlchemy version
(2.0.51) — an incidental behavior of *how* the query is issued, not a property
of the query itself. The underlying bug — an unfiltered, unselected join
causing row fan-out — was real; it just happened not to reach the caller in
this exact code path.

**Root cause.** `search_songs()` joins to `song_tags` for no functional reason,
causing SQL-level row duplication per tag. This is only invisible in the
current test suite because of an incidental quirk of the ORM API style used,
making it a latent bug one refactor away from resurfacing (e.g. switching to
`select()`, adding a second join, or a future SQLAlchemy version change).

**Fix.** Removed the `.outerjoin(song_tags, ...)` call entirely — it wasn't
needed for filtering or selection in the first place (`search_service.py`).
Also removed the now-unused `Tag` and `song_tags` imports.

**Verification against regressions.** Confirmed the fan-out is genuinely gone
(not just masked) by re-running the same before/after comparison: both the
legacy `query()` style and the modern `select()`/`execute()` style now return
exactly 1 result for the 3-tag song. Confirmed the `tags` list in the response
still populates correctly (it comes from `Song.tags`, unaffected by removing
the join). Ran the full suite: all `tests/test_search.py` cases pass
(matching songs, no-dup for 0/1/3-tag songs, empty-query-match case).

**AI usage.** Gave the AI the `search_songs` function and the two competing
query results (1 vs. 3) and asked "what's the structural difference between
these two query styles that would explain this?" to confirm my suspicion about
legacy-Query auto-uniquing before relying on it in the writeup. Diagnosis and
fix were done by reading and direct experimentation, not by asking the AI to
find the bug.

Commit: `fix: remove unnecessary song_tags join causing search result fan-out`

---

### Bug #5 — "The last song in a playlist never shows up" (`playlist_service.py`)

**Trace.** Call chain: `GET /playlists/<id>/songs` → `routes/playlists.py:
get_songs()` → `playlist_service.get_playlist_songs()`. Reading the function
top to bottom, the query itself looked correct — it joins `playlist_entries`,
filters by `playlist_id`, and orders by `position` ascending, exactly matching
the docstring ("Songs are returned in the order they were added"). The very
last line, though, contradicted the function's own `Note:` ("This function
returns all songs in the playlist"):

```python
return [song.to_dict() for song in songs[:-1]]
```

`songs[:-1]` drops the last element of whatever list precedes it — a stray
slice, not something the query or the docstring calls for.

**Verify.** Before fixing, I checked the boundary case a title like "the last
song never shows up" implies could be worse than described: a playlist with
exactly one song. `[<one song>][:-1]` evaluates to `[]`, so I confirmed this
directly against a real single-song playlist:

```python
get_playlist_songs(playlist_with_one_song)  # returned [] , not [that song]
```

So the bug isn't just "last song missing" — a single-song playlist appears
**entirely empty**, which is a more severe version of the reported symptom.

**Root cause.** A stray `[:-1]` slice on the final return line drops the last
song from every playlist's song list, unconditionally.

**Fix.** Removed the slice: `songs[:-1]` → `songs` (`playlist_service.py:66`).

**Verification against regressions.** Ran `tests/test_playlists.py` (all 3
cases, including the pre-existing empty-playlist case) — all pass. Then
checked both sides of the boundary manually: a single-song playlist now
correctly returns 1 song, and an empty playlist still correctly returns `[]`
(the slice was never the reason empty playlists worked — `[][:-1] == []` — so
removing it doesn't disturb that case). Checked other code that touches
playlist songs: `notification_service.add_to_playlist()` uses the
`playlist.songs` ORM relationship directly (not `get_playlist_songs`), so it
was never affected by this bug and needed no changes. Full suite: 13/13 pass.

**AI usage.** None needed for diagnosis — the docstring directly named the
correct behavior, and the contradicting line was the last line of the
function. Used a plain REPL call (no AI) to verify the single-song boundary
before writing the fix.

Commit: `fix: stop dropping the last song from playlist listings`
```

# Mixtape — Submission

## Codebase Map

### Main files and what they do

**`app.py`** — Flask application factory. `create_app(config)` builds the Flask
app, points SQLAlchemy at `sqlite:///mixtape.db` (or `DATABASE_URL`), registers
the four blueprints (`songs`, `playlists`, `users`, `feed`) under their URL
prefixes, and calls `db.create_all()` on startup. This is also where the
shared `db = SQLAlchemy()` instance lives — every other module imports `db`
from here.

**`models.py`** — All SQLAlchemy models and three association tables:
- `User` — has `listening_streak` / `last_listened_at` columns plus
  relationships to songs shared, ratings, listening events, notifications,
  playlists, and friends.
- `Song`, `Tag`, `ListeningEvent`, `Rating` (unique per user+song),
  `Playlist`, `Notification`.
- `friendships` — many-to-many, directional rows (`user_id → friend_id`).
  Symmetry (A is friends with B *and* B is friends with A) is not enforced
  by the schema — it only exists because `seed_data.py` inserts both
  directions by hand.
- `song_tags` — plain many-to-many between songs and tags.
- `playlist_entries` — many-to-many between playlists and songs, but with
  extra columns: `position`, `added_by`, `added_at`. This is what makes
  playlist songs ordered and audit-able, instead of just relying on
  insertion order.
- Every model has a `to_dict()` method, and routes call these directly
  when building JSON responses — there's no separate serialization layer.

**`routes/songs.py`** — `GET /songs/search`, `GET /songs/<id>`,
`POST /songs/<id>/rate`, `POST /songs/<id>/listen`. Each handler parses the
request, calls exactly one service function, and maps `ValueError` to an
error JSON response.

**`routes/playlists.py`** — `POST /playlists/`, `GET /playlists/<id>`,
`GET /playlists/<id>/songs`, `POST /playlists/<id>/songs`. Same thin-handler
pattern as above.

**`routes/users.py`** — `GET /users/<id>`, `GET /users/<id>/streak`,
`GET /users/<id>/notifications`, `POST /users/notifications/<id>/read`. Note
`get_user` is the one handler in the whole app that queries `User` directly
instead of going through a service.

**`routes/feed.py`** — `GET /feed/<id>/listening-now`,
`GET /feed/<id>/activity`.

**`services/streak_service.py`** — `record_listening_event()` creates a
`ListeningEvent` and calls `update_listening_streak()` to adjust the user's
streak counter; `get_streak()` just reads `user.listening_streak`.

**`services/feed_service.py`** — `get_friends_listening_now()` (last 24h,
deduped to one entry per friend) and `get_activity_feed()` (most recent N
events, no recency filter, no dedup). Both independently re-derive the
user's `friend_ids` from `user.friends`.

**`services/search_service.py`** — `search_songs()` (title/artist
`ILIKE` match) and `get_song()` (lookup by id).

**`services/notification_service.py`** — `create_notification()` (the one
place a `Notification` row actually gets written), `add_to_playlist()`
(mutates the playlist, then calls `create_notification`), `rate_song()`
(upserts a `Rating`), `get_notifications()`, `mark_as_read()`.

**`services/playlist_service.py`** — `create_playlist()`,
`get_playlist_songs()` (position-ordered), `get_playlist()` (metadata only),
`get_user_playlists()`.

**`seed_data.py`** — populates 5 users with bidirectional friendships, 25
songs (deliberately varying tag counts), 3 playlists, a mix of recent and
old `ListeningEvent`s, and one pre-existing "song added to playlist"
notification. Comments in this file call out which fixtures are meant to
exercise which of the five tracked issues.

**`tests/`** — `test_streaks.py`, `test_search.py`, `test_playlists.py`.
Each targets one service module and its associated issue.

### Data flow: adding a song to a playlist triggers a notification

This is the clearest end-to-end flow in the app, so I traced it fully:

```
POST /playlists/<playlist_id>/songs   { "song_id": ..., "added_by": ... }
  → routes/playlists.py: add_song(playlist_id)
      1. Parses song_id and added_by from the JSON body; 400s if either is missing.
      2. Calls services/notification_service.py: add_to_playlist(playlist_id, song_id, added_by)

         a. Looks up the Song, the adding User, and the Playlist —
            raises ValueError (→ 400 in the route) if any is missing.
         b. If the song isn't already in playlist.songs, appends it and
            commits. This is the actual "add" — it's a plain many-to-many
            append via the ORM relationship, not a raw insert into
            playlist_entries, so SQLAlchemy fills in the join row itself.
         c. If song.shared_by != added_by (i.e. you didn't add your own
            shared song), calls create_notification(user_id=song.shared_by,
            notification_type="song_added_to_playlist", body=...).
         d. create_notification() builds a Notification row, adds it,
            commits, and returns it (add_to_playlist doesn't use the
            return value — the route only cares that no exception was raised).
      3. Route returns {"message": "Song added to playlist"}, 201.
```

Later, the notified user retrieves it via a completely separate call path:
`GET /users/<id>/notifications` → `routes/users.py: notifications()` →
`notification_service.get_notifications(user_id, unread_only)`, which just
filters/order-bys the `Notification` table. There's no push mechanism —
notifications are only ever pulled on demand.

**Notable asymmetry:** `rate_song()` lives in the same file as
`create_notification()` and follows an identical existence-checking
pattern, but never calls `create_notification()` anywhere in its body —
rating a song does not notify the original sharer, even though adding that
same song to a playlist does. I'm flagging this as an observation from
reading the code, not as a fix yet.

### Patterns I noticed

- **Routes are thin, services own all business logic.** Every route handler
  does request parsing → one service call → response formatting. The only
  exception is `routes/users.py: get_user()`, which queries `User` directly.
- **Existence checks are done by convention, not a shared helper.** Nearly
  every service function starts with `db.session.get(Model, id)` followed by
  `if not x: raise ValueError(...)`, and every route catches `ValueError` and
  maps it to a 400/404. This is consistent but duplicated dozens of times
  rather than factored into a decorator or helper.
- **Local imports to dodge circular dependencies.** `notification_service.
  add_to_playlist()` imports `Playlist` and `playlist_service.
  get_playlist_songs` inside the function body rather than at module level
  (and doesn't actually use `get_playlist_songs`, interestingly) — this is
  the standard Flask/SQLAlchemy pattern for avoiding import cycles between
  `services/` modules and `models.py`.
- **String UUIDs everywhere.** Every model's primary key is a
  `db.String(36)` populated by `generate_uuid()`, rather than an
  auto-incrementing integer — consistent across all seven tables.
- **`to_dict()` as the only serialization layer.** Routes call
  `model.to_dict()` (or a service function that already returns dicts) and
  `jsonify()` it directly. There's no schema/marshalling library in play.
- **Association tables carry different amounts of metadata depending on
  need.** `friendships` and `song_tags` are bare join tables, but
  `playlist_entries` carries `position`, `added_by`, and `added_at` because
  playlist membership needs ordering and audit info that a plain M2M can't
  express.
- **Duplicated setup logic between near-identical functions.**
  `get_friends_listening_now()` and `get_activity_feed()` in
  `feed_service.py` each independently re-fetch the user and recompute
  `friend_ids` — the only difference is the recency filter and the dedup
  step.

## Root Cause Analyses

### Issue #1 — My listening streak keeps resetting

**How I reproduced it:** Before touching any code, I ran the existing suite
(`pytest tests/test_streaks.py -v`). `tests/test_streaks.py` already contains
`test_streak_increments_on_sunday`, which simulates exactly kenji's report —
listen on a Saturday, then listen again on the immediately following Sunday
— and asserts the streak goes from 1 to 2. It failed: the streak came back
as 1 instead of 2, which is the "consecutive days, but streak got thrown
away" symptom from the report reproduced with concrete dates
(`2024-06-15` → `2024-06-16`).

**How I found the root cause:** The read path
(`GET /users/<id>/streak` → `routes/users.py:streak()` →
`streak_service.get_streak()`) is a one-line getter that just returns
`user.listening_streak` — nothing to fix there, it's only reporting a value
that was already wrong. The value is actually computed on the write path:
`POST /songs/<id>/listen` → `routes/songs.py:listen()` →
`streak_service.record_listening_event()` → `update_listening_streak()`.
Reading `update_listening_streak()` line by line against its own docstring
("If the user listened yesterday: streak increments by 1") is what made me
confident I'd found the exact spot — the docstring describes a plain
one-day-gap check, but the code guarding the increment branch had an extra
clause not mentioned anywhere in the documented rules:
`elif days_since_last == 1 and today.weekday() != 6:`.

**The root cause:** Python's `datetime.weekday()` returns `0` for Monday
through `6` for Sunday. The increment branch required
`days_since_last == 1 and today.weekday() != 6` — i.e. "exactly one day
passed, AND today is not Sunday." Any time a user's consecutive-day listen
happened to land on a Sunday, `today.weekday() != 6` evaluated to `False`,
so the `elif` failed even though `days_since_last == 1` was true, and
execution fell through to the `else` branch (`user.listening_streak = 1`),
resetting the streak instead of incrementing it. This matches kenji's
report precisely: both resets he experienced happened on a Sunday, and
Monday's listen "bumped it to 2" because by then `days_since_last == 1`
again and `today.weekday() != 6` was true (Monday's `weekday()` is 0).

**My fix and side-effect check:** I removed the `and today.weekday() != 6`
clause, leaving `elif days_since_last == 1:` as the sole condition for
incrementing — a one-line change in `services/streak_service.py`, matching
the docstring's stated rules exactly (no day-of-week exception is
documented anywhere for this feature). After the fix, all 5 tests in
`test_streaks.py` pass, including `test_streak_increments_on_sunday`. I ran
the full test suite (`pytest tests/`) to check for side effects: the two
failures in `test_playlists.py` are pre-existing and unrelated (they belong
to Issue #5's `songs[:-1]` bug in `playlist_service.py`, a file this change
never touches — confirmed with `git diff --stat` showing only
`streak_service.py` modified). I also reviewed every other consumer of
`listening_streak` / `last_listened_at` / `ListeningEvent`
(`record_listening_event`, `get_streak`, the `/songs/<id>/listen` and
`/users/<id>/streak` routes, and `feed_service.py`/`seed_data.py`, which
read `ListeningEvent` and `last_listened_at` independently of this branch)
and confirmed none of them depend on the removed condition.

### Issue #3 (README "same song shows up twice in search") — investigated, not reproduced

Reported by simone: searching "Anthem" allegedly returned "Crown Heights
Anthem" (a song with 3 tags) three times. Before touching any code, I tried
to reproduce this against the real seeded database through the actual
`GET /songs/search?q=Anthem` endpoint (not just the unit tests) and it
returned the song exactly once. The existing test
`test_search_no_duplicates_multi_tag_song` in `tests/test_search.py`, which
encodes this exact scenario, also passes on the current code.

I dug into why: `search_service.search_songs()` does an
`outerjoin(song_tags, Song.id == song_tags.c.song_id)` without any
`.distinct()`, which — confirmed via raw SQL — genuinely fans out to one
row per matching tag (3 rows for a 3-tag song). But the function loads
results via `db.session.query(Song)...all()`, SQLAlchemy's *legacy* Query
API, which automatically de-duplicates full ORM-entity results by primary
key as a backward-compatibility behavior. That silently collapses the
3-row fan-out back into 1 `Song` object before `to_dict()` ever runs. (This
project pins `sqlalchemy>=2.0.0`; the equivalent 2.0-style `select()`
executed via `session.execute()` would *not* auto-dedupe and would need an
explicit `.unique()` call — but that's not the code path this function
uses.)

**Conclusion:** the join is fragile — it does unnecessary row fan-out at
the SQL level and would start producing real duplicates the moment this
function were rewritten to 2.0-style `select()`/`session.execute()` — but
it is not currently causing user-visible duplicates through the shipped
endpoint. Since I couldn't reproduce the reported symptom, I'm not counting
this as one of the 3+ required fixes and did not change any code here;
noting the investigation for completeness rather than writing a full RCA
for a fix that didn't happen.

### Issue #3 — Friends Listening Now shows people from yesterday

**How I reproduced it:** Before touching any code, I wrote a standalone
repro: created two friended users (nova, darius), gave darius a
`ListeningEvent` timestamped 10 hours in the past (mirroring "11pm last
night, checked at 9am"), and called
`feed_service.get_friends_listening_now(nova.id)` directly. It returned 1
entry — darius's stale event — matching the report exactly (a friend's
listen from the previous night still showing up hours later, well past
when "now" should mean).

**How I found the root cause:** `routes/feed.py:listening_now()` is a thin
passthrough with no logic of its own, so the actual filtering had to be in
`feed_service.get_friends_listening_now()`. Reading it top to bottom, the
recency filter is `ListeningEvent.listened_at >= cutoff` where
`cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD` and
`RECENT_THRESHOLD = timedelta(hours=24)` (module-level constant at the top
of the file). Once I saw the constant was a full day, the report made
sense immediately — an 11pm listen checked at 9am is only a ~10-hour gap,
well inside a 24-hour rolling window. I cross-checked this against
`seed_data.py`, which has a comment fixture explicitly built around a
different, much smaller boundary: events "within the past 30 minutes...
should appear in 'listening now'" and events starting at "2 hours" (part
of an "older, 1-14 days ago... should NOT appear" bucket). That fixture
design only makes sense if the intended threshold sits somewhere between
30 minutes and 2 hours — confirming `timedelta(hours=24)` itself, not the
comparison logic around it, was the defect.

**The root cause:** `RECENT_THRESHOLD` was set to `timedelta(hours=24)`, a
rolling 24-hour window, for a feature whose name and docstring both
describe near-real-time state ("Friends Listening Now" /
"listened to something recently"). A rolling 24-hour window doesn't
implement "now," "currently listening," or even "today" (a calendar-day
concept) — it just means "anything from the last full day," so an event
from last night stays visible until the exact same clock time the next
day, exactly as nova described ("hangs around... until the same time the
next day").

**My fix and side-effect check:** Changed `RECENT_THRESHOLD` from
`timedelta(hours=24)` to `timedelta(hours=1)` — a one-line change in
`services/feed_service.py`, landing inside the gap `seed_data.py`'s
fixtures were already designed around (30 min = show, 2+ hours = don't
show), so no existing fixture data had to change to accommodate it. I
reran the repro script after the fix: the 10-hour-old event is now
correctly excluded (0 entries). I then checked side effects two ways: (1)
ran the full test suite — the only failures are the same 2 pre-existing,
unrelated `test_playlists.py` failures from Issue #5; (2) reseeded the
real database and hit `GET /feed/<nova_id>/listening-now` through the
actual Flask endpoint — it correctly returned exactly the 3 friends from
the "within 30 minutes" fixture bucket (darius, simone, kenji) and
excluded everyone in the "2+ hours old" bucket. I also checked
`get_activity_feed()` in the same file, since it's the other consumer of
friend-listening data — it doesn't reference `RECENT_THRESHOLD` at all (by
design, per its own docstring, it's not recency-filtered), so it's
unaffected by this change.

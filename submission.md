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

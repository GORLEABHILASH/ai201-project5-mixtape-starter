# Project 5: Mixtape Bug Hunt — Submission

## AI Usage

I used AI as a guide and a pair of hands for reproduction, while keeping the diagnosis and fix decisions
my own. Concretely:

- **Codebase orientation.** I had the AI read `models.py` and every service and route file and give me a
  file-by-file summary of what each module is responsible for, plus a trace of one data flow (adding a
  song to a playlist, from the route down to the notification). This is the "File summary" and "Data flow
  trace" pattern from the project brief, and it's what my codebase map above is built on. I confirmed the
  summaries by reading the files myself rather than taking them on faith.

- **Building reproduction harnesses (not diagnoses).** For each bug I asked the AI to write small,
  isolated scripts that exercised a single service function with controlled inputs — for example, calling
  `update_listening_streak()` once per weekday against a synthetic "listened yesterday" user, or calling
  `rate_song()` in an in-memory DB and counting the sharer's notifications. The AI wrote the scripts, but
  I supplied the hypotheses about *where* the bug was and decided every fix. For Issue #1, I diagnosed
  the `today.weekday() != 6` clause myself; the AI's contribution was only confirming that Python's
  `datetime.weekday()` returns 6 for Sunday.

- **A place where the AI's first hypothesis was wrong and I had to course-correct.** For Issue #3
  (duplicate search results), the AI's initial explanation was that the `outerjoin` to `song_tags`
  multiplies rows for multi-tag songs and leaks duplicates. That sounded right, but when we actually ran
  a reproduction it did **not** duplicate — all five search tests passed. Rather than trust the
  explanation, I had it dig into *why*: it turned out `search_songs()` uses SQLAlchemy's legacy
  `db.session.query(Song)` API, which auto-deduplicates single-entity results by primary key, so the
  duplication is masked in this version of SQLAlchemy (2.0). Because the reported symptom couldn't be
  reproduced, I dropped Issue #3 and swapped in Issue #4 instead. This was the clearest example of the
  AI giving a plausible-but-unverified answer that only held up (or in this case, didn't) once I insisted
  on running the code.

- **Drafting documentation.** I used the AI to help draft these root-cause write-ups and the codebase map
  from my own analysis and our reproduction results, then reviewed and edited them so they reflect what I
  actually found and how I'd say it.

The overall workflow was: AI explains code I point it to and runs reproductions → I verify by reading and
by looking at the actual output → I decide the diagnosis and the fix. Asking the AI to *find* a bug
before reading the relevant code led somewhere plausible but wrong at least once (Issue #3), which is why
I kept reproduction and verification in my own hands.

---

## Codebase Map
<!-- Milestone 1. Write this in YOUR OWN WORDS from reading the code.
     Must cover: (a) main files and what each one does, (b) the data flow for at least one
     feature, (c) a pattern you noticed in how the app is organized.
     Should read as if written BEFORE any bug work. -->

### Main files and their roles

From my initial reading, the project is organized into an app factory, model definitions, a routing
layer, a service layer, and tests. The service files contain the application's business logic, while the
models define the database objects and their relationships.

- **`app.py`** — the Flask application factory (`create_app`). It configures the SQLite database,
  initializes SQLAlchemy, and registers four blueprints under URL prefixes: `/songs`, `/playlists`,
  `/users`, and `/feed`.
- **`models.py`** — defines the database entities and their relationships. The main models are `User`,
  `Song`, `Tag`, `ListeningEvent`, `Rating`, `Playlist`, and `Notification`. It also defines three
  association tables for the many-to-many relationships: `friendships` (a symmetric user-to-user link),
  `song_tags` (songs to tags), and `playlist_entries` (playlists to songs). `playlist_entries` is not a
  plain link table — it carries a `position` column, which means the order of songs in a playlist is
  stored explicitly rather than being just insertion order. Streak state (`listening_streak` and
  `last_listened_at`) lives directly on the `User` model.
- **`routes/`** — the routing layer (`songs.py`, `playlists.py`, `users.py`, `feed.py`). The routes are
  thin: each one parses the incoming request, calls the matching service function, and formats the
  response. They contain no business logic themselves.
- **`services/`** — where the application's business logic lives:
  - `playlist_service.py` is responsible for creating playlists and retrieving playlist information and
    songs.
  - `notification_service.py` handles creating notifications and user interactions such as rating songs
    or adding songs to playlists.
  - `streak_service.py` manages listening activity and keeps track of each user's listening streak.
  - `feed_service.py` builds the "Friends Listening Now" feed and a general activity feed.
  - `search_service.py` handles searching for songs by title or artist.
- **`tests/`** — verifies the expected behavior of the services (`test_streaks.py`, `test_search.py`,
  `test_playlists.py`).

### Data flow: Adding a song to a playlist

When a user adds a song to a playlist, the request enters at `POST /playlists/<playlist_id>/songs` in
`routes/playlists.py`, which calls `add_to_playlist()` in `notification_service.py`. The service first
loads the song, the user performing the action, and the target playlist from the database. If the song
is not already in the playlist, it is added and the database changes are committed. After the playlist
update, the service checks whether the song was originally shared by a different user. If so, it creates
a notification so the original sharer knows someone added their song to a playlist.

### Patterns I noticed

The project follows a consistent service-layer pattern. Each service function first validates its inputs,
loads the necessary database objects, performs the required business logic, commits any database changes,
and returns either a model object or a dictionary representation. The routes stay thin and delegate all
real work to a service.

I also noticed that each service focuses on a single responsibility. Playlist-related logic stays in the
playlist service, notification-related behavior stays in the notification service, and listening streak
logic stays in the streak service. This separation made the code relatively easy to navigate because
related functionality is grouped together in one place, and it also meant that when the README pointed
each open issue at a specific service file, I knew exactly where to start reading.

---

## Root Cause Analysis

### Issue #1 — My listening streak keeps resetting
- **How I reproduced it:** I isolated `update_listening_streak()` in a Python shell rather than firing
  HTTP requests, so the result wouldn't depend on the real clock. I built a synthetic user who had
  listened *exactly yesterday* with a streak of 3, then called the function with a controlled `now`
  anchored to each weekday (Mon–Sun). Per the docstring, a consecutive-day listen should increment the
  streak (3 → 4). The result: it incremented correctly Monday through Saturday, but on **Sunday** the
  streak reset to 1. That confirmed the reported "keeps resetting" behavior and narrowed it to a
  Sunday-only condition (which matches the fact that today, 2026-07-05, is a Sunday).
- **How I found the root cause (navigation):** The README maps this issue to `streak_service.py`, so I
  opened it and read `record_listening_event()` → `update_listening_streak()`. The three-way branch on
  `days_since_last` (lines 70–78) is where the increment-vs-reset decision is made. The `elif` on line 73
  stood out because it carried an extra clause unrelated to the day gap: `and today.weekday() != 6`.
  I confirmed with a one-liner that `datetime.weekday()` returns `6` for Sunday, and that was the moment
  I was sure — the reset was gated on the weekday, not on a skipped day.
- **The root cause:** On a valid consecutive-day listen `days_since_last == 1`. The increment branch was
  `elif days_since_last == 1 and today.weekday() != 6:`. Python's `datetime.weekday()` returns 6 for
  Sunday, so on Sundays `today.weekday() != 6` is `False`, the whole `elif` short-circuits to `False`,
  and execution falls through to the `else`, which sets `user.listening_streak = 1`. So a legitimate
  consecutive listen on a Sunday was misclassified as a broken streak and reset. The docstring's rules
  (lines 46–50) say "listened yesterday → increment" with no Sunday exception, so the code contradicted
  its own specification. The correct behavior requires the increment to depend *only* on the day gap,
  because "consecutive day" has nothing to do with which weekday it lands on.
- **My fix and side-effect check:** I removed the spurious clause so the branch reads
  `elif days_since_last == 1:`. Side-effect check: this branch is one of three, so I re-ran my weekday
  sweep across all three cases — consecutive (now increments on Sunday too, 3→4), same-day
  (`days_since_last == 0`, unchanged), and a 3-day gap (`> 1`, resets to 1) — and all passed on every
  weekday, confirming I didn't disturb the same-day or gap-reset boundaries. I also ran the repo's
  existing `tests/test_streaks.py` (5/5 pass) as an independent regression check.

### Issue #4 — Notified when a friend adds my song to a playlist, but not when they rate it
- **How I reproduced it:** In a clean in-memory DB I created a `sharer` who shares a song and a
  different `rater`. I called `rate_song(rater, song, 5)` and then queried the sharer's notifications:
  the rating saved fine but the sharer received **0** notifications. To prove the machinery wasn't
  broken, I called `create_notification()` directly and it produced a notification — isolating the
  defect to a missing call inside `rate_song()` rather than a broken helper.
- **How I found the root cause (navigation):** README maps this issue to `notification_service.py`. I
  put the two sibling actions side by side. `add_to_playlist()` (lines 35–70) ends with a guarded
  `create_notification(...)` block: `if song.shared_by != added_by_user_id:` → notify `song.shared_by`
  with type `song_added_to_playlist`. I then read all of `rate_song()` (lines 73–110) and saw it upserts
  the `Rating`, commits, and returns — with **no** `create_notification` call anywhere. The contrast
  between the two functions was the moment it was obvious.
- **The root cause:** `rate_song()` persists the rating but never notifies anyone. The notification
  system works (as `add_to_playlist` and a direct `create_notification` call both prove) — the rating
  code path simply omits the notification step. So the sharer is told when their song is added to a
  playlist but not when it's rated, exactly as reported. Correct behavior requires `rate_song` to mirror
  the playlist path: after saving, notify the song's original sharer.
- **My fix and side-effect check:** After the rating commit I added a guarded notification mirroring the
  playlist path: `if song.shared_by != user_id:` → `create_notification(user_id=song.shared_by,
  notification_type="song_rated", body=f"{rater.username} rated your song '{song.title}' {score}/5.")`.
  The guard prevents self-notification when someone rates their own shared song. Side-effect check: I
  verified (a) a rating by another user now yields exactly 1 `song_rated` notification, (b) the rating is
  still saved and returned, (c) re-rating updates in place — still exactly one `Rating` row (the
  `UniqueConstraint` upsert path is untouched), and (d) self-rating creates no notification. I also
  confirmed the two failing tests in the suite are the unrelated Issue #5 slice bug, not my change
  (my edit only touches `rate_song`, which the playlist code never calls).

### Issue #3 — (skipped) The same song keeps showing up twice in search
- **Why skipped:** I investigated this and found the reported symptom does **not** reproduce in this
  environment. `search_songs()` uses an unnecessary `outerjoin` to `song_tags`, which multiplies rows
  1-per-tag at the SQL level (I confirmed a 3-tag song returns 3 raw joined rows). However, the service
  uses SQLAlchemy's legacy `db.session.query(Song)` API, which auto-deduplicates single-entity results
  by primary key, so the duplication never reaches the caller — all 5 search tests pass, including
  `test_search_no_duplicates_multi_tag_song`. Per the milestone guidance ("if you can't reproduce a bug
  after a genuine attempt, try a different one"), I swapped this for Issue #4 above.

### Issue #5 — The last song in a playlist never shows up
- **How I reproduced it:** Two ways. First, the repo's own `tests/test_playlists.py` was already failing:
  `test_playlist_returns_all_songs` expected 5 songs but got 4, and `test_playlist_returns_songs_in_order`
  expected `Track 1..5` but lost `Track 5`. Second, I wrote a script that, for each seeded playlist,
  compared the `playlist_entries` rows in the DB against what `get_playlist_songs()` returned: every
  playlist had 7 songs in the DB but the service returned 6, and the missing one was always the
  highest-`position` song (`Free Throws`, `Harlem Renaissance`, `Lagos to London`).
- **How I found the root cause (navigation):** README maps this to `playlist_service.py`. I read
  `get_playlist_songs()` top-down: it queries all entries for the playlist, ordered by `position`
  ascending, into `songs`, then builds dicts. The `return` line (66) was the tell — it slices
  `songs[:-1]` before building the list. That single slice was the moment it clicked.
- **The root cause:** The return statement is `return [song.to_dict() for song in songs[:-1]]`. The
  slice `songs[:-1]` returns every element *except the last*. Because the query orders ascending by
  `position`, the last element is always the highest-position song — so the drop is deterministic, not
  random: the final song in playlist order is always omitted. This contradicts the function's own
  docstring, which says it returns *all* songs in order. Correct behavior requires iterating the full
  list, since every entry returned by the query is a real playlist member that should be shown.
- **My fix and side-effect check:** I removed the slice: `return [song.to_dict() for song in songs]`.
  Side-effect check: I re-ran the seeded repro (all playlists now return all 7 songs, nothing missing)
  and the full test suite (13/13 pass, including the two previously-failing playlist tests and the
  ordering test — so order is preserved). I specifically checked the boundary cases the slice also
  corrupted: a **1-song** playlist previously returned `[]` (`[x][:-1] == []`) and now correctly returns
  the one song; an **empty** playlist still returns `[]`. That empty case is why the bug hid — with zero
  songs there was nothing for the slice to drop.

---

## Commit History

Each bug fix is a separate commit on the `bugfix/mixtape` branch, using the conventional `fix:` prefix:

```
1f4e3cd fix: return all playlist songs, not all-but-last          (Issue #5)
13dc1d6 fix: notify song sharer when their song is rated          (Issue #4)
05f8100 fix: correct Sunday boundary in listening streak reset    (Issue #1)
2dfdeaa Add .gitignore file and update README with setup instructions
7b64551 initial commit
```

_(Screenshot of `git log --oneline` on the `bugfix/mixtape` branch to be attached.)_

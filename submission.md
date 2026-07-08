## AI Usage

I used Claude Code as an AI assistant while working on this project, but I stayed in control of the actual investigation and writing. Here is where it helped:

- Summarizing the codebase structure in Milestone 1 so I could get oriented faster.
- Tracing the route-to-service call chains for each bug (for example, `GET /playlists/<id>/songs` → `get_songs()` → `get_playlist_songs()`).
- Understanding what the suspicious service functions were actually doing.
- Comparing working and broken code paths — this mattered most for the notification bug, where lining up `rate_song()` against `add_to_playlist()` made the missing `create_notification()` call obvious.
- Drafting and revising parts of my codebase map and my root cause analysis entries.
- Checking whether my explanations were specific enough instead of vague.

I did not take the AI's output at face value. I verified its suggestions by reading the actual files, running `curl` commands against the running app, checking the seed data, and confirming the behavior manually before and after each fix.

One AI suggestion, about the search "duplicate results" bug, turned out to be incomplete: my first reproduction attempt (searching "Crown Heights Anthem") did not actually trigger a duplicate. Because I could not reproduce it with real terminal evidence, I did not document it as one of my bugs, and instead wrote up bugs I could actually reproduce and verify myself.

## Milestone 1: Codebase Map

### Main files and folders

The app is split into a few clear pieces. [app.py](app.py) is the main Flask setup file. It creates the app, sets `SQLALCHEMY_DATABASE_URI`, initializes `db = SQLAlchemy()`, registers the blueprints from [routes/songs.py](routes/songs.py), [routes/playlists.py](routes/playlists.py), [routes/users.py](routes/users.py), and [routes/feed.py](routes/feed.py), and calls `db.create_all()` inside the app context.

[models.py](models.py) is where the database structure lives. It defines the main SQLAlchemy models `User`, `Song`, `Playlist`, `Notification`, `Rating`, `ListeningEvent`, and `Tag`, plus the relationship tables `friendships`, `song_tags`, and `playlist_entries` that connect them.

[seed_data.py](seed_data.py) builds a sample dataset. It creates users, friendships, tags, songs, playlists, listening events, and notifications so the app has realistic data to work with, including the cases the README points to in the issue list.

[README.md](README.md) explains the assignment, the setup steps, and the five issue areas the bug hunt is built around. [requirements.txt](requirements.txt) lists the Python packages the app needs.

The [routes/](routes) folder contains the HTTP endpoints, split by feature: [routes/songs.py](routes/songs.py) handles `GET /songs/search`, `GET /songs/<song_id>`, `POST /songs/<song_id>/rate`, and `POST /songs/<song_id>/listen`; [routes/playlists.py](routes/playlists.py) handles `POST /playlists/`, `GET /playlists/<playlist_id>`, `GET /playlists/<playlist_id>/songs`, and `POST /playlists/<playlist_id>/songs`; [routes/users.py](routes/users.py) handles user profile, streak, and notification endpoints; and [routes/feed.py](routes/feed.py) handles `GET /feed/<user_id>/listening-now` and `GET /feed/<user_id>/activity`.

The [services/](services) folder has the actual app logic those routes call. For example, `routes/songs.py` calls `search_songs`, `get_song`, `rate_song`, and `record_listening_event`, while `routes/playlists.py` calls `create_playlist`, `get_playlist`, `get_playlist_songs`, `get_user_playlists`, and `add_to_playlist`. The [tests/](tests) folder has the focused tests for search, playlists, and streak behavior. The [instance/](instance) folder is the Flask instance folder for local app state.

### Data flow: adding a song to a playlist

One flow I traced is adding a song to a playlist.

The request starts at `POST /playlists/<playlist_id>/songs` in [routes/playlists.py](routes/playlists.py). That route reads the JSON body and expects `song_id` and `added_by`. Then it calls `add_to_playlist(playlist_id, song_id, added_by)` from [services/notification_service.py](services/notification_service.py).

Inside the service, the code looks up the `Song`, `User`, and `Playlist` records. If the song is not already in the playlist, it gets added through the `playlist.songs` relationship, which uses the `playlist_entries` table from [models.py](models.py). After that, if the person adding the song is not the original sharer, `create_notification()` writes a `Notification` row for the original sharer.

The route finishes by returning JSON with `{"message": "Song added to playlist"}` and a 201 status. So in this flow, the route is mostly the request/response wrapper, while the service is doing the database work and notification logic.

### Patterns I noticed

The biggest pattern I noticed is that the routes stay thin. In [routes/songs.py](routes/songs.py), the `rate()` route checks `user_id` and `score` and then calls `rate_song()`, and the `listen()` route calls `record_listening_event()`. In [routes/playlists.py](routes/playlists.py), the `add_song()` route checks `song_id` and `added_by` before calling `add_to_playlist()`. The services hold the actual behavior, so the business logic is separated from the HTTP layer.

Another pattern is that the models are built around relationships. This app uses many-to-many tables for friendships, song tags, and playlist songs, which makes sense because the whole app is built around social music sharing.

Most of the models also have a `to_dict()` method, which keeps the JSON response formatting pretty consistent across routes. The tests follow the same structure as the services, so the codebase is organized around features instead of being split into generic Flask pieces. The README also matches that structure by pointing directly to the main feature areas like `streak_service.py`, `feed_service.py`, `search_service.py`, `notification_service.py`, and `playlist_service.py`.

Overall, this feels like a feature-based Flask app where the route files are entry points and the service files hold the real behavior. Reading it that way made the code easier to follow.

### Issue overview

The README only gives short issue titles, so I matched those with the related tests and service files to figure out what each bug looks like in practice.

1. **My listening streak keeps resetting**
   - Plain-English summary: the streak count does not behave like a normal consecutive-day streak. It looks like listening on back-to-back days does not always keep the streak growing the way it should.
   - Relevant files: [services/streak_service.py](services/streak_service.py), [routes/songs.py](routes/songs.py), [routes/users.py](routes/users.py), and [tests/test_streaks.py](tests/test_streaks.py).
   - How I would reproduce it: create or use a user, call `POST /songs/<song_id>/listen` on two consecutive days, then check `GET /users/<user_id>/streak`. I would also test the Saturday-to-Sunday case since the streak tests cover that directly.

2. **Friends Listening Now shows people from yesterday**
   - Plain-English summary: the feed that is supposed to show recent listening activity is including old events, so it does not feel like a true “right now” feed.
   - Relevant files: [services/feed_service.py](services/feed_service.py), [routes/feed.py](routes/feed.py), [seed_data.py](seed_data.py), and [models.py](models.py) for `ListeningEvent` and `User`.
   - How I would reproduce it: use the seeded data, call `GET /feed/<user_id>/listening-now`, and check whether a friend who listened more than a day ago still shows up. The seed data already sets up recent and older listening events, so this should be easy to see.

3. **The same song keeps showing up twice in search**
   - Plain-English summary: one song can appear more than once in search results, especially when it has multiple tags.
   - Relevant files: [services/search_service.py](services/search_service.py), [routes/songs.py](routes/songs.py), [tests/test_search.py](tests/test_search.py), and [models.py](models.py) for `Song` and `Tag`.
   - How I would reproduce it: search for a song with multiple tags, like “Crown Heights Anthem,” through `GET /songs/search?q=Crown%20Heights`. Then count how many times that same title appears in the results.

4. **I got notified when a friend added my song to a playlist but not when they rated it**
   - Plain-English summary: the app creates a notification for playlist adds, but the matching notification does not seem to happen when someone rates a shared song.
   - Relevant files: [services/notification_service.py](services/notification_service.py), [routes/songs.py](routes/songs.py), [routes/users.py](routes/users.py), and [models.py](models.py) for `Notification`, `Song`, and `Rating`.
   - How I would reproduce it: have one user share a song, then have another user rate that song with `POST /songs/<song_id>/rate`. After that, check `GET /users/<user_id>/notifications` for the original sharer and see whether anything new shows up.

5. **The last song in a playlist never shows up**
   - Plain-English summary: when a playlist is fetched, it looks like the last song is missing from the returned list.
   - Relevant files: [services/playlist_service.py](services/playlist_service.py), [routes/playlists.py](routes/playlists.py), [tests/test_playlists.py](tests/test_playlists.py), and [models.py](models.py) for `Playlist`, `Song`, and `playlist_entries`.
   - How I would reproduce it: create or use a playlist with several songs, then call `GET /playlists/<playlist_id>/songs` and compare the returned count with the number of songs that were added.

**The three I would start with first**

- **#3 search duplicates** seems easiest because the search tests already describe the expected behavior clearly, and the bug is easy to spot with one search request.
- **#5 missing last playlist song** also looks pretty direct because the playlist tests already check song count and order, so it is simple to compare the API output with the data in the playlist.
- **#1 listening streak resets** is a good third choice because the streak tests already cover the edge cases I would check first, especially same-day listens and the Saturday-to-Sunday case.

### AI assistance disclosure

I used Claude Code to help me read and organize the files, but I checked the code myself and wrote these notes based on what is actually in this repo. There is a fuller account of how I used AI across all milestones in the "AI Usage" section at the top of this document.

## Milestone 2: Bug Reproduction

These are my reproduction notes for the bugs I picked. I focused only on showing that each bug actually happens. I did not look at root causes or fixes yet.

### Bug 1: The last song in a playlist never shows up

**How I reproduced it:**
- I ran `python seed_data.py` to load the sample data.
- I started the Flask app locally.
- I picked the seeded playlist "Late Night Vibes". Its id was `828f6cb3-162e-4151-b4d1-ecca9ab70c36`.
- I called `GET /playlists/828f6cb3-162e-4151-b4d1-ecca9ab70c36/songs`.
- I counted how many songs came back and checked the titles.

**Expected behavior:**
Late Night Vibes was seeded with 7 songs, so the endpoint should return `count: 7`, and the last song "Free Throws" should be included.

**Actual buggy behavior:**
The endpoint returned `count: 6`, and "Free Throws" (the last song) was missing. The other songs came back fine, so it is specifically the final song that gets dropped.

**Evidence from my terminal:**
```
count: 6
titles: ['Midnight Drive', 'Still Waters', 'First Light', 'Block Party', 'Late Night Session', 'Golden Hour']
Free Throws present: False
```

### Bug 2: Rating a song does not create a notification

**How I reproduced it:**
- I ran `python seed_data.py` to load the sample data.
- I used "nova" as the original song sharer and "darius" as the person rating the song.
  - nova id: `4d65c810-46c5-4a3e-90a2-7119a3c2cd78`
  - darius id: `c0616919-1f6e-43d7-a0e5-592bf5825ea0`
  - "Midnight Drive" song id: `942719c0-1a3b-43bb-8799-09d71061088b`
- Before rating, I checked nova's notifications with `GET /users/<nova_id>/notifications`.
- Then I sent `POST /songs/942719c0-1a3b-43bb-8799-09d71061088b/rate` with the body `{"user_id":"c0616919-1f6e-43d7-a0e5-592bf5825ea0","score":5}`.
- The rating request succeeded and returned a rating object with score 5.
- After rating, I checked nova's notifications again to see if a new one showed up.

**Expected behavior:**
When darius rates nova's shared song, nova should get a new notification about the rating, similar to how she gets one when someone adds her song to a playlist. So the notification count should go from 1 to 2.

**Actual buggy behavior:**
The rating was clearly saved (the POST returned a valid rating with score 5), but nova's notification count stayed at 1. No new "rated your song" notification was created. The only notification she had was still the old playlist-add one.

**Evidence from my terminal:**
```
Before count: 1
Before bodies: ["darius added your song 'Midnight Drive' to the playlist 'Late Night Vibes'."]

(rating POST returned score 5)

After count: 1
After bodies: ["darius added your song 'Midnight Drive' to the playlist 'Late Night Vibes'."]
```
The before and after are identical, which shows the rating went through but no notification was ever created.

### Bug 3: Friends Listening Now shows people from a while ago

For my third bug I first tried the search duplicates issue (#3 in the README). I searched for the multi-tag song "Crown Heights Anthem" with `GET /songs/search?q=Crown%20Heights`, since it has 3 tags and seemed most likely to duplicate, but it only came back once:
```
count: 1
titles: ['Crown Heights Anthem']
Crown Heights Anthem occurrences: 1
```
Since I could not reproduce a duplicate through the search endpoint, I switched to the "Friends Listening Now" issue instead, which I was able to reproduce clearly.

**How I reproduced it:**
- I ran `python seed_data.py` to load the sample data.
- I started the Flask app locally.
- I used "kenji" as the current user, because in the seed data his only friend with a listening event inside the window is "nova".
  - kenji id: `1b849c15-9506-4334-a6e7-25479173b6ec`
  - nova id: `00919c2d-3344-4b06-ae7b-6d311194e24e`
- I called `GET /feed/1b849c15-9506-4334-a6e7-25479173b6ec/listening-now`.
- I compared the `listened_at` timestamp in the response with the current server time.

**Expected behavior:**
"Friends Listening Now" should only show friends who are listening right now (or in the last few minutes). If a friend last listened hours ago, they should not be in this feed anymore.

**Actual buggy behavior:**
The feed still listed nova as "listening now" even though her listening event was about 2 hours old. Her `listened_at` was `2026-07-07T05:27:23`, but the server time when I made the request was `2026-07-07T07:29:12` — roughly 2 hours later. So someone who listened 2 hours ago is still being treated as listening right now.

**Evidence from my terminal:**
```
--- kenji listening-now ---
{"count":1,"feed":[{"friend":{"username":"nova", ...},
  "listened_at":"2026-07-07T05:27:23.171064",
  "song":{"title":"Midnight Drive", ...}}]}

--- server current time (UTC) ---
2026-07-07T07:29:12.985355+00:00
```
The one entry in the "listening now" feed has a `listened_at` about 2 hours before the current time, which shows the feed is including stale events instead of only current ones.

## Milestone 3: Root Cause Analysis and Fixes

I fixed three bugs: the missing last playlist song (Issue #5), the missing rating notification (Issue #4), and the stale "Friends Listening Now" feed (Issue #2). Each entry below covers how I reproduced it, how I found the cause, the exact cause, and my fix.

These are the same three bugs I reproduced in Milestone 2, just labeled by their README issue numbers: Bug 1 = Issue #5 (last playlist song), Bug 2 = Issue #4 (rating notification), Bug 3 = Issue #2 (Friends Listening Now).

### Issue #5: The last song in a playlist never shows up

**How I reproduced it:**
I ran `python seed_data.py`, started the Flask app, and queried the seeded playlist "Late Night Vibes" (id `828f6cb3-162e-4151-b4d1-ecca9ab70c36`) with `GET /playlists/828f6cb3-162e-4151-b4d1-ecca9ab70c36/songs`. The playlist was seeded with 7 songs, but the response came back with only 6:
```
count: 6
titles: ['Midnight Drive', 'Still Waters', 'First Light', 'Block Party', 'Late Night Session', 'Golden Hour']
Free Throws present: False
```
The last song, "Free Throws", was missing.

**How I found the root cause:**
The request hits `get_songs()` in [routes/playlists.py](routes/playlists.py), which is just a wrapper that calls `get_playlist_songs()` and returns it with a count, so there was no logic there to blame. I followed the call into `get_playlist_songs()` in [services/playlist_service.py](services/playlist_service.py). I read the database query first (it joins `Song` to the `playlist_entries` table, filters to the one playlist, and orders by `position` ascending) and confirmed it was correct and returned all the songs. That pointed me at the one line after the query — the return statement.

**The root cause:**
The function returned `[song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice means "every element except the last one," so after the query correctly fetched the full, position-ordered list, the code sliced off the final element right before returning. Because the list is ordered ascending by position, the dropped element is always the last song in the playlist. That is a classic off-by-one: the query was never wrong, but the slice discarded one good row. The function's own docstring even says it "returns all songs in the playlist," which the slice contradicted.

**My fix and side-effect check:**
I removed the `[:-1]` slice so the comprehension runs over the whole list: `return [song.to_dict() for song in songs]`. This fixes the cause because there is nothing else trimming the data — the query already returns everything in order, so once the slice is gone the full list is returned. I verified the fix by reseeding and re-running the same query I used to reproduce it:
```
count: 7
titles: ['Midnight Drive', 'Still Waters', 'First Light', 'Block Party', 'Late Night Session', 'Golden Hour', 'Free Throws']
Free Throws present: True
```
The count is back to 7 and "Free Throws" is present and last. I checked the related behavior too: ordering is still correct (ascending by position), an empty playlist still returns an empty list, and all three tests in `tests/test_playlists.py` pass. No other tests regressed.

### Issue #4: Rating a song does not create a notification

**How I reproduced it:**
I ran `python seed_data.py` and used "nova" as the song sharer and "darius" as the rater (nova id `4d65c810-46c5-4a3e-90a2-7119a3c2cd78`, darius id `c0616919-1f6e-43d7-a0e5-592bf5825ea0`, "Midnight Drive" song id `942719c0-1a3b-43bb-8799-09d71061088b`). I checked nova's notifications, then sent `POST /songs/942719c0-1a3b-43bb-8799-09d71061088b/rate` with `{"user_id":"c0616919-1f6e-43d7-a0e5-592bf5825ea0","score":5}`, then checked nova's notifications again:
```
Before count: 1
Before bodies: ["darius added your song 'Midnight Drive' to the playlist 'Late Night Vibes'."]

(rating POST returned score 5)

After count: 1
After bodies: ["darius added your song 'Midnight Drive' to the playlist 'Late Night Vibes'."]
```
The rating saved fine, but nova got no new notification.

**How I found the root cause:**
The route `rate()` in [routes/songs.py](routes/songs.py) only validates `user_id` and `score` and calls `rate_song()`, so it does no notification work. I followed it into `rate_song()` in [services/notification_service.py](services/notification_service.py) and read it top to bottom: it validates the score, loads the song and rater, saves the rating (new or updated), commits, and returns. That was the whole function — it never touched notifications. I then compared it to `add_to_playlist()` right above it in the same file, which does the "someone interacted with your shared song" case correctly by calling `create_notification()` for the song's sharer. That comparison made the missing step obvious.

**The root cause:**
`rate_song()` never called `create_notification()`. This was a missing call, not a broken one — the rating was persisted correctly (the POST returned a valid rating with score 5), but because no notification row was ever created, the sharer's notification count stayed the same. That is exactly why nova's count was 1 before and 1 after.

**My fix and side-effect check:**
I added a notification step at the end of `rate_song()`, mirroring the pattern already used in `add_to_playlist()`, using the existing `create_notification()` helper and the `song_rated` type its docstring already mentions:
```python
if is_new_rating and song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score} out of 5.",
    )
```
I scoped it with an `is_new_rating` flag so it only fires on a first-time rating (not when someone just changes their score), and with `song.shared_by != user_id` so nobody gets notified for rating their own song. This fixes the cause by restoring the missing interaction-to-notification link. I verified the fix by reseeding and replaying my reproduction, then checking the two edge cases:
```
Before count: 1
After count: 2
After bodies: ["darius rated your song 'Midnight Drive' 5 out of 5.",
               "darius added your song 'Midnight Drive' to the playlist 'Late Night Vibes'."]
After re-rate count: 2
After nova self-rates count: 2
```
darius rating nova's song raised her count from 1 to 2 with the new "rated your song" body; re-rating the same song kept the count at 2 (no duplicate); and nova rating her own song added nothing. Rating persistence itself was unchanged, and no tests regressed.

### Issue #2: Friends Listening Now shows people from a while ago

**How I reproduced it:**
I ran `python seed_data.py`, started the app, and used "kenji" as the current user (id `1b849c15-9506-4334-a6e7-25479173b6ec`) because his only friend with an event in the window was nova (id `00919c2d-3344-4b06-ae7b-6d311194e24e`). I called `GET /feed/1b849c15-9506-4334-a6e7-25479173b6ec/listening-now` and compared the `listened_at` with the server time:
```
--- kenji listening-now ---
{"count":1,"feed":[{"friend":{"username":"nova", ...},
  "listened_at":"2026-07-07T05:27:23.171064", ...}]}

--- server current time (UTC) ---
2026-07-07T07:29:12.985355+00:00
```
nova showed up as "listening now" even though her event was about 2 hours old.

**How I found the root cause:**
The route `listening_now()` in [routes/feed.py](routes/feed.py) just calls `get_friends_listening_now()` and wraps the result, so it does no filtering. Inside `get_friends_listening_now()` in [services/feed_service.py](services/feed_service.py), I traced the logic: it builds a cutoff as `now - RECENT_THRESHOLD`, queries friends' listening events where `listened_at >= cutoff`, and dedups to the most recent event per friend. The query, ordering, and dedup all looked right, so the only thing deciding "how recent counts as now" was the `RECENT_THRESHOLD` constant at the top of the file.

**The root cause:**
`RECENT_THRESHOLD` was set to `timedelta(hours=24)`. "Friends Listening Now" is meant to show who is listening right now, but a 24-hour window counts anyone who listened at any point in the past day as currently listening. That is why nova's 2-hour-old event still showed up — 2 hours is far inside a 24-hour window. The condition itself (`listened_at >= cutoff`) was fine; the window was just far too wide.

**My fix and side-effect check:**
I changed the constant to `RECENT_THRESHOLD = timedelta(minutes=30)`. The seed data confirms this is the intended window — its comments say events within the past 30 minutes should appear and older ones should not, and it seeds the "recent" events at 10-20 minutes ago and the rest at 2+ hours ago. With a 30-minute cutoff, the genuine recent events still pass the filter and the stale ones are excluded before dedup. I verified by reseeding and re-checking the same feeds:
```
kenji listening-now count: 0
nova listening-now count: 3
  darius: 10 min ago
  simone: 15 min ago
  kenji: 20 min ago
```
kenji's feed went from `count: 1` (nova, ~2 hours stale) to `count: 0`, and every entry left in nova's feed was within the last 20 minutes. I checked that real recent activity is preserved — nova's feed still shows her three friends who listened 10-20 minutes ago — and that the separate `activity` feed, which is intentionally not recency-filtered, was unaffected. No tests regressed.

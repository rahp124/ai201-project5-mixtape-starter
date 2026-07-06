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

I used Copilot to help me read and organize the files, but I checked the code myself and wrote these notes based on what is actually in this repo.

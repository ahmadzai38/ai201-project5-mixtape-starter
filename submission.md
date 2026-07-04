# Project 5: Mixtape Bug Hunt Submission

## AI Usage

## AI Usage

I used AI during codebase orientation and debugging in specific ways.

First, I pasted files like `app.py`, `models.py`, and the route files into AI and asked it to summarize what each file was responsible for. This helped me build the codebase map, but I verified the summaries myself by reading the files and checking how the routes imported service functions.

Second, I used AI to help trace call chains. For example, I traced `POST /songs/<song_id>/listen` from `routes/songs.py` to `record_listening_event()` and then to `update_listening_streak()` in `services/streak_service.py`. I verified the issue by running `python -m pytest tests/test_streaks.py`.

Third, for the notification issue, I used AI to compare `add_to_playlist()` and `rate_song()` in `services/notification_service.py`. The comparison helped me notice that playlist actions created notifications, but rating actions did not. I did not accept that as the answer automatically; I verified it by running `python seed_data.py` and then using a Python command to check the notification count before and after calling `rate_song()`.

AI helped me understand and trace the code, but I verified the fixes by reproducing the bugs and running the tests.

## Codebase Map

### Main Files and Roles

- `app.py`: Creates the Flask app using the `create_app()` factory function. It sets up the SQLite database connection, initializes SQLAlchemy, registers the route blueprints for songs, playlists, users, and feed, and creates the database tables inside the app context.
- `models.py`: Defines the database models for the app. The main models are `User`, `Song`, `Rating`, `ListeningEvent`, `Playlist`, `Tag`, and `Notification`. It also defines join tables for friendships, song tags, and playlist entries. The `playlist_entries` table is important because it connects playlists to songs and stores each song's `position`, who added it, and when it was added.
- `routes/songs.py`: Defines the song-related API routes. It handles searching songs, getting one song by ID, rating a song, and recording a listening event. This file does input checking, then calls service functions like `search_songs()`, `get_song()`, `rate_song()`, and `record_listening_event()`.
- `routes/playlists.py`: Defines the playlist-related API routes. It handles creating a playlist, getting playlist details, getting songs inside a playlist, and adding a song to a playlist. The route checks request data like `name`, `created_by`, `song_id`, and `added_by`, then calls service functions such as `create_playlist()`, `get_playlist()`, `get_playlist_songs()`, and `add_to_playlist()`.
- `routes/users.py`: Defines the user-related API routes. It handles getting a user profile, getting a user's listening streak, getting a user's notifications, and marking a notification as read. It calls service functions like `get_streak()`, `get_notifications()`, and `mark_as_read()`.
- `routes/feed.py`: Defines the feed-related API routes. It handles the “friends listening now” feed and the activity feed. It calls `get_friends_listening_now()` and `get_activity_feed()` from `services/feed_service.py`.
- `services/streak_service.py`: Handles listening streak logic. When a user listens to a song, `record_listening_event()` creates a `ListeningEvent`, then calls `update_listening_streak()` to update the user's streak based on the last day they listened. It also has `get_streak()` for returning a user's current streak.
- `services/feed_service.py`: Handles the feed logic. `get_friends_listening_now()` returns friends who listened recently and shows only the most recent song per friend. `get_activity_feed()` returns recent listening activity from friends without the same “listening now” filter.
- `services/search_service.py`: Handles song search logic. `search_songs()` searches songs by title or artist using a case-insensitive match, then returns the matching songs as dictionaries. `get_song()` retrieves one song by ID or raises an error if the song does not exist.
- `services/notification_service.py`: Handles notification logic. `create_notification()` creates a notification record. `add_to_playlist()` adds a song to a playlist and notifies the original song sharer if someone else added their song. `rate_song()` creates or updates a song rating. `get_notifications()` returns a user's notifications, and `mark_as_read()` marks one notification as read.
- `services/playlist_service.py`: Handles playlist creation and playlist retrieval logic. `create_playlist()` creates a playlist for a user. `get_playlist_songs()` gets the songs inside a playlist ordered by their playlist position. `get_playlist()` returns playlist metadata, and `get_user_playlists()` returns playlists created by one user.
- `seed_data.py`: Resets and fills the database with test data. It creates users, friendships, tags, songs, listening events, playlists, playlist entries, ratings, and notifications. This file is useful for reproducing the bugs because it creates the app state needed to test search duplicates, listening-now results, playlist songs, streaks, and notifications.
- `tests/`: Contains automated tests for some app features, including streaks, search, and playlists. These tests can help check whether a fix breaks existing behavior.

### Data Flow Example

Feature traced: A user listens to a song.

Steps:
1. The user sends a POST request to `/songs/<song_id>/listen`.
2. `routes/songs.py` receives the request in the `listen(song_id)` function.
3. The route reads `user_id` from the JSON request body.
4. If `user_id` is missing, the route returns a 400 error.
5. If `user_id` exists, the route calls `record_listening_event(user_id, song_id)` from `services/streak_service.py`.
6. The service function handles the real business logic: creating a listening event and updating the user's listening streak.
7. The route returns the listening event as JSON with status code 201.
8. Inside `update_listening_streak()`, the app compares today's date with the user's previous `last_listened_at` date.
9. If the user listened for the first time, the streak becomes 1.
10. If the user already listened today, the streak does not change.
11. If the user listened yesterday, the streak should increase by 1.
12. If the user skipped more than one day, the streak resets to 1.


### Patterns I Noticed

- The app uses SQLAlchemy models to represent the main objects in the music app: users, songs, ratings, listening events, playlists, tags, and notifications.
- Some relationships are many-to-many, like users having friends, songs having tags, and playlists having many songs.
- Playlist songs are not just connected randomly; the `playlist_entries` table also stores the song order using the `position` column.
- The app uses Flask blueprints to separate routes by feature: songs, playlists, users, and feed.
- `app.py` does not contain the business logic. It mainly connects the app, database, and route files together.
- The route files mostly do request handling: they read URL parameters, read JSON data, check for missing inputs, call service functions, and return JSON responses.
- The actual app logic is not inside the route file. For example, listening streak logic is handled in `services/streak_service.py`, not directly in `routes/songs.py`.
- User routes connect to both streak logic and notification logic.
- The `/users/<user_id>/streak` route does not calculate the streak itself. It calls `get_streak()` from `services/streak_service.py`.
- The `/users/<user_id>/notifications` route does not directly query notifications. It calls `get_notifications()` from `services/notification_service.py`.
- Feed routes also follow the same pattern: the route receives the request, calls a service function, and returns JSON.
- The bug “Friends Listening Now shows people from yesterday” is likely connected to `services/feed_service.py` because this route calls `get_friends_listening_now()`.
- `get_friends_listening_now()` filters listening events using a cutoff time, then removes duplicates so each friend only appears once.
- The activity feed and listening-now feed are similar, but they are not supposed to behave the same. Listening-now should be more time-sensitive, while activity feed can show older events.
- Search logic is handled in the service layer, not inside the route.
- The search route calls `search_songs(query)`, and the service returns a list of song dictionaries.
- `search_service.py` imports tag-related objects, but the current search filter only checks song title and artist.
- Notifications are stored as `Notification` rows in the database.
- `add_to_playlist()` creates a notification after a playlist action.
- Before the fix, `rate_song()` saved the rating but did not create a notification for the original song sharer. This helped identify Issue #4.- Playlist song order comes from the `position` column in the `playlist_entries` join table.
- `get_playlist_songs()` queries songs in order, then converts each song to a dictionary before returning them.
- `seed_data.py` creates realistic test data so the bugs can be reproduced without manually creating every user, song, and playlist.
- Some seed data is intentionally connected to the bugs. For example, songs with multiple tags help expose the duplicate search issue, older listening events help expose the listening-now issue, and existing playlist notifications show the expected notification pattern.
---

## Bug Fix 1

### Issue number and title

Issue #5: The last song in a playlist never shows up

### How I reproduced it

I reproduced this bug by running the playlist tests with:

`python -m pytest tests/test_playlists.py`

Before the fix, `test_playlist_returns_all_songs` failed because the test expected 5 songs but the function returned only 4. `test_playlist_returns_songs_in_order` also failed because the returned list ended at `Track 4` instead of including `Track 5`.

### How I found the root cause

I started from the route `GET /playlists/<playlist_id>/songs` in `routes/playlists.py`. That route calls `get_playlist_songs(playlist_id)` from `services/playlist_service.py`. Then I read `get_playlist_songs()` and saw that it queried the songs correctly in ascending playlist position, but the return statement used `songs[:-1]`.

### The root cause

The root cause was the slice `songs[:-1]` in `get_playlist_songs()`. In Python, `songs[:-1]` means “return every item except the last one.” So even though the database query found all songs in the playlist, the final song was removed right before the result was returned.

### My fix and side-effect check

I changed:

`return [song.to_dict() for song in songs[:-1]]`

to:

`return [song.to_dict() for song in songs]`

This fixes the bug because the function now returns every song from the query, including the last one. After the fix, I ran:

`python -m pytest tests/test_playlists.py`

All 3 playlist tests passed, including the test that checks the playlist returns all 5 songs and the test that checks the songs stay in order.

---

## Bug Fix 2

### Issue number and title

Issue #1: My listening streak keeps resetting

### How I reproduced it

I reproduced this bug by running the streak tests with:

`python -m pytest tests/test_streaks.py`

Before the fix, `test_streak_increments_on_sunday` failed. The test listened on Saturday first, then Sunday. The expected streak was 2, but the actual streak stayed at 1, meaning the streak reset instead of continuing.

### How I found the root cause

I traced the route `POST /songs/<song_id>/listen` in `routes/songs.py`. That route calls `record_listening_event()` in `services/streak_service.py`. Inside that service file, `record_listening_event()` creates the listening event and then calls `update_listening_streak()`. I found the problem inside `update_listening_streak()` where it checked the number of days since the user last listened.

### The root cause

The root cause was this condition:

`elif days_since_last == 1 and today.weekday() != 6:`

In Python, `weekday()` returns `6` for Sunday. That meant if a user listened on Saturday and then listened again on Sunday, the code did not increment the streak because Sunday was blocked by `today.weekday() != 6`. The app treated Sunday as a reset case even though it was only one day after Saturday.

### My fix and side-effect check

I changed:

`elif days_since_last == 1 and today.weekday() != 6:`

to:

`elif days_since_last == 1:`

This fixes the bug because any consecutive calendar day now increments the streak, including Sunday. After the fix, I ran:

`python -m pytest tests/test_streaks.py`

All 5 streak tests passed, including the Sunday test.


---

## Bug Fix 3

### Issue number and title

Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

### How I reproduced it

I reproduced this bug by resetting the seed data with:

`python seed_data.py`

Then I used a Python command to have `darius` rate the song `Midnight Drive`, which was originally shared by `nova`. Before calling `rate_song()`, I counted Nova's `song_rated` notifications. Then I called `rate_song(darius.id, song.id, 5)` and counted again.

Before the fix, the output was:

`before: 0`
`after: 0`

This showed that the rating was saved, but no notification was created for the original song sharer.

### How I found the root cause

I started from the route `POST /songs/<song_id>/rate` in `routes/songs.py`. That route calls `rate_song()` from `services/notification_service.py`. I compared `rate_song()` with `add_to_playlist()` in the same service file. `add_to_playlist()` had logic to notify the original song sharer when someone else added their song to a playlist, but `rate_song()` only saved the rating and returned it.

### The root cause

The root cause was that `rate_song()` did not create a notification after saving a rating. The app already had a notification pattern in `add_to_playlist()`, but the rating path was missing the same kind of notification logic. So when someone rated another user's song, the rating was saved, but the song's original sharer never received a `song_rated` notification.

### My fix and side-effect check

I added notification creation after the rating is committed. If the person rating the song is not the same person who originally shared the song, the app now creates a `song_rated` notification for the original sharer.

I checked the fix by running the reproduction command again. After the fix, the output changed to:

`before: 0`
`after: 1`

This confirmed that the notification was created. I also ran:

`python -m pytest tests/`

All 13 tests passed, so the fix did not break the existing playlist, search, or streak tests.


---

## Git Log Screenshot


I included `git-log-screenshot.png` in this repository. It shows `git log --oneline` on the `bugfix/mixtape` branch with three separate `fix:` commits.
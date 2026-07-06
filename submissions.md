# AI201 Project 5 – Mixtape Bug Hunt

## Codebase Map

### Overall Architecture

The application follows a layered architecture:

```
Client Request
      ↓
Flask Route (Blueprint)
      ↓
Service Layer
      ↓
SQLAlchemy Models
      ↓
SQLite Database
```

The route files are responsible for receiving HTTP requests, validating request data, and formatting JSON responses. Nearly all business logic is implemented in the `services/` layer, while the models define the application's database schema and relationships.

---

## Main Files

### app.py

- Creates the Flask application.
- Configures SQLAlchemy and application settings.
- Registers the four blueprints:
  - `/songs`
  - `/playlists`
  - `/users`
  - `/feed`
- Creates database tables during application startup.

---

### models.py

Defines every database model and relationship used throughout the application.

Main models:

- **User**
  - Stores user information, listening streak, and last listening timestamp.
  - Relationships:
    - Shared songs
    - Ratings
    - Listening events
    - Notifications
    - Playlists
    - Friends (many-to-many)

- **Song**
  - Stores song metadata including title, artist, album, genre, share note, and owner.
  - Has relationships with ratings, listening events, and tags.

- **ListeningEvent**
  - Records every time a user listens to a song.
  - Used by listening streaks and activity feeds.

- **Rating**
  - Stores user ratings for songs.
  - A unique constraint prevents the same user from rating the same song more than once.

- **Playlist**
  - Stores playlist information.
  - Connected to songs through the `playlist_entries` association table.

- **Notification**
  - Stores notifications sent to users.
  - Tracks notification type, message body, creation time, and read status.

Association tables:

- `friendships`
- `song_tags`
- `playlist_entries`

The `playlist_entries` table stores additional metadata including playlist position, who added the song, and when it was added.

---

## Route Files

### routes/songs.py

Handles:

- Song search
- Song details
- Rating songs
- Recording listening events

Calls:

- `search_service`
- `notification_service`
- `streak_service`

---

### routes/playlists.py

Handles:

- Playlist creation
- Playlist retrieval
- Listing playlist songs
- Adding songs to playlists

Calls:

- `playlist_service`
- `notification_service`

---

### routes/users.py

Handles:

- User profile
- Listening streak
- User notifications
- Marking notifications as read

Calls:

- `streak_service`
- `notification_service`

---

### routes/feed.py

Handles:

- Friends Listening Now
- Activity Feed

Calls:

- `feed_service`

---

## Data Flow Example – User Rates a Song

```
POST /songs/<song_id>/rate
        ↓
routes/songs.py
        ↓
notification_service.rate_song()
        ↓
Creates Rating record
        ↓
Creates Notification (expected behavior)
        ↓
Database
        ↓
Returns JSON response
```

The route validates the request and delegates all business logic to the notification service.

---

## Data Flow Example – User Listens to a Song

```
POST /songs/<song_id>/listen
        ↓
routes/songs.py
        ↓
streak_service.record_listening_event()
        ↓
Creates ListeningEvent
Updates User listening streak
Updates last_listened_at
        ↓
Database
        ↓
Returns listening event
```

The listening event is stored and the user's streak information is updated through the service layer.

---

## Design Patterns Observed

- Routes are intentionally lightweight.
- Business logic is centralized inside the `services/` directory.
- SQLAlchemy models represent the application's data and relationships.
- Each route typically delegates work to a single service function.
- Models provide `to_dict()` methods so routes can easily return JSON responses.
- Many-to-many relationships are implemented using association tables, with `playlist_entries` storing additional metadata such as song order.

---

## Initial Understanding Before Debugging

Based on the README and the project structure:

- Listening streak functionality is implemented through `ListeningEvent` and `User`.
- Friends Listening Now likely uses recent `ListeningEvent` records.
- Search functionality is isolated inside `search_service`.
- Playlist retrieval is handled entirely through `playlist_service`.
- Notification creation is centralized in `notification_service`, which is used when users rate songs or add songs to playlists.

This organization makes the service layer the primary location for debugging the five reported issues.

## Rough plan:

### Issue #1 – My listening streak keeps resetting

**How I reproduced it**

- Started the application with the seeded database.
- Followed the steps described in the issue.
- Sent a POST request to the listening endpoint.
- Retrieved the user's streak using the streak endpoint.
- Observed that the streak reset unexpectedly under the reported condition.


Root Cause

The bug is in update_listening_streak():

elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1

today.weekday() == 6 means Sunday.

The code explicitly prevents the streak from incrementing on Sundays, even if the user listened on Saturday. As a result, the streak resets to 1 every Sunday instead of continuing.

Fix

Remove the unnecessary weekday check so that any consecutive day increments the streak:

elif days_since_last == 1:
    user.listening_streak += 1

 
### Issue #2 – Friends Listening Now shows people from yesterday

**How I reproduced it**

- Started the application with the seeded database.
- Followed the steps described in the issue.
- Accessed the Friends Listening Now endpoint for a user.
- Compared the returned listening activity with the expected "currently listening" behavior.
- Observed that users with listening activity from the previous day were still included in the results.
 

Root Cause

The service defines:

RECENT_THRESHOLD = timedelta(hours=24)

and filters using:

cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD

This means any listening event from the last 24 hours is considered "Listening Now."

For example:

Yesterday at 9:00 PM
Today at 8:00 PM

is only 23 hours apart, so yesterday's event still appears.

The issue description specifically says users from yesterday should not appear.

Fix

The filtering should compare calendar dates (or whatever the issue specifies), rather than a rolling 24-hour window. We'll implement the exact logic after confirming the intended behavior from the issue description.

---

### Issue #5 – The last song in a playlist never shows up

**How I reproduced it**

- Started the application with the seeded database.
- Used a playlist containing multiple songs.
- Requested the songs for that playlist through the playlist endpoint.
- Compared the returned list with the songs stored in the playlist.
- Observed that the final song in the playlist was missing from the response. 

Root Cause

The bug is immediately visible here:

return [song.to_dict() for song in songs[:-1]]

songs[:-1] returns every element except the last one.


Fix

Return the complete list:

return [song.to_dict() for song in songs]


-----------

# Root Cause Analysis

## Issue #1 – My listening streak keeps resetting

### How I reproduced it

* Started the application using the seeded database.
* Sent a `POST /songs/<song_id>/listen` request to record a listening event.
* Retrieved the user's streak using `GET /users/<user_id>/streak`.
* Repeated the test using dates that crossed from Saturday to Sunday.
* Observed that the listening streak reset to 1 instead of incrementing on Sunday.

### How I found the root cause

I started at `routes/songs.py` and traced the `POST /songs/<song_id>/listen` endpoint. The route calls `record_listening_event()` in `services/streak_service.py`, which then calls `update_listening_streak()`. Reading that function revealed a conditional statement that handled consecutive listening days. The condition explicitly checked `today.weekday() != 6`, which immediately stood out because `datetime.weekday()` returns `6` for Sunday. Since the bug report mentioned Sunday behavior, this confirmed the root cause.

### The root cause

The streak increment logic incorrectly excluded Sundays. The code only incremented the streak when `days_since_last == 1` **and** `today.weekday() != 6`. Since `datetime.weekday()` returns `6` on Sunday, a user who listened on Saturday and again on Sunday had their streak reset instead of incremented.

### My fix and side-effect check

I removed the unnecessary weekday check so that any consecutive calendar day increments the streak.

Changed:

```python
elif days_since_last == 1 and today.weekday() != 6:
```

to:

```python
elif days_since_last == 1:
```

After the change, I verified that:

* listening multiple times on the same day does not increase the streak,
* listening on consecutive days increments the streak,
* missing more than one day still resets the streak.


---------------
## Issue #2 – Friends Listening Now shows people from yesterday

### How I reproduced it

* Started the application with the seeded database.
* Accessed the `GET /feed/<user_id>/listening-now` endpoint.
* Used listening events whose timestamps were from the previous calendar day but less than 24 hours old.
* Observed that those users still appeared in the "Friends Listening Now" feed.

### How I found the root cause

I began with `routes/feed.py`, which routes the request to `get_friends_listening_now()` in `services/feed_service.py`. Reading the function showed that it filtered recent listening events using a cutoff of `datetime.now(timezone.utc) - timedelta(hours=24)`. Comparing this logic with the issue description revealed that the implementation treated "now" as the previous 24 hours instead of restricting results to the intended current listening period.

### The root cause

The service used a rolling 24-hour window to determine whether a friend was "listening now." Because of this, listening events from the previous calendar day were still returned if they occurred within the last 24 hours. This caused users from yesterday to incorrectly appear in the current listening feed.

### My fix and side-effect check

I updated the filtering logic so that only events matching the intended "current" time period are returned instead of every event from the last 24 hours.

After making the change, I verified that:

* current listening events still appear,
* yesterday's listening events no longer appear,
* the separate activity feed continues to return historical listening events because it intentionally is not filtered by recency.


-----------------

## Issue #5 – The last song in a playlist never shows up

### How I reproduced it

* Started the application using the seeded database.
* Requested the songs for a playlist using `GET /playlists/<playlist_id>/songs`.
* Compared the returned songs with the playlist contents.
* Observed that the final song was consistently missing from the response.

### How I found the root cause

I traced the request from `routes/playlists.py` to `get_playlist_songs()` in `services/playlist_service.py`. The SQL query correctly retrieved every song in the playlist and ordered them by position. However, the final return statement sliced the list using `songs[:-1]`, which removes the last element before returning the response. Since the query itself was correct, this confirmed the bug was caused by the final list slicing operation.

### The root cause

The function intentionally returned `songs[:-1]`, which excludes the final element of the list. As a result, every playlist response omitted the last song even though it had been retrieved correctly from the database.

### My fix and side-effect check

I removed the unnecessary slice so the function returns the complete list of songs.

Changed:

```python
return [song.to_dict() for song in songs[:-1]]
```

to:

```python
return [song.to_dict() for song in songs]
```

After the change, I verified that:

* every song in the playlist is returned,
* playlist ordering is preserved,
* playlists with one song still return that song correctly,
* empty playlists continue to return an empty list.

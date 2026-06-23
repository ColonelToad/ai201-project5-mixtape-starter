# Mixtape Bug Hunt Submission

## AI Usage

During this project, I collaborated with an AI assistant to accelerate my understanding of the codebase and trace execution flows.

* **Orientation & Mapping:** I provided the AI with `models.py` and asked it to summarize the relationships and architectural patterns. This helped me quickly grasp concepts like the dynamic calculation of ratings and the use of rich join tables (like `playlist_entries`). I also used it to trace the data flow for the `/rate` endpoint across `routes/songs.py` and `services/notification_service.py`.
* **Understanding Root Causes:** After locating suspicious code, I used the AI to clarify specific Python and SQLAlchemy behaviors. For example, I asked it why `[:-1]` drops the last item in a list, and how SQL handles 1-to-many `.outerjoin()` queries without a `.distinct()` clause.
* **Verification:** I did not rely on the AI to blindly guess the bugs; I first reproduced them locally and verified the AI's explanations by reading the service files and testing the API endpoints myself to ensure the data conditions matched reality.

---

## Codebase Map

### App Structure & Main Files

* **`app.py`**: The application factory. It initializes the Flask app and configures the SQLAlchemy database connection.
* **`models.py`**: Defines the database schema. It dictates the shape of the data and the exact relationships (one-to-many, many-to-many) between all entities.
* **`routes/` (e.g., `songs.py`, `playlists.py`)**: The API gatekeepers. These files handle incoming HTTP requests, parse JSON payloads or URL parameters, and return formatted responses. They strictly delegate complex work to the services layer.
* **`services/` (e.g., `notification_service.py`, `streak_service.py`)**: The brain of the application. These files contain the actual business logic, handle database transactions, and enforce the rules of the app.

### Core Data Models (`models.py`)

| Model | Primary Responsibility | Key Characteristics |
| --- | --- | --- |
| **User** | Manages user profiles and state. | Tracks `listening_streak` and `last_listened_at`. Uses a self-referential many-to-many join table for `friendships`. |
| **Song** | Represents a shared track. | Stores metadata (title, artist) and links to the `User` who shared it (`shared_by`). |
| **ListeningEvent** | Acts as an activity log. | Records exactly when a specific user listened to a specific song. |
| **Rating** | Tracks user sentiment on songs. | Stores a 1–5 score. Has a unique constraint ensuring a user can only rate a specific song once. |
| **Playlist** | Manages collections of songs. | Belongs to a creator but has an `is_collaborative` flag to allow others to contribute. |
| **Notification** | Manages user alerts. | Stores a `notification_type`, the message body, and a `read` boolean status. |
| **Tag** | Categorizes songs. | Simple text label (e.g., "chill", "workout"). |

### Architectural Patterns

* **Strict Separation of Concerns:** Routes *only* handle HTTP/web concerns (parsing inputs, returning 200/400/404 statuses). Business logic and database commits are strictly isolated within the `services/` directory.
* **Dynamic Calculation:** The app generally avoids storing derived data. For instance, a song's average rating isn't a column on the `Song` model; it relies on the `Rating` table.
* **Rich Join Tables:** The many-to-many relationships are not always simple links. The `playlist_entries` table connecting Playlists and Songs includes extra data like `position` (for explicit ordering) and `added_by`.

### Data Flow Trace: Rating a Song (`POST /songs/<song_id>/rate`)

1. **The Request:** A client sends a `POST` request to `/songs/<song_id>/rate` with a JSON payload containing a `user_id` and a `score`.
2. **The Route (`routes/songs.py`):** The `rate(song_id)` function extracts the payload, validates that the required fields exist, and delegates the work by calling `rate_song(user_id, song_id, score)`.
3. **The Service (`services/notification_service.py`):** The `rate_song()` function enforces business rules (score must be 1-5). It queries the database to check if the user has already rated this song. If they have, it updates the existing score. If not, it creates a new `Rating` object.
4. **The Database:** `db.session.commit()` is called, persisting the transaction to the database.
5. **The Response:** The service returns the rating object to the route, which formats it via `rating.to_dict()` and sends it back to the client with an HTTP `201 Created` status.

---

## Root Cause Analyses

### Issue #1: My listening streak keeps resetting

* **How you reproduced it:** I opened a Python REPL to simulate listening events. I forced a user's `last_listened_at` date to be a Saturday. I then triggered a new listening event for that same user on a Sunday. Instead of the streak incrementing, it reset to 1.
* **How you found the root cause:** I traced the `/listen` route to `record_listening_event` in `streak_service.py`, which then delegates to `update_listening_streak`. I read through the conditional logic that determines whether a streak increments, holds, or resets. The specific line checking for `days_since_last == 1` caught my eye because it had a secondary date condition attached to it.
* **The root cause:** The streak increment logic explicitly included the condition `and today.weekday() != 6`. In Python's `datetime` module, `weekday()` returns 6 for Sunday. This meant that if a user listened to a song on Sunday (exactly 1 day after Saturday), the increment condition failed, fell through to the `else` block, and incorrectly reset the streak to 1 instead of continuing it.
* **Your fix and side-effect check:** I removed the `and today.weekday() != 6` condition entirely, leaving just `elif days_since_last == 1:`. I verified the fix by simulating a Saturday-to-Sunday listening transition and confirming the streak correctly incremented. I also simulated a Sunday-to-Monday transition to ensure no other day boundaries were negatively affected.

### Issue #2: Friends Listening Now shows people from yesterday

* **How you reproduced it:** I checked the "Friends Listening Now" feed via the API. The response included friends who had a `listened_at` timestamp from over 20 hours ago, making the feed look like a daily history log rather than a live "now" indicator.
* **How you found the root cause:** I looked at `get_friends_listening_now` in `services/feed_service.py`. The function calculates a `cutoff` time by subtracting a `RECENT_THRESHOLD` variable from the current time. I scrolled to the top of the file to see how that threshold was defined, and the issue was immediately apparent.
* **The root cause:** `RECENT_THRESHOLD` was hardcoded to `timedelta(hours=24)`. Because the database query grabs any listening events that happened *after* the cutoff, defining the cutoff as 24 hours ago mathematically forced yesterday's listening activity into a feed that is supposed to represent real-time or very recent activity.
* **Your fix and side-effect check:** I changed `RECENT_THRESHOLD` from `timedelta(hours=24)` to `timedelta(hours=1)` to better reflect the concept of "listening now." I verified the fix by checking the feed and confirming that events from 2+ hours ago were successfully filtered out. I also explicitly checked the `get_activity_feed` function in the same file to ensure it didn't accidentally break; it remained unaffected because it relies on a hard row limit (`limit=20`) rather than the recency threshold.

### Issue #3: The same song keeps showing up twice in search

* **How you reproduced it:** I used the `/search?q=` endpoint in my browser, searching for a common vowel like "a". In the JSON response, I found multiple identical song objects (sharing the exact same UUID) populating the `results` array.
* **How you found the root cause:** I navigated to `search_songs` in `services/search_service.py`. Since the project brief hinted this was conditional, I looked closely at the SQLAlchemy query. The query uses an `.outerjoin` on `song_tags`. Joining a 1-to-many relationship without grouping will return a row for every match on the "many" side.
* **The root cause:** The SQLAlchemy query was joining the `Song` table with the `song_tags` join table. If a song matched the search query and had three tags (e.g., "pop", "workout", "upbeat"), the database returned three identical song rows. Because the Python code just iterated over the raw results, it appended the same song to the final list three times.
* **Your fix and side-effect check:** I chained `.distinct()` to the SQLAlchemy query immediately before `.all()`. This commands the database to filter out duplicate `Song` records before handing the result set back to Python. I verified the fix by re-running my original search query and confirming that each unique song ID only appeared exactly once in the response. I also checked that songs with *zero* tags still appeared in the search results.

### Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

* **How you reproduced it:** I sent a POST request to `/songs/<song_id>/rate` using a `user_id` rating a song shared by a different user. I then checked the database for the original sharer and verified that no `Notification` record was generated for the rating event.
* **How you found the root cause:** I traced the data flow starting from `routes/songs.py`, which led me to the `rate_song` function in `services/notification_service.py`. I compared the implementation of `rate_song` side-by-side with `add_to_playlist` (which was successfully generating notifications). I realized the rating function was completely missing the notification logic block.
* **The root cause:** The `rate_song` function correctly saved the rating to the database but lacked the architectural step to call `create_notification()`. It simply committed the rating and returned, whereas other interaction functions explicitly built and saved a notification object for the original sharer.
* **Your fix and side-effect check:** I added an `if` block at the end of `rate_song` (after the commit) that checks if the rater is different from the `shared_by` user. If so, it calls `create_notification` with a "song_rated" type. I verified the fix by rating a song and confirming the notification appeared in the database, and checked that a user rating their *own* shared song does not trigger a spam notification.

### Issue #5: The last song in a playlist never shows up

* **How you reproduced it:** I fetched the songs for an existing playlist using the `GET /playlists/<id>/songs` endpoint. I compared the JSON response array length to the actual number of rows tied to that playlist in the `playlist_entries` join table in the database. The API consistently returned one fewer song than the database actually held.
* **How you found the root cause:** I looked at `routes/playlists.py`, which pointed me to `get_playlist_songs` in `services/playlist_service.py`. I checked the SQLAlchemy query, which correctly queried and ordered all the songs. Looking at the final return statement, the error was obvious.
* **The root cause:** The function was deliberately truncating the results before returning them. The return statement applied a Python list slice `[:-1]` to the `songs` list. In Python, this slice means "all elements from the start up to, but not including, the final element," guaranteeing the last song would always be dropped from the response.
* **Your fix and side-effect check:** I removed the `[:-1]` slice from the return statement, so it simply returns `[song.to_dict() for song in songs]`. I verified the fix by fetching a playlist and confirming the total count matched the database. I also tested it on an empty playlist to ensure the list comprehension didn't throw an index error when returning an empty array.
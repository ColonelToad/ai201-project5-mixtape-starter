# Mixtape Codebase Map

## App Structure & Main Files

* **`app.py`**: The application factory. It initializes the Flask app and configures the SQLAlchemy database connection.
* **`models.py`**: Defines the database schema. It dictates the shape of the data and the exact relationships (one-to-many, many-to-many) between all entities.
* **`routes/` (e.g., `songs.py`, `playlists.py`)**: The API gatekeepers. These files handle incoming HTTP requests, parse JSON payloads or URL parameters, and return formatted responses. They strictly delegate complex work to the services layer.
* **`services/` (e.g., `notification_service.py`, `streak_service.py`)**: The brain of the application. These files contain the actual business logic, handle database transactions, and enforce the rules of the app.

## Core Data Models (`models.py`)

| Model | Primary Responsibility | Key Characteristics |
| --- | --- | --- |
| **User** | Manages user profiles and state. | Tracks `listening_streak` and `last_listened_at`. Uses a self-referential many-to-many join table for `friendships`. |
| **Song** | Represents a shared track. | Stores metadata (title, artist) and links to the `User` who shared it (`shared_by`). |
| **ListeningEvent** | Acts as an activity log. | Records exactly when a specific user listened to a specific song. |
| **Rating** | Tracks user sentiment on songs. | Stores a 1–5 score. Has a unique constraint ensuring a user can only rate a specific song once. |
| **Playlist** | Manages collections of songs. | Belongs to a creator but has an `is_collaborative` flag to allow others to contribute. |
| **Notification** | Manages user alerts. | Stores a `notification_type`, the message body, and a `read` boolean status. |
| **Tag** | Categorizes songs. | Simple text label (e.g., "chill", "workout"). |

## Architectural Patterns

* **Strict Separation of Concerns:** Routes *only* handle HTTP/web concerns (parsing inputs, returning 200/400/404 statuses). Business logic and database commits are strictly isolated within the `services/` directory.
* **Dynamic Calculation:** The app generally avoids storing derived data. For instance, a song's average rating isn't a column on the `Song` model; it relies on the `Rating` table.
* **Rich Join Tables:** The many-to-many relationships are not always simple links. The `playlist_entries` table connecting Playlists and Songs includes extra data like `position` (for explicit ordering) and `added_by`.

## Data Flow Trace: Rating a Song (`POST /songs/<song_id>/rate`)

1. **The Request:** A client sends a `POST` request to `/songs/<song_id>/rate` with a JSON payload containing a `user_id` and a `score`.
2. **The Route (`routes/songs.py`):** The `rate(song_id)` function extracts the payload, validates that the required fields exist, and delegates the work by calling `rate_song(user_id, song_id, score)`.
3. **The Service (`services/notification_service.py`):** The `rate_song()` function enforces business rules (score must be 1-5). It queries the database to check if the user has already rated this song. If they have, it updates the existing score. If not, it creates a new `Rating` object.
4. **The Database:** `db.session.commit()` is called, persisting the transaction to the database.
5. **The Response:** The service returns the rating object to the route, which formats it via `rating.to_dict()` and sends it back to the client with an HTTP `201 Created` status.
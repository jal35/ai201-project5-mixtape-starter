# AI Usage Log

**Tool Used:** Gemini / Claude (Learning Mode)

*   **Issue #4 (Notifications):** I used AI to help implement the fix by writing a descriptive comment explaining what should happen right after the database commit (`db.commit()`), and allowed the AI to generate the specific notification syntax.
*   **Issue #1 (Streaks) & Issue #5 (Playlists):** I used the AI in a guided learning/tutor capacity to understand how specific data structures and methods work (such as Python's day-of-the-week indexing and negative list slicing). Instead of being handed a direct solution, we engaged in a back-and-forth dialogue where I asked questions, tested assumptions, and reasoned my way to the correct fixes.
*   **Verification:** I verified all generated logic and explanations manually by running the project test suite (`pytest tests/`) to ensure the changes fixed the root causes without creating side effects.

# Codebase Map

## File Architecture & Responsibilities

| File / Directory | Main Responsibility |
|---|---|
| **`app.py`** | Acts as the Flask application factory and initializes the database setup. |
| **`models.py`** | Defines all SQLAlchemy database models (`User`, `Song`, `Playlist`, and the `playlist_entries` association table). |
| **`routes/`** | Contains the endpoint routers (like `songs.py` and `playlists.py`). These parse incoming HTTP requests and format the JSON responses. |
| **`services/`** | House the core business logic of the app, ensuring data formatting, validation, and calculations happen away from the routes. |

---

## Data Flow Trace: Rating a Song & Triggering a Notification

When a user rates a track, data flows through the application layers step-by-step:

1. **Request Entry:** A client sends a `POST` request to `/songs/<song_id>/rate` with a payload containing a numeric score.
2. **Routing Layer:** `routes/songs.py` captures the request, extracts the `song_id` and score, and applies the rating directly to the `Song` model record.
3. **Service Orchestration:** Right after updating the song data, the route calls `notification_service.notify_song_rated()`, passing along the `song_id` and the database transaction state.
4. **Data Persistence:** The service uses the `Song` relationship to identify who originally shared the track and creates a new `Notification` row in the database before committing the transaction.

## Root Cause Analysis (RCA)

### Issue #4 — Missing rating notifications
- We ran the automated test suite using `pytest tests/`, which highlighted a failure in the notification tracking system. Specifically, when a rating action occurred, the expected notification record was missing from the user's notification list.
- We navigated to `services/notification_service.py` to inspect the rating handling function. By comparing it to the functioning playlist notification logic, we observed that while the database transaction successfully saved the rating, it omitted the code block needed to instantiate and add a new `Notification` model instance to the session.
- The application architecture expects a `Notification` entry to be explicitly created and added to `db.session` whenever an event occurs that a friend needs to see. The rating service successfully updated the song data but completely lacked the lines of code required to generate a notification payload for the song's original creator.
- We implemented the missing logic directly after the data update, explicitly building a new `Notification` row with the correct user IDs and committing it to the database. We verified the fix by running `pytest tests/`, ensuring the notification tests passed perfectly without breaking any existing song rating tracking.

### Issue #1 — My listening streak keeps resetting
- We ran the test suite using `pytest tests/`. The streak testing environment simulated a user listening to a song on a Saturday and then listening again on Sunday morning. Instead of the streak incrementing from 12 to 13, the test failed because the streak reset to 1.
- We opened `services/streak_service.py` to examine how daily streaks are calculated. We located the conditional block that updates the streak based on the number of days since the user's last listen.
- The root cause:** Inside the `elif` statement that checked if a single day had passed (`days_since_last == 1`), the code included an extra conditional check: `today.weekday() == 6` (Python's representation for Sunday). This logic mistakenly assumed special week-boundary rules applied to Sundays, preventing the streak from incrementing normally and forcing a reset to 1 even when the user listened on consecutive days.
- We simplified the condition to solely check if exactly one day had passed (`elif days_since_last == 1:`), removing the weekday restriction entirely. We verified the fix by running `pytest tests/`, confirming that the streak tests now pass perfectly across all days of the week.

### Issue #5 — The last song in a playlist never shows up
- We ran the automated test suite using `pytest tests/`. The tests verified that when retrieving a playlist containing a specific number of tracks, the total count returned was short by exactly one song, consistently dropping the most recently added track.
- We navigated to `services/playlist_service.py` and inspected the `get_playlist_songs` function. We looked closely at the return statement at the very end of the function to see how the list of songs was being processed before being returned.
- The return statement used a Python list slice syntax: `[song.to_dict() for song in songs[:-1]]`. The negative index slice `[:-1]` explicitly instructs Python to stop before the very last item in the list, causing the final track of every playlist to be omitted from the results.
- We removed the slice modifier entirely, updating the loop to iterate through the entire `songs` variable directly. We ran `pytest tests/` to verify that all 13 tests now pass successfully and playlists return their complete track listings.
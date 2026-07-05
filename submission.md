## Code Base Map

### `app.py`

The app.py is where the app lives and all the routes / configurations are initially registered upon start up. The app is also running in debug mode.


--- 
### `model.py`

The `model.py` file serves as the core schema definition, managing the relationships between the 6 primary models through 3 association tables.

#### Association Tables

| Table | Description |
| --- | --- |
| **`friendships`** | A many-to-many join table enabling the self-referential "friends" feature for users. |
| **`song_tags`** | Maps multiple tags to a single song (and vice versa). |
| **`playlist_entries`** | An association object linking `Playlist` to `Song` with metadata (`position`, `added_by`, `add_at`). |

#### Data Models

| Model | Relationships | Description |
| --- | --- | --- |
| **`User`** | Relationships with all models (except `Tag`). Self-referential friendship relationship configured with `lazy='dynamic'`. | The individual users of the mixtape app. |
| **`Song`** | Relationships with `Rating` and `Tag`. Uses `lazy='subquery'` for tags to prevent N+1 performance issues. | The individual songs available in the app. |
| **`Playlist`** | Associated with `Song` via the `playlist_entries` bridge table. | Playlists with support for custom ordering via `position`. |
| **`ListeningEvent`** | Standalone model. | Records user listening history (e.g., *User X listened to Song Y at Time Z*). |
| **`Tag`** | Standalone model. | Categorization tags applied to songs. |
| **`Rating`** | Standalone model. | User ratings can only be from 1-5 inclusive. Enforces a unique constraint on `(user_id, song_id)` to ensure one rating per song. |
| **`Notification`** | Standalone model. | User alerts with a body and a `read` state (defaulting to `False`). |

#### Data Flow

A user creates a playlist -> `POST /playlists/` in `routes/playlists.py` which calls `create.create_playlist()`. That functions creates a playlists record which has the playlist creator, name, and whether other people can add to the playlist or not.

#### Pattern

I noticed that the all most endpoints don't just use the default `/` the only that does is `/playlists/`. Which is a interesting decision.

#### Bug Report / Root Cause Analysis

#### Bug 1: My listening streak keeps resetting

**App state to reproduce**
1. The app will need to record streaks for a user on saturday
2. The app will need to record streaks for a user on sunday
3. Observe that the user's streak was reset back to 1

**How the bug was caught**
- Unit Test.

**Root Cause Analysis**
```
elif days_since_last == 1 and today.weekday() != 6: # if the day is sunday then fall through to the else and reset it the user streak to 1
    user.listening_streak += 1
else:
    user.listening_streak = 1
```
```
elif days_since_last == 1: # fix remove the sunday check and simple increment no matter what day of the week it is.
    user.listening_streak += 1
else:
    user.listening_streak = 1
```

----

#### Bug 2: Friends Listening Now shows people from yesterday

**Steps to reproduce**
1. Send a this curl request `curl http://127.0.0.1:5000/feed/988b8515-a0fd-4435-b121-e4c87c0b2800/listening-now` for test user `nova` *note* `988b8515-a0fd-4435-b121-e4c87c0b2800` is the test user_id.
2. Observe that `last_listened_at` is both null and includes yesterday. 

**Hypothesis**
- The app is not updating both `listened_at` and `last_listen_at`. In addition, to not filtering out users who have `last_listened_at` as null and is not the current date.

**How the bug was caught**
- Reported by user / support.

---

#### Bug 3: The last song in a playlist never shows up

**Steps to reproduce**
1. Create a playlist with X number of songs
2. Send a `GET` Request to `/playlists/{playlist_id}/song`
3. Observe that the # of songs returns is X-1 instead of X. 

**How the bug was caught**
1. Unit Test.
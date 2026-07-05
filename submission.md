## AI USAGE

I asked Claude to explain some suspicious-looking functions. For example, I asked it to confirm my suspicion that the `feed_service` was not properly filtering for active listening users based on their `last_listened_at` timestamp. Claude was helpful, confirmed my suspicion, and proposed a solution. I then reviewed its suggestion and implemented it. For the most part claude was able to help me identify suspicious looking code and identify the root cause which I then reviewed and pushed back until I got to a solution or root cause that I believe to be true.

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

Unit test failed. I investigated by tracing through `services/streak_service.py` to understand the streak logic. In the `update_listening_streak()` function (lines 70-76), I found the problem: the condition `today.weekday() != 6` was explicitly preventing streak increments on Sundays. Any listen on a Sunday would fall through to the `else` block and reset the streak to 1, even if the user had listened on Saturday. The fix was straightforward: remove the weekday check so the streak increments regardless of the day of the week.

**Root Cause Analysis**
The problem was that the logic wrongly reset user streaks when they record a streak on sunday. i.e. record on a saturday so that the days_since_last is 1 and record on sunday days_since_last is = to 1 but not the weekday is 6 so it skips and resets to 1.
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

- ran the unit test again after to ensure existing test didn't break.

----

#### Bug 2: Friends Listening Now shows people from yesterday

**Steps to reproduce**
1. Send a this curl request `curl http://127.0.0.1:5000/feed/988b8515-a0fd-4435-b121-e4c87c0b2800/listening-now` for test user `nova` *note* `988b8515-a0fd-4435-b121-e4c87c0b2800` is the test user_id.
2. Observe that `last_listened_at` is both null and includes yesterday. 

**How the bug was caught**

User reported that the Friends Listening Now endpoint was returning stale data from yesterday. I started by examining `services/feed_service.py` to understand the filtering logic in `get_friends_listening_now()` (lines 16-62). I noticed the function was retrieving a list of friend IDs but had no filter for `last_listened_at` — it only checked if listening events existed within the past 24 hours. I then checked `models.py` to confirm that the User model had a `last_listened_at` field (line 46). The issue was clear: the field existed but wasn't being used. I added a filter on line 33 to only include friends where `last_listened_at` was not null and fell within the 24-hour recency window, ensuring stale friends wouldn't appear in the feed.

**Root Cause Analysis**
The problem was that the logic didn't include any filter for last_listened_at inside of the feed_service.get_friends_listening_now
```
# The code didn't account for user activity instead just relied on listening event. Which is not as accurate as last_listened_at.
friend_ids = [f.id for f in user.friends]
```
```
# Add a filter to the friend_ids list so that only friends with non null and >= cutoff time are included in the list.
friend_ids = [f.id for f in user.friends if f.last_listened_at and f.last_listened_at.replace(tzinfo=timezone.utc) >= cutoff]
```

- ran the unit test again after to ensure existing test didn't break.

---

#### Bug 3: The last song in a playlist never shows up

**Steps to reproduce**
1. Create a playlist with X number of songs
2. Send a `GET` Request to `/playlists/{playlist_id}/song`
3. Observe that the # of songs returns is X-1 instead of X. 

**How the bug was caught**

Unit tests revealed that playlists were returning fewer songs than expected. I traced through `services/playlist_service.py` to find the `get_songs()` function. At the return statement (line ~118), I found the problematic slice: `return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice was explicitly removing the last song from every playlist. This was clearly unintentional. I removed the slice so all songs are returned, and verified the fix by re-running the tests.

**Root Cause Analysis**
The problem was in playlist_service at the return statement at the end.
```
# removed the last song
return [song.to_dict() for song in songs[:-1]]
```
```
# removed the slice [:-1] so we will now return all songs
return [song.to_dict() for song in songs]
```

- ran the unit test this time all test pass where previous attempts there were still 2 failing test related to the playlist.

![screen shot of git log](/Screenshot%202026-07-05%20at%2017.36.02.png)
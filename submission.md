## AI USAGE

1. I asked Claude to explain some suspicious-looking functions. For example, I asked it to confirm my suspicion that the `feed_service` was not properly filtering for active listening users based on their `last_listened_at` timestamp. Claude was helpful, confirmed my suspicion, and proposed a solution. I then reviewed its suggestion and implemented it. For the most part claude was able to help me identify suspicious looking code and identify the root cause which I then reviewed and pushed back until I got to a solution or root cause that I believe to be true.

2. I also asked claude to help me identify the data path and help me build the correct test curl commands to reproduce the bugs. One specific example was when claude helped me build the query to fix the bug where users would see wrong listening now friend results. I was able to identify from the response that null and expired last_listening_at fields were the main source of the problem.

## Code Base Map

### Project Structure

```
mixtape/
├── app.py                 # Flask application initialization and route registration
├── models.py              # SQLAlchemy ORM models and association tables
├── services/              # Business logic for streaks, feeds, playlists, and search
│   ├── streak_service.py
│   ├── feed_service.py
│   ├── playlist_service.py
│   └── search_service.py
├── routes/                # API endpoints organized by resource
│   ├── playlists.py
│   ├── songs.py
│   ├── users.py
│   ├── ratings.py
│   ├── notifications.py
│   └── search.py
├── instance/              # Application instance (database)
└── tests/                 # Unit tests
```

### Core Components

#### `app.py`
Flask application entry point. Initializes the database, registers blueprints for all routes, and enables debug mode. All route configurations are registered at startup.

#### `models.py`
Defines 7 SQLAlchemy ORM models with 3 association tables managing many-to-many relationships.

**Association Tables:**

| Table | Purpose |
| --- | --- |
| **`friendships`** | Self-referential many-to-many linking users as friends |
| **`song_tags`** | Maps songs to tags for categorization |
| **`playlist_entries`** | Links playlists to songs with ordering metadata (`position`, `added_by`, `added_at`) |

**Data Models:**

| Model | Key Fields | Relationships | Purpose |
| --- | --- | --- | --- |
| **`User`** | `id`, `username`, `email`, `listening_streak`, `last_listened_at`, `created_at` | Friends (self-ref, `lazy='dynamic'`), songs, ratings, events, notifications, playlists | Individual app users with streak tracking |
| **`Song`** | `id`, `title`, `artist`, `album`, `genre`, `shared_by`, `shared_at` | Ratings, listening events, tags (`lazy='subquery'`) | Songs shared in the app |
| **`Playlist`** | `id`, `name`, `created_by`, `created_at`, `is_collaborative` | Songs via `playlist_entries`, creator (User) | User-created collections with custom ordering |
| **`ListeningEvent`** | `id`, `user_id`, `song_id`, `listened_at` | User, Song | Tracks when users listen to songs |
| **`Rating`** | `id`, `user_id`, `song_id`, `score`, `rated_at` | User, Song | 1-5 score per user/song with uniqueness constraint |
| **`Tag`** | `id`, `name` | Songs (many-to-many) | Categorization labels for songs |
| **`Notification`** | `id`, `user_id`, `notification_type`, `body`, `created_at`, `read` | User | User alerts with read state |

#### `services/` Directory
Contains business logic isolated from routes:
- **`streak_service.py`**: Manages listening streaks and listening event recording
- **`feed_service.py`**: Handles "Friends Listening Now" (recent friends) and activity feeds
- **`playlist_service.py`**: Playlist creation, song addition/removal, and retrieval
- **`search_service.py`**: Song search by title and artist

#### `routes/` Directory
API endpoints organized by resource:
- Users, songs, ratings, playlists, notifications, and search endpoints
- Each route typically calls a corresponding service function
- Most endpoints use resource-specific paths (e.g., `/playlists/`, `/songs/`) rather than a root `/` pattern

### Data Flow Example

User creates a playlist: `POST /playlists/` → `routes/playlists.py` → `services/create.create_playlist()` → Creates `Playlist` record with creator, name, and collaboration settings.

---

## Bug Report / Root Cause Analysis

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

**Side-Effect Check**

Verified that removing the Sunday check doesn't break other streak logic:
- Tested that streaks properly reset when `days_since_last > 1` (skipping 2+ days): Confirmed the `else` branch still resets to 1
- Tested that consecutive days still increment: Confirmed `days_since_last == 1` logic is unaffected
- Examined `record_listening_event()` in streak_service.py to ensure it still calls `update_listening_streak()` correctly

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

**Side-Effect Check**

Verified that the new filter doesn't break other feed functionality:
- Checked `get_activity_feed()` in feed_service.py (lines 65-105) to ensure it's unaffected — it doesn't use the `last_listened_at` filter, so it still returns all friend listening events regardless of recency
- Traced through the `User.friends` relationship to confirm the filter correctly handles null `last_listened_at` values without crashing
- Verified that timezone-naive datetimes in the database are properly converted with `.replace(tzinfo=timezone.utc)` before comparison with the UTC cutoff
- Examined the original listening event filter (`ListeningEvent.listened_at >= cutoff`) to ensure it still works alongside the new user-level filter

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

**Side-Effect Check**

Verified that removing the slice doesn't break edge cases or related functionality:
- Checked behavior with single-song playlists: A playlist with 1 song now correctly returns that 1 song (previously would have returned 0)
- Verified empty playlists still work: An empty playlist correctly returns an empty list (the slice `songs[:-1]` on an empty list returns an empty list, as does `songs`)
- Confirmed that playlist ordering via the `position` field is maintained: No changes to the sort order, only removing the truncation
- Ran the unit test and confirmed all 2 previously failing tests related to playlists now pass.

![screen shot of git log](/Screenshot%202026-07-05%20at%2017.36.02.png)
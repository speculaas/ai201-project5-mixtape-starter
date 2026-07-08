# Mixtape Bug Hunt — Submission

> **Status:** Pre-fix observations only. Root cause analyses and fix commits go below after each bug is fixed.

---

## AI Usage

*(Fill in after project — describe how you used AI for orientation, explanation, and debugging. Note where you verified or overrode AI output.)*

- Used Cursor to:
- Verified myself:

---

## Codebase Map (written before bug fixes)

### Main files

| File / directory | Role |
|------------------|------|
| `app.py` | Flask app factory, DB init, blueprint registration |
| `models.py` | SQLAlchemy models (`User`, `Song`, `Playlist`, `Tag`, join tables) |
| `routes/` | HTTP layer — parses requests, calls services, returns JSON |
| `services/` | Business logic (where the five bugs live) |
| `tests/` | pytest coverage for streak, search, playlist services |
| `seed_data.py` | Populates `mixtape.db` with users, songs, playlists, events |

### Pattern

Routes are thin; services hold the logic. Example: `GET /playlists/<id>/songs` → `routes/playlists.py` → `playlist_service.get_playlist_songs()`.

### Data flow example — song added to playlist + notification

1. `POST /playlists/<playlist_id>/songs` (`routes/playlists.py`)
2. → `notification_service.add_to_playlist()`
3. → adds song to playlist; if adder ≠ sharer, `create_notification()` with type `song_added_to_playlist`

### Data flow example — listening streak

1. `POST /songs/<song_id>/listen` → `streak_service.record_listening_event()`
2. → creates `ListeningEvent`, calls `update_listening_streak(user, now)`
3. → updates `user.listening_streak` and `user.last_listened_at`

---

## Pre-Fix Observations (baseline, before any service edits)

**Captured:** 2026-07-07  
**Evidence files:** `../observations/` — regenerate with `../capture-observations.sh`  
**pytest baseline:** 3 failed, 10 passed  
**Branch:** `bugfix/mixtape` *(create before first fix)*

**Current seed IDs (2026-07-07):**

| username | id |
|----------|-----|
| nova | `07e44d55-6f84-450b-a01b-2e79ebe9a5c5` |
| darius | `4c52bbd2-5a12-4a3e-a959-9112c19b9a60` |
| Late Night Vibes playlist | `588519f8-e0d4-44c4-b5ba-7d0bf09dc95a` (7 songs in DB) |

### Chosen bugs (plan: fix at least 3)

| Issue | Title | Service | Repro method |
|-------|-------|---------|--------------|
| #1 | Listening streak keeps resetting | `streak_service.py` | pytest (Sunday case) |
| #5 | Last song in playlist never shows up | `playlist_service.py` | pytest + HTTP |
| #2 / #3 / #4 | *(pick third)* | `feed` / `search` / `notification` | HTTP + code trace |

---

### Issue #1 — Listening streak keeps resetting

#### How I reproduced it

```bash
cd ai201-project5-mixtape-starter
source .venv/bin/activate
pytest tests/test_streaks.py::test_streak_increments_on_sunday -v
```

#### Observed behavior

- After `update_listening_streak` on Saturday 2024-06-15, streak = 1 (correct).
- After listening again on Sunday 2024-06-16 (one calendar day later), streak stays **1**.
- Test failure: `assert 1 == 2`.

#### Expected behavior

- Consecutive **calendar days** should increment the streak (Saturday → Sunday → streak 2).
- Docstring in `update_listening_streak()` promises: listened yesterday → increment; same day → no change; skipped day → reset.

#### Trigger condition

- `days_since_last == 1` **and** `today.weekday() == 6` (Sunday).

#### Call chain traced

| Step | Location | Notes |
|------|----------|-------|
| 1 | `tests/test_streaks.py` | Calls `update_listening_streak` directly |
| 2 | `services/streak_service.py` → `update_listening_streak()` | Computes `days_since_last`, branches on streak rules |

#### Docstring vs code (before fix)

- **Docstring:** consecutive calendar days; yesterday → increment.
- **Code:** still reading — look for date arithmetic vs `weekday()` checks.

#### Hypothesis (before fix)

- Streak logic may treat Sunday differently from other consecutive days (weekday-based check instead of pure calendar-day difference).

---

### Issue #5 — Last song in playlist never shows up

#### How I reproduced it

**pytest:**

```bash
pytest tests/test_playlists.py -v
```

**HTTP (after `python seed_data.py`; use IDs from `observations/00-seed-ids.txt`):**

```bash
curl "http://127.0.0.1:5000/playlists/<LATE_NIGHT_VIBES_ID>/songs"
```

#### Observed behavior

- pytest: 5-song test playlist returns **4** songs; titles end at `Track 4` (missing `Track 5`).
- HTTP (seeded "Late Night Vibes", 7 songs in DB): API `"count": 6` — one fewer than stored.

#### Expected behavior

- All songs in playlist order; count matches number of `playlist_entries` rows.

#### Trigger condition

- Any playlist with ≥1 song; **last** song consistently omitted.

#### Call chain traced

| Step | Location | Notes |
|------|----------|-------|
| 1 | `GET /playlists/<id>/songs` | `routes/playlists.py` |
| 2 | `playlist_service.get_playlist_songs()` | Queries songs ordered by `position` |
| 3 | Return value | Compare row count vs length of returned list |

#### Docstring vs code (before fix)

- **Docstring:** "returns all songs in the playlist."
- **Code:** still reading — check whether return statement slices the list.

#### Hypothesis (before fix)

- Off-by-one or slice on the final element drops the last song after a correct query.

---

### Issue #3 — Same song shows up twice in search *(conditional — investigate)*

#### How I reproduced it

```bash
curl "http://127.0.0.1:5000/songs/search?q=Crown"
```

#### Observed behavior (baseline)

- `Crown Heights Anthem` (3 tags) appears **once** in API results; search unit tests **pass**.
- Bug may be conditional per project hint — needs more investigation before fix.

#### Expected behavior

- Each song appears once regardless of tag count.

#### Notes

- Inspect `search_service.py` join on `song_tags`; compare raw SQL row count vs ORM result count.

---

### Issue #2 — Friends Listening Now shows stale listeners *(to investigate)*

#### How I reproduced it

```bash
curl "http://127.0.0.1:5000/feed/<NOVA_USER_ID>/listening-now"
```

#### Observed behavior (baseline)

- Returns friends with recent `listened_at` timestamps (see `observations/02-api-repro.txt`).
- Compare each `listened_at` to 24-hour cutoff in `feed_service.RECENT_THRESHOLD`.

#### Expected behavior

- Only friends who listened within the last 24 hours.

---

### Issue #4 — Missing rating notification *(to investigate)*

#### How I reproduced it

```bash
curl "http://127.0.0.1:5000/users/<NOVA_USER_ID>/notifications"
```

#### Observed behavior (baseline)

- One notification: `song_added_to_playlist` (darius added Midnight Drive).
- No `song_rated` notification present.

#### Expected behavior

- When a friend rates your shared song, sharer receives a notification (mirror `add_to_playlist` pattern).

#### Next repro step

- `POST /songs/<song_id>/rate` as friend user; re-check notifications.

---

## Root Cause Analyses

*(Add one section per bug after fixing — all 5 fields per `projects.md`.)*

### Issue #1 — *(pending)*

### Issue #5 — *(pending)*

### Issue #3 / #2 / #4 — *(pending — pick third bug)*

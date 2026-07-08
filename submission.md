# Mixtape Bug Hunt — Submission

> **Status:** Three bugs fixed on `bugfix/mixtape`. RCAs complete. Ready for portal submit after `git log` screenshot.

---

## AI Usage

- **Cursor:** Oriented the repo, ran `capture-observations.sh`, applied fixes #1/#5/#4 on `bugfix/mixtape`, and drafted RCAs with Mermaid diagrams.
- **Microsoft Copilot (BookClub Tinker thread):** Debugging workflow — trace route → service → model, tests as contracts, diagnose before fixing, contract-mismatch diagrams.
- **Verified myself:** Ran `pytest tests/ -v` before and after (3 failed → 13 passed); HTTP repro from laptop (`05-remote-http-repro.txt`); read service files to confirm docstring-vs-code mismatches before editing; manual script to verify `song_rated` notification after fix #4.

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

### Architecture (route → service → model)

```mermaid
flowchart TD
    Client["Client / curl / pytest"] --> Routes["routes/*.py"]
    Routes --> Services["services/*.py"]
    Services --> Models["models.py"]
    Models --> DB[(SQLite mixtape.db)]
    Seed[seed_data.py] --> DB
    Tests[tests/*.py] --> Services
```

### Data flow — song added to playlist + notification

1. `POST /playlists/<playlist_id>/songs` (`routes/playlists.py`)
2. → `notification_service.add_to_playlist()`
3. → adds song to playlist; if adder ≠ sharer, `create_notification()` with type `song_added_to_playlist`

### Data flow — listening streak

1. `POST /songs/<song_id>/listen` → `streak_service.record_listening_event()`
2. → creates `ListeningEvent`, calls `update_listening_streak(user, now)`
3. → updates `user.listening_streak` and `user.last_listened_at`

Additional diagrams: [`docs/diagrams.md`](docs/diagrams.md)

---

## Evidence baseline (captured before service edits)

| Artifact | Result |
|----------|--------|
| `pytest tests/ -v` | **3 failed, 10 passed** (see `observations/01-pytest-baseline.txt`) |
| Local API (test client) | `observations/02-api-repro.txt` |
| Remote HTTP (laptop → Mac mini) | `observations/05-remote-http-repro.txt` |
| Seed IDs | `observations/00-seed-ids.txt` |

**pytest failures:** `test_streak_increments_on_sunday`, `test_playlist_returns_all_songs`, `test_playlist_returns_songs_in_order`

**Chosen bugs to fix (≥3):** #1 streak, #5 playlist, plus #4 notifications (third — clear HTTP/code comparison path).

---

## Pre-Fix Observations & Diagnosis

### Issue #1 — Listening streak keeps resetting

#### How I reproduced it

```bash
cd ai201-project5-mixtape-starter
source .venv/bin/activate
pytest tests/test_streaks.py::test_streak_increments_on_sunday -v
```

#### Observed vs expected

| | |
|---|---|
| **Observed** | Saturday listen → streak 1. Sunday listen (next calendar day) → streak stays **1**. Failure: `assert 1 == 2`. |
| **Expected** | Consecutive calendar days increment streak (Saturday → Sunday → **2**). |
| **Trigger** | `days_since_last == 1` on a **Sunday** (`today.weekday() == 6`). |

#### Call chain

```mermaid
flowchart TD
    A["POST /songs/{song_id}/listen"] --> B["routes/songs.py listen()"]
    B --> C["streak_service.record_listening_event()"]
    C --> D["create ListeningEvent"]
    C --> E["update_listening_streak(user, now)"]
    E --> F{"days_since_last = (today - last_date).days"}
    F -->|0| G["no change"]
    F -->|1 and weekday != 6| H["streak += 1"]
    F -->|else| I["streak = 1"]
    J["Sat → Sun: days_since_last = 1"] --> F
    F -->|Sunday fails weekday check| I
    I --> K["BUG: streak reset instead of increment"]
```

#### Contract vs implementation

```mermaid
flowchart LR
    A["Docstring contract:<br/>consecutive calendar days increment streak"] --> C{"Implementation honors<br/>calendar-day difference only?"}
    B["Code line 73:<br/>days_since_last == 1 AND weekday != 6"] --> C
    C -->|No| D["Sunday + yesterday listen<br/>falls through to reset branch"]
    D --> E["Diagnosis: weekday guard blocks<br/>valid consecutive-day increment"]
```

#### Pre-fix diagnosis (before editing)

| | |
|---|---|
| **Docstring says** | Yesterday → increment; same day → no change; skip → reset. Based on **calendar days**. |
| **Code does** | Computes `days_since_last` correctly, but only increments when `today.weekday() != 6`. |
| **Mismatch** | Line 73 treats Sunday as a special case incompatible with the Saturday→Sunday test. |
| **Bug location** | `services/streak_service.py` line 73 |
| **Planned fix** | Increment whenever `days_since_last == 1`; remove weekday check. |
| **Verify** | `pytest tests/test_streaks.py -v` (all 5 pass) |

---

### Issue #5 — Last song in playlist never shows up

#### How I reproduced it

**pytest:**

```bash
pytest tests/test_playlists.py -v
```

**HTTP (remote laptop → `http://10.88.191.102:5000`):**

```bash
curl "http://10.88.191.102:5000/playlists/588519f8-e0d4-44c4-b5ba-7d0bf09dc95a/songs"
```

Evidence: `observations/05-remote-http-repro.txt`

#### Observed vs expected

| | |
|---|---|
| **pytest** | 5-song playlist returns **4** songs; titles `Track 1`…`Track 4` (missing `Track 5`). |
| **HTTP** | "Late Night Vibes" has **7** songs in DB; API `"count": 6`. Last song ("Free Throws" / position 7) missing. |
| **Expected** | Count matches all `playlist_entries` rows; ordered by `position`. |
| **Trigger** | Any playlist with ≥1 song; **last** entry always dropped. |

#### Call chain

```mermaid
flowchart TD
    A["GET /playlists/{id}/songs"] --> B["routes/playlists.py get_songs()"]
    B --> C["playlist_service.get_playlist_songs()"]
    C --> D["JOIN playlist_entries ORDER BY position"]
    D --> E["songs = query.all()  → N rows"]
    E --> F["return songs[:-1]"]
    F --> G["API count = N - 1"]
    G --> H["BUG: last song missing"]
```

#### Contract vs implementation

```mermaid
flowchart LR
    A["Docstring + test contract:<br/>return ALL songs in playlist order"] --> C{"Return length == query length?"}
    B["Code line 66:<br/>songs[:-1]"] --> C
    C -->|No| D["Last element sliced off every time"]
    D --> E["Diagnosis: off-by-one in return,<br/>not in SQL query"]
```

#### Pre-fix diagnosis (before editing)

| | |
|---|---|
| **Docstring says** | Returns all songs in playlist order. |
| **Code does** | Query is correct; return uses `songs[:-1]`. |
| **Mismatch** | Line 66 excludes the final song after a successful query. |
| **Bug location** | `services/playlist_service.py` line 66 |
| **Planned fix** | Return `songs` without `[:-1]`. |
| **Verify** | `pytest tests/test_playlists.py -v`; re-curl playlist endpoint (expect count 7). |

---

### Issue #3 — Same song shows up twice in search *(not reproduced on baseline)*

#### How I reproduced it

```bash
curl "http://10.88.191.102:5000/songs/search?q=Crown"
```

#### Observed vs expected

| | |
|---|---|
| **Observed** | `Crown Heights Anthem` (3 tags) appears **once**; `"count": 1`. All `test_search.py` tests **pass**. |
| **Expected (per test comment)** | Multi-tag song should appear exactly once — bug comment says it *would* appear 3× without fix. |
| **Status** | Conditional / latent — investigate join before fixing. |

#### Fault-isolation diagram

```mermaid
flowchart TD
    A["Multi-tag song has 3 tag rows"] --> B["search_songs outerjoins song_tags"]
    B --> C["Raw SQL can return 3 rows per song"]
    C --> D{"SQLAlchemy query(Song).all()<br/>deduplicates by primary key?"}
    D -->|Yes| E["Current tests + HTTP pass"]
    D -->|No / other code path| F["Duplicate results in API"]
    E --> G["Still inspect join — unnecessary<br/>and risky if query shape changes"]
```

#### Pre-fix diagnosis

- `search_service.py` joins `song_tags` but only filters on title/artist — join may be dead code or latent duplicate source.
- **Not chosen as primary fix target** until reproduced; may fix defensively if time permits.

---

### Issue #2 — Friends Listening Now shows people from yesterday

#### How I reproduced it

```bash
curl "http://10.88.191.102:5000/feed/07e44d55-6f84-450b-a01b-2e79ebe9a5c5/listening-now"
```

Evidence: `observations/05-remote-http-repro.txt` — `"count": 3`

#### Observed vs expected

| | |
|---|---|
| **Observed** | 3 friends returned; e.g. darius `listened_at` ~20 min ago (within 24h window). |
| **Expected (issue brief)** | "Listening **Now**" should not show yesterday's listeners. |
| **Investigation** | `feed_service.RECENT_THRESHOLD = timedelta(hours=24)` may be too wide for "now". |

#### Threshold diagram

```mermaid
flowchart TD
    A["get_friends_listening_now()"] --> B["cutoff = now - 24 hours"]
    B --> C["Friend listened yesterday<br/>but within 24h"]
    C --> D["Event passes filter"]
    D --> E["Appears in Listening Now"]
    E --> F["Product mismatch:<br/>24h window ≠ 'now'"]
```

#### Pre-fix diagnosis

- Need project brief's intended window (e.g. 1 hour vs same calendar day).
- **Backup third bug** if #4 is fixed first.

---

### Issue #4 — Missing notification when friend rates song

#### How I reproduced it

```bash
curl "http://10.88.191.102:5000/users/07e44d55-6f84-450b-a01b-2e79ebe9a5c5/notifications"
```

#### Observed vs expected

| | |
|---|---|
| **Observed** | 1 notification: `song_added_to_playlist` (darius added Midnight Drive). |
| **Expected** | When a friend rates your shared song, sharer gets `song_rated` notification. |
| **Next step** | `POST /songs/<song_id>/rate` as darius; re-check notifications. |

#### Working vs broken path

```mermaid
flowchart TD
    subgraph Works
        A1["POST /playlists/.../songs"] --> B1["add_to_playlist()"]
        B1 --> C1["create_notification()"]
        C1 --> D1["song_added_to_playlist ✅"]
    end
    subgraph Broken
        A2["POST /songs/.../rate"] --> B2["rate_song()"]
        B2 --> C2["save Rating"]
        C2 --> D2["No create_notification()"]
        D2 --> E2["Missing song_rated ❌"]
    end
```

#### Pre-fix diagnosis

- Compare `rate_song()` to `add_to_playlist()` — working path notifies sharer; rating path does not.
- **Chosen as third fix** — architectural pattern mismatch, not a one-character typo.

---

## Root Cause Analyses

### Issue #1 — Listening streak keeps resetting

**1. Issue number and title:** #1 — Listening streak keeps resetting (`streak_service.py`)

**2. How I reproduced it**

```bash
pytest tests/test_streaks.py::test_streak_increments_on_sunday -v
```

Saturday 2024-06-15 → streak 1. Sunday 2024-06-16 (one calendar day later) → streak stayed 1. Failure: `assert 1 == 2`.

**3. How I found the root cause**

- Read failing test in `tests/test_streaks.py` (Sunday boundary case).
- Opened `services/streak_service.py` → `update_listening_streak()`.
- Compared docstring (consecutive **calendar days**) to branch on line 73.
- Confirmed `days_since_last` is computed correctly with `(today - last_date).days` — bug is not in date math but in the extra `weekday()` guard.

**4. The root cause**

Line 73 required `days_since_last == 1 and today.weekday() != 6`. On Sunday (`weekday() == 6`), listening after listening yesterday satisfied `days_since_last == 1` but failed the weekday check, falling through to `streak = 1` instead of incrementing. The docstring promises calendar-day streaks; the weekday check imposed an undocumented Sunday exception.

**5. Fix and side-effect check**

- **Change:** `elif days_since_last == 1:` (removed `and today.weekday() != 6`).
- **Why it works:** Any exactly-one-day gap now increments, including Saturday→Sunday.
- **Verified:** `pytest tests/test_streaks.py` — 5/5 pass. Full suite: 13/13 pass. Other streak cases (same day, skip day, consecutive weekday) unchanged.

---

### Issue #5 — Last song in playlist never shows up

**1. Issue number and title:** #5 — Last song in playlist never shows up (`playlist_service.py`)

**2. How I reproduced it**

```bash
pytest tests/test_playlists.py -v
curl "http://10.88.191.102:5000/playlists/<playlist_id>/songs"
```

pytest: 5-song playlist returned 4 (`Track 1`…`Track 4`). HTTP: "Late Night Vibes" had 7 songs in DB; API `"count": 6`.

**3. How I found the root cause**

- `tests/test_playlists.py` comment: "Bug causes this to return 4."
- Traced `GET /playlists/<id>/songs` → `playlist_service.get_playlist_songs()`.
- Query joins `playlist_entries`, orders by `position` — correct.
- Return line 66 used `songs[:-1]`, dropping the final element every time.

**4. The root cause**

`get_playlist_songs()` ran a correct ordered query but returned `songs[:-1]`, which always omits the last song. For N entries, the API returned N−1 songs. This is a display-layer slice bug, not a SQL or ordering bug.

**5. Fix and side-effect check**

- **Change:** `return [song.to_dict() for song in songs]` (removed `[:-1]`).
- **Why it works:** Return list now matches query length.
- **Verified:** `pytest tests/test_playlists.py` — 3/3 pass. Full suite: 13/13 pass. Empty playlist test still passes.

---

### Issue #4 — Missing notification when friend rates song

**1. Issue number and title:** #4 — Missing notification when friend rates song (`notification_service.py`)

**2. How I reproduced it**

```bash
# Before fix — seed data
curl "http://10.88.191.102:5000/users/<nova_id>/notifications"
# Only song_added_to_playlist present

# After identifying pattern, isolated in Python:
# rate_song(friend_id, song_id, 5) → sharer notifications stayed empty
```

Compared to working path: `add_to_playlist()` creates `song_added_to_playlist` when a friend adds your song.

**3. How I found the root cause**

- Issue brief + `GET /users/<id>/notifications` showed playlist-add notification but no rating notification.
- Compared `add_to_playlist()` (calls `create_notification`) vs `rate_song()` (saved `Rating` only).
- `routes/songs.py` → `rate_song()` — architectural omission, not a typo in notification text.

**4. The root cause**

`rate_song()` persisted the rating and returned, but never called `create_notification()` for the song's original sharer. `add_to_playlist()` already had the correct pattern: if the actor is not the sharer, notify `song.shared_by`.

**5. Fix and side-effect check**

- **Change:** After commit, if `song.shared_by != user_id`, call `create_notification(..., "song_rated", ...)`.
- **Why it works:** Mirrors the working playlist-add path; sharer learns when someone rates their song.
- **Verified:** Manual test — `rate_song` by friend creates one `song_rated` notification for sharer. Self-rating does not notify. Full `pytest tests/` — 13/13 pass (no regressions).

```mermaid
flowchart LR
    A["rate_song() saves Rating"] --> B{"rater == sharer?"}
    B -->|No| C["create_notification song_rated"]
    B -->|Yes| D["no notification"]
    C --> E["sharer sees notification"]
```

---

## Git commit history (`bugfix/mixtape`)

Course Portal has no separate screenshot upload field, so the screenshot is included in this file on the `bugfix/mixtape` branch.

### Screenshot

<!-- Save terminal screenshot as docs/git-log-bugfix-mixtape.png then commit on bugfix/mixtape -->

![git log --oneline on bugfix/mixtape](docs/git-log-bugfix-mixtape.png)

### Monospace copy (for graders / search)

```text
$ git checkout bugfix/mixtape
$ git log --oneline

f93fd50 docs: complete RCAs, git log, and demo script in submission.md
e6c2131 fix: notify song sharer when a friend rates their song
110145c fix: return all playlist songs instead of slicing off the last one
ca08670 fix: allow streak increment on Sunday after consecutive day
b1018af docs: draft pre-fix submission from observation captures
9901cc3 docs: add Mermaid diagrams for workflow and bug investigation
2bf5010 docs: add pre-fix submission draft before bug fixes
1e1215b feat: bind Flask dev server to all network interfaces
6213062 codepath ai201 unit 5 projects ; 17K 2026-07-01 01:03:51.852849000 -0600 projects.md
2dfdeaa Add .gitignore file and update README with setup instructions
7b64551 initial commit
```

**Three `fix:` commits** (one per bug): `ca08670`, `110145c`, `e6c2131`.

---

## Optional demo walkthrough script (not required for submission)

This project does **not** require a video demo. If you want to rehearse explaining your work aloud (study group, office hours, or portfolio), use this ~3-minute script:

| Step | Say | Do |
|------|-----|-----|
| 1 | "Mixtape is a Flask API — routes call services, bugs live in services/." | Open `README.md` structure section |
| 2 | "I reproduced bugs before fixing — pytest showed 3 failures." | `pytest tests/ -v` (or show `01-pytest-baseline.txt`) |
| 3 | "Issue 1: streak should increment Sat→Sun; weekday guard blocked Sunday." | `pytest tests/test_streaks.py::test_streak_increments_on_sunday -v` |
| 4 | "Issue 5: playlist query was right but return sliced off the last song." | `curl .../playlists/<id>/songs` — show count matches DB |
| 5 | "Issue 4: rating saved but never notified the sharer — I matched add_to_playlist." | `POST .../rate` then `GET .../notifications` |
| 6 | "One fix per commit on bugfix/mixtape." | `git log --oneline` |

**Is `submission.md` alone enough?** Almost — the portal also wants your **GitHub fork link** (on `bugfix/mixtape`) and the **`git log` screenshot**. Everything else (codebase map, AI usage, RCAs) lives in `submission.md` in the repo root.

---

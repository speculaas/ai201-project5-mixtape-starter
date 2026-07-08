# Mixtape — Mermaid Diagrams

Supporting diagrams for `submission.md`. GitHub renders Mermaid in Markdown previews.

> **Note:** Diagrams for Issues #1 and #5 below describe **pre-fix investigation** (symptoms and call flow). Update or add post-fix diagrams in each RCA section after you commit the fix.

---

## 1. Project workflow (what this homework asks you to do)

```mermaid
flowchart TD
    A[Read README and codebase] --> B[Write codebase map]
    B --> C[Read all 5 issues]
    C --> D[Pick at least 3 bugs]
    D --> E[Reproduce bug]
    E --> F[Trace route → service → model]
    F --> G[Compare docstring vs code]
    G --> H[Write diagnosis before editing]
    H --> I[Make targeted fix]
    I --> J[Verify fix and side effects]
    J --> K[Write RCA entry in submission.md]
    K --> L["Commit: fix: ..."]
    L --> M{Need more fixes?}
    M -- Yes --> E
    M -- No --> N[Final review and submit]
```

---

## 2. Codebase architecture (layers)

```mermaid
flowchart LR
    Client[Client / curl / pytest] --> Routes[routes/*.py]
    Routes --> Services[services/*.py]
    Services --> Models[models.py]
    Models --> DB[(SQLite mixtape.db)]
    Seed[seed_data.py] --> DB
    Tests[tests/*.py] --> Services

    subgraph Routes
        R1[users.py]
        R2[songs.py]
        R3[playlists.py]
        R4[feed.py]
    end

    subgraph Services
        S1[streak_service.py]
        S2[playlist_service.py]
        S3[search_service.py]
        S4[feed_service.py]
        S5[notification_service.py]
    end
```

---

## 3. Per-bug debugging lifecycle

```mermaid
stateDiagram-v2
    [*] --> Reproduce
    Reproduce --> Trace
    Trace --> Diagnose
    Diagnose --> Fix
    Fix --> Verify
    Verify --> DocumentRCA
    DocumentRCA --> Commit
    Commit --> [*]

    Reproduce: Trigger bug before editing code
    Trace: Follow route → service → model
    Diagnose: Docstring contract vs implementation
    Fix: Smallest targeted change
    Verify: Failing test passes; no regressions
    DocumentRCA: All 5 required fields
    Commit: One conventional fix: commit per bug
```

---

## 4. Diagnosis workflow (diagnose before guessing)

```mermaid
flowchart TD
    A[Suspicious code found] --> B{Compare to docstring contract?}
    B -->|No| C[Guess and change code]
    C --> D{Symptom gone?}
    D -->|Maybe| E[Uncertain if root cause was found]
    B -->|Yes| F[Write diagnosis in plain English]
    F --> G[Apply minimal fix]
    G --> H[Re-run repro + related tests]
    H --> I{Still wrong?}
    I -->|Yes| A
    I -->|No| J[RCA + commit]
```

---

## 5. Issue #1 — Streak (Saturday → Sunday)

**Repro:** `pytest tests/test_streaks.py::test_streak_increments_on_sunday -v`

```mermaid
flowchart TD
    A[update_listening_streak user, now] --> B[last_listened_at set?]
    B -->|No| C[streak = 1]
    B -->|Yes| D["days_since_last = (today - last_date).days"]
    D --> E{days_since_last == 0?}
    E -->|Yes| F[no change]
    E -->|No| G{days_since_last == 1 AND weekday != 6?}
    G -->|Yes| H[streak += 1]
    G -->|No| I[streak = 1]

    J[Saturday listen: streak = 1] --> K[Sunday listen: days_since_last = 1]
    K --> G
    G -->|Sunday fails weekday check| I
    I --> L[BUG: streak stays 1 instead of 2]

    style L fill:#f9d,stroke:#333
```

**Contract (docstring):** consecutive **calendar days** increment the streak.  
**Investigation focus:** `weekday()` is weekday-based, not calendar-day based.

---

## 6. Issue #5 — Playlist missing last song

**Repro:** `pytest tests/test_playlists.py -v` or `GET /playlists/<id>/songs`

```mermaid
flowchart TD
    A[get_playlist_songs playlist_id] --> B[Query Song JOIN playlist_entries]
    B --> C[ORDER BY position ASC]
    C --> D["songs = query.all()  → N rows"]
    D --> E["return songs[:-1]"]
    E --> F[API count = N - 1]
    F --> G[BUG: last song always dropped]

    H[Example: 5 songs in DB] --> D
    F --> I[Returns 4 songs: Track 1..4]

    style G fill:#f9d,stroke:#333
```

**Contract (docstring):** returns **all** songs in playlist order.  
**Investigation focus:** compare query row count vs returned list length.

---

## 7. Issue #3 — Search duplicates (conditional)

**Repro:** `GET /songs/search?q=...` — unit tests may pass; bug can be conditional.

```mermaid
flowchart TD
    A[search_songs query] --> B[query Song]
    B --> C[OUTER JOIN song_tags]
    C --> D[FILTER title OR artist ILIKE]
    D --> E{Song has 3 tags?}
    E -->|Yes| F[SQL can emit 3 rows for 1 song]
    E -->|No| G[1 row per song]
    F --> H{ORM deduplicates by Song.id?}
    H -->|Sometimes| I[API shows 1 result — tests pass]
    H -->|No / other path| J[BUG: duplicate songs in results]

    style J fill:#ff9,stroke:#333
```

**Investigation focus:** raw SQL row count vs `query(Song).all()` length vs JSON `count`.

---

## 8. Issue #2 — Friends Listening Now (stale events)

```mermaid
sequenceDiagram
    participant C as curl
    participant R as routes/feed.py
    participant F as feed_service
    participant DB as ListeningEvent

    C->>R: GET /feed/{user_id}/listening-now
    R->>F: get_friends_listening_now(user_id)
    F->>F: cutoff = now - RECENT_THRESHOLD
    F->>DB: friend events where listened_at >= cutoff
    DB-->>F: events (check timestamps vs 24h window)
    F-->>R: deduped feed list
    R-->>C: JSON feed + count
```

**Investigation focus:** `RECENT_THRESHOLD` and whether old events pass the cutoff filter.

---

## 9. Issue #4 — Missing rating notification

```mermaid
flowchart LR
    subgraph Works
        A1[POST playlist add song] --> B1[add_to_playlist]
        B1 --> C1[create_notification song_added_to_playlist]
    end

    subgraph Missing
        A2[POST rate song] --> B2[rate_song]
        B2 --> C2[save Rating]
        C2 --> D2[No notification created]
    end

    style D2 fill:#f9d,stroke:#333
```

**Investigation focus:** compare `rate_song()` to the working `add_to_playlist()` notification pattern.

---

## 10. RCA entry structure (required fields)

```mermaid
mindmap
  root((RCA Entry))
    Issue number and title
    How you reproduced it
      Commands
      Inputs
      Observed vs expected
    How you found root cause
      Files traced
      Docstring vs code
    The root cause
      Exact wrong condition
      Why user saw the bug
    Fix and side-effect check
      What changed
      Tests re-run
      Related behavior checked
```

---

## 11. Git commit model (one fix per commit)

```mermaid
gitGraph
    commit id: "starter"
    branch bugfix/mixtape
    checkout bugfix/mixtape
    commit id: "docs: submission baseline"
    commit id: "fix: streak Sunday boundary"
    commit id: "fix: playlist off-by-one"
    commit id: "fix: third bug"
```

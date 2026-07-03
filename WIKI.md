# hermes-dat — Discord Activity Tracker

Plugin for tracking Discord presence via the Lanyard REST API. Polls presence data, stores daily JSONL logs, and exposes a `discord_activity` tool for querying status, sessions, stats, Spotify listening, and raw history.

**Plugin name:** `hermes-discord-activity-tracker`
**Location:** `~/.hermes/plugins/hermes-dat/` (symlinked from profile)
**Version:** 1.0.0
**Author:** Aura

---

## Architecture

```
┌─────────────┐     REST (60s)      ┌──────────────┐
│   Lanyard   │ ◄────────────────── │  ActivityPoller│
│   API       │ ──presence JSON───► │  (daemon thd)  │
└─────────────┘                     └──────┬───────┘
                                           │ append JSONL
                                           ▼
                                  ┌──────────────────┐
                                  │ logs/discord-     │
                                  │ activity/         │
                                  │  YYYY-MM-DD.jsonl │
                                  └────────┬─────────┘
                                           │ read
                                           ▼
                                  ┌──────────────────┐
                                  │ discord_activity  │
                                  │ tool (5 queries)  │
                                  └──────────────────┘
```

---

## Files

| File | Lines | Purpose |
|------|-------|---------|
| `plugin.yaml` | 18 | Config: env vars, tool declaration |
| `__init__.py` | 78 | `register()` — loads config, registers tool, starts poller |
| `poller.py` | 190 | `ActivityPoller` — background REST poller with PID lock |
| `schemas.py` | 25 | Tool JSON schema for `discord_activity` |
| `tools.py` | 521 | All query handlers (status, sessions, stats, spotify, history) |

---

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DISCORD_ACTIVITY_USER_ID` | **Yes** | — | Discord user ID to track |
| `DISCORD_ACTIVITY_POLL_INTERVAL` | No | `60` | Seconds between REST polls |
| `DISCORD_ACTIVITY_LANYARD_API` | No | `https://api.lanyard.rest/v1` | Lanyard API base URL |
| `DISCORD_ACTIVITY_IGNORE_SPOTIFY` | No | `false` | Strip Spotify data from results (1/true/yes/on) |
| `DISCORD_ACTIVITY_SESSION_GAP_MINUTES` | No | `30` | Disconnect threshold for session segmentation |

---

## Data Flow

### Poller (`poller.py`)

`ActivityPoller` runs as a daemon thread. Uses PID file lock (`~/.hermes/logs/discord-activity/.poller.pid`) to prevent duplicates when CLI and gateway coexist.

**Polling loop:**
1. `GET {api_base}/users/{user_id}` → Lanyard JSON response
2. Convert to JSONL entry via `_make_entry()`
3. Append to `~/.hermes/logs/discord-activity/YYYY-MM-DD.jsonl`
4. Sleep `poll_interval` seconds, repeat

**HTTP client:** Falls back between `requests` library and `urllib.request`. Custom `_FakeResponse` wrapper for urllib to match requests API.

### JSONL Entry Schema

Each line in `YYYY-MM-DD.jsonl`:
```json
{
  "timestamp": "2026-07-03T10:30:00+07:00",
  "discord_status": "online",
  "activities": [
    {
      "name": "VALORANT",
      "type": 0,
      "details": "In Game",
      "state": "Competitive",
      "timestamps": { "start": 1234567890 },
      "assets": { "large_text": "..." }
    }
  ],
  "spotify": {
    "track_id": "...",
    "song": "...",
    "artist": "...",
    "album": "...",
    "timestamps": { "start": ..., "end": ... }
  },
  "platforms": {
    "desktop": true,
    "mobile": false,
    "web": false
  },
  "listening_to_spotify": false
}
```

---

## Tool: `discord_activity`

**Parameters:**
- `query` (required): `status` | `sessions` | `stats` | `spotify` | `history`
- `days` (optional): Time range in days for stats/sessions (default: 1)
- `minutes` (optional): Time range in minutes for history/spotify (default: 60)

### Query: `status`

Returns the most recent presence entry. Reads only the last line of the latest JSONL file — fast, no scanning.

**Output:**
```json
{
  "timestamp": "...",
  "discord_status": "online",
  "activities": [{ "name": "...", "type": "playing", "details": "...", ... }],
  "spotify": { "song": "...", "artist": "...", ... },
  "platforms": { "desktop": true, ... }
}
```

### Query: `sessions`

Segments continuous app usage into sessions. Groups consecutive entries for the same app. Gap > `SESSION_GAP_MINUTES` starts a new session.

**Output:**
```json
{
  "apps": {
    "VALORANT": {
      "total_duration": "2.3h",
      "sessions": [
        {
          "start": "2026-07-03T10:30:00+07:00",
          "end": "2026-07-03T12:45:00+07:00",
          "duration": "2.3h",
          "details": ["In Game", "Competitive"]
        }
      ]
    }
  }
}
```

### Query: `stats`

Aggregated statistics over the time window. Uses `SESSION_GAP_MINUTES` for active time calculation.

**Output:**
```json
{
  "period_days": 1,
  "total_entries": 120,
  "elapsed_minutes": 720.0,
  "status_minutes": { "online": 480.0, "idle": 240.0 },
  "platform_minutes": { "desktop": 600.0, "mobile": 120.0 },
  "activities": {
    "top_apps": [{ "name": "VALORANT", "minutes": 180.5 }],
    "top_content": [{ "app": "VALORANT", "details": "Competitive", "minutes": 120.0 }]
  },
  "spotify": {
    "listening_minutes": 95.3,
    "unique_tracks": 28,
    "unique_artists": 12,
    "top_songs": [{ "name": "Song", "artist": "Artist", "ms": 240000 }],
    "top_artists": [{ "name": "Artist", "ms": 600000 }]
  }
}
```

### Query: `spotify`

Recent Spotify listening history. Deduplicates by `(track_id, song, artist)`. Returns empty when `IGNORE_SPOTIFY` is set.

**Output:**
```json
{
  "songs": [
    {
      "track_id": "...",
      "song": "Midnight City",
      "artist": "M83",
      "album": "Hurry Up, We're Dreaming",
      "timestamps": { "start": ..., "end": ... }
    }
  ],
  "total_entries": 42
}
```

### Query: `history`

Raw recent entries from the JSONL logs.

**Output:**
```json
{
  "entries": [
    {
      "timestamp": "...",
      "discord_status": "online",
      "activities": [...],
      "spotify": {...},
      "platforms": {...}
    }
  ],
  "total": 15
}
```

---

## Key Implementation Details

### tools.py Functions

| Function | Line | Purpose |
|----------|------|---------|
| `_get_log_dir()` | 38 | Get/create log directory |
| `_load_entries(log_dir, cutoff)` | 48 | Load JSONL entries within time window, sorted by `_dt` |
| `_load_file(entries, filepath, cutoff)` | 66 | Parse single JSONL file, add `_dt` datetime field |
| `_activity_names(entry)` | 94 | Get unique non-Spotify activity names from entry |
| `_extract_activities(entry)` | 107 | Rich activity extraction — name, type, details, state, timestamps, assets |
| `_extract_spotify(entry)` | 183 | Extract Spotify info (respects `IGNORE_SPOTIFY`) |
| `_get_current_status(log_dir)` | 204 | Latest presence (last line read) |
| `_get_sessions(log_dir, days)` | 244 | Session segmentation by app with gap detection |
| `_get_stats(log_dir, days)` | 330 | Full aggregated stats (status, platform, activities, spotify) |
| `_get_spotify(log_dir, minutes, days)` | 448 | Spotify listening history |
| `_get_history(log_dir, minutes)` | 479 | Raw entry dump |
| `discord_activity(args)` | 499 | Entry point — routes to query handlers |

### Activity Type Map

```python
_ACTIVITY_TYPE_MAP = {
    0: "playing",
    1: "streaming",
    2: "listening",
    3: "watching",
    4: "custom",
    5: "competing",
}
```

### Session Segmentation Logic

- Entries sorted by `_dt` (parsed from `timestamp` field)
- Gap between consecutive entries > `SESSION_GAP_MINUTES` → new session
- Sessions track `_start_dt`, `_last_seen`, `duration_minutes`, and `details` (set of activity detail strings)
- Filtered: sessions with `duration_minutes < 1` are dropped
- Duration formatted as `"Xm"` or `"X.Xh"`

### LoL Special Case

League of Legends entries get special handling in `_extract_activities()` — detects `timestamps.start` to determine if player is currently in a match.

### YouTube Detail Pruning

YouTube activity details are cleaned to remove noise (per commit `1907acf`).

### Stats: Active Time Calculation

Stats uses the same `SESSION_GAP_MINUTES` threshold to determine "active time" — consecutive entries within the gap count as active; gaps > threshold are idle time.

---

## Installation

```bash
cd ~/.hermes/plugins/
git clone <repo-url> hermes-dat
```

Set required env in `~/.hermes/profiles/aura/.env`:
```
DISCORD_ACTIVITY_USER_ID=197798264291065857
```

---

## Dependencies

- Python stdlib only (`json`, `os`, `threading`, `datetime`, `pathlib`, `collections`)
- Optional: `requests` (falls back to `urllib.request`)
- External: Lanyard API (https://api.lanyard.rest/v1) — requires joining Lanyard's Discord server or self-hosting

---

## Log Location

`~/.hermes/logs/discord-activity/YYYY-MM-DD.jsonl`

One file per day. Each line is a JSON poll snapshot. Files grow ~60 lines/hour (at 60s poll interval).

---

## PID Lock

`~/.hermes/logs/discord-activity/.poller.pid`

Prevents duplicate pollers. Cross-platform process check (`_is_process_alive()`). Handles stale PIDs on Windows via `OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION)`.

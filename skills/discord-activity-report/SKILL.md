---
name: discord-activity-report
description: Generate and deliver daily Discord activity reports
tags: [discord, activity, report, daily]
---

# Discord Activity Report

Generate a daily Discord activity report and deliver to the home channel.

## Steps

1. **Query data using the discord_activity tool:**
   - `discord_activity(query="stats", days=1)` — get aggregated stats
   - `discord_activity(query="timeline", days=1)` — get activity timeline

2. **Parse the output and extract key metrics:**
   - Total online time
   - Spotify listening time and top songs
   - Games/activities detected
   - Platform usage (desktop/mobile/web)
   - Status distribution (online/idle/offline)

3. **Format a clean Discord report using markdown:**
   - Use bold headers
   - Use bullet lists
   - Keep it concise but informative

4. **Send the report** to the home channel using `send_message` with target="discord:1506522557690679368"

## Report Template

```
**Daily Discord Activity Report**

**Online Time:** X hours Y minutes
**Platforms:** Desktop: Xm | Mobile: Xm | Web: Xm

**Spotify:**
- Top tracks: ...
- Total listening: Xm

**Activities:**
- Game: X sessions (Ym)
- ...

**Status:**
- Online: Xm
- Idle: Xm
- Offline: Xm

**Timeline:**
- **08:30–09:15** · 🟢 · Playing Valorant · 🎵 Days of Thunder — The Midnight
- **09:15–11:40** · 🟢 · Playing Valorant
- **13:05–15:30** · 🟢 · Listening to Spotify · 🎵 The Night — Avicii
```

## Timeline Formatting Rules

1. **Status indicators:**
   - 🟢 = online
   - 🟡 = idle
   - ⚫ = off/disconnected

2. **Activity display:**
   - Main activity: "Playing {name}" for games, just the name for others
   - Spotify: "Listening to Spotify" with 🎵 track info

3. **Spotify:** 🎵 {song} — {artist}

4. **Time format:** HH:MM in 24h format (WIB), start–end: "08:30–09:15"

## Notes

- The timeline filters out idle/offline periods and "Online (no activity)" — only real activity is shown
- Spotify tracks are grouped into listening blocks (not individual tracks)
- Status distribution (online/idle/offline) is always included in the stats

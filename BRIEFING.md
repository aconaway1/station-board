# GBB Arrivals Board — Project Briefing

## What This Is

A self-contained, browser-based train arrivals board for Amtrak stations. The initial target station is **Galesburg, IL (GBB)**. The app fetches live train data from the Amtraker API, displays upcoming arrivals with real-time status (on time, late, early), and auto-refreshes every 10 minutes.

The app runs as a local HTML file opened directly in a browser — no server required, no build step, no dependencies to install. It can also be served via a simple `python3 -m http.server` for access from other devices on the local network.

---

## Current State

A working single-file `index.html` exists with:
- Dark "departure board" aesthetic (amber on near-black, monospace fonts)
- Live clock and auto-refresh countdown with progress bar
- Color-coded rows (green = on time, red = late, blue = early, gray = scheduled)
- Fetches 8 train numbers hardcoded for GBB
- Falls back to hardcoded scheduled times when trains aren't active
- Scanline overlay effect for atmosphere
- Google Fonts: Bebas Neue (headers), Share Tech Mono (data), DM Sans (body)

**The problem with the current approach:** Train numbers, routes, directions, and scheduled times are all hardcoded for GBB. Changing stations would require significant manual work.

---

## The Rebuild Plan

We are rebuilding the app from scratch using a **self-learning schedule** approach. Here is the full architecture:

### Core Concept: Learn the Schedule Over Time

Instead of hardcoding train schedules, the app builds its own schedule database by watching the API. Every time it sees a train stop at the station, it records that train's scheduled time, route name, and direction. This data persists in `localStorage`. Over a few days of running, the app will have a complete picture of every train serving the station — with no manual data entry.

### Station Agnostic

The app should work for **any Amtrak station**, not just GBB. The station code should be configurable (ideally settable in the UI, or at minimum easy to change). When pointed at a new station, the app starts with no schedule knowledge and builds it up over time.

### Data Flow (per refresh cycle)

1. Call `GET https://api-v3.amtraker.com/v3/stations/{CODE}`
   - Returns station metadata and a list of currently active train run IDs (e.g. `["4-24", "4-25", "3-26", "5-26", "6-25"]`)
   - Also returns station name, address, lat/lon — use these to populate the UI header dynamically

2. Extract unique train numbers from those run IDs (e.g. `4-24` → train `4`)

3. Call `GET https://api-v3.amtraker.com/v3/trains/{NUM}` for each unique train number

4. For each train response, find the stop matching the current station code

5. **Upsert into localStorage:** For each train seen at the station, save/update:
   ```json
   {
     "4": {
       "schArr": "11:26 AM",
       "route": "Southwest Chief",
       "dir": "→ Chicago",
       "firstSeen": "2026-03-26",
       "lastSeen": "2026-03-27"
     }
   }
   ```
   Always update `schArr` and `lastSeen` on each sighting — this keeps the schedule current if Amtrak changes timetables.

6. **Determine next arrival for each known train:**
   - If the train has an active run where the station stop is `Enroute` or `Station`, use the estimated arrival time from that run
   - If all runs have `Departed` at this station, or the train isn't currently active, show the learned scheduled time with date label "TOMORROW"
   - If a train has never been seen (not yet in localStorage), it won't appear in the board at all — it gets added naturally the first time it shows up in the API

7. Render the board sorted by next estimated arrival time, with live/active trains sorted before scheduled ones

### localStorage Schema

Use a key per station to keep data isolated:

```
amtrak_schedule_GBB  →  { "3": {...}, "4": {...}, "5": {...}, ... }
amtrak_schedule_CHI  →  { ... }
```

Each train entry:
```json
{
  "schArr": "11:26 AM",
  "schDep": "11:28 AM",
  "route": "Southwest Chief",
  "dir": "→ Chicago",
  "destCode": "CHI",
  "destName": "Chicago Union",
  "origCode": "LAX",
  "origName": "Los Angeles Union",
  "firstSeen": "2026-03-26",
  "lastSeen": "2026-03-27"
}
```

Derive `dir` from `destName` — format as `→ {destName}`.

---

## The Amtraker API

**Base URL:** `https://api-v3.amtraker.com/v3`

**No authentication required. CORS is open for local file:// origins.**

### Station endpoint
`GET /v3/stations/{CODE}`

Example response for GBB:
```json
{
  "GBB": {
    "name": "Galesburg",
    "code": "GBB",
    "tz": "America/Chicago",
    "lat": 40.944678,
    "lon": -90.364106,
    "hasAddress": true,
    "address1": "225 South Seminary Street",
    "address2": " ",
    "city": "Galesburg",
    "state": "IL",
    "zip": "61401",
    "trains": ["3-25", "3-26", "4-24", "4-25", "4-26", "5-25", "5-26", "6-25", "6-26"]
  }
}
```

The `trains` array contains run IDs in the format `{trainNum}-{dayOfMonth}`.

### Train endpoint
`GET /v3/trains/{NUM}`

Returns an array of active run objects for that train number. Each run has:
- `routeName` — e.g. "Southwest Chief"
- `trainNum` — e.g. "4"
- `trainID` — e.g. "4-24"
- `lat`, `lon` — current GPS position
- `heading` — e.g. "NE"
- `velocity` — current speed in mph
- `eventCode` — station code of most recent event
- `eventName` — station name of most recent event
- `origCode`, `origName` — origin station
- `destCode`, `destName` — destination station
- `trainState` — "Active"
- `stations` — array of all stops (see below)

Each station stop in the `stations` array:
```json
{
  "name": "Galesburg",
  "code": "GBB",
  "tz": "America/Chicago",
  "bus": false,
  "schArr": "2026-03-26T11:26:00-05:00",
  "schDep": "2026-03-26T11:28:00-05:00",
  "arr": "2026-03-26T22:22:00-05:00",
  "dep": "2026-03-26T22:22:00-05:00",
  "status": "Enroute"
}
```

**Station stop statuses:**
- `"Enroute"` — train hasn't reached this stop yet
- `"Station"` — train is currently at this stop
- `"Departed"` — train has left this stop

**Key insight on lateness:** Compare `arr` (estimated/actual) to `schArr` (scheduled) to calculate minutes late/early. Both are ISO 8601 timestamps with timezone offsets.

---

## Trains Observed at GBB (as of initial build)

These are the trains known to serve GBB. The rebuilt app will learn these on its own, but this is useful context:

| # | Route | Direction | Sched Arr |
|---|-------|-----------|-----------|
| 3 | Southwest Chief | → Los Angeles | 5:01 PM |
| 4 | Southwest Chief | → Chicago | 11:26 AM |
| 5 | California Zephyr | → Emeryville | 4:43 PM |
| 6 | California Zephyr | → Chicago | 11:38 AM |
| 380 | Illinois Zephyr | → Chicago | 7:37 AM |
| 381 | Carl Sandburg | → Chicago | 8:15 AM |
| 382 | Carl Sandburg | → Quincy | 6:56 PM |
| 383 | Illinois Zephyr | → Quincy | 8:41 PM |

Trains 380–383 run the Chicago–Quincy corridor and are often not active in the API outside of their operating window.

---

## UI / Aesthetic Direction

Keep the existing visual design — it works well and the user likes it:

- **Background:** `#0a0c0f` (near black)
- **Panel:** `#0f1318`
- **Amber accent:** `#f5a623` (primary color for station name, train numbers, clock)
- **Green:** `#39d98a` (on time)
- **Red:** `#ff4d4d` (late)
- **Blue:** `#4db8ff` (early)
- **Muted:** `#4a5568` (labels, secondary text)
- **Fonts:** Bebas Neue (display), Share Tech Mono (data/mono), DM Sans (body)
- Scanline overlay via CSS `repeating-linear-gradient`
- Left border stripe on each row indicating status color
- Shimmer skeleton loading state

### Header (dynamic)
The header should populate from the station API response:
- Station name (large, Bebas Neue, amber)
- Station code · address · city, state (small, monospace, muted)
- Live clock top right (updates every second)
- Date below clock

### Status bar
- Animated dot (green when live, amber when loading)
- "LAST UPDATED HH:MM:SS" after successful fetch
- "NEXT REFRESH IN MM:SS" countdown on the right

### Refresh progress bar
- Thin bar below status bar
- Drains left-to-right over 10 minutes
- Amber gradient

### Board columns
`TRAIN | ROUTE | DIRECTION | NEXT ARRIVAL | STATUS`

Each row:
- Train number (large Bebas Neue, amber)
- Route name + "NR: {nearest station}" when live data available
- Direction (→ destination)
- Estimated arrival time + TODAY/TOMORROW/date label
- Status badge (ON TIME / X MIN LATE / X MIN EARLY / SCHEDULED)

---

## Suggested Feature Roadmap (post-rebuild)

These were discussed and are planned for future iterations:

1. **Manual refresh button** — don't wait 10 minutes, click to refresh now
2. **Row detail expand** — click a train row to see full station-by-station status for that run
3. **Proximity alert** — visual/audio alert when a train is N stops away from the station
4. **Station switcher UI** — input field to change station code without editing code
5. **Python local server instructions** in README — serve on local network for phone access
6. **Train map** — plot current GPS positions on a simple map

---

## Project Structure (recommended)

```
station-board/
├── index.html        # main app (single file, self-contained)
├── README.md         # what it is, how to use it, how to change stations
├── BRIEFING.md       # this file
└── .gitignore
```

As the app grows, consider splitting into:
```
├── index.html
├── css/
│   └── style.css
├── js/
│   ├── api.js        # Amtraker API calls
│   ├── schedule.js   # localStorage schedule management
│   ├── render.js     # DOM rendering
│   └── app.js        # main entry point / refresh loop
└── assets/
```

---

## Key Design Decisions Made

- **No build step, no bundler, no framework** — plain HTML/CSS/JS, open in browser and it works
- **localStorage for persistence** — no backend, no database, works offline for learned schedule display
- **Always update schArr on each sighting** — keeps schedule accurate through Amtrak timetable changes
- **Station-agnostic from the start** — the station code drives everything, no hardcoded station data
- **Show trains not yet learned simply by omitting them** — the board grows naturally over time
- **CORS works from file:// origin** — confirmed working locally; the Amtraker API is permissive

---

## Notes / Gotchas

- The Amtraker API can lag up to ~60 minutes behind real time — the `lastValTS` field on each run shows when data was last updated by the source
- Multiple runs of the same train number can be active simultaneously (e.g. `4-24`, `4-25`, `4-26`) — always use the first run where the station status is `Enroute` or `Station`; ignore runs where the station is `Departed`
- Train 4 (Southwest Chief) was observed running ~11 hours late during this session — normal lateness on freight-dominated corridors
- The Illinois Zephyr/Carl Sandburg trains (380-383) run the same Chicago–Quincy route and share all stops between Chicago and Galesburg
- Station timezone is provided in each stop object — use it when formatting times
- `schArr` in the API response is an ISO 8601 string with timezone offset — parse with `new Date()` directly

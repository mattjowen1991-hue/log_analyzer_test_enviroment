# Hubstaff Audit Log Analyzer v34

A browser-based tool for analyzing Hubstaff `audit.log` files to understand tracking sessions, crashes, idle time, and more.

![Version](https://img.shields.io/badge/version-v34-green)
![Privacy](https://img.shields.io/badge/privacy-100%25%20local-blue)

---

## 🔒 Privacy First

**All analysis happens entirely in your browser.** No data is ever sent to any server. When you close the tab or click "Clear," all data is removed from memory.

---

## Table of Contents

- [Quick Start](#quick-start)
- [Features](#features)
- [Understanding the Interface](#understanding-the-interface)
- [How It Works](#how-it-works)
- [Warning Types Explained](#warning-types-explained)
- [Timestamp Handling](#timestamp-handling)
- [Keyboard Shortcuts](#keyboard-shortcuts)
- [Technical Reference](#technical-reference)
- [Troubleshooting](#troubleshooting)

---

## Quick Start

1. **Open the HTML file** in any modern browser
2. **Paste** the contents of an `audit.log` file into the text area
3. **Click "Analyze"** (or press `Ctrl/Cmd + Enter`)
4. **Review** the KPIs, timeline, intervals, and warnings

### Sample Data

Two sample buttons are provided for testing:
- **Sample: Crash Recovered** — Shows a successful crash recovery scenario
- **Sample: Time Lost** — Shows a scenario where time was lost due to a crash

---

## Features

### Core Analysis
- **Tracked Time Calculation** — Sums all intervals between START and STOP events
- **Interval Breakdown** — Detailed table of each tracking session with start/stop times, duration, project, and stop reason
- **Crash Detection** — Identifies app crashes and whether time was recovered
- **Idle Time Tracking** — Shows idle time kept vs. discarded, with detailed decision history table

### Visual Timeline
- **Interactive timeline** showing tracking blocks, idle periods, and app events
- **Zoom control** to expand/collapse timeline detail
- **Click-to-inspect** any block for detailed information
- **Cursor placement** for pinpointing specific times

### Smart Warnings
- **Unclosed intervals** — Timer started but never stopped
- **Quick start/stop** — Likely accidental double-clicks
- **Crash recovery status** — Whether time was saved or lost
- **Configuration stops** — When tracking was stopped by system settings
- **Clickable warnings** — Jump directly to the relevant interval or event

### User Timezone Support
- Convert all displayed times to the customer's timezone
- Enter offset like `08:00`, `+5.5`, or `-05:00`
- Times shown in user's timezone are marked with ★

### Date Filtering
- Filter results to a specific date range
- Useful for logs spanning multiple days

---

## Understanding the Interface

### KPI Cards (Top Row)

| KPI | Description |
|-----|-------------|
| **Tracked Time (est.)** | Sum of all interval durations. May differ from dashboard by 1-2 minutes (normal). |
| **Intervals** | Number of START→STOP pairs found |
| **Idle Kept** | Total seconds the user chose to keep as idle time |
| **Idle Discarded** | Total seconds the user chose to discard |
| **Warnings** | Number of issues detected |

### Visual Timeline

The timeline has two tracks:

**Tracking Track (top)**
| Color | Meaning |
|-------|---------|
| 🟢 Green | Normal tracking (started by USER) |
| 🟠 Orange | Idle time (kept) |
| 🟣 Purple | Resumed after crash |
| ⬜ Striped gray | Discarded time |
| 🟢→🔴 Green with red end | Stopped by configuration |

**Events Track (bottom)**
| Symbol | Meaning |
|--------|---------|
| 🟢 ● | Clean app startup |
| 🔴 ✖ | Unclean startup (crash detected) |
| ⚫ ● | Normal shutdown |

### Interval Table

Each row shows:
- **#** — Interval number (with RECOVERED badge if applicable)
- **Start** — When tracking began (clickable to jump on timeline)
- **Stop** — When tracking ended
- **Duration** — Length of the interval
- **Project** — Project name if available
- **Task** — Task ID if available
- **Stop Reason** — Why tracking stopped (USER, IDLE, SHUTDOWN, CONFIGURATION, etc.)

### Idle Time Decisions Table

Shows every idle prompt response with full details:

| Column | Description |
|--------|-------------|
| **Time** | When the user responded to the idle prompt |
| **Idle Duration** | How long the user was idle before responding |
| **Response Time** | How many seconds the user took to answer the prompt |
| **User Action** | What the user chose (see below) |
| **Raw Values** | The actual `KeepIdle` and `StopTracking` values from the log |

**User Action Types:**

| Display | Raw Values | Meaning |
|---------|------------|---------|
| ✅ KEPT - User clicked YES | `KeepIdle: 1 / StopTracking: 0` | User chose to keep the idle time |
| ❌ DISCARDED - User clicked NO | `KeepIdle: 0 / StopTracking: 0` | User discarded idle time but continued tracking |
| 🛑 DISCARDED + STOPPED | `KeepIdle: 0 / StopTracking: 1` | User discarded idle time and stopped tracking |

**⚠️ >1hr Warning:** If idle duration exceeds 1 hour, a warning badge appears. Idle time over 1 hour cannot be kept regardless of user choice.

### Events Table

Shows all non-tracking events:
- Startups (clean/unclean)
- Shutdowns
- Idle prompts and answers
- Crash recovery events
- Location events
- Network errors

---

## How It Works

### Interval Detection

1. All log lines are parsed and sorted by timestamp
2. The analyzer loops through looking for `START_TRACKING` events
3. When found, it's held as an "open start"
4. When `STOP_TRACKING` or `SHUTDOWN` is found, the interval is closed
5. Duration = stop timestamp − start timestamp

### Crash & Resume Logic

When the app crashes while tracking:
1. On restart, `STARTUP_UNCLEAN` is logged
2. If recoverable, `RESUME_DETECTED` shows the original start time
3. The analyzer replaces the `[RESUMED]` event's timestamp with the original
4. This ensures the interval includes all tracked time

**Resume limits:**
- Desktop: Won't auto-resume if > 1 hour passed
- Mobile: Won't auto-resume if > 8 hours passed

### Idle Time Calculation

1. Find all `IDLE_ANSWER` events
2. Look backwards for the most recent `IDLE_WAKE` event
3. Extract the idle duration from the `IDLE_WAKE` "after X seconds" value
4. Extract the response time from the `IDLE_ANSWER` "after X seconds with" value
5. Determine decision type based on `KeepIdle` and `StopTracking` flags:
   - `KeepIdle: 1` → User kept idle time
   - `KeepIdle: 0 / StopTracking: 1` → User discarded and stopped tracking
   - `KeepIdle: 0 / StopTracking: 0` → User discarded but continued tracking
6. Add to "Kept" or "Discarded" totals accordingly
7. Record full decision details in the Idle Time Decisions table

---

## Warning Types Explained

### 🔴 Unclosed START
Timer started but never stopped. Usually indicates:
- App crashed (look for `STARTUP_UNCLEAN`)
- Timer still running
- Incomplete logs

### 🟡 Unmatched STOP
Stop found without a matching start. Usually means:
- Logs are incomplete
- Start event in an older log file

### ⚡ Quick Start/Stop
Timer started and stopped within 5 seconds. Almost always:
- Accidental double-click on record button

### ✅ Crash Recovered
App crashed but time was successfully recovered. No action needed.

### ⚠️ Crash Time Lost
App crashed and time couldn't be recovered because:
- Too much time passed (> 1 hour desktop, > 8 hours mobile)
- Resume data was corrupted
- App was reinstalled

### ⚙️ Stopped by Configuration
Tracking stopped automatically due to:
- User removed from project
- Time/budget limit reached
- Weekly/daily limit reached

### 🔗 Stopped by URL Permission
Tracking stopped because URL permission wasn't granted but the project requires it.

### 🚨 Stopped by Error
Tracking stopped due to an internal error. Check `hubstaff.log` for details.

---

## Timestamp Handling

### Supported Formats

The analyzer handles these timestamp formats:
```
2025-12-04 12:39:01.667205+1100    (timezone without colon)
2025-12-04 12:39:01.667205+11:00   (timezone with colon)
2025-12-04T12:39:01.667205Z        (UTC/Zulu time)
```

### Display Timezone

By default, times are displayed in your browser's local timezone.

To show times in the customer's timezone:
1. Check "Show user's timezone"
2. Enter their UTC offset (e.g., `+05:30` or `-08:00`)
3. Times will show with a ★ marker

---

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl/Cmd + Enter` | Analyze logs |

---

## Technical Reference

### Event Types Parsed

| Event | Description |
|-------|-------------|
| `START_TRACKING` | User or system started the timer |
| `STOP_TRACKING` | User or system stopped the timer |
| `SHUTDOWN` | App closed normally |
| `STARTUP_CLEAN` | App opened normally |
| `STARTUP_UNCLEAN` | App opened after crash/force quit |
| `RESUME_DETECTED` | Found previous session to recover |
| `RESUME_IGNORED` | Previous session too old to recover |
| `IDLE` | User went idle |
| `IDLE_WAKE` | User returned from idle |
| `IDLE_ANSWER` | User responded to idle prompt |
| `START_BREAK` / `STOP_BREAK` | Break started/ended |
| `LOCATIONS_STATUS` | Location permission changed |
| `GEOFENCE_ENTER` / `GEOFENCE_EXIT` | Job Site entered/exited |
| `DISCARD` | Time rejected by server |
| `NETWORK_ERROR` | Connection issue |
| `SENTRY` | Crash reported to Sentry |

### Stop Reasons

| Reason | Meaning |
|--------|---------|
| `USER` | User clicked stop |
| `IDLE` | Stopped due to idle timeout |
| `SHUTDOWN` | App closed |
| `RESUMED` | Recovered after crash |
| `CONFIGURATION` | Stopped by config/limit |
| `PROJECT_CONFIGURATION_URL` | URL permission required |
| `PROJECT_CONFIGURATION_LOCATION` | Left required Job Site |
| `ERROR` | Unexpected error |

### Timeline Position Calculation

```
position = ((eventTime - minTime) / totalDuration) * 100%
```

Where:
- `minTime` = earliest event − 5 minutes
- `maxTime` = latest event + 5 minutes
- `totalDuration` = maxTime − minTime

---

## Troubleshooting

### "No valid events found"
- Ensure you're pasting the contents of `audit.log`, not `hubstaff.log`
- Check that the log file isn't empty
- Verify timestamps are in a supported format

### Times don't match dashboard
Expected variance: 0-120 seconds. The dashboard uses slot-based calculations (10-minute slots), while this tool uses exact START/STOP timestamps.

For exact dashboard time, search `hubstaff.log` for: `Applied activity. Total:`

### Timeline is too compressed
Use the zoom slider to expand the timeline. Click "Reset" to return to default view.

### Customer's timezone shows "Invalid"
Enter the offset in one of these formats:
- `+05:30` or `-08:00` (with colon)
- `5.5` or `-8` (decimal hours)
- `08:00` (assumes positive)

---

## Version History

- **v34** — Current version with full crash recovery analysis, clickable warnings, date filtering, user timezone support, and idle time decisions table

---

## Support

This tool is designed for Hubstaff support teams to diagnose client tracking issues. For questions or feature requests, contact your team lead or the tool maintainer.

# Todo Timer App — Design Spec
**Date:** 2026-06-30  
**Delivery:** Single `todo.html` file (HTML + embedded CSS + JS, no build step)

---

## Overview

A dark-mode todo list with per-task time tracking. Tasks have priority levels. One timer runs at a time. History page shows per-task time totals filtered by day/week/month. All data persists in `localStorage`.

---

## Data Model

Two `localStorage` keys:

### `tasks` — `Task[]`

```ts
interface Task {
  id: string           // crypto.randomUUID()
  text: string
  priority: 'high' | 'medium' | 'low'
  completed: boolean
  createdAt: number    // epoch ms
  completedAt: number | null
  accumulatedMs: number  // sum of all stopped timer sessions
  activeTimer: {
    startTimestamp: number  // epoch ms when current session started
  } | null
}
```

### `timerSessions` — `Session[]`

```ts
interface Session {
  taskId: string
  startedAt: number   // epoch ms
  endedAt: number     // epoch ms
  durationMs: number
}
```

**Live time formula:**  
`displayMs = task.accumulatedMs + (task.activeTimer ? Date.now() - task.activeTimer.startTimestamp : 0)`

Sessions are immutable once written. They are the source of truth for history aggregation.

---

## Timer Logic

### Start timer
1. Call `stopAll()` — stops any currently running timer, saves its session
2. Set `task.activeTimer = { startTimestamp: Date.now() }`
3. Save to localStorage
4. `setInterval` at 1000ms drives display refresh only (not the clock)

### Stop timer
1. Compute `durationMs = Date.now() - task.activeTimer.startTimestamp`
2. `task.accumulatedMs += durationMs`
3. Push `Session { taskId, startedAt: task.activeTimer.startTimestamp, endedAt: Date.now(), durationMs }` to `timerSessions`
4. Set `task.activeTimer = null`
5. Save to localStorage

**Minimize-safe:** `Date.now()` is wall-clock time — unaffected by Chrome tab throttling. The `setInterval` may fire late when minimized, but the displayed value is always recomputed from timestamps on each tick, so it self-corrects instantly when the tab is foregrounded.

---

## UI Structure

Two views, toggled in-place (no navigation, no reload):

### Tasks View

- Header: app name + `[Tasks]` `[History]` nav tabs
- Input row: text field + priority dropdown (`High / Medium / Low`) + Add button
- Task list sorted: High → Medium → Low, completed tasks sink to bottom within each group
- Each task row:
  - Checkbox (toggles `completed`)
  - Task text
  - Live timer display `HH:MM:SS`
  - ▶ Start / ■ Stop button
  - 🗑 Delete button
- Active timer row: pulsing green dot indicator

### History View

- Filter tabs: `[Day]` `[Week]` `[Month]`
  - Day = sessions with `startedAt` in today (local midnight → now)
  - Week = last 7 days
  - Month = last 30 days
- Table columns: Task name | Total Time
- Grand total row at bottom
- Tasks with zero time in the selected window are hidden

---

## Architecture (single file)

```
todo.html
├── <style>        CSS variables for dark theme, layout, animations
└── <script>
    ├── Storage    load() / save() — JSON serialize/deserialize localStorage
    ├── State      tasks[], sessions[], intervalId
    ├── Timer      startTimer(id), stopTimer(id), stopAll(), getLiveMs(task)
    ├── Render     renderTasks(), renderHistory(filter), tick()
    └── Events     DOMContentLoaded wires all input/button handlers
```

### Dark theme CSS variables
```css
--bg: #0f0f0f
--surface: #1a1a1a
--border: #2a2a2a
--text: #e0e0e0
--muted: #666
--accent: #4ade80     /* green — active timer */
--high: #f87171       /* red */
--medium: #fb923c     /* orange */
--low: #60a5fa        /* blue */
```

---

## Edge Cases

- **Delete task with active timer:** stop the timer first (save session), then remove the task and all its sessions from storage
- **Start timer on completed task:** allowed — completed tasks can accumulate more time
- **App reload with active timer:** on load, if any task has `activeTimer` set, resume displaying live time immediately (the timestamp is already saved)

---

## Constraints & Non-Goals

- No frameworks, no external dependencies, no CDN links
- No multi-device sync
- No task editing after creation (delete + re-add)
- No sub-tasks
- Completed tasks can still run timers (rework/review tracking)

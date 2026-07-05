# Daily Target Feature — Design Spec

**Date:** 2026-07-06  
**Status:** Approved

## Overview

Add per-task daily time targets to TaskFlow. Tasks with a target display an animated water-fill background that rises as time is logged. Timer continues past 100% — the target is a goal, not a cap. Progress resets at midnight each day.

## Data Model

One new field on each task object:

```js
dailyTargetMs: null | number  // null = no target; number = target in ms
```

Stored in existing `tf_tasks` localStorage key. No migration needed — missing field treated as `null`.

**Daily progress computation:**

```js
function getDailyMs(taskId) {
  const midnight = new Date(); midnight.setHours(0,0,0,0);
  const fromSessions = state.sessions
    .filter(s => s.taskId === taskId && s.startedAt >= midnight.getTime())
    .reduce((acc, s) => acc + s.durationMs, 0);
  const task = state.tasks.find(t => t.id === taskId);
  const liveMs = (task?.activeTimer)
    ? Math.max(0, Date.now() - Math.max(task.activeTimer.startTimestamp, midnight.getTime()))
    : 0;
  return fromSessions + liveMs;
}
```

Resets naturally at midnight since it's always computed on read.

## Task Row UI

`.task-row` gains `position: relative; overflow: hidden`.

A `<div class="fill-bar">` is injected as the first child of each task row that has a target. It sits behind all content via `position: absolute`.

**Fill bar CSS:**

```css
.fill-bar {
  position: absolute;
  left: 0; top: 0;
  height: 100%;
  width: 0%;
  pointer-events: none;
  transition: width 0.8s cubic-bezier(0.4, 0, 0.2, 1);
}
.fill-bar.filling  { background: rgba(74, 222, 128, 0.12); }
.fill-bar.complete { background: rgba(74, 222, 128, 0.28); }
```

Fill width = `Math.min(dailyProgressMs / dailyTargetMs * 100, 100)%` — capped at 100% visually, timer runs past.

**Target badge** — shown between timer display and play button:

- Format: `⊙ 30m`
- Color: `var(--medium)` (amber) when target not yet met; `var(--accent)` (green) when met
- Hidden on tasks with no target set
- Clicking badge opens inline edit input

## Setting & Editing Targets

**At creation:** small optional `<input id="target-input" placeholder="Target (30m, 1h…)">` (~80px) added to the add-row between priority select and Add button. Leave blank = no target.

**After creation:** click the `⊙ Xm` badge → badge replaced with a small `<input>` prefilled with current value → Enter or blur saves. For tasks with no target, a `⊕` icon appears on row hover (hidden by default) — clicking it opens the same inline input to set a target.

**Parser** (`parseTargetInput(str) → ms | null`):

| Input | Result |
|-------|--------|
| `30m` | 1,800,000 ms |
| `1h`  | 3,600,000 ms |
| `1.5h` | 5,400,000 ms |
| `90`  | 5,400,000 ms (bare number = minutes) |
| empty / invalid | `null` |

## Render & Tick Integration

`renderTasks()` — for each task:
1. Call `getDailyMs(task.id)`
2. If `dailyTargetMs` set: compute `pct`, inject `.fill-bar` with correct class and `style="width:${pct}%"`
3. Inject target badge with color class based on met/unmet state

`tick()` already calls `renderTasks()` every second when a timer is active. CSS transition on `.fill-bar` produces smooth water-rise animation without an extra interval.

## Scope

- All changes confined to `todo.html` (single file, no external deps)
- Add-row: one new `<input id="target-input">` element
- No new storage keys
- No changes to History view, PiP, or Pause logic

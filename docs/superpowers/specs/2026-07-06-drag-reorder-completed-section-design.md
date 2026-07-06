# Drag Reorder + Completed Section — Design Spec

**Date:** 2026-07-06  
**Status:** Approved

## Overview

Two features for TaskFlow:
1. **Drag-and-drop reordering** — incomplete tasks can be dragged to any position; manual order replaces priority-based sorting.
2. **Collapsible completed section** — completed tasks collapse to a toggle at the bottom; hidden by default.

## Data Model

No new fields. `state.tasks` array position is the display order. Existing `priority` field becomes a label only — no longer drives sort order.

**Render split:**
```js
const incomplete = state.tasks.filter(t => !t.completed);
const completed  = state.tasks.filter(t => t.completed);
```

Incomplete tasks render in array order. Completed tasks render in a separate collapsible section below.

**New tasks:** `state.tasks.unshift(task)` — prepended to array, appears at top of incomplete list.

**Uncompleting a task:** `task.completed = false` — task reappears at its current array position in the incomplete list (no splice needed).

**Completed section toggle state:**
```js
let completedExpanded = false;
```
Persisted in `localStorage` as `tf_completed_expanded` (string `'true'`/`'false'`). Loaded on init.

## Completed Tasks Section

Rendered below incomplete tasks in `renderTasks()`. Hidden entirely when `completed.length === 0`.

**Collapsed state (default):**
```
▶ Completed (3)
```

**Expanded state:**
```
▼ Completed (3)
  [completed task rows]
```

Toggle header: `.completed-toggle` — dark surface, muted border, chevron (`▶`/`▼`) + count. Click toggles `completedExpanded`, persists to localStorage, re-renders.

Completed task rows: same structure as incomplete rows but `draggable="false"`, no fill-bar animation (fill-bar still renders if target has `dailyTargetMs`, showing today's progress as a static visual).

## Drag & Drop

**State variable:**
```js
let dragSrcId = null;
```

**Task row changes:** Each incomplete row gets `draggable="true"`.

**Event delegation on `#task-list`:**

| Event | Action |
|-------|--------|
| `dragstart` | Set `dragSrcId = id`, add `.dragging` to row |
| `dragover` | `e.preventDefault()`, add `.drag-over` to target row |
| `dragleave` | Remove `.drag-over` from target row |
| `drop` | Splice `state.tasks`, `persist()`, `renderTasks()` |
| `dragend` | Clear `dragSrcId`, remove `.dragging`/`.drag-over` from all rows |

**Drop splice logic:**
```js
function reorderTask(srcId, targetId) {
  const srcIdx = state.tasks.findIndex(t => t.id === srcId);
  const tgtIdx = state.tasks.findIndex(t => t.id === targetId);
  if (srcIdx === -1 || tgtIdx === -1 || srcIdx === tgtIdx) return;
  const [task] = state.tasks.splice(srcIdx, 1);
  state.tasks.splice(tgtIdx, 0, task);
  persist();
}
```

**CSS additions:**
```css
.task-row[draggable="true"] { cursor: grab; }
.task-row[draggable="true"]:active { cursor: grabbing; }
.task-row.dragging { opacity: 0.4; }
.task-row.drag-over { border-color: var(--accent); border-style: dashed; }
```

The priority dot (`.priority-dot`) gains a `title="Drag to reorder"` hint since it's the leftmost visual anchor.

## renderTasks Changes

HTML structure in `#view-tasks`:
```html
<div id="task-list"></div>          <!-- incomplete only, drag events here -->
<div id="completed-section"></div>  <!-- toggle + completed rows, no drag events -->
```

`renderTasks()`:
1. Sets `#task-list` innerHTML to `incomplete.map(task => rowHtml(task, true))` — `draggable="true"`
2. Sets `#completed-section` innerHTML to toggle header + (if expanded) completed rows — `draggable="false"`

Drag events attached only to `#task-list` — completed rows never receive drag events.

## Scope

- All changes in `todo.html` only
- No new localStorage keys except `tf_completed_expanded`
- History, PiP, Pause, daily target features unchanged
- `addTask` call site: `unshift` instead of `push`

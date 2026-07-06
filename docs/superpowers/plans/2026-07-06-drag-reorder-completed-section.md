# Drag Reorder + Completed Section Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add drag-and-drop reordering of incomplete tasks and a collapsible completed tasks section at the bottom of the task list.

**Architecture:** `state.tasks` array order becomes the display order (priority sort removed). `renderTasks` splits tasks into incomplete/completed and renders them into two separate DOM containers (`#task-list` and `#completed-section`). A `taskRowHtml` helper is extracted to serve both containers. Drag events delegate from `#task-list` only; `reorderTask` splices the array and persists.

**Tech Stack:** Vanilla JS, HTML5 Drag and Drop API, CSS, localStorage — no external dependencies.

## Global Constraints

- All changes confined to `todo.html` (single file, no external deps)
- No new localStorage keys except `tf_completed_expanded`
- Existing History, PiP, Pause, and daily target features must remain unchanged
- New tasks prepend to top of list (`unshift`)
- Drag operates on incomplete tasks only

---

### Task 1: CSS — drag styles + completed toggle

**Files:**
- Modify: `todo.html` — `<style>` block, before closing `</style>`

**Interfaces:**
- Produces: `.task-row[draggable="true"]`, `.task-row.dragging`, `.task-row.drag-over`, `.completed-toggle` — used by Tasks 2 and 4

- [ ] **Step 1: Add drag and completed toggle CSS**

Insert the following immediately before the `#target-input:focus` rule (just before the closing `</style>` tag):

```css
    .task-row[draggable="true"] { cursor: grab; }
    .task-row[draggable="true"]:active { cursor: grabbing; }
    .task-row.dragging { opacity: 0.4; }
    .task-row.drag-over { border-color: var(--accent); border-style: dashed; }

    .completed-toggle {
      display: flex;
      align-items: center;
      gap: 8px;
      padding: 10px 14px;
      background: var(--surface);
      border: 1px solid var(--border);
      border-radius: 8px;
      color: var(--muted);
      font-size: 0.875rem;
      cursor: pointer;
      margin-bottom: 8px;
      width: 100%;
      text-align: left;
      transition: all 0.15s;
    }
    .completed-toggle:hover { color: var(--text); border-color: var(--accent); }
```

- [ ] **Step 2: Verify CSS loads without errors**

Open `todo.html` in browser. DevTools → Console: zero errors. DevTools → Elements: verify `.completed-toggle` rule exists in computed styles panel.

- [ ] **Step 3: Commit**

```bash
git add todo.html
git commit -m "style: add drag and completed-toggle CSS"
```

---

### Task 2: HTML structure + taskRowHtml helper + renderTasks refactor

**Files:**
- Modify: `todo.html` — HTML `#view-tasks`, JS `renderTasks` function

**Interfaces:**
- Consumes: `getDailyMs`, `formatTarget`, `Timer.getLiveMs`, `escapeHtml`, `formatMs` — all existing
- Produces:
  - `taskRowHtml(task: object, draggable: boolean) → string` — HTML string for one task row
  - `renderCompletedSection(completed: object[]) → void` — renders `#completed-section`
  - Updated `renderTasks()` — renders incomplete into `#task-list`, calls `renderCompletedSection`

- [ ] **Step 1: Add `#completed-section` div to HTML**

Find the `#view-tasks` HTML block (around line 366):

```html
        <div id="task-list"></div>
      </div>
```

Change to:

```html
        <div id="task-list"></div>
        <div id="completed-section"></div>
      </div>
```

- [ ] **Step 2: Extract `taskRowHtml` helper function**

Find the `// ── Render: Tasks` comment. Insert `taskRowHtml` immediately before `function renderTasks()`:

```js
    function taskRowHtml(task, draggable) {
      const liveMs = Timer.getLiveMs(task);
      const isActive = !!task.activeTimer;

      const dailyMs = task.dailyTargetMs ? getDailyMs(task.id) : 0;
      const pct = task.dailyTargetMs
        ? Math.min(dailyMs / task.dailyTargetMs * 100, 100)
        : 0;
      const targetMet = task.dailyTargetMs && dailyMs >= task.dailyTargetMs;

      const fillBarHtml = task.dailyTargetMs
        ? `<div class="fill-bar ${targetMet ? 'complete' : 'filling'}" style="width:${pct.toFixed(2)}%"></div>`
        : '';

      const targetBadgeHtml = task.dailyTargetMs
        ? `<button class="target-badge ${targetMet ? 'met' : 'unmet'}" data-id="${task.id}" data-action="edit-target">⊙ ${formatTarget(task.dailyTargetMs)}</button>`
        : `<button class="target-badge add-target" data-id="${task.id}" data-action="edit-target">⊕</button>`;

      return `<div class="task-row${task.completed ? ' completed' : ''}${isActive ? ' active' : ''}" data-id="${task.id}"${draggable ? ' draggable="true"' : ''}>
        ${fillBarHtml}
        <span class="priority-dot priority-${task.priority}"></span>
        <input type="checkbox" class="task-check" data-id="${task.id}"${task.completed ? ' checked' : ''} />
        <span class="task-text">${escapeHtml(task.text)}</span>
        <span class="task-timer${isActive ? ' running' : ''}">${formatMs(liveMs)}</span>
        ${targetBadgeHtml}
        <button class="timer-btn" data-id="${task.id}" data-action="${isActive ? 'stop' : 'start'}">${isActive ? '■' : '▶'}</button>
        <button class="delete-btn" data-id="${task.id}">🗑</button>
      </div>`;
    }

    function renderCompletedSection(completed) {
      const section = document.getElementById('completed-section');
      if (!section) return;
      if (completed.length === 0) { section.innerHTML = ''; return; }
      const chevron = completedExpanded ? '▼' : '▶';
      const rowsHtml = completedExpanded
        ? completed.map(task => taskRowHtml(task, false)).join('')
        : '';
      section.innerHTML = `
        <button class="completed-toggle" id="completed-toggle-btn">
          ${chevron} Completed (${completed.length})
        </button>
        ${rowsHtml}
      `;
    }
```

- [ ] **Step 3: Replace `renderTasks` body**

Replace the entire body of `renderTasks` (keep the function declaration, replace everything inside `{}`):

```js
    function renderTasks() {
      const list = document.getElementById('task-list');
      if (!list) return;

      const incomplete = state.tasks.filter(t => !t.completed);
      const completed  = state.tasks.filter(t => t.completed);

      if (incomplete.length === 0 && completed.length === 0) {
        list.innerHTML = '<p class="empty-state">No tasks yet. Add one above.</p>';
      } else {
        list.innerHTML = incomplete.map(task => taskRowHtml(task, true)).join('');
      }

      renderCompletedSection(completed);
      renderTotalTarget();
    }
```

- [ ] **Step 4: Verify tasks render without errors**

Open `todo.html`. Add two tasks. Verify both appear at the top. Open DevTools → Console: zero errors. Verify tasks have `draggable="true"` attribute in Elements panel.

- [ ] **Step 5: Commit**

```bash
git add todo.html
git commit -m "refactor: extract taskRowHtml, split renderTasks into incomplete/completed sections"
```

---

### Task 3: State changes — unshift, completedExpanded, toggle wiring

**Files:**
- Modify: `todo.html` — `addTask`, state variables, `DOMContentLoaded` init block

**Interfaces:**
- Consumes: `renderCompletedSection` (Task 2), `renderTasks` (Task 2)
- Produces: `completedExpanded: boolean` — read by `renderCompletedSection`

- [ ] **Step 1: Add `completedExpanded` and `dragSrcId` state variables**

Find the state variable declarations (around `let state = { ... }`):

```js
    let state = { tasks: [], sessions: [], pauseSessions: [], activePause: null, lastActiveTaskId: null };
    let pipWindow = null;
```

Change to:

```js
    let state = { tasks: [], sessions: [], pauseSessions: [], activePause: null, lastActiveTaskId: null };
    let pipWindow = null;
    let completedExpanded = false;
    let dragSrcId = null;
```

- [ ] **Step 2: Change `addTask` to prepend with `unshift`**

Find `state.tasks.push(task);` inside `addTask` and change to:

```js
      state.tasks.unshift(task);
```

- [ ] **Step 3: Load `completedExpanded` from localStorage on init**

In the `DOMContentLoaded` handler, find the block where state is loaded from `Storage.load()`:

```js
      state.tasks = loaded.tasks;
      state.sessions = loaded.sessions;
      state.pauseSessions = loaded.pauseSessions || [];
      state.activePause = loaded.activePause || null;
      state.lastActiveTaskId = loaded.lastActiveTaskId || null;
```

Add one line after it:

```js
      completedExpanded = localStorage.getItem('tf_completed_expanded') === 'true';
```

- [ ] **Step 4: Wire up completed section toggle click handler**

In `DOMContentLoaded`, after the existing `document.getElementById('task-list').addEventListener('click', ...)` handler, add:

```js
      document.getElementById('completed-section').addEventListener('click', e => {
        const btn = e.target.closest('#completed-toggle-btn');
        if (btn) {
          completedExpanded = !completedExpanded;
          localStorage.setItem('tf_completed_expanded', String(completedExpanded));
          renderTasks();
          return;
        }
        const id = e.target.dataset.id;
        if (!id) return;
        if (e.target.classList.contains('task-check')) {
          toggleComplete(id);
          renderTasks();
        } else if (e.target.classList.contains('timer-btn')) {
          if (e.target.dataset.action === 'start') Timer.startTimer(id);
          else Timer.stopTimer(id);
          renderTasks();
        } else if (e.target.classList.contains('delete-btn')) {
          deleteTask(id);
          renderTasks();
        } else if (e.target.dataset.action === 'edit-target') {
          openTargetEdit(id);
        }
      });
```

- [ ] **Step 5: Verify completed section in browser**

1. Add a task, check it off. Verify `▶ Completed (1)` toggle appears below incomplete tasks.
2. Click toggle. Verify it expands to show the completed task with `▼ Completed (1)`.
3. Reload page. Verify expanded state persists (`tf_completed_expanded` in localStorage).
4. Add a new task. Verify it appears at the **top** of the incomplete list.

- [ ] **Step 6: Commit**

```bash
git add todo.html
git commit -m "feat: collapsible completed section and new tasks prepend to top"
```

---

### Task 4: Drag-and-drop reordering

**Files:**
- Modify: `todo.html` — new `reorderTask` function, drag event handlers in `DOMContentLoaded`

**Interfaces:**
- Consumes: `dragSrcId: string | null` (Task 3), `state.tasks`, `persist`, `renderTasks`
- Produces: `reorderTask(srcId: string, targetId: string) → void`

- [ ] **Step 1: Add `reorderTask` function**

Insert immediately before `function openTargetEdit(taskId)`:

```js
    function reorderTask(srcId, targetId) {
      if (srcId === targetId) return;
      const srcIdx = state.tasks.findIndex(t => t.id === srcId);
      const tgtIdx = state.tasks.findIndex(t => t.id === targetId);
      if (srcIdx === -1 || tgtIdx === -1) return;
      const [task] = state.tasks.splice(srcIdx, 1);
      state.tasks.splice(tgtIdx, 0, task);
      persist();
    }
```

- [ ] **Step 2: Wire up drag event handlers on `#task-list`**

In `DOMContentLoaded`, after the `#task-list` click handler, add:

```js
      const taskList = document.getElementById('task-list');

      taskList.addEventListener('dragstart', e => {
        const row = e.target.closest('.task-row');
        if (!row) return;
        dragSrcId = row.dataset.id;
        row.classList.add('dragging');
        e.dataTransfer.effectAllowed = 'move';
      });

      taskList.addEventListener('dragover', e => {
        e.preventDefault();
        const row = e.target.closest('.task-row');
        if (!row || row.dataset.id === dragSrcId) return;
        document.querySelectorAll('.task-row.drag-over').forEach(r => r.classList.remove('drag-over'));
        row.classList.add('drag-over');
      });

      taskList.addEventListener('dragleave', e => {
        const row = e.target.closest('.task-row');
        if (row) row.classList.remove('drag-over');
      });

      taskList.addEventListener('drop', e => {
        e.preventDefault();
        const row = e.target.closest('.task-row');
        if (!row || !dragSrcId) return;
        reorderTask(dragSrcId, row.dataset.id);
        dragSrcId = null;
        renderTasks();
      });

      taskList.addEventListener('dragend', () => {
        dragSrcId = null;
        document.querySelectorAll('.task-row.dragging, .task-row.drag-over')
          .forEach(r => r.classList.remove('dragging', 'drag-over'));
      });
```

- [ ] **Step 3: Verify drag reordering in browser**

1. Add three tasks: "A", "B", "C". Verify order is C (top), B, A (bottom) — newest first.
2. Drag "A" to the top. Verify order is A, C, B.
3. Reload page. Verify order A, C, B persists (saved to localStorage).
4. Check DevTools Console: zero errors during drag.
5. Drag a task while another is running (active timer). Verify timer continues after reorder.

- [ ] **Step 4: Commit**

```bash
git add todo.html
git commit -m "feat: drag-and-drop task reordering"
```

---

## Self-Review Checklist

**Spec coverage:**
- [x] Manual order replaces priority sort → `renderTasks` uses array order, no sort
- [x] New tasks at top → `unshift` in Task 3
- [x] Completed collapsed by default → `completedExpanded = false` in Task 3
- [x] Toggle persists → `localStorage.setItem('tf_completed_expanded', ...)` in Task 3
- [x] Completed rows not draggable → `taskRowHtml(task, false)` in `renderCompletedSection`
- [x] Drag events on `#task-list` only, not `#completed-section` → Task 4
- [x] `reorderTask` splice logic → Task 4 Step 1
- [x] `dragSrcId` cleared on `dragend` → Task 4 Step 2
- [x] Completed section click handler (uncheck, delete, timer, edit-target) → Task 3 Step 4
- [x] `renderTotalTarget()` still called at end of `renderTasks` → Task 2 Step 3

**No placeholders:** All steps contain exact code.

**Type consistency:** `taskRowHtml(task, draggable)` — used in Task 2 (`renderTasks`, `renderCompletedSection`) with `true`/`false`. `reorderTask(srcId, targetId)` called in `drop` handler with `row.dataset.id` strings. `completedExpanded` declared in Task 3 Step 1, read in `renderCompletedSection` (Task 2). `dragSrcId` declared in Task 3 Step 1, set/read in Task 4. All consistent.

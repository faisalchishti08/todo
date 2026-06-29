# Todo Timer App Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single `todo.html` file — dark-mode todo list with priority levels, per-task timestamp-based timers (minimize-safe), and a history page with day/week/month filters.

**Architecture:** One HTML file with embedded CSS and vanilla JS. JS is organized into IIFE modules (Storage, Timer) and top-level functions (CRUD, render, events). No build step — open directly in Chrome.

**Tech Stack:** Vanilla HTML5, CSS3, JavaScript (ES2020), localStorage, `crypto.randomUUID()`, `Date.now()`

## Global Constraints

- No external dependencies, no CDN links, no frameworks
- Single file: `todo.html` in project root
- Dark mode only — no light mode toggle
- One active timer at a time — starting a new one auto-stops the current one
- `Date.now()` drives all timer math — display tick only refreshes the DOM
- localStorage keys: `tf_tasks` and `tf_sessions`

---

### Task 1: HTML Skeleton + CSS Dark Theme

**Files:**
- Create: `todo.html`

**Interfaces:**
- Produces: Full HTML structure with IDs used by all later JS tasks. CSS variables and all visual classes defined here.

- [ ] **Step 1: Create `todo.html` with skeleton and all CSS**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>TaskFlow</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

    :root {
      --bg: #0f0f0f;
      --surface: #1a1a1a;
      --border: #2a2a2a;
      --text: #e0e0e0;
      --muted: #666;
      --accent: #4ade80;
      --high: #f87171;
      --medium: #fb923c;
      --low: #60a5fa;
    }

    body {
      background: var(--bg);
      color: var(--text);
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
      min-height: 100vh;
    }

    #app {
      max-width: 720px;
      margin: 0 auto;
      padding: 0 16px 80px;
    }

    header {
      display: flex;
      align-items: center;
      justify-content: space-between;
      padding: 24px 0 20px;
      border-bottom: 1px solid var(--border);
      margin-bottom: 24px;
    }

    header h1 {
      font-size: 1.4rem;
      font-weight: 700;
      color: var(--accent);
      letter-spacing: -0.5px;
    }

    nav { display: flex; gap: 4px; }

    .nav-btn {
      background: none;
      border: 1px solid var(--border);
      color: var(--muted);
      padding: 6px 14px;
      border-radius: 6px;
      cursor: pointer;
      font-size: 0.875rem;
      transition: all 0.15s;
    }

    .nav-btn.active, .nav-btn:hover {
      background: var(--surface);
      color: var(--text);
      border-color: var(--accent);
    }

    .hidden { display: none !important; }

    /* Add row */
    .add-row {
      display: flex;
      gap: 8px;
      margin-bottom: 24px;
    }

    .add-row input {
      flex: 1;
      background: var(--surface);
      border: 1px solid var(--border);
      color: var(--text);
      padding: 10px 14px;
      border-radius: 8px;
      font-size: 0.9rem;
      outline: none;
      transition: border-color 0.15s;
    }

    .add-row input:focus { border-color: var(--accent); }

    .add-row select {
      background: var(--surface);
      border: 1px solid var(--border);
      color: var(--text);
      padding: 10px 10px;
      border-radius: 8px;
      font-size: 0.875rem;
      cursor: pointer;
      outline: none;
    }

    .add-row button {
      background: var(--accent);
      border: none;
      color: #000;
      padding: 10px 18px;
      border-radius: 8px;
      font-size: 0.875rem;
      font-weight: 600;
      cursor: pointer;
      transition: opacity 0.15s;
    }

    .add-row button:hover { opacity: 0.85; }

    /* Task rows */
    .task-row {
      display: flex;
      align-items: center;
      gap: 10px;
      background: var(--surface);
      border: 1px solid var(--border);
      border-radius: 8px;
      padding: 12px 14px;
      margin-bottom: 8px;
      transition: border-color 0.2s;
    }

    .task-row.active { border-color: var(--accent); }

    .task-row.completed .task-text {
      text-decoration: line-through;
      color: var(--muted);
    }

    .priority-dot {
      width: 8px;
      height: 8px;
      border-radius: 50%;
      flex-shrink: 0;
    }

    .priority-dot.priority-high   { background: var(--high); }
    .priority-dot.priority-medium { background: var(--medium); }
    .priority-dot.priority-low    { background: var(--low); }

    .task-check {
      width: 16px;
      height: 16px;
      cursor: pointer;
      accent-color: var(--accent);
      flex-shrink: 0;
    }

    .task-text {
      flex: 1;
      font-size: 0.9rem;
      word-break: break-word;
    }

    .task-timer {
      font-family: 'SF Mono', 'Fira Code', monospace;
      font-size: 0.85rem;
      color: var(--muted);
      min-width: 70px;
      text-align: right;
      flex-shrink: 0;
    }

    .task-timer.running { color: var(--accent); }

    @keyframes pulse {
      0%, 100% { opacity: 1; }
      50% { opacity: 0.4; }
    }

    .task-timer.running::before {
      content: '● ';
      animation: pulse 1.2s ease-in-out infinite;
    }

    .timer-btn, .delete-btn {
      background: none;
      border: 1px solid var(--border);
      color: var(--muted);
      width: 30px;
      height: 30px;
      border-radius: 6px;
      cursor: pointer;
      font-size: 0.8rem;
      display: flex;
      align-items: center;
      justify-content: center;
      flex-shrink: 0;
      transition: all 0.15s;
    }

    .timer-btn:hover { color: var(--accent); border-color: var(--accent); }
    .delete-btn:hover { color: var(--high); border-color: var(--high); }

    /* History */
    .filter-tabs {
      display: flex;
      gap: 4px;
      margin-bottom: 20px;
    }

    .filter-btn {
      background: none;
      border: 1px solid var(--border);
      color: var(--muted);
      padding: 6px 16px;
      border-radius: 6px;
      cursor: pointer;
      font-size: 0.875rem;
      transition: all 0.15s;
    }

    .filter-btn.active, .filter-btn:hover {
      background: var(--surface);
      color: var(--text);
      border-color: var(--accent);
    }

    #history-table table {
      width: 100%;
      border-collapse: collapse;
    }

    #history-table th, #history-table td {
      text-align: left;
      padding: 10px 12px;
      border-bottom: 1px solid var(--border);
      font-size: 0.875rem;
    }

    #history-table th {
      color: var(--muted);
      font-weight: 500;
    }

    #history-table tfoot td {
      font-weight: 600;
      color: var(--accent);
      border-top: 1px solid var(--border);
      border-bottom: none;
    }

    .empty-state {
      color: var(--muted);
      font-size: 0.875rem;
      text-align: center;
      padding: 40px 0;
    }
  </style>
</head>
<body>
  <div id="app">
    <header>
      <h1>✓ TaskFlow</h1>
      <nav>
        <button id="nav-tasks" class="nav-btn active">Tasks</button>
        <button id="nav-history" class="nav-btn">History</button>
      </nav>
    </header>

    <main>
      <div id="view-tasks">
        <div class="add-row">
          <input type="text" id="task-input" placeholder="Add a task..." />
          <select id="priority-select">
            <option value="high">High</option>
            <option value="medium" selected>Medium</option>
            <option value="low">Low</option>
          </select>
          <button id="add-btn">+ Add</button>
        </div>
        <div id="task-list"></div>
      </div>

      <div id="view-history" class="hidden">
        <div class="filter-tabs">
          <button class="filter-btn active" data-filter="day">Day</button>
          <button class="filter-btn" data-filter="week">Week</button>
          <button class="filter-btn" data-filter="month">Month</button>
        </div>
        <div id="history-table"></div>
      </div>
    </main>
  </div>

  <script>
    // JS modules added in subsequent tasks
  </script>
</body>
</html>
```

- [ ] **Step 2: Open `todo.html` in Chrome and verify**
  - Dark background renders correctly
  - Header shows "✓ TaskFlow" in green
  - Tasks / History nav buttons visible
  - Input row shows text field, priority dropdown, Add button
  - No console errors

- [ ] **Step 3: Commit**

```bash
git init
git add todo.html
git commit -m "feat: HTML skeleton and CSS dark theme"
```

---

### Task 2: Storage Module

**Files:**
- Modify: `todo.html` — replace `// JS modules added in subsequent tasks` comment with Storage IIFE

**Interfaces:**
- Produces: `Storage.load()` → `{ tasks: Task[], sessions: Session[] }` and `Storage.save(tasks, sessions)` → `void`

- [ ] **Step 1: Replace the JS comment with the Storage module**

Replace `// JS modules added in subsequent tasks` with:

```js
const Storage = (() => {
  const TASKS_KEY = 'tf_tasks';
  const SESSIONS_KEY = 'tf_sessions';

  function load() {
    return {
      tasks: JSON.parse(localStorage.getItem(TASKS_KEY) || '[]'),
      sessions: JSON.parse(localStorage.getItem(SESSIONS_KEY) || '[]'),
    };
  }

  function save(tasks, sessions) {
    localStorage.setItem(TASKS_KEY, JSON.stringify(tasks));
    localStorage.setItem(SESSIONS_KEY, JSON.stringify(sessions));
  }

  return { load, save };
})();

// State, Timer, CRUD, Render, Events added below
```

- [ ] **Step 2: Verify in Chrome DevTools console**

Open Chrome DevTools → Console and run:
```js
Storage.save([{id:'test'}], []);
Storage.load(); // → { tasks: [{id:'test'}], sessions: [] }
localStorage.getItem('tf_tasks'); // → '[{"id":"test"}]'
Storage.save([], []);
```
Expected: no errors, values match above.

- [ ] **Step 3: Commit**

```bash
git add todo.html
git commit -m "feat: add Storage module (localStorage read/write)"
```

---

### Task 3: State Variables + Utility Helpers

**Files:**
- Modify: `todo.html` — add after Storage module

**Interfaces:**
- Produces:
  - `state` — `{ tasks: Task[], sessions: Session[] }`
  - `persist()` — saves current state to localStorage
  - `formatMs(ms: number): string` — returns `"HH:MM:SS"`
  - `escapeHtml(str: string): string` — escapes `& < > "`

- [ ] **Step 1: Add state and helpers after the Storage module (before the `// State...` comment)**

Replace `// State, Timer, CRUD, Render, Events added below` with:

```js
let state = { tasks: [], sessions: [] };

function persist() {
  Storage.save(state.tasks, state.sessions);
}

function formatMs(ms) {
  const totalSecs = Math.floor(ms / 1000);
  const h = Math.floor(totalSecs / 3600);
  const m = Math.floor((totalSecs % 3600) / 60);
  const s = totalSecs % 60;
  return `${String(h).padStart(2, '0')}:${String(m).padStart(2, '0')}:${String(s).padStart(2, '0')}`;
}

function escapeHtml(str) {
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;');
}

// Timer, CRUD, Render, Events added below
```

- [ ] **Step 2: Verify in console**

```js
formatMs(0);       // → "00:00:00"
formatMs(3661000); // → "01:01:01"
formatMs(59999);   // → "00:00:59"
escapeHtml('<b>hi</b>'); // → "&lt;b&gt;hi&lt;/b&gt;"
state; // → { tasks: [], sessions: [] }
```

- [ ] **Step 3: Commit**

```bash
git add todo.html
git commit -m "feat: add state, persist(), formatMs(), escapeHtml()"
```

---

### Task 4: Timer Module

**Files:**
- Modify: `todo.html` — add Timer IIFE after state/helpers

**Interfaces:**
- Consumes: `state.tasks`, `persist()`
- Produces:
  - `Timer.getLiveMs(task: Task): number` — total ms including live session
  - `Timer.startTimer(id: string): void` — stops all, starts timer on task
  - `Timer.stopTimer(id: string): void` — stops timer, saves session
  - `Timer.stopAll(): void` — stops any running timer

- [ ] **Step 1: Add Timer module — replace `// Timer, CRUD, Render, Events added below`**

```js
const Timer = (() => {
  function getLiveMs(task) {
    if (!task.activeTimer) return task.accumulatedMs;
    return task.accumulatedMs + (Date.now() - task.activeTimer.startTimestamp);
  }

  function stopTimer(id) {
    const task = state.tasks.find(t => t.id === id);
    if (!task || !task.activeTimer) return;
    const now = Date.now();
    const durationMs = now - task.activeTimer.startTimestamp;
    task.accumulatedMs += durationMs;
    state.sessions.push({
      taskId: id,
      startedAt: task.activeTimer.startTimestamp,
      endedAt: now,
      durationMs,
    });
    task.activeTimer = null;
    persist();
  }

  function stopAll() {
    // copy ids first — stopTimer mutates task in place
    state.tasks.filter(t => t.activeTimer).forEach(t => stopTimer(t.id));
  }

  function startTimer(id) {
    stopAll();
    const task = state.tasks.find(t => t.id === id);
    if (!task) return;
    task.activeTimer = { startTimestamp: Date.now() };
    persist();
  }

  return { getLiveMs, startTimer, stopTimer, stopAll };
})();

// CRUD, Render, Events added below
```

- [ ] **Step 2: Verify in console**

```js
// Seed a fake task
state.tasks = [{ id: 'a', accumulatedMs: 5000, activeTimer: null }];
Timer.getLiveMs(state.tasks[0]); // → 5000

Timer.startTimer('a');
state.tasks[0].activeTimer; // → { startTimestamp: <number> }

// Wait 2 seconds, then:
Timer.getLiveMs(state.tasks[0]); // → ~7000

Timer.stopTimer('a');
state.tasks[0].activeTimer;    // → null
state.tasks[0].accumulatedMs;  // → ~7000
state.sessions.length;         // → 1
state.sessions[0].durationMs;  // → ~2000

// Cleanup
state.tasks = []; state.sessions = []; Storage.save([],[]);
```

- [ ] **Step 3: Commit**

```bash
git add todo.html
git commit -m "feat: add Timer module (timestamp-based, minimize-safe)"
```

---

### Task 5: CRUD Functions

**Files:**
- Modify: `todo.html` — add CRUD functions after Timer module

**Interfaces:**
- Consumes: `state`, `persist()`, `Timer.stopTimer()`
- Produces:
  - `addTask(text: string, priority: 'high'|'medium'|'low'): Task`
  - `deleteTask(id: string): void`
  - `toggleComplete(id: string): void`

- [ ] **Step 1: Add CRUD functions — replace `// CRUD, Render, Events added below`**

```js
function addTask(text, priority) {
  const task = {
    id: crypto.randomUUID(),
    text,
    priority,
    completed: false,
    createdAt: Date.now(),
    completedAt: null,
    accumulatedMs: 0,
    activeTimer: null,
  };
  state.tasks.push(task);
  persist();
  return task;
}

function deleteTask(id) {
  Timer.stopTimer(id);
  state.tasks = state.tasks.filter(t => t.id !== id);
  state.sessions = state.sessions.filter(s => s.taskId !== id);
  persist();
}

function toggleComplete(id) {
  const task = state.tasks.find(t => t.id === id);
  if (!task) return;
  task.completed = !task.completed;
  task.completedAt = task.completed ? Date.now() : null;
  persist();
}

// Render, Events added below
```

- [ ] **Step 2: Verify in console**

```js
const t = addTask('Buy milk', 'high');
state.tasks.length;   // → 1
t.priority;           // → 'high'
t.completed;          // → false

toggleComplete(t.id);
state.tasks[0].completed;   // → true
state.tasks[0].completedAt; // → <number>

toggleComplete(t.id);
state.tasks[0].completed;   // → false

Timer.startTimer(t.id); // start a timer so we can test delete with active timer
deleteTask(t.id);
state.tasks.length;   // → 0
state.sessions.length; // session saved from the interrupted timer → 1
Storage.save([],[]); state.sessions = [];
```

- [ ] **Step 3: Commit**

```bash
git add todo.html
git commit -m "feat: add addTask, deleteTask, toggleComplete"
```

---

### Task 6: renderTasks

**Files:**
- Modify: `todo.html` — add renderTasks after CRUD functions

**Interfaces:**
- Consumes: `state.tasks`, `Timer.getLiveMs()`, `formatMs()`, `escapeHtml()`
- Produces: `renderTasks(): void` — redraws `#task-list` innerHTML

- [ ] **Step 1: Add renderTasks — replace `// Render, Events added below`**

```js
function renderTasks() {
  const list = document.getElementById('task-list');
  if (!list) return;

  const PRIORITY_ORDER = ['high', 'medium', 'low'];

  const sorted = [...state.tasks].sort((a, b) => {
    // Primary: priority order
    const pi = PRIORITY_ORDER.indexOf(a.priority) - PRIORITY_ORDER.indexOf(b.priority);
    if (pi !== 0) return pi;
    // Secondary: incomplete before complete
    return (a.completed ? 1 : 0) - (b.completed ? 1 : 0);
  });

  if (sorted.length === 0) {
    list.innerHTML = '<p class="empty-state">No tasks yet. Add one above.</p>';
    return;
  }

  list.innerHTML = sorted.map(task => {
    const liveMs = Timer.getLiveMs(task);
    const isActive = !!task.activeTimer;
    return `<div class="task-row${task.completed ? ' completed' : ''}${isActive ? ' active' : ''}" data-id="${task.id}">
      <span class="priority-dot priority-${task.priority}"></span>
      <input type="checkbox" class="task-check" data-id="${task.id}"${task.completed ? ' checked' : ''} />
      <span class="task-text">${escapeHtml(task.text)}</span>
      <span class="task-timer${isActive ? ' running' : ''}">${formatMs(liveMs)}</span>
      <button class="timer-btn" data-id="${task.id}" data-action="${isActive ? 'stop' : 'start'}">${isActive ? '■' : '▶'}</button>
      <button class="delete-btn" data-id="${task.id}">🗑</button>
    </div>`;
  }).join('');
}

// Render History, Events added below
```

- [ ] **Step 2: Verify in console**

```js
// Seed some tasks
addTask('High priority', 'high');
addTask('Low priority', 'low');
addTask('Medium priority', 'medium');
renderTasks();
// Expected: task-list shows 3 rows, sorted High → Medium → Low
// Each row has priority dot, checkbox, text, 00:00:00 timer, ▶ and 🗑 buttons
```

Then in the browser — confirm the UI renders correctly with visible rows.

- [ ] **Step 3: Commit**

```bash
git add todo.html
git commit -m "feat: add renderTasks with priority sort and timer display"
```

---

### Task 7: renderHistory

**Files:**
- Modify: `todo.html` — add renderHistory after renderTasks

**Interfaces:**
- Consumes: `state.tasks`, `state.sessions`, `Timer.getLiveMs()`, `formatMs()`, `escapeHtml()`
- Produces: `renderHistory(filter: 'day'|'week'|'month'): void` — redraws `#history-table` innerHTML

- [ ] **Step 1: Add renderHistory — replace `// Render History, Events added below`**

```js
function renderHistory(filter) {
  const container = document.getElementById('history-table');
  if (!container) return;

  const now = Date.now();
  let windowStart;
  if (filter === 'day') {
    const d = new Date();
    d.setHours(0, 0, 0, 0);
    windowStart = d.getTime();
  } else if (filter === 'week') {
    windowStart = now - 7 * 24 * 60 * 60 * 1000;
  } else {
    windowStart = now - 30 * 24 * 60 * 60 * 1000;
  }

  // Sum completed sessions in window
  const totals = {};
  state.sessions
    .filter(s => s.startedAt >= windowStart)
    .forEach(s => {
      totals[s.taskId] = (totals[s.taskId] || 0) + s.durationMs;
    });

  // Add live time for the currently active timer if its session started in window
  state.tasks.forEach(t => {
    if (t.activeTimer && t.activeTimer.startTimestamp >= windowStart) {
      const live = Date.now() - t.activeTimer.startTimestamp;
      totals[t.id] = (totals[t.id] || 0) + live;
    }
  });

  const entries = Object.entries(totals);
  if (entries.length === 0) {
    container.innerHTML = '<p class="empty-state">No activity in this period.</p>';
    return;
  }

  let grandTotal = 0;
  const rows = entries.map(([taskId, ms]) => {
    grandTotal += ms;
    const task = state.tasks.find(t => t.id === taskId);
    const name = task ? escapeHtml(task.text) : '(deleted task)';
    return `<tr><td>${name}</td><td>${formatMs(ms)}</td></tr>`;
  }).join('');

  container.innerHTML = `<table>
    <thead><tr><th>Task</th><th>Total Time</th></tr></thead>
    <tbody>${rows}</tbody>
    <tfoot><tr><td>Total</td><td>${formatMs(grandTotal)}</td></tr></tfoot>
  </table>`;
}

// Events added below
```

- [ ] **Step 2: Verify in console**

```js
// Seed a task and sessions
const t = addTask('Test task', 'medium');
state.sessions.push({ taskId: t.id, startedAt: Date.now() - 3000, endedAt: Date.now(), durationMs: 3000 });
renderHistory('day');
// Expected: table shows "Test task | 00:00:03" and "Total | 00:00:03"

renderHistory('week');   // same data, same result
renderHistory('month');  // same data, same result

// Test empty state
state.sessions = [];
renderHistory('day');
// Expected: "No activity in this period."

// Cleanup
state.tasks = []; state.sessions = []; Storage.save([],[]);
```

- [ ] **Step 3: Commit**

```bash
git add todo.html
git commit -m "feat: add renderHistory with day/week/month aggregation"
```

---

### Task 8: Event Wiring, Tick Loop, and Init

**Files:**
- Modify: `todo.html` — add all event handling and init after renderHistory

**Interfaces:**
- Consumes: all previously defined functions and modules
- Produces: fully wired, interactive app

- [ ] **Step 1: Add events + init — replace `// Events added below`**

```js
let currentView = 'tasks';
let currentHistoryFilter = 'day';

function showView(view) {
  currentView = view;
  document.getElementById('view-tasks').classList.toggle('hidden', view !== 'tasks');
  document.getElementById('view-history').classList.toggle('hidden', view !== 'history');
  document.getElementById('nav-tasks').classList.toggle('active', view === 'tasks');
  document.getElementById('nav-history').classList.toggle('active', view === 'history');
  if (view === 'tasks') renderTasks();
  else renderHistory(currentHistoryFilter);
}

function tick() {
  const hasActive = state.tasks.some(t => t.activeTimer);
  if (currentView === 'tasks' && hasActive) {
    renderTasks();
  } else if (currentView === 'history') {
    renderHistory(currentHistoryFilter);
  }
}

document.addEventListener('DOMContentLoaded', () => {
  // Load persisted state
  const loaded = Storage.load();
  state.tasks = loaded.tasks;
  state.sessions = loaded.sessions;

  // Add task button
  document.getElementById('add-btn').addEventListener('click', () => {
    const input = document.getElementById('task-input');
    const priority = document.getElementById('priority-select').value;
    const text = input.value.trim();
    if (!text) return;
    addTask(text, priority);
    input.value = '';
    renderTasks();
  });

  // Enter key submits
  document.getElementById('task-input').addEventListener('keydown', e => {
    if (e.key === 'Enter') document.getElementById('add-btn').click();
  });

  // Task list — event delegation for check, timer, delete buttons
  document.getElementById('task-list').addEventListener('click', e => {
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
    }
  });

  // Nav tabs
  document.getElementById('nav-tasks').addEventListener('click', () => showView('tasks'));
  document.getElementById('nav-history').addEventListener('click', () => showView('history'));

  // History filter tabs
  document.querySelectorAll('.filter-btn').forEach(btn => {
    btn.addEventListener('click', () => {
      document.querySelectorAll('.filter-btn').forEach(b => b.classList.remove('active'));
      btn.classList.add('active');
      currentHistoryFilter = btn.dataset.filter;
      renderHistory(currentHistoryFilter);
    });
  });

  // Start display tick (1s interval — drives DOM refresh only, not the clock)
  setInterval(tick, 1000);

  // Initial render
  renderTasks();
});
```

- [ ] **Step 2: Full smoke test in Chrome**

Open `todo.html` in Chrome. Verify each behavior:

1. **Add tasks:** Type "Buy groceries", select High, click Add → row appears with red dot
2. **Priority sort:** Add "Write docs" (Medium) → appears below High task
3. **Complete task:** Check "Buy groceries" → strikethrough, sinks below uncompleted
4. **Start timer:** Click ▶ on "Write docs" → timer ticks up, green pulse, border glows green
5. **Auto-stop:** Click ▶ on a second task → first timer stops, second starts
6. **Stop timer:** Click ■ → timer freezes at current value
7. **Minimize Chrome:** Start a timer, minimize Chrome for 5 seconds, restore → timer shows correct elapsed time (not frozen or slow)
8. **Delete with active timer:** Start timer on a task, click 🗑 → task gone, session saved (check History)
9. **Reload persistence:** Add tasks, start+stop a timer, reload page → tasks and history preserved
10. **History — Day:** Switch to History tab, verify task appears with correct time in Day filter
11. **History — Week/Month:** Switch filters, same task appears
12. **Reload with active timer:** Start a timer, reload page → timer resumes counting from correct timestamp

- [ ] **Step 3: Commit**

```bash
git add todo.html
git commit -m "feat: wire events, tick loop, init — app fully functional"
```

---

## Summary

```
todo.html
├── <style>      — CSS variables, layout, animations (Task 1)
└── <script>
    ├── Storage  — load() / save() localStorage (Task 2)
    ├── state    — tasks[], sessions[]; persist() (Task 3)
    ├── helpers  — formatMs(), escapeHtml() (Task 3)
    ├── Timer    — getLiveMs(), startTimer(), stopTimer(), stopAll() (Task 4)
    ├── CRUD     — addTask(), deleteTask(), toggleComplete() (Task 5)
    ├── Render   — renderTasks(), renderHistory() (Tasks 6-7)
    └── Events   — DOMContentLoaded, showView(), tick() (Task 8)
```

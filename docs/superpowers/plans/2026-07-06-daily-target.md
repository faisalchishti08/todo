# Daily Target Feature Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add optional per-task daily time targets to TaskFlow, with an animated water-fill background showing progress toward the target each day.

**Architecture:** All changes confined to `todo.html`. A `getDailyMs` helper computes today's progress from `state.sessions` filtered to after midnight. `renderTasks()` injects a `.fill-bar` div and a target badge into each task row; the CSS transition on `width` produces the water-fill animation. A `openTargetEdit` function handles inline editing by replacing the badge with an `<input>`.

**Tech Stack:** Vanilla JS, CSS, localStorage — no external dependencies.

## Global Constraints

- Single file: all changes in `todo.html` only
- No new localStorage keys — `dailyTargetMs` field stored inside existing `tf_tasks` array
- No external dependencies
- Existing History, PiP, and Pause logic must remain unchanged
- Timer continues past 100% — target is a goal not a cap
- Progress resets at midnight (computed on read, never stored)

---

### Task 1: CSS — fill-bar, target badge, inline edit input

**Files:**
- Modify: `todo.html` — `<style>` block (around line 151 `.task-row` rule and end of style block)

**Interfaces:**
- Produces: `.fill-bar`, `.fill-bar.filling`, `.fill-bar.complete`, `.target-badge`, `.target-badge.unmet`, `.target-badge.met`, `.target-badge.add-target`, `.target-edit-input` — used by Tasks 4 and 5

- [ ] **Step 1: Add `position: relative; overflow: hidden` to `.task-row`**

Find the `.task-row` rule (currently around line 151). Change it from:

```css
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
```

To:

```css
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
  position: relative;
  overflow: hidden;
}
```

- [ ] **Step 2: Add fill-bar, target badge, and edit input CSS**

Insert the following block immediately before the closing `</style>` tag (just before `</style></head>`):

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

    .target-badge {
      font-family: 'SF Mono', 'Fira Code', monospace;
      font-size: 0.75rem;
      padding: 2px 6px;
      border-radius: 4px;
      border: 1px solid currentColor;
      background: none;
      cursor: pointer;
      flex-shrink: 0;
      white-space: nowrap;
      transition: opacity 0.15s;
    }
    .target-badge.unmet { color: var(--medium); }
    .target-badge.met   { color: var(--accent); }
    .target-badge.add-target {
      color: var(--muted);
      border-color: var(--border);
      opacity: 0;
    }
    .task-row:hover .target-badge.add-target { opacity: 1; }

    .target-edit-input {
      font-family: 'SF Mono', 'Fira Code', monospace;
      font-size: 0.75rem;
      background: var(--surface);
      border: 1px solid var(--accent);
      color: var(--text);
      padding: 2px 6px;
      border-radius: 4px;
      width: 70px;
      outline: none;
      flex-shrink: 0;
    }
```

- [ ] **Step 3: Verify CSS loads without errors**

Open `todo.html` in a browser. Open DevTools → Console. Confirm zero errors. Open DevTools → Elements, find a `.task-row` element, confirm computed styles include `position: relative` and `overflow: hidden`.

- [ ] **Step 4: Commit**

```bash
git add todo.html
git commit -m "style: add fill-bar and target badge CSS"
```

---

### Task 2: JS helpers — parseTargetInput, formatTarget, getDailyMs + addTask update

**Files:**
- Modify: `todo.html` — `<script>` block, Storage/State/Helpers section and `addTask` function

**Interfaces:**
- Produces:
  - `parseTargetInput(str: string) → number | null` — parses user input to ms
  - `formatTarget(ms: number) → string` — formats ms to "30m" / "1h" / "1h30m"
  - `getDailyMs(taskId: string) → number` — today's accumulated ms for a task
  - `addTask(text, priority, dailyTargetMs?)` — now accepts optional third param

- [ ] **Step 1: Add three helper functions after the `escapeHtml` function**

Find the line `function escapeHtml(str) {` (around line 384). After the closing `}` of `escapeHtml`, insert:

```js
    function parseTargetInput(str) {
      const s = (str || '').trim().toLowerCase();
      if (!s) return null;
      const mMatch = s.match(/^(\d+\.?\d*)m$/);
      if (mMatch) return Math.round(parseFloat(mMatch[1]) * 60000);
      const hMatch = s.match(/^(\d+\.?\d*)h$/);
      if (hMatch) return Math.round(parseFloat(hMatch[1]) * 3600000);
      const numMatch = s.match(/^(\d+\.?\d*)$/);
      if (numMatch) return Math.round(parseFloat(numMatch[1]) * 60000);
      return null;
    }

    function formatTarget(ms) {
      const totalMins = Math.round(ms / 60000);
      if (totalMins < 60) return `${totalMins}m`;
      const h = Math.floor(totalMins / 60);
      const m = totalMins % 60;
      return m === 0 ? `${h}h` : `${h}h${m}m`;
    }

    function getDailyMs(taskId) {
      const midnight = new Date();
      midnight.setHours(0, 0, 0, 0);
      const midnightTs = midnight.getTime();
      const fromSessions = state.sessions
        .filter(s => s.taskId === taskId && s.startedAt >= midnightTs)
        .reduce((acc, s) => acc + s.durationMs, 0);
      const task = state.tasks.find(t => t.id === taskId);
      const liveMs = task?.activeTimer
        ? Math.max(0, Date.now() - Math.max(task.activeTimer.startTimestamp, midnightTs))
        : 0;
      return fromSessions + liveMs;
    }
```

- [ ] **Step 2: Update `addTask` to accept `dailyTargetMs`**

Find the `addTask` function (around line 458). Change it from:

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
```

To:

```js
    function addTask(text, priority, dailyTargetMs = null) {
      const task = {
        id: crypto.randomUUID(),
        text,
        priority,
        completed: false,
        createdAt: Date.now(),
        completedAt: null,
        accumulatedMs: 0,
        activeTimer: null,
        dailyTargetMs,
      };
```

- [ ] **Step 3: Verify helpers in browser console**

Open `todo.html`, open DevTools Console, run:

```js
parseTargetInput('30m')   // → 1800000
parseTargetInput('1h')    // → 3600000
parseTargetInput('1.5h')  // → 5400000
parseTargetInput('90')    // → 5400000
parseTargetInput('')      // → null
parseTargetInput('abc')   // → null
formatTarget(1800000)     // → "30m"
formatTarget(3600000)     // → "1h"
formatTarget(5400000)     // → "1h30m"
```

All results must match comments exactly.

- [ ] **Step 4: Commit**

```bash
git add todo.html
git commit -m "feat: add parseTargetInput, formatTarget, getDailyMs helpers and dailyTargetMs to task model"
```

---

### Task 3: Add-row — target input at creation time

**Files:**
- Modify: `todo.html` — HTML add-row, JS add-btn click handler

**Interfaces:**
- Consumes: `parseTargetInput` (Task 2), `addTask` updated signature (Task 2)

- [ ] **Step 1: Add target input to the add-row HTML**

Find the add-row HTML (around line 313):

```html
        <div class="add-row">
          <input type="text" id="task-input" placeholder="Add a task..." />
          <select id="priority-select">
            <option value="high">High</option>
            <option value="medium" selected>Medium</option>
            <option value="low">Low</option>
          </select>
          <button id="add-btn">+ Add</button>
        </div>
```

Change to:

```html
        <div class="add-row">
          <input type="text" id="task-input" placeholder="Add a task..." />
          <select id="priority-select">
            <option value="high">High</option>
            <option value="medium" selected>Medium</option>
            <option value="low">Low</option>
          </select>
          <input type="text" id="target-input" placeholder="Target (30m, 1h…)" style="width:110px;background:var(--surface);border:1px solid var(--border);color:var(--text);padding:10px 10px;border-radius:8px;font-size:0.875rem;outline:none;" />
          <button id="add-btn">+ Add</button>
        </div>
```

- [ ] **Step 2: Update the add-btn click handler to read target input**

Find the add-btn click handler (around line 780):

```js
      document.getElementById('add-btn').addEventListener('click', () => {
        const input = document.getElementById('task-input');
        const priority = document.getElementById('priority-select').value;
        const text = input.value.trim();
        if (!text) return;
        addTask(text, priority);
        input.value = '';
        renderTasks();
      });
```

Change to:

```js
      document.getElementById('add-btn').addEventListener('click', () => {
        const input = document.getElementById('task-input');
        const priority = document.getElementById('priority-select').value;
        const targetInput = document.getElementById('target-input');
        const text = input.value.trim();
        if (!text) return;
        const dailyTargetMs = parseTargetInput(targetInput.value);
        addTask(text, priority, dailyTargetMs);
        input.value = '';
        targetInput.value = '';
        renderTasks();
      });
```

- [ ] **Step 3: Add focus-style for target input**

The inline style above handles basic appearance. Add a focus border by inserting into the `<style>` block (before `</style>`):

```css
    #target-input:focus { border-color: var(--accent) !important; }
```

- [ ] **Step 4: Verify creation flow in browser**

1. Type a task name, set priority, type `30m` in target field, click Add.
2. Open DevTools Console, run `JSON.parse(localStorage.getItem('tf_tasks'))`. Last entry must have `dailyTargetMs: 1800000`.
3. Add another task with no target. Its `dailyTargetMs` must be `null`.

- [ ] **Step 5: Commit**

```bash
git add todo.html
git commit -m "feat: add target input to task creation form"
```

---

### Task 4: renderTasks — inject fill-bar and target badge

**Files:**
- Modify: `todo.html` — `renderTasks` function

**Interfaces:**
- Consumes: `getDailyMs` (Task 2), `formatTarget` (Task 2), CSS classes from Task 1
- Produces: task rows with `.fill-bar` and `.target-badge` / `.add-target` elements with `data-action="edit-target"`

- [ ] **Step 1: Replace the task row template in `renderTasks`**

Find the `list.innerHTML = sorted.map(task => {` block (around line 512). Replace the entire map callback — from the opening backtick of the template literal to its closing — with:

```js
      list.innerHTML = sorted.map(task => {
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

        return `<div class="task-row${task.completed ? ' completed' : ''}${isActive ? ' active' : ''}" data-id="${task.id}">
          ${fillBarHtml}
          <span class="priority-dot priority-${task.priority}"></span>
          <input type="checkbox" class="task-check" data-id="${task.id}"${task.completed ? ' checked' : ''} />
          <span class="task-text">${escapeHtml(task.text)}</span>
          <span class="task-timer${isActive ? ' running' : ''}">${formatMs(liveMs)}</span>
          ${targetBadgeHtml}
          <button class="timer-btn" data-id="${task.id}" data-action="${isActive ? 'stop' : 'start'}">${isActive ? '■' : '▶'}</button>
          <button class="delete-btn" data-id="${task.id}">🗑</button>
        </div>`;
      }).join('');
```

- [ ] **Step 2: Verify fill-bar renders in browser**

1. Add a task with target `1m` (1 minute).
2. Start its timer.
3. Within 30 seconds, open DevTools → Elements, find the task's `.fill-bar`. Its `width` style must be between `0%` and `50%` and increasing.
4. After 1 minute, `.fill-bar` must have class `complete` and `width: 100%`.
5. A task with no target must have no `.fill-bar` child and must show `⊕` (hidden until hover).

- [ ] **Step 3: Verify badge colors**

1. Task with target not yet met: badge text is amber (matches `var(--medium)` = `#fb923c`).
2. Task with target met: badge text is green (matches `var(--accent)` = `#4ade80`).

- [ ] **Step 4: Commit**

```bash
git add todo.html
git commit -m "feat: render fill-bar and target badge in task rows"
```

---

### Task 5: Inline target editing — click badge to set or change target

**Files:**
- Modify: `todo.html` — new `openTargetEdit` function, task-list click handler

**Interfaces:**
- Consumes: `parseTargetInput` (Task 2), `formatTarget` (Task 2), `renderTasks`, `persist`
- Consumes: `data-action="edit-target"` on badge elements (Task 4)

- [ ] **Step 1: Add `openTargetEdit` function**

Insert this function immediately before the `// ── Events + Init` comment (around line 712):

```js
    function openTargetEdit(taskId) {
      const task = state.tasks.find(t => t.id === taskId);
      if (!task) return;
      const badge = document.querySelector(`[data-id="${taskId}"][data-action="edit-target"]`);
      if (!badge) return;
      const currentVal = task.dailyTargetMs ? formatTarget(task.dailyTargetMs) : '';
      const input = document.createElement('input');
      input.className = 'target-edit-input';
      input.value = currentVal;
      input.placeholder = '30m, 1h…';
      badge.replaceWith(input);
      input.focus();
      input.select();

      let saved = false;
      function save() {
        if (saved) return;
        saved = true;
        task.dailyTargetMs = parseTargetInput(input.value);
        persist();
        renderTasks();
      }

      input.addEventListener('keydown', e => {
        if (e.key === 'Enter') save();
        if (e.key === 'Escape') { saved = true; renderTasks(); }
      });
      input.addEventListener('blur', save);
    }
```

- [ ] **Step 2: Wire up the click handler**

Find the task-list click handler (around line 794):

```js
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
```

Change to:

```js
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
        } else if (e.target.dataset.action === 'edit-target') {
          openTargetEdit(id);
        }
      });
```

- [ ] **Step 3: Verify inline editing in browser**

1. **Edit existing target:** Click the `⊙ Xm` badge on a task with a target. An input must appear prefilled with the current value. Type `45m`, press Enter. Badge must update to `⊙ 45m`.
2. **Clear target:** Click badge, clear input, press Enter. Badge disappears, `⊕` icon shows on hover.
3. **Set target on no-target task:** Hover over a task with no target, click the `⊕` icon. Input appears. Type `1h`, press Enter. Badge `⊙ 1h` appears.
4. **Escape cancels:** Click badge, change value, press Escape. Original value restored, no persist call.
5. **Persistence:** Reload page. All set targets must survive reload. Check `JSON.parse(localStorage.getItem('tf_tasks'))` — `dailyTargetMs` values must be present.

- [ ] **Step 4: Commit**

```bash
git add todo.html
git commit -m "feat: inline target editing via badge click"
```

---

## Self-Review Checklist

**Spec coverage:**
- [x] `dailyTargetMs` field on task → Task 2
- [x] `getDailyMs` clamped to midnight → Task 2
- [x] `.fill-bar` with CSS transition → Task 1 + Task 4
- [x] `filling` / `complete` classes → Task 1 + Task 4
- [x] Target badge amber (unmet) / green (met) → Task 1 + Task 4
- [x] `⊕` visible on hover for no-target tasks → Task 1 + Task 4
- [x] Target input at creation → Task 3
- [x] Inline edit after creation → Task 5
- [x] Escape cancels edit → Task 5 step 3
- [x] Timer runs past 100% → `Math.min(..., 100)` caps fill only, timer unaffected
- [x] Resets at midnight → `getDailyMs` always reads from midnight on read

**No placeholders:** All steps contain exact code.

**Type consistency:** `parseTargetInput` → `number | null` used in Task 3 (creation) and Task 5 (edit save). `formatTarget` → `string` used in Task 4 (badge) and Task 5 (prefill). `getDailyMs` → `number` used in Task 4. All consistent.

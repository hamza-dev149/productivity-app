# Project Plan — Productivity App

A small web app for managing tasks, running focus sessions with the Pomodoro
technique, and measuring progress over time.

---

## 1. Goal

Build a single-page productivity dashboard where a user can:

1. **Add, edit, and remove tasks** quickly.
2. **Run a timer / Pomodoro cycle** linked to a task.
3. **See progress** — task completion, focus time, Pomodoros, and streaks.

The app must work offline, keep data after a page reload, and feel fast on a
phone as well as a desktop.

**Out of scope for v1:** real user accounts, server/database, team sharing,
mobile app, payments. Login/register pages stay as static placeholders.

---

## 2. Current state

| Item | Status |
| --- | --- |
| `index.html`, `login.html`, `register.html`, `dashboard.html` | Empty files |
| `css/*.css` | Empty files |
| `javascript/main.js` | Empty file |
| `docs/project-plan.md` | This plan |
| `assests/icons/Glossy Orange Folded T Logo.png` | Logo available |
| Git repo | Initialised, 2 commits |

**Tech stack:** plain HTML5, CSS3 (custom properties, Flexbox/Grid), and
vanilla JavaScript (ES2020 modules). Data is stored in `localStorage`.
No build step and no framework for v1.

**Dev server:** use a local static server (e.g. VS Code Live Server or
`python3 -m http.server`) because ES modules do not load from `file://`.

---

## 3. Feature scope

### 3.1 Tasks — add / edit / remove

**Task fields**

| Field | Type | Rules |
| --- | --- | --- |
| `id` | string | `crypto.randomUUID()` |
| `title` | string | Required, trimmed, 1–120 chars |
| `notes` | string | Optional, max 500 chars |
| `priority` | enum | `low` \| `medium` \| `high` (default `medium`) |
| `estimatePomodoros` | number | 0–20, default 1 |
| `completedPomodoros` | number | Starts at 0 |
| `dueDate` | string \| null | `YYYY-MM-DD` |
| `status` | enum | `todo` \| `done` |
| `createdAt` / `updatedAt` | number | Unix ms |
| `completedAt` | number \| null | Set when marked done |

**Add**

- Input at the top of the task list: title + optional details (priority,
  estimate, due date).
- Press **Enter** or click **Add** to create.
- Empty/whitespace title → inline error, no task created.
- New task appears at the top of the list; list re-renders immediately.
- Input clears and keeps focus so several tasks can be added in a row.

**Edit**

- Click a task (or its edit button) to open an edit form (inline or modal).
- All fields editable; **Save** validates and updates `updatedAt`.
- **Cancel** / `Esc` discards changes.
- Editing title to empty is blocked with an inline error.

**Remove**

- Delete button on each task.
- Confirmation step (small inline "Delete? Yes / No" or a modal).
- After delete, show an **Undo** toast for ~5 seconds (restores the task).
- Deleting a task linked to a running timer keeps the session running and
  clears the link (`taskId = null`).

**Task list behaviour**

- Toggle done with a checkbox; done tasks move to a "Completed" section or
  sink to the bottom with a strikethrough.
- Filter tabs: **All**, **Active**, **Done**.
- Sort: newest first, due date, priority.
- Counter: "3 of 8 done".
- Empty state: friendly message + hint to add the first task.
- Data survives reload via `localStorage`.

**Edge cases to handle**

- Very long titles (wrap, do not break layout).
- Duplicate titles are allowed (IDs are unique).
- Invalid/absent due date.
- Corrupt or missing `localStorage` data → fall back to `[]` and log a warning.
- Multi-tab edits: listen to the `storage` event and re-render (nice-to-have).

### 3.2 Timer and Pomodoro

**Modes**

| Mode | Default duration |
| --- | --- |
| Focus | 25 min |
| Short break | 5 min |
| Long break | 15 min |

Durations are configurable in Settings.

**Controls**

- **Start / Pause / Resume**
- **Reset** (back to full duration of current mode)
- **Skip** (jump to the next mode)
- Mode switcher: Focus / Short break / Long break

**Pomodoro cycle logic**

1. Start a Focus session.
2. When it finishes → short break.
3. After every **4 completed focus sessions** → long break instead.
4. Long break ends → cycle counter resets.
5. Optional auto-start for breaks and for the next focus session.
6. Show cycle dots, e.g. `● ● ○ ○` (4 per long break).

**Linking a task**

- The user can select a task for the current focus session.
- A completed focus session increments that task's `completedPomodoros`.
- The timer panel shows the linked task title.

**Accuracy and persistence**

- Count down using an end timestamp (`endsAt`), not by decrementing a counter,
  so background tabs and throttling do not cause drift.
- Persist timer state (`mode`, `remainingSeconds`, `isRunning`, `endsAt`,
  `cycleCount`, `taskId`) so a reload restores the session.
- On reload while running: recompute remaining time from `endsAt`; if it has
  already ended, complete the session once (no duplicate credits).

**Alerts**

- On finish: play a short sound (Web Audio API, no audio file needed) and,
  if permission is granted, show a browser notification.
- Update the page title with the remaining time while running, e.g.
  `24:59 — Focus`.
- Flash or highlight the timer card when a session completes.

**Settings**

| Setting | Default |
| --- | --- |
| Focus duration | 25 min |
| Short break | 5 min |
| Long break | 15 min |
| Sessions before long break | 4 |
| Auto-start breaks | off |
| Auto-start focus | off |
| Sound | on |
| Notifications | off |
| Daily focus goal | 120 min |

**Edge cases to handle**

- Switching mode while running → confirm, then reset cleanly.
- Closing/reloading the tab mid-session → restore state.
- System clock changes → clamp remaining time to `[0, duration]`.
- Timer must never credit the same session twice.
- Zero/negative/invalid settings values → validate and clamp.
- Long-running sessions across midnight → attribute focus minutes to the
  actual day the session ended (or split, but pick one rule and document it).

### 3.3 Measuring progress

**Metrics (definitions must be exact)**

| Metric | Formula / rule |
| --- | --- |
| Task completion rate | `doneTasks / totalTasks × 100` (0 if no tasks) |
| Today's tasks done | Tasks with `completedAt` on today's local date |
| Focus minutes today | Sum of `durationSeconds` of completed focus sessions today ÷ 60 |
| Pomodoros today | Count of completed focus sessions today |
| Daily goal progress | `focusMinutesToday / dailyGoalMinutes × 100`, capped at 100 |
| Estimate accuracy | `completedPomodoros / estimatePomodoros` per task (if estimate > 0) |
| Streak | Consecutive local days (ending today or yesterday) with ≥ 1 completed focus session |
| Weekly focus | Focus minutes per day for the last 7 days |

**Where it is shown**

- **Stat cards** at the top of the dashboard:
  Today's focus time, Pomodoros today, Tasks done, Current streak.
- **Progress ring / bar** for the daily focus goal.
- **Task progress bar**: "3 of 8 done — 37%".
- **7-day bar chart** of focus minutes (pure CSS bars or `<canvas>`; no chart
  library needed for v1).
- **Per-task progress**: `completedPomodoros / estimatePomodoros`.

**Data source**

- Keep an append-only session log so stats are recalculated rather than
  guessed:
  ```js
  Session = {
    id, taskId, type: "focus" | "shortBreak" | "longBreak",
    startedAt, endedAt, durationSeconds, completed: boolean
  }
  ```
- Recompute all stats from tasks + sessions on load; cache in memory only.
- "Today" uses the browser's local timezone and rolls over at local midnight.

---

## 4. Data model (localStorage)

| Key | Contents |
| --- | --- |
| `pa.tasks.v1` | `Task[]` |
| `pa.sessions.v1` | `Session[]` |
| `pa.settings.v1` | Settings object |
| `pa.timer.v1` | Persisted timer state |

Rules:

- Every read goes through `storage.js` with `try/catch` and a schema/version
  check; invalid data is reset to defaults (never crash the UI).
- Every write updates the in-memory state first, then saves.
- Add a `version` field so migrations are possible later.
- Keep writes debounced where they happen often (e.g. timer ticks).

```js
// Task
{
  id: "uuid",
  title: "Write project plan",
  notes: "",
  priority: "medium",
  estimatePomodoros: 2,
  completedPomodoros: 0,
  dueDate: null,
  status: "todo",
  createdAt: 1730000000000,
  updatedAt: 1730000000000,
  completedAt: null
}
```

---

## 5. Pages and layout

| File | Purpose |
| --- | --- |
| `index.html` | Landing / marketing page with a "Get started" CTA |
| `login.html` | Static login form (no real auth in v1) |
| `register.html` | Static register form (no real auth in v1) |
| `dashboard.html` | The app: timer panel + task list + stats |

**Dashboard layout**

- Desktop: two columns — left = timer + today's stats, right = task list.
- Mobile: single column, timer first, task list below.
- Sticky timer card so it stays visible while scrolling tasks.
- Header with logo, app name, and a settings button.
- Settings panel as a modal/drawer.

**Design tokens** (`css/style.css`)

- Colour palette derived from the orange logo (orange accent + neutral greys).
- CSS variables for colour, spacing, radius, font size, shadow.
- System font stack; no external font dependency.
- Dark mode is a nice-to-have (Phase 6), not required for v1.

**Accessibility**

- All controls reachable by keyboard; visible focus ring.
- Labels on every input; buttons have text or `aria-label`.
- Timer uses `aria-live="polite"` (announce sparingly, not every second).
- Colour contrast ≥ 4.5:1; do not rely on colour alone for priority.
- Respect `prefers-reduced-motion`.

---

## 6. File structure (target)

```
productivity-app/
├── index.html
├── login.html
├── register.html
├── dashboard.html
├── css/
│   ├── style.css        # reset, tokens, shared components
│   ├── home.css
│   ├── login.css
│   ├── register.css
│   └── dashboard.css    # timer, tasks, stats
├── javascript/
│   ├── main.js          # entry point, wiring
│   ├── storage.js       # load/save, schema guards
│   ├── tasks.js         # CRUD + filtering/sorting
│   ├── timer.js         # pomodoro state machine
│   ├── stats.js         # progress calculations
│   └── ui.js            # rendering + DOM helpers
├── assests/             # existing folder name (kept to avoid broken paths)
│   ├── icons/
│   └── images/
└── docs/
    └── project-plan.md
```

> Note: the folder is currently spelled `assests`. Keep it for now so the logo
> path keeps working; renaming to `assets` is an optional cleanup task that must
> update all references.

---

## 7. Implementation phases

### Phase 0 — Setup and design (≈ half a day)

- [ ] Define colour palette, spacing scale, typography in `css/style.css`.
- [ ] Build shared components: buttons, inputs, cards, modal, toast.
- [ ] Set up `storage.js` with safe load/save and defaults.
- [ ] Confirm local dev server workflow.

### Phase 1 — Static dashboard layout (≈ 1 day)

- [ ] Header, timer card, task list card, stats row.
- [ ] Responsive layout for mobile and desktop.
- [ ] Empty states and loading skeleton (optional).
- [ ] Wire navigation between `index`, `login`, `register`, `dashboard`.

### Phase 2 — Task CRUD (≈ 1–2 days)

- [ ] Add task with validation.
- [ ] Edit task (inline or modal) with cancel.
- [ ] Delete with confirm + undo toast.
- [ ] Toggle done, filters, sort, counter.
- [ ] Persist to `localStorage`; restore on reload.
- [ ] Handle corrupt/missing storage gracefully.

### Phase 3 — Timer and Pomodoro (≈ 2 days)

- [ ] Timer state machine: idle → running → paused → finished.
- [ ] Timestamp-based countdown (no drift).
- [ ] Start/pause/reset/skip and mode switching.
- [ ] Pomodoro cycle: short/long break after 4 focus sessions.
- [ ] Link a task; increment `completedPomodoros` once per finished session.
- [ ] Persist/restore timer state across reloads.
- [ ] Sound, notification, page-title countdown.
- [ ] Settings panel with validation.

### Phase 4 — Progress tracking (≈ 1–2 days)

- [ ] Session log written on every completed focus session.
- [ ] `stats.js` calculations (completion rate, today's focus, Pomodoros,
      goal progress, streak, weekly series).
- [ ] Stat cards + daily goal ring.
- [ ] 7-day bar chart.
- [ ] Task progress bar and per-task estimate progress.
- [ ] Recalculate on task/session changes and at midnight.

### Phase 5 — Polish and QA (≈ 1 day)

- [ ] Accessibility pass (keyboard, labels, contrast, reduced motion).
- [ ] Cross-browser check (Chrome, Firefox, Safari, mobile).
- [ ] Performance: avoid full re-render where it causes flicker.
- [ ] Error handling and user-facing messages.
- [ ] Write short usage notes in the README.

### Phase 6 — Optional / future

- [ ] Dark mode.
- [ ] Drag-and-drop reordering.
- [ ] Recurring tasks and reminders.
- [ ] Export/import JSON backup.
- [ ] Real authentication + backend sync (replaces placeholder login pages).
- [ ] PWA install + offline caching.

**Estimated total for v1:** ~6–8 working days.

---

## 8. Testing checklist

**Tasks**

- [ ] Add with empty title is blocked.
- [ ] Add, edit, delete, undo all persist after reload.
- [ ] Filters and sorting show the correct tasks.
- [ ] Deleting a task linked to the timer does not break the timer.
- [ ] 100+ tasks still render and scroll smoothly.

**Timer**

- [ ] Pause/resume keeps the correct remaining time.
- [ ] Switching tabs for 10 minutes does not drift.
- [ ] Reload mid-session restores the timer.
- [ ] Finished session is credited exactly once.
- [ ] Long break appears after the 4th focus session.
- [ ] Skip and reset always leave a valid state.
- [ ] Invalid settings are rejected/clamped.

**Progress**

- [ ] Completion rate matches the task list.
- [ ] Focus minutes equal the sum of completed sessions.
- [ ] Streak counts consecutive days correctly, including across midnight.
- [ ] Weekly chart has 7 buckets and today is the last one.
- [ ] No division-by-zero when there are no tasks or no sessions.

**General**

- [ ] Works with `localStorage` cleared and with corrupt JSON.
- [ ] Layout usable at 320 px, 768 px, and 1440 px widths.
- [ ] Keyboard-only navigation works for all main actions.
- [ ] No console errors.

---

## 9. Definition of done (v1)

- A user can add, edit, complete, and delete tasks, and the data survives reload.
- A user can run a full Pomodoro cycle with focus and break modes, linked to a
  task, with sound/notification on completion.
- The dashboard shows accurate task completion, focus minutes, Pomodoros,
  daily goal progress, and a 7-day chart.
- The app is responsive, keyboard accessible, and has no blocking console errors.
- The plan and a short README describe how to run the project.

---

## 10. Risks and decisions

| Risk / decision | Mitigation |
| --- | --- |
| Timer drift in background tabs | Use `endsAt` timestamps, recompute on tick/visibility change |
| `localStorage` data loss or corruption | Schema versioning, `try/catch`, defaults, optional export |
| Double-crediting a session on reload | Store a `completed` flag + session id; check before writing |
| Scope creep (auth, backend) | Keep v1 local-only; placeholders stay static |
| Vanilla JS becoming messy | Split modules by responsibility (tasks/timer/stats/ui) |
| Folder typo `assests` | Keep for v1; rename later with a full reference update |

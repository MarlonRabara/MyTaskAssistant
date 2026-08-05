# MyTaskAssistant

**Version:** v4.6.8

A local-first, single-file task manager for organizing projects, tracking effort, monitoring AI-assisted productivity, and producing weekly HTML timesheets. No installation, no backend, no database, no account — just open one HTML file in a modern browser.

## Quick Start

1. Clone or download this repository.
2. Double-click `MyTaskAssistant.html` (or open it in Chrome, Edge, or another modern desktop browser).
3. Click **+ Add Task** to start, or **Load Tasks** to load an existing saved task file.
4. Click **Save** to persist your tasks to a local JSON file. The browser may ask for permission to read/write the file — this is expected for local file handling.

> Tip: A Chromium-based browser (Chrome/Edge) is recommended for the best save experience via the File System Access API.

## Key Features

- **Projects and tasks** — organize tasks by project, with subtasks, parent/child relationships, cloning, statuses (including Waiting, Blocked, Deferred, Canceled, and Closed), priorities, due dates, and explicit completion dates.
- **Clear top actions** — use **Load Tasks** to open a saved task file. Every top-toolbar action includes a hover tooltip that explains what it does.
- **Mirrored bottom actions** — the main toolbar commands, including **+ Add Task**, are repeated below the task grid and call the same command handlers as the top toolbar.
- **Unsaved indicator** — task additions, deletions, and modifications immediately show a red **(unsaved)** marker beside **My Tasks** and in the left-panel JSON file area until Save completes successfully.
- **Parent task rollups** — parent Estimated Hours, Time Spent, and Estimated Effort Without AI are read-only sums of all child and nested-child tasks. Parent AI Assistance is a read-only Estimated Effort Without AI-weighted average. A parent cannot be completed until every child and nested child is complete.
- **Terminal-only reporting** — rollups, Weekly Timesheet, and AI Analysis count terminal work tasks only. Parent and intermediate summary tasks never add duplicate hours or AI savings to report totals.
- **Parent status summary** — parent rows use a compact format such as `In Progress · 1/3`; hover over it for a plain-language explanation, calculated progress, and terminal status counts.
- **Dependency details and task-grid scrolling** — hover or focus a dependency indicator to view dependency items and their statuses. The task grid supports horizontal scrolling when the browser is narrower than the table.
- **Readable grid values** — Hours and AI Hours Saved display to one decimal place; AI Hours Saved stays on one line without bold emphasis. The compact **Due** column retains a readable standard font size and shows a self-contained green checkmark when a completed task was on time (or has no due date), or a sad-face icon when it was completed late. The grid does not show a separate Completed date column; hover the icon for completion details.
- **Clear hierarchy markers** — a green leaf beside Priority identifies a childless sub-task; a brown branch identifies a parent task with child tasks; and a blue standalone-item icon identifies a task with neither a parent nor child tasks. Hover a marker to see its parent-child relationship, including the direct child count or parent task name.
- **Task-row actions** — Actions are ordered Goto Link, Clone, Create Child Task, Edit, Archive/Unarchive, and Delete. Goto Link is blue when Notes has a hyperlink, opens the first link in a new window, and is disabled when Notes has no link.
- **Archive** — archive a task after an **Are you sure?** confirmation that names the task and shows how many descendants will be included. Archiving a parent archives all descendants so completed task sets can be retired together. Archived tasks disappear from normal views, reports, and exports, and can be restored from the left-panel **Archive** view with the Unarchive action. Cloning an archived child or adding a child to an archived parent automatically restores the parent chain.
- **Filters and views** — search, project/status/due-date filters (including Due Yesterday, Due Last Week, and Due Next Month), Active/Completed/Overdue navigation, Most Recent and Recently Created views, an Archive view, and hide-completed toggle.
- **Most Recent** — a left-panel view showing up to 10 tasks with the newest creation dates, newest first, after active filters are applied. Hover over it in the app to see the filtering criteria.
- **Recently Created** — a left-panel view showing tasks created during the rolling previous 24 hours, newest first, after active filters are applied.
- **Due Next Month** — a Due Date dropdown option that shows tasks due in the next calendar month. It is not a left-panel navigation view.
- **Due Last Week** — a Due Date dropdown option that shows tasks due in the complete Sunday-through-Saturday week before the current week.
- **Sub-Tasks tab** — when editing a saved parent task, select **Sub-Tasks** beside **Task Detail** to replace the detail form with full-width rows for direct child tasks. Each row shows title, status, and Time Spent; select a row to load that child into the same task editor window.
- **Task editor layout** — below AI Assistance, Parent Task and Dependencies are grouped in the left column, while Due Date and Completed Date are grouped in the right column. Priority is immediately above Notes, and Time Spent aligns with Estimated Effort Without AI using equal-height controls.
- **Terminal validation** — Completed tasks require both Estimated Hours and Time Spent to be greater than zero. Closed tasks may have zero effort. A positive AI Assistance percentage requires a positive Estimated Effort Without AI value; estimates may validly be lower than, equal to, or greater than Time Spent.
- **Terminal synchronization** — setting progress to 100% defaults Status to Completed only when Closed was not explicitly selected; setting Status to Completed or Closed sets progress to 100%. An explicit Closed status is preserved during completion-date and progress synchronization.
- **Completion dates** — a Completed or Closed task with no completion date is assigned today's local date. Changing to a non-terminal status clears the date and automatically reduces 100% progress to 99% so the task is no longer terminal. Entering a completion date of today or earlier automatically sets Status to Completed and progress to 100%.
- **Progress validation** — a task with progress greater than 0% cannot be saved with a Not Started status; select an active status before recording progress.
- **Notes** — task rows preview up to five note lines; hover a note to view its complete text. A bold **...(see more)** link appears for longer notes and opens the task editor; the editor provides an Expand Notes control for comfortable long-note reading and editing.
- **Time and AI productivity tracking** — per task, capture:
  - `estimatedHours` — your original estimate
  - `timeSpent` — actual hours worked
  - `aiAssistancePercentage` — how much AI assisted (0–100%)
  - `estimatedEffortWithoutAI` — hours the task would have taken without AI
- **AI Efficiency** — the statistics section displays weighted AI efficiency for qualifying completed or Closed tasks: total estimated time saved divided by total AI-assisted hours. The AI Analysis view also reports estimated time saved, AI-assisted hours, overall time reduction, productivity multiplier, AI utilization, and qualifying-task count. Negative time-saved results remain visible when AI-assisted work exceeded the estimated effort without AI.
- **AI Hours Saved** — computed as `estimatedEffortWithoutAI − timeSpent`; MyTaskAssistant measures AI *productivity*, not AI usage.
- **🤖 AI Analysis** — a read-only weekly view for qualifying non-archived Completed or Closed AI-assisted terminal tasks. It reports weighted AI Efficiency, AI-Assisted Hours, Estimated Time Saved, Overall Time Reduction, Productivity Multiplier, AI Utilization, and qualifying-task count in a downloadable standalone HTML report.
- **Weekly Timesheet** — generate a printable, standalone HTML timesheet for any Sunday-through-Saturday week from completed non-archived terminal tasks, with editable daily allocations initially placed on each task's completion date.
- **Exports and reports** — export the current filtered view or all active non-archived tasks as JSON, or use **Export Report** to download a standalone HTML report bound to the same current-view task set and hierarchy order. Archived tasks are included only from the Archive view.

## Data Format

Task data lives in a single human-readable JSON file that you choose. Time-capture fields look like:

```json
{
	"completedDate": "2026-03-20",
	"archived": false,
  "estimatedHours": 8,
  "timeSpent": 5.5,
  "aiAssistancePercentage": 40,
  "estimatedEffortWithoutAI": 9.5
}
```

Older files are supported. Load Tasks accepts `tasks`, `taskList`, `items`, or a top-level task array and normalizes common aliases including `hoursSpent`, `actualEffort`, `percentageOfAIAssistance`, `estimatedHumanOnlyEffort`, `estimatedHumanEffort`, `taskStatus`, `progress`, `due`, `targetDate`, `description`, `detail`, `dependencies`, and `parentTaskId`. Imported tasks without `archived` default to `false`. Imported Completed tasks without `completedDate` use their due date when available. Saves always use the current schema.

## Repository Contents

| File | Purpose |
| --- | --- |
| `MyTaskAssistant.html` | The application. Open this file in your browser. |
| `ReadMe.md` | This overview for repository browsers. |
| `ReadMe.html` | The full user guide (open in a browser). |
| `specs/MyTaskAssistant_Specification.md` | Technical and functional specification for maintainers. |
| `testdata/Mortgage_Servicing_TestData.json` | Synthetic mortgage and servicing fixture for testing hierarchy, dates, reports, AI productivity, dependencies, and long notes. |

## Privacy

MyTaskAssistant runs entirely locally in your browser. It does not send task data to any remote service. Your data stays in the JSON file you choose.

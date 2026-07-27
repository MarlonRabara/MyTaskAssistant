# MyTaskAssistant

**Version:** v3.5.4

A local-first, single-file task manager for organizing projects, tracking effort, monitoring AI-assisted productivity, and producing weekly HTML timesheets. No installation, no backend, no database, no account — just open one HTML file in a modern browser.

## Quick Start

1. Clone or download this repository.
2. Double-click `MyTaskAssistant.html` (or open it in Chrome, Edge, or another modern desktop browser).
3. Click **+ Add Task** to start, or **Open JSON** to load an existing data file.
4. Click **Save** to persist your tasks to a local JSON file. The browser may ask for permission to read/write the file — this is expected for local file handling.

> Tip: A Chromium-based browser (Chrome/Edge) is recommended for the best save experience via the File System Access API.

## Key Features

- **Projects and tasks** — organize tasks by project, with subtasks, parent/child relationships, cloning, statuses, priorities, and due dates.
- **Parent task rollups** — parent Estimated Hours, Time Spent, and Estimated Effort Without AI are read-only sums of all child and nested-child tasks. Parent AI Assistance is a read-only Estimated Effort Without AI-weighted average. A parent cannot be completed until every child and nested child is complete.
- **Filters and views** — search, project/status/due-date filters (including Due Next Month), Active/Completed/Overdue navigation, a Most Recent view, and hide-completed toggle.
- **Most Recent** — a left-panel view showing up to 10 tasks with the newest creation dates, newest first, after active filters are applied. Hover over it in the app to see the filtering criteria.
- **Due Next Month** — a Due Date dropdown option that shows tasks due in the next calendar month. It is not a left-panel navigation view.
- **Time and AI productivity tracking** — per task, capture:
  - `estimatedHours` — your original estimate
  - `timeSpent` — actual hours worked
  - `aiAssistancePercentage` — how much AI assisted (0–100%)
  - `estimatedEffortWithoutAI` — hours the task would have taken without AI
- **AI Hours Saved** — computed as `estimatedEffortWithoutAI − timeSpent`; MyTaskAssistant measures AI *productivity*, not AI usage.
- **🤖 AI Analysis** — a read-only weekly view summarizing AI-assisted tasks: Time Spent, Estimated Effort Without AI, and AI Hours Saved, with totals and a downloadable standalone HTML report.
- **Weekly Timesheet** — generate a printable, standalone HTML timesheet for any Sunday-through-Saturday week with editable daily allocations.
- **JSON exports** — export the current filtered view or all active tasks as JSON.

## Data Format

Task data lives in a single human-readable JSON file that you choose. Time-capture fields look like:

```json
{
  "estimatedHours": 8,
  "timeSpent": 5.5,
  "aiAssistancePercentage": 40,
  "estimatedEffortWithoutAI": 9.5
}
```

Older files using legacy keys (`hoursSpent`, `percentageOfAIAssistance`, `estimatedHumanOnlyEffort`, `estimatedHumanEffort`, `actualEffort`) are mapped automatically on import; saves always use the current schema.

## Repository Contents

| File | Purpose |
| --- | --- |
| `MyTaskAssistant.html` | The application. Open this file in your browser. |
| `ReadMe.md` | This overview for repository browsers. |
| `ReadMe.html` | The full user guide (open in a browser). |
| `specs/MyTaskAssistant_Specification.md` | Technical and functional specification for maintainers. |

## Privacy

MyTaskAssistant runs entirely locally in your browser. It does not send task data to any remote service. Your data stays in the JSON file you choose.

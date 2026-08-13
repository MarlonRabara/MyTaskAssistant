# MyTaskAssistant Specification
**Application Version:** v4.7.16
**Last Updated:** 2026-08-01

## Overview

MyTaskAssistant is a lightweight, local-first task management application.

Design goals:

- Single self-contained HTML application
- Single JSON data file
- No backend
- No database
- No authentication
- Offline-first
- Human-readable JSON
- Easy for both humans and AI assistants to maintain

---

# Versioning and File Naming

The visible version displayed in the application header is the authoritative application version.

The application, specification, and README files are under source control and are updated in place. Version-suffixed filename copies are no longer generated.

The source-controlled filenames are:

- `MyTaskAssistant.html`
- `specs/MyTaskAssistant_Specification.md`
- `ReadMe.html` / `ReadMe.md`

When the application version changes:

1. Update the visible version badge in the HTML.
2. Update `metadata.applicationVersion` when JSON is saved.
3. Update the specification's application version.
4. Do not create new version-suffixed filenames; source control history tracks versions.

Legacy sequence-based names such as `Final_v9` should not be used.

---

# Architecture

## Application

- HTML
- Embedded CSS
- Embedded JavaScript
- Embedded SVG icons
- Self-contained favicon
- No external libraries required

## Self-Contained Favicon

The application includes a favicon in the HTML document using a `data:` URL so no external image file is required.

Requirements:

- The favicon is declared in the `<head>` using a `<link rel="icon">` element.
- The favicon uses an inline SVG encoded as a `data:image/svg+xml` URL.
- The icon visually reflects MyTaskAssistant's purpose as a personal task manager.
- The visual metaphor should include task-oriented imagery such as a checklist, checkmark, task card, or productivity assistant mark.
- The favicon must remain self-contained and must not reference external `.ico`, `.png`, `.svg`, or web resources.

## Data

The application uses a single JSON file.

The HTML is the application.
The JSON is the persistent data and source of truth for task content.

## Runtime System State

The application maintains an in-memory `saved` state that indicates whether the current browser session has unsaved task-record changes.

Rules:

- The `saved` state is runtime-only and must never be written to the JSON data file.
- On initial application load, before a data file is opened, `saved` is `true`.
- Immediately after a task data file is loaded, `saved` is `true`.
- When the Save action successfully writes the task data file, `saved` is set to `true`.
- The following task-record changes immediately set `saved` to `false`:
  - Adding a task
  - Deleting a task
  - Modifying a task
- Task modification includes changes made through the task editor, completion checkbox, priority reordering, archive/unarchive action, clone creation, child-task creation, parent/dependency changes, and any other operation that changes a task record.

---

# Task Model

Each task contains:

- `id` (immutable)
- `title`
- `projectId`
- `parentId`
- `dependsOn[]`
- `status`
- `percentComplete`
- `estimatedHours`
- `timeSpent`
- `estimatedEffortWithoutAI`
- `aiAssistancePercentage`
- `priorityOrder`
- `dueDate`
- `completedDate`
- `archived`
- `notes`
- `createdAt`
- `updatedAt`

Unlimited hierarchy is supported using `parentId`.

## Parent Task Rollups and Completion

Tasks with one or more direct child tasks are parent tasks. The application calculates parent values from terminal descendants only: a terminal task has no children. Intermediate parent values and stored direct parent values are excluded from rollups, which prevents double counting. Calculated values are displayed dynamically and are not persisted as duplicate rollup data.

For a parent task, these editor fields are read-only and show calculated terminal-descendant values:

- `estimatedHours`
- `timeSpent`
- `estimatedEffortWithoutAI`

`percentComplete`, `status`, and `aiAssistancePercentage` are also read-only for a parent. Progress is Estimated Hours-weighted when every terminal descendant has Estimated Hours; otherwise it uses a task-count average. Parent AI Assistance is calculated as the Estimated Effort Without AI-weighted average of terminal descendant AI Assistance percentages with a positive Estimated Effort Without AI value:

```text
Σ (child AI Assistance % × child Estimated Effort Without AI)
÷ Σ child Estimated Effort Without AI
```

The parent slider tooltip identifies the number of included terminal tasks, total Estimated Effort Without AI, formula, and calculated percentage. Missing numeric values contribute zero to summed rollups.

Parent rows use calculated Estimated Hours, Time Spent, Estimated Effort Without AI, AI Assistance, AI Hours Saved, Percent Complete, and Status. Parent AI Hours Saved is the sum of individually valid terminal-task AI Hours Saved values; incomplete AI data is excluded and identified in the tooltip. Parent status is derived from terminal statuses and shown as a compact summary, such as `In Progress · 1/3`, with a detailed tooltip. Parent direct stored values remain unchanged when the parent is saved and become editable again if all children are removed.

A parent task cannot be marked `Completed` until every terminal descendant task is complete. This is enforced through the task-row completion checkbox, the task-editor Mark Complete actions, and saving the parent with status `Completed` or 100% progress. The parent completion controls are disabled while terminal work remains incomplete and direct the user to the Sub-Tasks tab.

## Closed Status

`Closed` is a terminal task status for work that has been intentionally declined, canceled without completion, determined to be unnecessary, or otherwise ended without successful completion. It lets users close a task without representing it as successfully completed.

- `Closed` is available anywhere a task status can be selected.
- Setting Status to `Closed` sets `percentComplete` to `100` and assigns `completedDate` to the current local date when it is blank, using the same synchronization behavior as `Completed`.
- A task whose status is `Closed` is treated as complete for completion checks, parent rollups, parent completion eligibility, active/completed filtering, overdue calculations, export/report inclusion, and dependency-satisfaction logic.
- `Completed` and `Closed` are separate terminal statuses. A task at `100%` progress may have either status.
- Selecting `Closed` is authoritative: after the user selects and saves `Closed`, the application must preserve `status: "Closed"` and `percentComplete: 100`. It must not silently rewrite the status to `Completed` during change handling, completion-date synchronization, validation, rendering, normalization, saving, loading, filtering, reporting, or parent-rollup calculation.
- Setting progress to `100%` without an explicit terminal-status choice may set Status to `Completed` as the default. This default applies only when the task is not already `Closed`; it must never overwrite an existing or explicitly selected `Closed` status.
- Entering or changing `completedDate` may default a non-terminal task to `Completed`, but it must preserve `Closed` when the task is already Closed or Closed is selected in the editor.
- Moving a `Closed` task to any non-terminal status clears `completedDate` and reduces `percentComplete` from `100` to `99`, matching the existing behavior for moving a completed task back to active work.
- Unlike `Completed`, `Closed` does not require `estimatedHours`, `timeSpent`, or AI-productivity fields to be present or greater than zero. Existing entered effort values are retained and remain available for historical reporting.
- A parent task may be marked `Closed` only when every terminal descendant is in a complete terminal state (`Completed` or `Closed`). Parent rollups continue to derive status and progress from terminal descendants.
- Task-grid and report status presentation must display `Closed` distinctly from `Completed`; it must not use language or icons that imply the task was successfully completed.

## Terminal Status AI Validation

Before a terminal task is saved as `Completed` or `Closed`, including through a task-row completion command, a task-editor action, Status selection, or setting progress to `100%`, the application validates AI productivity fields.

- When `estimatedEffortWithoutAI` is present and greater than zero, `aiAssistancePercentage` must be greater than `0`.
- A terminal task with an Estimated Effort Without AI value and AI Assistance of `0%`, blank, missing, or otherwise non-positive must be rejected with a clear validation message.
- The user must either enter a positive AI Assistance percentage or clear the Estimated Effort Without AI value before completing or closing the task.
- Existing validation remains in force: whenever AI Assistance is greater than `0%`, Estimated Effort Without AI is required and must be greater than zero.
- This validation applies to terminal work tasks. Parent terminal-status eligibility remains governed by descendant completion and calculated rollup behavior.

## AI Productivity Fields

### Estimated Hours

`estimatedHours`

- Decimal hours
- The planned estimate for the task
- Optional

### Estimated Effort Without AI

`estimatedEffortWithoutAI`

- Decimal hours
- Estimate of how long the task would have taken if AI had not been available
- Optional when AI Assistance is 0%
- Required whenever AI Assistance is greater than 0%
- Must be greater than Time Spent whenever AI Assistance is greater than 0%
- Must be greater than zero to calculate AI Hours Saved
- Editor label: `Estimated Effort Without AI (hours)`
- Editor helper text: `Estimate how long this task would have taken if AI had not been available. Required whenever AI Assistance is greater than 0%.`

Backward compatibility: older JSON files may contain `estimatedHumanOnlyEffort` (or the earlier `estimatedHumanEffort`). On import, those values are automatically mapped into `estimatedEffortWithoutAI` so no data is lost. When saving JSON, only `estimatedEffortWithoutAI` is written; the legacy property names are never saved again.

### Actual (Time Spent)

`timeSpent`

- Decimal hours
- Represents the actual effort required to complete or work on the task
- Optional
- Must be less than Estimated Effort Without AI whenever AI Assistance is greater than 0%

### Percentage of AI Assistance

`aiAssistancePercentage`

- Whole-number percentage from 0 through 100
- The estimated percentage of recorded Time Spent that was performed with AI assistance
- Record AI Assistance only when AI produced measurable productivity gains
- Captured through a slider
- Slider step is 5%
- Defaults to 0%
- Stored as a whole number

### AI-Assisted Time Portion

`aiAssistancePercentage` must be used to calculate the AI-assisted portion of a task's recorded Time Spent. This value is calculated at runtime and is not stored separately in JSON.

```text
AI-Assisted Time = Time Spent × (AI Assistance Percentage ÷ 100)
```

Example:

```text
Estimated Effort Without AI: 100 hours
Time Spent: 10 hours
AI Assistance Percentage: 90%

AI-Assisted Time: 10 × 0.90 = 9 hours
```

The remaining portion of recorded time is the non-AI-assisted portion:

```text
Non-AI-Assisted Portion = Time Spent − AI-Assisted Time
```

For the example above, the non-AI-assisted portion is `1 hour`. This does not mean that a human was absent from the AI-assisted portion; the task remains human-directed work in which AI assisted with the stated percentage of the recorded effort. When Time Spent is greater than zero and AI Assistance is `0%`, all recorded Time Spent is human-only time. AI Assistance percentage is not a replacement for Time Spent and must not be interpreted as a percentage of Estimated Effort Without AI.

### Task Editor Layout

Below the Percentage of AI Assistance slider, the editor places Parent Task above Dependencies in the left column and Due Date above Completed Date in the right column. Priority order appears immediately above Notes. Time Spent and Estimated Effort Without AI are presented together in aligned columns with equal-height input controls.

### Validation

When a task is saved with AI Assistance greater than 0%:

- `estimatedEffortWithoutAI` is required
- `estimatedEffortWithoutAI` must be greater than zero
- `estimatedEffortWithoutAI` may be less than, equal to, or greater than Time Spent (`timeSpent`)

An estimate below Time Spent produces a valid negative time-saved and AI-efficiency result. An estimate equal to Time Spent produces a valid zero time-saved and AI-efficiency result. The application must not reject either result or coerce negative calculated values to zero.

Validation message when the estimate is missing or non-positive:

```text
Estimated Effort Without AI must be greater than zero whenever AI Assistance is greater than 0%.

AI Assistance represents measurable productivity gains.
```

These fields appear in the Add/Edit Task dialog but do not appear as individual columns in the task row.

---

# AI Hours Saved

The task grid includes a compact icon-only Robot column.

## Column Header

- Displays only a robot SVG icon
- Tooltip:

```text
AI Hours Saved is calculated automatically.

AI Hours Saved =
Estimated Effort Without AI − Time Spent.

Represents the estimated productivity gained through meaningful AI assistance.
```

- Accessible label: `AI Hours Saved`

## Calculation

AI Hours Saved is calculated at runtime and is not stored in JSON.

```text
AI Hours Saved =
Estimated Effort Without AI - Time Spent
```

AI Hours Saved is also called **Estimated Time Saved** in aggregate AI-efficiency metrics. It may be positive, zero, or negative; negative values indicate the task took longer with AI assistance than the estimated effort without AI.

Example:

```text
Estimated Effort Without AI: 9.5 hours
Actual (Time Spent): 5.5 hours
Percentage of AI Assistance: 40%

AI Hours Saved: 9.5 - 5.5 = 4.00 h
```

## Display Rules

The Robot column displays AI Hours Saved in decimal hours.

- If both effort fields are absent and AI assistance resolves to `0%`, as with normalized legacy JSON, display `0.00 h`.
- If some AI-tracking data exists but Estimated Effort Without AI is missing or zero, display blank.
- If some AI-tracking data exists but Actual (Time Spent) is missing, display blank.
- If Percentage of AI Assistance is zero, blank, missing, `null`, or otherwise not captured, treat it as `0%` and display `0.00 h` once the effort baseline is available.
- If all required values are present, calculate and display AI Hours Saved, including negative values.
- The value is displayed with two decimal places followed by `h`.

## Tooltip

Hovering over a displayed AI Hours Saved value shows a native browser tooltip containing:

- AI Hours Saved
- The automatic-calculation explanation
- Estimated Effort Without AI
- Actual (Time Spent)
- Percentage of AI Assistance
- The calculation used
- A note that the value represents the estimated productivity gained through meaningful AI assistance

When AI assistance is zero or was not captured, the tooltip explains that the default assumption is that AI produced no measurable productivity gain and therefore AI Hours Saved is `0.00 h`.

For legacy tasks where none of the AI-tracking fields exist, the tooltip explains that no AI-productivity tracking data was captured and the default assumption is that AI produced no measurable productivity gain.

Example tooltip:

```text
AI Hours Saved: 4.00 h

AI Hours Saved is calculated automatically.

AI Hours Saved =
Estimated Effort Without AI − Time Spent.

Estimated Effort Without AI: 9.50 h
Actual (Time Spent): 5.50 h
AI Assistance: 40%

Calculation:
9.50 − 5.50 = 4.00 h

Represents the estimated productivity gained through meaningful AI assistance.
```

---

# AI Efficiency

## Purpose and Source Fields

AI Efficiency measures estimated time saved relative to actual work hours during which AI materially assisted the user. All task and aggregate calculations use only the authoritative source fields below; derived values are calculated at runtime, use full numeric precision, and are never stored as independent source-of-truth fields.

- `timeSpent` — actual human work hours spent on the task.
- `aiAssistancePercentage` — percentage from `0` through `100` of actual Time Spent during which AI materially assisted the user. Missing, blank, null, or invalid values normalize to `0` for backward compatibility.
- `estimatedEffortWithoutAI` — estimated human work hours required to complete the same task without AI assistance.

```text
AI Usage Rate = AI Assistance Percentage ÷ 100
AI-Assisted Hours = Time Spent × AI Usage Rate
Estimated Time Saved = Estimated Effort Without AI − Time Spent
Task AI Efficiency = (Estimated Time Saved ÷ AI-Assisted Hours) × 100
```

Task AI Efficiency represents estimated human-work hours saved per AI-assisted hour, expressed as a percentage. It must not be described as a completion percentage or as the task being a corresponding percentage faster. For example, Time Spent `4`, AI Assistance `50%`, and Estimated Effort Without AI `8` produces AI-Assisted Hours `2`, Estimated Time Saved `4`, and Task AI Efficiency `200%`.

When AI Assistance is `0%`, AI-Assisted Hours and Task AI Efficiency are `0`; the task does not contribute to an aggregate AI-efficiency denominator. When Time Spent is `0` and AI Assistance is greater than `0%`, Task AI Efficiency is undefined and represented as `0` or null according to the application numeric model. The record is excluded from AI-efficiency aggregates and is identified as having insufficient time data. Division by zero must never produce `NaN` or `Infinity`.

## Qualifying Task Rules and Scope

A task qualifies for official AI-efficiency aggregate metrics only when all of the following are true:

- Time Spent is finite and greater than `0`.
- AI Assistance is finite, greater than `0`, and no greater than `100`.
- Estimated Effort Without AI is finite and greater than `0`.
- The task is terminal with final actual-time values: Status is `Completed` or `Closed`.

A **terminal item** is any task whose status is `Completed` or `Closed`, regardless of whether it is archived. Archived terminal tasks remain terminal items and are included when a UI count, label, tooltip, or calculation scope is defined as all terminal items. Terminal describes a task's workflow state only; it does not mean the task was deleted.

In-progress tasks may display provisional task-level results, but they are excluded from official completed-task aggregate metrics unless the aggregate is explicitly labeled provisional. Aggregate metrics must respect their active reporting scope, such as the current filtered task set, selected project, or selected date range. Every numerator and denominator in an individual aggregate must be calculated from the same qualifying task set.

## Aggregate AI Efficiency Calculation

The official **AI Efficiency** is the estimated percentage reduction in total effort attributable to AI involvement. It is calculated from totals and must never be a simple arithmetic average of task-level percentages.

```text
Total AI-Assisted Hours = Σ(Time Spent × AI Usage Rate)
Total Estimated Time Saved = Σ(Estimated Effort Without AI − Time Spent)
Total Estimated Effort Without AI = Σ(Estimated Effort Without AI)
AI Efficiency = (Total Estimated Time Saved ÷ Total Estimated Effort Without AI) × 100
```

AI Efficiency measures the portion of estimated no-AI effort that was saved through AI involvement. It is distinct from AI utilization and must not use Total AI-Assisted Hours as its denominator.

The following calculations are prohibited for aggregate AI Efficiency:

```text
Σ(Task AI Efficiency) ÷ Number of Tasks

Total Estimated Time Saved ÷ Total AI-Assisted Hours
```

If Total Estimated Effort Without AI is `0`, AI Efficiency is `0` and the UI displays `No qualifying AI-assisted work has been recorded.` Negative time-saved and negative efficiency values are valid and must be displayed with neutral wording, such as `AI-assisted work exceeded the estimated effort without AI for the selected tasks.`

## Supporting Aggregate Metrics

For the same qualifying AI-assisted terminal-item set used by AI Efficiency, calculate:

```text
Productivity Multiplier = Total Estimated Effort Without AI ÷ Total Time Spent
AI Utilization = (Total AI-Assisted Hours ÷ Total Time Spent) × 100
```

`Productivity Multiplier` is a separate analysis of potential no-AI effort compared with actual effort. For example, `1.50×` means the selected completed or Closed work was estimated to require 1.5 times as many hours without AI as it actually required.

The application may provide a general productivity multiplier or AI utilization for all terminal items with valid actual-time data, including tasks with `0%` AI assistance, only when the UI label and tooltip explicitly identify that broader scope. No intermediate derived value is rounded; display hours to two decimal places, percentages to one or two decimal places, and productivity multipliers to two decimal places.

## Statistics Section and AI Analysis UI

The top statistics section contains `Total Tasks`, `Active`, `Overdue`, `Overall Progress`, and an `AI Efficiency` box immediately after `Overall Progress`.

- The main-view `AI Efficiency` box displays AI Efficiency for qualifying terminal items across the full task dataset, including archived tasks. Its qualifying-task count and supporting label must use that same full-dataset scope.
- The box may label the count as `qualifying terminal items`; this count includes only terminal items that also satisfy the AI-efficiency source-value rules. A separate terminal-item count, when displayed, includes every Completed or Closed task regardless of archive state or AI data completeness.
- Hovering or focusing the main-view `AI Efficiency` box displays the overall Productivity Multiplier for the same qualifying terminal-item scope. The tooltip identifies the formula `Total Estimated Effort Without AI ÷ Total Time Spent` and formats the result as a multiplier, such as `1.50×`.
- The box tooltip also states: `Estimated time saved due to AI involvement, divided by the total estimated effort without AI.`
- When no task qualifies, the box displays `0%` and exposes the no-qualifying-work empty-state explanation.

The AI Analysis section displays AI Efficiency, Total Estimated Time Saved, Total Estimated Effort Without AI, Total AI-Assisted Hours, Productivity Multiplier, AI Utilization, and Qualifying Task Count. Tooltips define: Total Estimated Time Saved as the difference between Estimated Effort Without AI and Time Spent; AI Efficiency as Total Estimated Time Saved divided by Total Estimated Effort Without AI; Productivity Multiplier as Estimated Effort Without AI divided by Time Spent; and AI Utilization as the weighted portion of actual work time associated with AI assistance. The downloaded AI Analysis HTML export includes a `Productivity Multiplier` column for each task, calculated as that task's Estimated Effort Without AI divided by its Time Spent.

## Reference Aggregate Example

| Task | Time Spent | AI Assistance | Estimated Effort Without AI | AI-Assisted Hours | Estimated Time Saved |
| --- | ---: | ---: | ---: | ---: | ---: |
| A | 4 | 50% | 8 | 2 | 4 |
| B | 10 | 20% | 12 | 2 | 2 |
| C | 2 | 100% | 7 | 2 | 5 |

```text
Total AI-Assisted Hours = 6
Total Estimated Time Saved = 11
Total Estimated Effort Without AI = 27
AI Efficiency = (11 ÷ 27) × 100 = 40.74%
Productivity Multiplier = 27 ÷ 16 = 1.69×
```

This result is intentionally not the arithmetic mean of task-level percentages and does not use AI-Assisted Hours as the AI Efficiency denominator.

---

# Projects

Supports:

- Create
- Edit
- Delete
- Assign tasks

---

# Hierarchy

Supports unlimited:

- Parent tasks
- Child tasks
- Grandchildren
- Deeper nesting

Rendering uses depth-first traversal so moving a parent keeps all descendants together.

Sorting preserves hierarchy by sorting tasks within their sibling group rather than flattening the full task tree.

---

# Priority Ordering

Sibling tasks are ordered by `priorityOrder` when no column sort is active.

Features:

- Drag and drop
- Automatic sibling renumbering
- Reordering does not change parent-child relationships

---

# Task Features

Supports:

- Create
- Edit
- Delete
- Archive / Unarchive
- Mark Complete
- Clone
- Create Child Task

## Clone Task

Creates a new task that:

- Generates a new unique ID
- Copies all task properties, including AI productivity fields
- Appends `" - Copy"` to the title
- Preserves:
  - Project
  - Parent
  - Dependencies
  - Status
  - Progress
  - Hours
  - Estimated Effort Without AI
  - Actual (Time Spent)
  - Percentage of AI Assistance
  - Due date
  - Notes
- Places the clone at the end of its sibling group

Child tasks are not cloned.

An archived source task may be cloned from the Archive view, but the clone is created with `archived: false` so it appears as a new active task record.

If the archived source task is a child task, cloning it automatically sets each archived ancestor in its parent chain to `archived: false` so the unarchived clone has visible hierarchy context in normal task views.

### Task Editor Clone Action

When editing an existing saved task, the task editor provides a `Clone Task` action:

- The bottom `Clone Task` button appears immediately before `Save Task`.
- The editor header includes a matching compact Clone Task action that invokes the same clone behavior as the bottom button.
- Selecting Clone Task uses the existing clone rules, creates a new task record, updates the runtime unsaved state, refreshes dependent UI, and opens the cloned task in the same editor dialog.
- Clone Task is unavailable while creating a new, unsaved task because no source task record exists to clone.
- Clone Task does not save pending, unsaved edits currently entered in the editor; it clones the persisted source task record.

## Create Child Task

Selecting Create Child:

- Opens the normal task dialog
- Automatically sets:
  - `parentId`
  - `projectId`, inherited from the parent
- Allows the user to modify the remaining fields before saving

If the selected parent task is archived, saving the new child task automatically sets the parent task and any archived ancestors in its parent chain to `archived: false`. This keeps the newly created child visible with its hierarchy context in normal task views.

---

# Task Archiving

Archiving provides a reversible alternative to deleting a task record.

## Data Model and Compatibility

- Each task stores an `archived` Boolean property.
- `archived: true` identifies an archived task; `archived: false` identifies a non-archived task.
- New tasks default to `archived: false`.
- When loaded data omits `archived`, or supplies a value that cannot be normalized to a Boolean, the task is treated as not archived and normalized to `archived: false`.
- Saving writes the normalized `archived` property for every task.

## Archive Command

- Every non-archived task row provides an Archive icon in the Actions area.
- Selecting Archive must display a confirmation prompt containing the question `Are you sure?` before changing the record.
- The confirmation prompt must identify the task being archived by title.
- If the selected task has child or descendant tasks, the confirmation prompt must state how many child and descendant task records will also be archived.
- Canceling the confirmation leaves the task unchanged.
- Confirming sets `archived` to `true`, updates `updatedAt`, and immediately removes the task from the current non-Archive results as if it had been deleted from that view. The task data is retained.
- When a parent task is archived, every child and descendant task is also archived in the same operation.
- Parent archive cascading is intended to keep the normal task list small as completed sets of related tasks are retired together.

## Archive Cascade and Reactivation Rules

- Archiving a parent cascades downward to all descendants.
- Archiving a child does not archive its parent or siblings.
- Unarchiving a parent does not automatically unarchive descendants; descendants remain archived until explicitly unarchived or reactivated by a child-related operation.
- Cloning an archived child task creates a non-archived clone and automatically unarchives its parent chain so the clone is visible in normal views.
- Adding a new child to an archived parent automatically unarchives that parent chain so the new child is visible in normal views.
- Automatic parent-chain unarchiving updates `updatedAt` on each affected parent or ancestor task.

## Unarchive Command

- In the Archive view, the same action position and button changes to an Unarchive icon and command for archived tasks.
- Selecting Unarchive sets `archived` to `false`, updates `updatedAt`, and immediately removes the task from the Archive view.
- Unarchiving does not require the archive confirmation prompt because it does not remove or archive data.

## Archive View and Visibility

- The left navigation includes an `Archive` view with a count of archived task records.
- The Archive view is the only main task-grid view in which archived tasks may appear.
- All other left-panel views and normal task-grid results exclude archived tasks before applying search, project, status, due-date, and hide-completed filters.
- In the Archive view, those main search and filtering controls operate only on archived tasks.
- The parent-context rule must not cause an archived parent to appear in a non-Archive view or a non-archived parent to appear in the Archive view. Parent context is limited to records whose `archived` value matches the selected view.
- Archived tasks are excluded from normal task metrics and navigation counts. The Archive navigation count reports the number of archived records.

## Reporting and Export Exclusions

- Archived tasks never participate in Weekly Timesheet eligibility, preview rows, totals, or downloaded timesheet HTML.
- Archived tasks never participate in AI Analysis eligibility, summary metrics, detail rows, totals, or downloaded AI Analysis HTML.
- `Export Active` excludes archived tasks.
- `Export View` excludes archived tasks in every non-Archive view. When the Archive view is selected, `Export View` may export the archived records currently visible after the active main-grid filters are applied.

---

# Sorting

The task table supports sorting by:

- Task name
- Project
- Status
- Due date

Behavior:

- First click sorts ascending.
- Second click sorts descending.
- The active sort direction is shown in the column header.
- Sorting occurs within sibling groups.
- Parent-child hierarchy remains intact.
- Tasks without due dates sort after dated tasks when due date is sorted ascending.

---

# Search and Filtering

Supports:

- Search text
- Project filter
- Status filter
- Hide completed
- Active view
- Overdue view
- Due-date filter
- Archive view
- Recently Created view
- Recently Accessed view

## Recently Created View

The left navigation includes a `Recently Created` view that shows tasks created during the trailing 24-hour period.

- A task qualifies when its `createdAt` timestamp is greater than or equal to the current local date/time minus 24 hours.
- The comparison is rolling rather than calendar-day based; it is evaluated when the task grid is rendered or refreshed.
- Tasks are ordered newest first by `createdAt`; tasks with missing or invalid `createdAt` values do not qualify.
- The view respects the current search, project, status, due-date, and Hide Completed filters.
- Normal archive boundaries remain in force: non-archived tasks participate in the normal `Recently Created` view, while archived tasks participate only when the Archive view is selected.
- Parent-context rendering continues to apply: a visible recently created child includes its eligible parent hierarchy context in the main task grid without causing a parent to count as a Recently Created match.
- The view label includes a tooltip or accessible description stating that it shows tasks created in the last 24 hours.

## Task Access Tracking and Recently Accessed View

Each task record includes an optional `lastAccessedAt` timestamp that tracks the most recent time the task was accessed.

- Accessing a task means opening that task's details in the task editor, including through a grid Edit action, row double-click, a Notes `...(see more)` action, parent-task navigation, sub-task navigation, or opening a newly created clone after the clone record is created.
- When a task is accessed, the application sets that task's `lastAccessedAt` value to the current timestamp and updates its `updatedAt` value.
- The access timestamp is persisted in the task JSON data and is preserved by normal load, save, export, clone, archive, and unarchive workflows.
- Existing task records without `lastAccessedAt` remain valid and do not qualify for recently accessed results until they are accessed.
- Loading a dataset whose task records omit `lastAccessedAt` must succeed without errors or crashes. The application treats the missing value as unavailable access history rather than requiring a migration or rejecting the dataset.
- Updating access tracking marks the runtime dataset unsaved, so the existing unsaved-state indicator and browser exit confirmation apply until Save completes successfully.

The left navigation includes a `Recently Accessed` view that shows tasks accessed during the trailing 24-hour period.

- A task qualifies when its `lastAccessedAt` timestamp is greater than or equal to the current local date/time minus 24 hours.
- The comparison is rolling rather than calendar-day based and is evaluated when the task grid is rendered or refreshed.
- Qualifying tasks are ordered by most recently accessed first. Tasks with missing or invalid `lastAccessedAt` values do not qualify.
- The view respects the current search, project, status, due-date, and Hide Completed filters.
- Normal archive boundaries remain in force: non-archived tasks participate in the normal Recently Accessed view, while archived tasks participate only when the Archive view is selected.
- Parent-context rendering continues to apply: a visible recently accessed child includes its eligible parent hierarchy context in the main task grid without causing a parent to count as a Recently Accessed match.
- The view label includes a tooltip or accessible description stating that it shows tasks accessed in the last 24 hours.

## Parent Context in Main Results

When a child task satisfies the active filters and is therefore visible in the main task-grid results, its parent task must also be shown even when the parent does not satisfy one or more active filters. The parent is contextual only; it does not count as a filter match. Higher ancestors are included recursively when available so the visible child retains its complete hierarchy path. Parent context never crosses the archive boundary: normal views include only non-archived ancestors, and the Archive view includes only archived ancestors. This rule applies to the main task-grid rendering only and does not change the filtered task set used by **Export View**.

### Parent Direct-Match Expansion

When a parent task itself directly satisfies the active search text or another active filter:

- The main task grid displays all of that matching parent's direct child tasks, even when an individual child does not independently satisfy the active filters.
- The displayed child rows are contextual children; they do not count as independent filter matches and do not alter filtered task sets used by exports, metrics, or counts.
- A direct matching child continues to display its parent hierarchy context, but does not cause its sibling tasks to appear unless the parent directly matches or a sibling independently matches.
- The expansion applies recursively only to the direct children of each directly matching parent; descendants beyond those direct children appear only when they independently match, are required parent context, or are direct children of another directly matching parent.
- Archive boundaries remain in force: normal views expand only non-archived children, while the Archive view expands only archived children.

## Search Match Highlighting

When Search text contributes to a task matching the main-grid results:

- Matching text within visible task text is rendered in bright red and bold, including matches within a larger word. For example, searching for `our` renders the `our` portion of `your` in `Find your brother` in bright red bold text.
- Highlighting is case-insensitive and preserves the original stored text casing.
- All occurrences of the search text within a rendered text value are highlighted.
- Highlighting must safely encode task content before rendering and must not introduce unsafe HTML injection.
- The highlight treatment targets text that is visibly rendered from the matched task, beginning with task titles. Context-only parent and child rows that do not independently match the search text are not highlighted solely because they are shown for hierarchy context.
- Highlight color is supplemental: bold emphasis and existing task labels/structure remain available so meaning is not communicated by color alone.

## Empty Main-Grid Search Guidance

When active search text and filters produce no main-grid results:

- The empty-state message continues to state that no tasks match the current filters and also identifies the currently selected left-panel navigation context.
- The context is expressed as `You are currently on '[view name]'`, using the visible navigation label, such as `All Tasks`, `Active`, `Completed`, `Overdue`, `Most Recent`, `Recently Created`, `Recently Accessed`, or `Archive`.
- The current-context phrase is rendered in bold red text so users can quickly recognize that switching the left-panel view may reveal the expected task.
- The message remains understandable without color alone through its explicit wording and bold emphasis.
- The current navigation context is derived dynamically so the guidance remains accurate for every supported left-panel view.

## Root-Level Task Row Emphasis

The main task grid visually distinguishes root-level tasks, meaning tasks whose `parentId` is absent or empty:

- Root-level task rows use a subtle, aesthetically restrained cool-blue background distinct from the standard white child-task rows. The treatment should support scanning hierarchy without competing with status, due-date, selection, hover, focus, or validation states.
- The root-level task title uses the same blue token and visual color as the completed-task checkmark, with sufficient contrast against its row background and targeting WCAG 2.2 AA text contrast.
- The root-level row background must be lighter than the previous cool-blue treatment so the oval status badge, including the `In Progress` badge, retains clearly visible shape, fill, and text contrast. The background remains supplemental hierarchy emphasis and must not visually merge with any status-pill color.
- The treatment applies to both root parent tasks and standalone tasks, because neither has a parent task.
- Child-task rows retain their existing standard background and title treatment.
- The same styling applies in all main-grid views, including filtered, Recently Created, Recently Accessed, and Archive views.
- The hierarchy marker, indentation, task type text, and accessible labels remain in place; color is additional visual emphasis rather than the only indicator of hierarchy.

## Due-Date Filter

Available options:

- All Due Dates
- Past Due
- Due Last Week
- Due Today
- Due Yesterday
- Due Tomorrow
- Due This Week
- Due Next Week
- Due Next Month

### Week Definition

A week runs from Sunday through Saturday.

### Due This Week

Shows tasks with a due date from today through the final day of the current week.

### Due Yesterday

Shows tasks whose due date is exactly the preceding local calendar day. This option is distinct from `Past Due`: `Due Yesterday` excludes tasks due before yesterday and applies regardless of whether the task is complete, subject to any other active filters.

### Due Next Week

Shows tasks due during the full calendar week immediately following the current week.

### Due Last Week

Shows tasks due during the complete Sunday-through-Saturday calendar week immediately before the current week. This filter includes completed and incomplete tasks, subject to any other active filters.

### Due Next Month

Shows tasks with a due date in the next calendar month, from the first day through the final day of that month. For example, when the current month is July, this filter includes tasks due from August 1 through August 31. This option is available only in the Due Date dropdown filter; it is not a left-panel navigation view.

### Past Due

Shows incomplete tasks whose due date is before today.

---

# Import and Export

Supports:

- JSON Import
- Full JSON Save
- Export Active
- Export Current View
- Export Report

## Legacy Import Compatibility

Load Tasks accepts task arrays from `tasks`, `taskList`, `items`, or a top-level JSON array. It accepts project arrays from `projects` or `projectList`.

The importer normalizes common legacy aliases before rendering:

- `taskStatus` → `status`; `progress` → `percentComplete`; `estimatedEffort` → `estimatedHours`
- `hoursSpent` or `actualEffort` → `timeSpent`
- `estimatedHumanOnlyEffort` or `estimatedHumanEffort` → `estimatedEffortWithoutAI`
- `percentageOfAIAssistance` → `aiAssistancePercentage`
- `due` or `targetDate` → `dueDate`; `description` or `detail` → `notes`
- `dependencies` → `dependsOn`; `parentTaskId` → `parentId`

The importer normalizes `archived` to a Boolean. A missing or invalid `archived` value defaults to `false` for backward compatibility.

When imported data has status `Completed` but no `completedDate`, the importer uses its due date as the completion date when one is available.

Missing optional values are normalized safely. Saving writes the current schema only.

## Export Current View

The `Export View` action exports only the tasks currently visible in the task grid.

It respects the active:

- Search text
- Project filter
- Status filter
- Due-date filter
- Hide-completed setting
- Navigation view: All, Active, Completed, Overdue, Most Recent, or Archive

The exported task order matches the hierarchy-preserving order shown in the grid.

The export includes:

- `exportType: "current-view"`
- Export timestamp
- Application version
- Snapshot of the selected filters
- Visible task count
- Projects referenced by visible tasks
- Visible tasks only

The downloaded filename uses the format:

```text
mytaskassistant-current-view-YYYY-MM-DD.json
```

## Export Report

The `Export Report` action creates a downloadable standalone HTML report using the tasks currently visible in the task grid.

Behavior:

- `Export Report` is available next to `Export View` in both the top and bottom command bars.
- It uses the same current-view task set and hierarchy-preserving order as `Export View`.
- It respects the active search, project, status, due-date, hide-completed, and navigation filters.
- Archived tasks are included only when the Archive view is selected, matching `Export View` behavior.
- The report is generated from the visible task data at the time the command is selected.
- The report downloads as a self-contained HTML file that can be opened independently in a browser.

### Export Report HTML Template Binding

`Export Report` must bind the current-view data into the provided standalone HTML report template whose document title begins with `MyTaskAssistant Report` and whose hero heading is `Projects, priorities, and execution health.`

Template binding requirements:

- The generated report uses the supplied report template structure, styling, and client-side rendering script.
- The template remains self-contained: embedded CSS, embedded JavaScript, and no external network or file dependencies.
- The generated report replaces the template's sample JSON with the actual current-view report payload in:

```html
<script type="application/json" id="embeddedData">...</script>
```

- The `embeddedData` payload contains:
  - `metadata`
  - `projects`
  - `tasks`
- `metadata` includes at minimum:
  - `application: "MyTaskAssistant"`
  - `applicationVersion`
  - `schemaVersion`
  - `updatedAt`
  - Report-generation timestamp when available
  - Current filter snapshot when available
- `projects` contains the projects referenced by the visible tasks.
- `tasks` contains only the currently visible task records in the same hierarchy-preserving order used by `Export View`.
- The template's placeholder/sample JSON data must never be emitted as report data unless it is the actual current application data.
- The template script is responsible for rendering the report from `embeddedData` into the existing template regions.
- The report template includes the following report sections and bound regions:
  - Source card: `sourceApp`, `sourceVersion`, `sourceUpdated`, `sourceSchema`
  - KPI grid: `kpiGrid`
  - Executive summary: `summaryCopy`, `insightList`
  - Status distribution: `statusDonut`, `activeTaskCount`, `statusLegend`
  - Project portfolio: `projectGrid`
  - Attention needed: `attentionList`
  - Deadline timeline: `timeline`
  - Task hierarchy and register: `taskTableBody`
  - Effort and utilization: `effortSummary`, `effortChart`
  - AI contribution: `aiSummary`, `aiChart`
  - Data quality audit: `auditGrid`, `auditFindings`
  - Footer timestamp: `footerGenerated`
- The generated report should include enough task information to be useful outside the application, including project, task title, status, progress, due date, days remaining or completion state, hours, AI Hours Saved or AI contribution values when available, dependencies, notes, and update timestamps.

Default filename:

```text
MyTaskAssistant-[The name of the project on the top of the list]-datetimestamp.html
```

Filename rules:

- `[The name of the project on the top of the list]` is taken from the project name of the first visible task in the report order.
- If the first visible task has no project, use `NoProject`.
- If no tasks are visible, the export should either be disabled or use `NoTasks` in the filename.
- The project-name portion must be sanitized for safe filesystem use.
- `datetimestamp` uses the local date and time at export generation.

`Export Active` remains available and exports all incomplete, non-archived tasks regardless of the current view filters.

When data is saved, metadata includes:

```json
{
  "applicationVersion": "v3.5.2"
}
```

The application may update metadata when saving older JSON files.

No formal schema migration framework is required.

Older task records that do not contain AI productivity fields remain valid. Missing fields are normalized as:

- `estimatedHours`: `null`
- `estimatedEffortWithoutAI`: `null`
- `timeSpent`: `0`
- `aiAssistancePercentage`: `0` (default assumption: AI produced no measurable productivity gain)
- `archived`: `false`

Older JSON files may contain legacy property names. On import, they are automatically mapped:

- `estimatedHumanOnlyEffort` or `estimatedHumanEffort` → `estimatedEffortWithoutAI`
- `hoursSpent` or `actualEffort` → `timeSpent`
- `percentageOfAIAssistance` → `aiAssistancePercentage`

When saving, only the new property names are written; the legacy names are never saved again.

Example effort-related task properties:

```json
{
    "estimatedHours": 8.0,
    "timeSpent": 5.5,
    "aiAssistancePercentage": 40,
    "estimatedEffortWithoutAI": 9.5
}
```

AI Hours Saved is NOT stored. It is always calculated:

```text
AI Hours Saved =
estimatedEffortWithoutAI - timeSpent
```

---

# User Interface

## Header

Displays:

- Application logo
- Application name
- Visible application version badge
- Browser favicon that reflects the task-management purpose of the application
- When runtime `saved` is `false`, the heading area next to `My Tasks` displays `(unsaved)` in the same font style and size as the `My Tasks` text.
- The `(unsaved)` indicator is red.
- When runtime `saved` is `true`, the `(unsaved)` indicator is hidden.
- The left-panel JSON file information area also displays a red `(unsaved)` indicator when runtime `saved` is `false`.
- The left-panel `(unsaved)` indicator follows the same state transitions as the `My Tasks` heading indicator: hidden when `saved` is `true`, visible when `saved` is `false`, and updated immediately after task changes, file load, and successful Save.

## Main Toolbar Mirroring

The primary top action bar commands are also available in a mirrored bottom action bar below the task grid.

Mirrored commands:

- Load Tasks
- Export View
- Export Report
- Weekly Timesheet
- 🤖 AI Analysis
- Export Active
- Save
- Projects
- + Add Task

Behavior and implementation requirements:

- The bottom action bar appears below the grid/workspace area.
- The bottom action bar uses the same command order as the top action bar.
- The bottom action bar aligns visually with the top action bar so the page layout remains balanced and aesthetically consistent.
- The bottom buttons use the same button labels, styling, tooltips, and accessibility intent as the corresponding top buttons.
- The implementation must not duplicate the command logic behind the buttons.
- Top and bottom buttons must call the same common JavaScript functions for each command.
- If a command's behavior changes, updating the shared command function must update both the top and bottom button behavior.
- The `+ Add Task` button is included in both the top and bottom command bars so a new task can be created even when the user is scrolled to the bottom of the grid.

## Task Row Actions

Compact inline SVG action icons:

1. Goto Link — link
2. Clone Task — document with plus
3. Create Child Task — hierarchy with plus
4. Edit Task — pencil
5. Archive or Unarchive — archive-state icon
6. Delete Task — trash

Characteristics:

- Embedded SVG
- Native browser tooltips
- Approximately 28-pixel click targets
- Tight spacing of approximately 3 pixels
- Neutral styling
- The archive-state icon and accessible label show Archive for non-archived tasks and Unarchive for archived tasks
- Archive requires an `Are you sure?` confirmation before the state changes
- Delete icon highlights red on hover

## Main Task Grid Hours Display

- The main task grid's Hours value displays the task's Estimated Hours, including calculated Estimated Hours for parent-task rollups.
- Hovering or focusing the Hours value continues to show both Estimated Hours and Time Spent.
- When both Estimated Hours and Time Spent are greater than zero and Time Spent exceeds Estimated Hours, the displayed Hours value is bold red to identify an overrun without relying on color alone.
- The standalone current-view HTML report mirrors this Hours display, tooltip, and overrun treatment for its Estimated Hours column. Its separate Time Spent column remains unchanged.

## AI Assistance Slider

The Add/Edit Task dialog includes a compact slider:

- Minimum: 0%
- Maximum: 100%
- Step: 5%
- Default: 0%
- Live percentage displayed beside the label
- End labels display 0% and 100%

## Task Editor Top Actions

The Add/Edit Task dialog includes a compact action bar in the dialog header so users can act without scrolling to the bottom.

The top action bar mirrors the bottom actions:

- Mark Complete
- Cancel and Close
- Clone Task
- Save Task
- Save and Close

Behavior and presentation:

- Uses compact embedded SVG icons
- Uses approximately 28-pixel click targets
- Includes native browser tooltips
- Includes accessible labels
- Invokes the same underlying actions as the bottom buttons
- Mark Complete is shown only when editing an incomplete task
- Clone Task is shown only when editing an existing saved task
- The bottom text buttons remain available

## Task Editor Save Actions

The Add/Edit Task dialog provides distinct save actions at the bottom of the dialog:

- `Save Task` persists the current task details to the in-memory main task dataset, updates the runtime unsaved state, and refreshes dependent UI values without closing the task editor. It supports iterative note editing and saving other task-detail changes while the dialog remains open.
- `Save and Close` performs the same persistence, validation, unsaved-state update, and UI refresh behavior as `Save Task`, then closes the task editor after a successful save.
- `Save and Close` is positioned immediately to the right of `Save Task` in the bottom action area.
- If validation fails, neither action persists changes or closes the task editor.
- The task editor's top actions include corresponding Save Task and Save and Close controls that invoke the same shared command logic as the bottom actions.
- The existing Cancel and Close action does not save changes made since the most recent successful Save Task or Save and Close action.

## Task Editor Notes Placement

The task editor places the Notes control directly below Task Title:

- The Notes label, text area, and existing Expand Notes / Collapse Notes control move from their prior lower-form position to immediately after the Task Title field.
- Expand Notes and Collapse Notes retain their existing behavior, accessible state, labels, and keyboard operation after relocation.
- No other task-editor fields, rows, grouping, alignment, order, or layout behavior is changed by the Notes relocation.
- The Notes control remains full width within the Task Detail form.

## Task Editor Auto-Update Percent Complete

The task editor provides a runtime-only `Auto-Update Percent Complete` checkbox to help users keep progress aligned with recorded effort.

- The checkbox is positioned directly below the Time Spent field, in the left side of the existing Time Spent / Estimated Effort Without AI row. It appears to the left of the Estimated Effort Without AI instructions. The existing task-editor layout and field order must not otherwise be rearranged.
- The checkbox is selected by default whenever a task is loaded or opened in the task editor, including when opening an existing task, a clone, or a new task.
- The checkbox value is editor-only runtime state. It is not stored in task JSON, exports, or any other persisted dataset.
- While selected, changing Time Spent automatically recalculates Percent Complete only when both Time Spent and Estimated Hours have valid values and Estimated Hours is greater than zero.
- Percent Complete is calculated as `(Time Spent ÷ Estimated Hours) × 100` and is constrained to the editor's valid `0` through `100` percent range.
- If Time Spent or Estimated Hours is missing or invalid, or Estimated Hours is zero, changing Time Spent does not automatically change Percent Complete.
- When the checkbox is cleared, changing Time Spent does not change Percent Complete; the user remains responsible for entering Percent Complete manually.
- The automatic update follows the existing percent-complete/status synchronization rules after the calculated value is applied.

## Task Editor Parent Navigation

When editing an existing child task, the displayed selected parent task is a navigation target:

- Clicking the parent task name opens that parent task in the same task editor dialog, replacing the currently displayed child-task details.
- Parent navigation follows the same in-dialog editor-navigation behavior used when a user clicks a sub-task in the Sub-Tasks tab.
- The parent-task navigation control has an accessible name that identifies the parent task it opens and supports keyboard activation.
- The existing control for removing the parent relationship remains a separate action and must not be triggered when navigating to the parent task.
- The parent task is unavailable as a navigation target when the task has no parent or when the dialog is creating a new, unsaved task.

## Task Editor Dependency Lookup Ordering

When the Dependency lookup opens with no search text, it displays eligible dependency tasks. As the user types, it filters those same eligible tasks by the existing dependency search fields. In both cases, the filtered suggestions are ordered by relevance before the result limit is applied:

1. Tasks that share the edited task's non-empty parent task, sorted by task name.
2. Remaining tasks in the edited task's project, sorted by task name.
3. All remaining eligible tasks, sorted by task name.

The first group is omitted when the edited task has no parent. A task appears in only its highest-priority applicable group. Existing dependency eligibility rules, selected-dependency exclusion, selected-parent exclusion, search matching, and result limits remain unchanged.

## Task Editor Sub-Task Creation

When editing an existing saved task, the Sub-Tasks tab provides an `Add Sub-Task` action:

- The Add Sub-Task action is available while the Sub-Tasks tab is selected and is positioned with the tab's sub-task content.
- Selecting Add Sub-Task opens the task editor for a new task in the same dialog.
- The new task is initialized with the currently edited task as its parent, so its Parent Task field is preselected to that task.
- The new sub-task also inherits the current task's project when a project is assigned.
- The action does not create or persist a task record until the user saves the new sub-task through the normal task-editor save flow.
- Add Sub-Task is unavailable when creating a new, unsaved task because a parent task record does not yet exist.
- Opening the new sub-task follows the existing in-dialog navigation behavior used for selecting a sub-task or parent task.

## Robot Column

The Robot column:

- Uses an icon-only header
- Displays only the calculated AI Hours Saved value in each task row
- Remains blank when calculation inputs are incomplete
- Uses native tooltips to explain the calculation
- Does not display the underlying effort fields

---

# Design Principles

- Fast
- Minimal
- Compact
- Desktop-application feel
- Keyboard and mouse friendly
- Local-first
- No unnecessary complexity
- Easy to extend
- Human-readable JSON
- Self-contained deployment

## AI Productivity

MyTaskAssistant measures the productivity gains achieved through AI-assisted work rather than simply recording AI tool usage.

All AI metrics quantify measurable efficiency improvements, allowing organizations to report AI-driven time savings across projects and reporting periods.

---

# Future Enhancements

Potential future improvements:

- Sorting or filtering by AI Hours Saved
- AI productivity reporting by project
- Aggregate estimated time saved
- Recurring tasks
- Multiple saved views
- Tags
- Attachments
- Rich notes
- Keyboard shortcuts
- Optional dark and light themes

These enhancements are optional and should not compromise the application's simplicity.


## v2.6 Legacy JSON Correction

Imported legacy tasks are normalized with `percentageOfAIAssistance` set to `0`, even when the original JSON contained none of the AI fields.

The runtime therefore identifies a legacy, untracked task when:

- `estimatedEffortWithoutAI` is absent
- `timeSpent` is absent
- `aiAssistancePercentage` resolves to `0`

Such tasks display `0.00 h` rather than blank.


## v2.8 Effort Field Consolidation

`hoursSpent` is now the single source of truth for actual effort.

Changes:

- The separate Actual Effort field was removed from the Add/Edit Task dialog.
- AI Hours Saved now uses `hoursSpent` as the actual-effort value.
- New and updated tasks no longer store `actualEffort`.
- During import, a legacy `actualEffort` value is copied into `hoursSpent` only when a valid `hoursSpent` value is unavailable.
- Hovering over the Hours value in a task row shows:
  - Estimated: `(hours)` or `Not captured`
  - Actual: `(hours)`

The Hours tooltip uses Time Spent as Actual.


## v2.9 Due This Week Correction

The `Due This Week` filter now uses the full current calendar week, Sunday through Saturday.

It includes:

- Tasks due earlier in the current week
- Tasks due today
- Tasks due later in the current week
- Past-due tasks whose due date falls within the current week

It does not include tasks due before the start of the current week.


## v3.0 Weekly Timesheet HTML Export

A new `Weekly Timesheet` action opens a week-selection and preview dialog.

### Included tasks

The timesheet includes tasks that:

- Have Time Spent greater than zero
- Are In Progress or Completed

### Row layout

Each row displays:

- Project name in bold
- Task name beneath the project
- Total hours
- Sunday-through-Saturday daily hour columns

### Project grouping and subtotals

The Weekly Timesheet preview groups eligible task rows by project:

- Rows are grouped under their project name, with projects ordered alphabetically and task rows ordered alphabetically within each project.
- Tasks without an assigned project are grouped under `No Project`.
- Every task row retains its individual editable daily-hour inputs and its per-task total.
- After the task rows in each project group, the preview displays a Project Total row that contains the project's combined total hours and combined hours for each Sunday-through-Saturday day column.
- Editing a task's daily-hour input immediately recalculates its task total, its Project Total row, daily totals, and the Grand Total.
- Project grouping does not change which tasks qualify for the timesheet or the existing daily-allocation behavior.

The downloaded standalone Weekly Timesheet HTML uses the same project grouping, task ordering, Project Total rows, day totals, and Grand Total as the reviewed preview. Downloaded task rows are read-only display values rather than editable inputs.

### Daily allocation

Because tasks currently store total Time Spent rather than dated time entries, the preview initially places the full task total on:

1. The task's due date, provided that the due date falls inside the selected week

Every daily value is editable in the preview. Row totals, daily totals, and the grand total update immediately.

### Output

The export downloads a standalone, printable HTML file named:

```text
mytaskassistant-timesheet-YYYY-MM-DD-to-YYYY-MM-DD.html
```


## v3.1 Weekly Timesheet Layout Refinement

- Expanded the dialog to approximately 94% of viewport width and 90% of viewport height.
- Moved week selection and actions into one compact toolbar.
- Reduced guidance to one muted line.
- Added live task-count and total-hours badges.
- Made the preview fill the remaining dialog area.
- Added sticky table headers and a sticky Grand Total row.
- Increased column widths and added responsive sizing.


## v3.2 Weekly Timesheet Week Switching and Close Button

- Changing the selected week immediately clears the prior preview and regenerates the new week.
- If the selected week has no qualifying tasks, the old rows are removed and an empty-state message is shown.
- Weekly rows include only tasks whose due date falls inside the selected Sunday-through-Saturday week.
- The close button is positioned consistently in the upper-right corner of the dialog header.
- The close button now has a larger click target and visible hover treatment.


## v3.3 Weekly Timesheet Due-Date Eligibility Correction

Weekly Timesheet eligibility is now based exclusively on the task's due date.

A task is included only when:

- Time Spent is greater than zero
- Status is anything except Not Started
- The task has a valid due date
- The due date falls within the selected Sunday-through-Saturday week

The task's `updatedAt` timestamp is no longer used for timesheet eligibility or initial day allocation.

The full Time Spent value is initially placed on the task's due-date column. The user may redistribute the hours across the selected week before downloading the HTML timesheet.


## v3.4 Weekly Timesheet Status Eligibility

Weekly Timesheet status eligibility now includes every task status except `Not Started`.

A task is included when:

- Time Spent is greater than zero
- Status is not `Not Started`
- The task has a valid due date
- The due date falls within the selected Sunday-through-Saturday week

This permits time spent on blocked, deferred, canceled, or otherwise interrupted work to appear in the timesheet.


## v3.5.1 Estimated Effort Without AI and AI Hours Saved

- Renamed `estimatedHumanOnlyEffort` (and the earlier `estimatedHumanEffort`) to `estimatedEffortWithoutAI`. Legacy names are mapped automatically on import and are never saved again.
- Renamed `hoursSpent` to `timeSpent` and `percentageOfAIAssistance` to `aiAssistancePercentage`; legacy names (including `actualEffort`) are mapped automatically on import and are never saved again.
- Added the stored `estimatedHours` field with an `Estimated hours` input in the task editor.
- Renamed the editor input `estimatedHumanOnlyEffortInput` to `estimatedEffortWithoutAIInput`.
- Replaced the AI Efficiency percentage with AI Hours Saved, calculated as `estimatedEffortWithoutAI - timeSpent`. AI Hours Saved is never stored.
- Estimated Effort Without AI is required and must be strictly greater than Time Spent whenever AI Assistance is greater than 0%; equality is invalid.
- Updated labels, helper text, and tooltips to reflect AI Productivity rather than AI Usage: the application measures productivity gains rather than tool usage.


## v3.5.2 AI Analysis Restoration and In-Place Versioning

- Restored the `AI Analysis` view from v3.5. A `🤖 AI Analysis` toolbar button opens a read-only, week-based dialog.
- A task is included in the AI Analysis when:
  - Its status is not `Not Started`
  - Its due date falls within the selected Sunday-through-Saturday week
  - `timeSpent` is greater than zero
  - `aiAssistancePercentage` is greater than zero
  - `estimatedEffortWithoutAI` has a value
- The dialog shows summary badges (Tasks Using AI, Time Spent, Without AI, AI Hours Saved, Average AI %) and a per-task table with a grand-total footer.
- AI Hours Saved per task is `estimatedEffortWithoutAI - timeSpent` and is never stored.
- The analysis can be downloaded as a standalone HTML file named `AI_Analysis_<weekStart>_to_<weekEnd>.html`.
- Source-controlled files (`MyTaskAssistant.html`, `specs/MyTaskAssistant_Specification.md`, `ReadMe.html`, `ReadMe.md`) are now updated in place; version-suffixed filename copies are no longer generated.


## v3.5.3 Parent Task Rollups and Completion Guard

- Parent tasks calculate their displayed Estimated Hours, Time Spent, and Estimated Effort Without AI as sums of all descendant tasks.
- Parent AI Assistance is read-only and displayed as an Estimated Effort Without AI-weighted average of descendant AI Assistance percentages.
- Parent task rows use the calculated Time Spent and AI Hours Saved values.
- A parent cannot be completed until all descendant tasks are complete, regardless of whether completion is attempted from the row checkbox, the task editor, or a saved `Completed` status.


## v3.5.4 Most Recent and Due Next Month Views

- Added the `Most Recent` left-panel view. It shows the 10 tasks with the newest `createdAt` values, ordered newest first, after any active search, project, status, due-date, and hide-completed filters are applied. Its hover tooltip describes these criteria.
- Added `Due Next Month` to the Due Date dropdown. It includes all tasks due during the next calendar month and is not a left-panel navigation view.


## v3.5.5 Due Last Week, Sub-Tasks Tab, and Completion Validation

- Added `Due Last Week` to the Due Date dropdown. It includes tasks due in the full Sunday-through-Saturday week immediately before the current week.
- The task editor provides `Task Detail` and `Sub-Tasks` tabs. The Sub-Tasks tab lists direct child tasks. Selecting a child opens that child in the task editor.
- Mark Complete now uses the same editor validations as Save Task, including AI productivity validation.
- A task cannot be completed unless both Estimated Hours and Time Spent are greater than zero. This rule applies to task-editor completion actions, saving a Completed status or 100% progress, and the task-row completion checkbox.


## v3.5.6 Sub-Tasks Tab Interaction Refinement

- The task editor opens on the `Task Detail` tab, which retains the full-width, two-column detail layout.
- Selecting `Sub-Tasks` replaces the Task Detail content in the same editor window; it does not append child content below the detail form.
- The Sub-Tasks tab lists only direct children as full-width task-style rows showing title, status, and Time Spent.
- Selecting a Sub-Tasks row loads that child into the same task editor window.


## v3.5.7 Timesheet and AI Analysis Dialog Spacing

- Refined the Weekly Timesheet and AI Analysis dialog gutters, header alignment, helper-text spacing, and report-preview padding.
- The dialogs retain usable table width and horizontal scrolling, with compact responsive spacing on smaller screens.


## v3.5.8 Task-Row Note Preview and Expandable Editor Notes

- Task-row Notes display up to five lines of text.
- When a note exceeds five lines, a bold `...(see more)` action appears after the preview. Selecting it opens that task in the task editor.
- The task editor Notes field provides an Expand Notes / Collapse Notes control so longer notes can be read and edited with a larger text area.


## v3.5.9 Completion Status and Progress Synchronization

- Setting task progress to `100%` automatically sets status to `Completed`.
- Setting status to `Completed` automatically sets task progress to `100%`.
- Save validation prevents an inconsistent completed state: a task cannot be persisted as `Completed` below 100% progress, and a task at 100% progress cannot be persisted with a non-Completed status.


## v3.5.10 Top Toolbar Labels and Tooltips

- Renamed the `Open JSON` toolbar action to `Load Tasks` to use task-focused, non-technical language.
- Every top-toolbar action provides a tooltip describing its behavior: Load Tasks, Export View, Weekly Timesheet, AI Analysis, Export Active, Save, Projects, and Add Task.


## v3.5.11 Not Started Progress Validation

- Save validation prevents a task from retaining `Not Started` status when Percent Complete is greater than `0%`.
- The user must select an appropriate active status before saving progress greater than zero.


## v3.6.0 Terminal Task Rollups and Report Totals

- Parent calculations use terminal descendants only. Intermediate parent values and stored direct parent values never contribute to parent rollups, preventing double counting.
- Parent Estimated Hours, Time Spent, and Estimated Effort Without AI are sums of terminal descendant values. Missing numeric values contribute zero.
- Parent Percent Complete is an Estimated Hours-weighted average of terminal descendant progress. If one or more terminal descendants lack Estimated Hours, it falls back to a task-count average. Parent progress is displayed as a whole number.
- Parent status is calculated from terminal descendants: Completed only when all are completed; Not Started only when all are Not Started at 0%; Canceled only when all are canceled; Blocked or Deferred only when all unfinished terminal work meets that state; otherwise In Progress.
- Parent status rows display a concise terminal-descendant summary and a tooltip with status counts and calculated progress.
- Parent AI Assistance is calculated from positive Estimated Effort Without AI terminal values. Parent AI Hours Saved is the sum of individually valid terminal AI Hours Saved values; incomplete AI productivity records are excluded and identified in the tooltip.
- Parent calculated fields are read-only. Parent completion checkboxes and Mark Complete actions are disabled while terminal descendants remain incomplete, with an explanation directing the user to Sub-Tasks.
- Weekly Timesheet and AI Analysis reports include terminal work tasks only. Parent rows do not contribute to report grand totals, preventing double counting.


## v3.6.1 Condensed Parent Status Summary

- Parent rows use the compact status format `Status · completed/terminal`, for example `In Progress · 1/3`.
- The parent status tooltip expands the compact value into plain language, including completed and blocked terminal task counts, calculated progress, and the per-status breakdown.


## v3.6.2 Dependency Detail and Responsive Task Grid

- Dependency indicators provide a detailed hover/focus overlay listing every dependency item, its current status, and any unresolved dependency references.
- The task grid constrains its workspace and exposes horizontal overflow when the browser viewport is narrower than the task table.


## v3.6.3 Task-Row Action Ordering

- The Goto Link action is the first task-row action and uses the accent-blue active state when a Notes hyperlink is available.
- Clone Task and Create Child Task are adjacent actions.


## v4.0.0 Consolidated Task-Management Release

- Promoted the source-controlled standalone application to the v4.0.0 feature baseline.
- Main task rows include a green leaf for terminal work tasks and a brown branch for parent tasks beside the priority number.
- The Actions area is ordered Goto Link, Clone Task, Create Child Task, Edit, and Delete. Goto Link is blue when Notes contains a link, disabled otherwise, and opens the first Notes hyperlink in a new window.
- The `Dep?` column uses an embedded SVG dependency indicator with a count badge. Its hover/focus detail reports dependency totals, satisfaction/blocking state, unresolved references, and each dependency task/status.
- Notes preview up to five lines in task rows. Longer notes expose a bold `...(see more)` editor shortcut; the editor supports Expand Notes / Collapse Notes.
- Load Tasks uses non-technical language, detailed toolbar tooltips, and expanded legacy JSON compatibility.
- The task grid supports horizontal scrolling whenever its columns exceed the available content area.
- Parent rollups, Timesheet, and AI Analysis use terminal tasks only, preventing duplicate report totals.

### v4.0.0 Final Grid Presentation and Test Fixture Details

- The task-row Hours and AI Hours Saved values are displayed to one decimal place. Stored task values and editor fields retain their full precision.
- AI Hours Saved uses standard font weight rather than emphasis and is constrained to a single line in the task grid.
- The due-date column is labeled `Due`; its values retain the standard task-grid font size.
- The task-grid horizontal scrollbar is available whenever the table exceeds the workspace width, including when the left navigation remains visible.
- The synthetic test fixture is `testdata/Mortgage_Servicing_TestData.json`. It provides five mortgage, servicing, analytics, and technology projects; parent/child hierarchy; dependencies; long notes; statuses; varied AI productivity; and dates spanning last week, this week, next week, and next month.


## v4.1.0 Explicit Completion Dates

- Tasks store an editable `completedDate` in local `YYYY-MM-DD` format. The task grid and task editor display the field.
- When status is set to `Completed`, a missing completion date is set to the current local date. Setting progress to 100% follows the same completion synchronization.
- Setting a non-completed status clears `completedDate`. If the task was previously complete and progress is 100%, progress is automatically reduced below completion (default `99%`) so the task is no longer complete. The user can then adjust progress as needed.
- Entering a completion date that is today or earlier automatically sets status to `Completed` and progress to 100%. A completed task may retain a user-edited future completion date.
- Weekly Timesheet eligibility uses terminal tasks with Time Spent greater than zero and a `completedDate` in the selected Sunday-through-Saturday week. The initial daily allocation places the full Time Spent value on the completion-date column.

## v4.2.0 Main Result View Completion Indicator

- The main task grid does not display a separate Completed date column.
- A self-contained SVG completion indicator appears beside the Due date when a task has a `completedDate`.
- A green checkmark indicates the task was completed on or before its due date, or that no due date was set.
- A sad-face icon indicates the task was completed after its due date.
- The indicator tooltip identifies the completion date and whether the task was completed on time, late, or had no due date.

## v4.3.0 Main Result View Standalone Task Marker

- In the main task grid, a task that has neither a parent nor child tasks is a standalone task.
- Standalone tasks display a blue, self-contained SVG marker beside Priority instead of the green leaf used for childless sub-tasks or the brown branch used for parent tasks.
- The standalone marker is an outlined rounded square with a centered filled circle and exposes the accessible label `Standalone item`.
- Main task-grid hierarchy-marker tooltips identify the task relationship: a parent shows `Parent task with [count] child task(s)`, a child shows `Child task of '[parent task name]'`, and a standalone task shows `Standalone task`.

## v4.1.1 Task-Row Note Hover

- Hovering a task-row Notes preview displays the full task note in a native browser tooltip, including text that exceeds the five-line preview.
- The existing note preview, hyperlink behavior, and `...(see more)` editor action remain unchanged.

## v4.4.0 Unsaved Task Change Indicator

- The application maintains a runtime-only `saved` state that is not persisted to JSON.
- Initial application load and successful data-file load set `saved` to `true`.
- A successful Save action sets `saved` to `true`.
- Adding, deleting, or modifying a task immediately sets `saved` to `false`.
- When `saved` is `false`, the `My Tasks` heading displays a red `(unsaved)` indicator in the same font style and size as the heading text.
- The left-panel JSON file information area displays a matching red `(unsaved)` indicator driven by the same runtime `saved` state.

## v4.5.0 Bottom Main Toolbar Mirror

- The commands Load Tasks, Export View, Weekly Timesheet, 🤖 AI Analysis, Export Active, Save, and Projects are mirrored below the task grid.
- The bottom command bar uses the same order, visual alignment, labels, tooltips, and styling intent as the top command bar.
- Top and bottom command buttons share common JavaScript command functions; command logic is not duplicated.

## v4.5.1 Add Task Bottom Toolbar Mirror

- The `+ Add Task` command is included in both the top and bottom command bars.
- Both `+ Add Task` buttons call the same common JavaScript command function.
- The bottom `+ Add Task` button lets users create a task after scrolling to the bottom of the grid and helps the bottom toolbar visually align with the top toolbar.

## v4.5.2 Self-Contained Favicon

- The application includes a self-contained SVG favicon declared with a `data:image/svg+xml` URL in the HTML `<head>`.
- The favicon reflects the personal task-manager purpose of the application through task/checklist-oriented imagery.
- No external favicon image file is required.

## v4.6.0 Export Report

- Added `Export Report` beside `Export View` in both the top and bottom command bars.
- `Export Report` uses the currently visible task-grid data and hierarchy-preserving order to generate a downloadable standalone HTML report.
- `Export Report` binds current-view data into the supplied standalone report HTML template by replacing the template's `embeddedData` JSON payload.
- The default report filename is `MyTaskAssistant-[The name of the project on the top of the list]-datetimestamp.html`.

## v4.6.1 Closed Status Specification

- Added the `Closed` terminal status specification.
- `Closed` closes a task at 100% while allowing Estimated Hours, Time Spent, and AI-productivity fields to be zero or absent.
- It participates in completion-equivalent workflow behavior while remaining visibly distinct from successfully completed work.

## v4.6.2 Recently Created View Specification

- Added the left-navigation `Recently Created` view for tasks created during the trailing 24 hours.
- The view is rolling, orders qualifying tasks newest first, honors active main-grid filters, and retains the existing archive and parent-context rules.

## v4.6.3 Terminal Status AI Validation Specification

- Completing or closing a terminal task now validates that a positive Estimated Effort Without AI value is accompanied by AI Assistance greater than `0%`.
- A user must provide a positive AI Assistance percentage or clear the Estimated Effort Without AI value before the terminal status can be saved.

## v4.6.4 Due Yesterday Filter Specification

- Added `Due Yesterday` to the Due Date dropdown filter.
- The filter returns tasks due exactly on the preceding local calendar day and is intentionally distinct from `Past Due`, which includes all incomplete tasks due before today.

## v4.6.5 AI-Assisted Time Portion Specification

- AI Assistance percentage is used to calculate the AI-assisted portion of recorded Time Spent: `timeSpent × (aiAssistancePercentage ÷ 100)`.
- For example, 10 hours of Time Spent with 90% AI Assistance contains 9 AI-assisted hours and a 1-hour non-AI-assisted portion, regardless of the Estimated Effort Without AI value. This remains human-directed partnership work throughout.
- When Time Spent is greater than zero and AI Assistance is `0%`, the full recorded Time Spent is human-only time.

## v4.6.6 Closed Status Terminology

- Replaced the `Won't Do` terminal-status concept with `Closed`.
- `Closed` retains the same terminal, completion-equivalent, validation, rollup, filtering, dependency, and zero-effort rules while supporting a broader range of non-completion closure reasons.

## v4.6.7 Weighted AI Efficiency and Statistics

- Added the `AI Efficiency` statistics box immediately after `Overall Progress`.
- Replaced any arithmetic averaging of task-level AI-efficiency percentages with the weighted calculation `Total Estimated Time Saved ÷ Total AI-Assisted Hours`.
- Added qualifying-task rules, zero and negative efficiency handling, supporting aggregate metrics, UI tooltips, precision rules, and the AI Analysis metric set.
- Replaced the invalid requirement that Estimated Effort Without AI exceed Time Spent; a positive estimate may now produce positive, zero, or negative time saved.

## v4.6.8 Closed Status Preservation

- Clarified that `Closed` and `Completed` are separate terminal statuses even though both use `100%` progress.
- Saving, loading, synchronization, normalization, rendering, filtering, reporting, and parent-rollup logic must preserve an explicitly selected or stored `Closed` status and must not revert it to `Completed`.
- `Completed` remains only the default terminal status when 100% progress or a completion date is entered without an explicit `Closed` selection.

## v4.6.9 Main AI Efficiency Terminal-Item Scope

- Defined a terminal item as any `Completed` or `Closed` task, including archived tasks. Terminal describes workflow state and does not imply deletion.
- The main-view AI Efficiency statistic uses qualifying terminal items from the full task dataset, including archived records, and labels supporting counts according to that same scope.

## v4.7.0 Task Editor Save and Close

- Added `Save and Close` immediately to the right of the bottom `Save Task` button.
- `Save Task` saves valid task changes without closing the editor, supporting iterative Notes and task-detail editing.
- `Save and Close` uses the same save behavior and closes the editor only after a successful save.
- Top and bottom task-editor save controls use shared command logic.

## v4.7.1 AI Efficiency Denominator and Productivity Multiplier

- Replaced the aggregate AI Efficiency denominator: AI Efficiency is now `Total Estimated Time Saved ÷ Total Estimated Effort Without AI`.
- AI-Assisted Hours remains a supporting utilization metric and is not the aggregate AI Efficiency denominator.
- The AI report displays both AI Efficiency and Productivity Multiplier, where Productivity Multiplier is `Total Estimated Effort Without AI ÷ Total Time Spent`.

## v4.7.2 AI Efficiency Hover and Export Productivity Multiplier

- Hovering or focusing the main-view AI Efficiency statistic displays the overall Productivity Multiplier for the same qualifying terminal-item scope.
- The downloaded AI Analysis HTML export includes a per-task Productivity Multiplier column using `Estimated Effort Without AI ÷ Time Spent`.

## v4.7.3 Unsaved Browser Exit Confirmation

- When the runtime `saved` state is `false`, the application registers browser `beforeunload` behavior for closing, reloading, or navigating away from the page.
- The handler requests the browser's standard unsaved-changes confirmation dialog, allowing the user to remain in the application or proceed and discard unsaved changes.
- The application must not attempt to supply custom dialog text or claim it can prevent the user from leaving after the user confirms the browser prompt.
- When `saved` is `true`, no exit confirmation is requested.

## v4.7.4 Task Editor Parent Navigation

- Added in-dialog navigation from a child task's displayed parent task to that parent task's editor details.
- Parent navigation matches existing sub-task editor navigation and keeps parent-relationship removal as a separate action.

## v4.7.5 Task Editor Clone Action

- Added Clone Task to the task editor before Save Task, with matching top and bottom controls.
- The action clones the persisted source task through the existing clone rules, marks the dataset unsaved, and opens the new clone in the same editor.
- Clone Task is unavailable for a new, unsaved task and does not persist pending editor changes before cloning.

## v4.7.6 Task Editor Add Sub-Task

- Added an Add Sub-Task action to the Sub-Tasks tab for saved tasks.
- The action opens a new task in the same editor with the current task preselected as Parent Task and its project inherited when available.
- The new task is not persisted until saved through the standard task-editor save actions.

## v4.7.7 Task Access Tracking and Recently Accessed View

- Added persisted `lastAccessedAt` tracking whenever a task is opened in the task editor.
- Added a left-panel Recently Accessed view for tasks opened during the rolling prior 24 hours, ordered from most recently accessed to least recently accessed.
- Clarified backward-compatible loading: datasets without `lastAccessedAt` load without errors and simply have no recently accessed history until tasks are opened.
- Access tracking participates in the existing unsaved-state and browser exit-confirmation behavior until saved.

## v4.7.8 Task Editor Auto-Update Percent Complete

- Added a runtime-only Auto-Update Percent Complete checkbox below Time Spent without rearranging the existing editor layout.
- It defaults to selected each time a task is loaded into the editor and is never persisted.
- When selected, changing Time Spent recalculates Percent Complete from Time Spent divided by Estimated Hours when both are valid and Estimated Hours is greater than zero. When cleared, progress remains manual.

## v4.7.9 Root-Level Task Row Emphasis

- Added a cool-blue row-background and dark-blue task-title treatment for root-level tasks in the main grid.
- Child-task rows retain their existing appearance, while hierarchy semantics and accessible labels remain available independently of color.

## v4.7.10 Root-Level Row Color Refinement

- Root-level task titles use the same blue as the completed-task checkmark.
- The root-level cool-blue background is lightened to preserve the visibility and contrast of status pills, including In Progress.

## v4.7.11 Weekly Timesheet Project Grouping

- Added project-grouped task rows and Project Total rows to the editable Weekly Timesheet preview.
- Individual task-day editing remains available and updates project, daily, and grand totals immediately.
- The downloaded standalone timesheet HTML mirrors the same project grouping and subtotal structure.

## v4.7.12 Main Search Hierarchy Expansion and Match Highlighting

- A direct parent match expands the main grid to include all of that parent's direct child tasks, while a direct child match continues to show only its parent context and not unrelated siblings.
- Added case-insensitive bright-red bold highlighting for all visible search-text matches within matching task text, including substring matches within words.

## v4.7.13 Empty Search Context Guidance

- Enhanced the no-results main-grid message with the dynamically identified current left-panel context in bold red text, helping users recognize when another context view may contain the intended task.

## v4.7.14 Task Editor Notes Placement

- Moved the full-width Notes control, including Expand Notes and Collapse Notes behavior, directly below Task Title.
- Preserved every other task-editor field's existing grouping, ordering, alignment, and behavior.

## v4.7.15 Task Editor Dependency Lookup Ordering

- Dependency lookup suggestions prioritize tasks that share the edited task's parent, then remaining tasks in its project, then all other eligible tasks.
- Each priority group is sorted by task name before the existing suggestion limit is applied, both before and after a user enters search text.

## v4.7.16 Main Task Grid Estimated Hours and Overrun Indicator

- The main task grid now displays Estimated Hours instead of Time Spent in its Hours column while retaining the Estimated/Actual hover details.
- Estimated Hours is bold red when positive Time Spent exceeds positive Estimated Hours.
- The standalone current-view HTML report applies the same Estimated Hours display, tooltip, and overrun treatment.

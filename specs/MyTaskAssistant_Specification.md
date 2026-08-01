# MyTaskAssistant Specification
**Application Version:** v4.4.0
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
- No external libraries required

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
- The estimated percentage of work that AI accelerated or meaningfully improved
- Record AI Assistance only when AI produced measurable productivity gains
- Captured through a slider
- Slider step is 5%
- Defaults to 0%
- Stored as a whole number

### Task Editor Layout

Below the Percentage of AI Assistance slider, the editor places Parent Task above Dependencies in the left column and Due Date above Completed Date in the right column. Priority order appears immediately above Notes. Time Spent and Estimated Effort Without AI are presented together in aligned columns with equal-height input controls.

### Validation

When a task is saved with AI Assistance greater than 0%:

- `estimatedEffortWithoutAI` is required
- `estimatedEffortWithoutAI` must be strictly greater than Time Spent (`timeSpent`); equality is invalid

Validation message:

```text
Estimated Effort Without AI must be greater than Time Spent whenever AI Assistance is greater than 0%.

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
- If all required values are present, calculate and display AI Hours Saved.
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

## Parent Context in Main Results

When a child task satisfies the active filters and is therefore visible in the main task-grid results, its parent task must also be shown even when the parent does not satisfy one or more active filters. The parent is contextual only; it does not count as a filter match. Higher ancestors are included recursively when available so the visible child retains its complete hierarchy path. Parent context never crosses the archive boundary: normal views include only non-archived ancestors, and the Archive view includes only archived ancestors. This rule applies to the main task-grid rendering only and does not change the filtered task set used by **Export View**.

## Due-Date Filter

Available options:

- All Due Dates
- Past Due
- Due Last Week
- Due Today
- Due Tomorrow
- Due This Week
- Due Next Week
- Due Next Month

### Week Definition

A week runs from Sunday through Saturday.

### Due This Week

Shows tasks with a due date from today through the final day of the current week.

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
- When runtime `saved` is `false`, the heading area next to `My Tasks` displays `(unsaved)` in the same font style and size as the `My Tasks` text.
- The `(unsaved)` indicator is red.
- When runtime `saved` is `true`, the `(unsaved)` indicator is hidden.

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
- Save Task

Behavior and presentation:

- Uses compact embedded SVG icons
- Uses approximately 28-pixel click targets
- Includes native browser tooltips
- Includes accessible labels
- Invokes the same underlying actions as the bottom buttons
- Mark Complete is shown only when editing an incomplete task
- The bottom text buttons remain available

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

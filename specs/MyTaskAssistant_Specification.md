# MyTaskAssistant Specification
**Application Version:** v3.5.8  
**Last Updated:** 2026-07-23

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
- `notes`
- `createdAt`
- `updatedAt`

Unlimited hierarchy is supported using `parentId`.

## Parent Task Rollups and Completion

Tasks with one or more direct child tasks are parent tasks. The application calculates parent values from all descendant tasks, including nested descendants. Calculated values are displayed dynamically and are not persisted as duplicate rollup data.

For a parent task, these editor fields are read-only and show the sum of descendant values:

- `estimatedHours`
- `timeSpent`
- `estimatedEffortWithoutAI`

`aiAssistancePercentage` is also read-only for a parent. It is calculated as the Estimated Effort Without AI-weighted average of descendant AI Assistance percentages:

```text
Σ (child AI Assistance % × child Estimated Effort Without AI)
÷ Σ child Estimated Effort Without AI
```

The parent slider tooltip identifies the number of included descendants, total Estimated Effort Without AI, formula, and calculated percentage.

Parent rows use the calculated Time Spent and AI Hours Saved values. The parent task's direct stored values remain unchanged when the parent is saved.

A parent task cannot be marked `Completed` until every descendant task is complete. This is enforced through the task-row completion checkbox, the task-editor Mark Complete actions, and saving the parent with status `Completed`.

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

## Create Child Task

Selecting Create Child:

- Opens the normal task dialog
- Automatically sets:
  - `parentId`
  - `projectId`, inherited from the parent
- Allows the user to modify the remaining fields before saving

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

## Export Current View

The `Export View` action exports only the tasks currently visible in the task grid.

It respects the active:

- Search text
- Project filter
- Status filter
- Due-date filter
- Hide-completed setting
- Navigation view: All, Active, Completed, or Overdue

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

`Export Active` remains available and exports all incomplete tasks regardless of the current view filters.

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

## Task Row Actions

Compact inline SVG action icons:

1. Clone Task — document with plus
2. Create Child Task — hierarchy with plus
3. Edit Task — pencil
4. Delete Task — trash

Characteristics:

- Embedded SVG
- Native browser tooltips
- Approximately 28-pixel click targets
- Tight spacing of approximately 3 pixels
- Neutral styling
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

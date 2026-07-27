# MyTaskAssistant Specification
**Application Version:** v3.4  
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

Generated application and specification filenames must match that visible version.

For version `v2.2`, the expected filenames are:

- `MyTaskAssistant_v3.4.html`
- `MyTaskAssistant_Specification_v3.4.md`

When the application version changes:

1. Update the visible version badge in the HTML.
2. Update `metadata.applicationVersion` when JSON is saved.
3. Generate the HTML using the same version in its filename.
4. Generate the specification using the same version in its filename.
5. Update the specification's application version.

Legacy sequence-based names such as `Final_v9` should not be used for future generated releases.

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
- `hoursSpent`
- `estimatedHumanEffort`
- `hoursSpent`
- `percentageOfAIAssistance`
- `priorityOrder`
- `dueDate`
- `notes`
- `createdAt`
- `updatedAt`

Unlimited hierarchy is supported using `parentId`.

## AI Productivity Fields

### Estimated Human Effort

`estimatedHumanEffort`

- Decimal hours
- Represents the expected human effort if the task were completed without AI assistance
- Optional
- Must be greater than zero to calculate AI efficiency

### Actual (Time Spent)

`hoursSpent`

- Decimal hours
- Represents the actual effort required to complete or work on the task
- Optional
- May be greater than the estimate, which can produce negative AI efficiency

### Percentage of AI Assistance

`percentageOfAIAssistance`

- Whole-number percentage from 0 through 100
- Captured through a slider
- Slider step is 5%
- Defaults to 0%
- Stored as a whole number

These fields appear in the Add/Edit Task dialog but do not appear as individual columns in the task row.

---

# AI Efficiency

The task grid includes a compact icon-only Robot column.

## Column Header

- Displays only a robot SVG icon
- Tooltip: `AI Efficiency`
- Accessible label: `AI Efficiency`

## Calculation

AI efficiency is calculated at runtime and is not stored in JSON.

```text
Human Time Saved =
Estimated Human Effort - Actual (Time Spent)

AI Efficiency % =
((Estimated Human Effort - Actual (Time Spent)) / Estimated Human Effort)
x Percentage of AI Assistance
```

The assistance percentage is treated as a percentage value from 0 through 100.

Example:

```text
Estimated Human Effort: 10 hours
Actual (Time Spent): 4 hours
Percentage of AI Assistance: 80%

Human Time Saved: 6 hours
AI Efficiency: ((10 - 4) / 10) x 80% = 48%
```

## Display Rules

The Robot column displays a whole-number percentage.

- If both effort fields are absent and AI assistance resolves to `0%`, as with normalized legacy JSON, display `0%`.
- If some AI-tracking data exists but Estimated Human Effort is missing or zero, display blank.
- If some AI-tracking data exists but Actual (Time Spent) is missing, display blank.
- If Percentage of AI Assistance is zero, blank, missing, `null`, or otherwise not captured, treat it as `0%` and display `0%` once the effort baseline is available.
- If all required values are present, calculate and display AI efficiency.
- Negative values are allowed when Actual (Time Spent) exceeds Estimated Human Effort.
- The calculated result is rounded to the nearest whole percentage.

## Tooltip

Hovering over a displayed AI-efficiency percentage shows a native browser tooltip containing:

- AI Efficiency
- Estimated Human Effort
- Actual (Time Spent)
- Human Time Saved
- Percentage of AI Assistance
- The calculation used
- The final result

When AI assistance is zero or was not captured, the tooltip explains that the default assumption is no AI assistance and therefore AI Efficiency is `0%`.

For legacy tasks where none of the AI-tracking fields exist, the tooltip explains that no AI-effort tracking data was captured and the default assumption is no AI assistance.

Example tooltip:

```text
AI Efficiency: 48%

Estimated Human Effort: 10.00 h
Actual (Time Spent): 4.00 h
Human Time Saved: 6.00 h
AI Assistance: 80%

Calculation:
((10.00 - 4.00) / 10.00) x 80%
= 48%
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
  - Estimated Human Effort
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
- Due Today
- Due Tomorrow
- Due This Week
- Due Next Week

### Week Definition

A week runs from Sunday through Saturday.

### Due This Week

Shows tasks with a due date from today through the final day of the current week.

### Due Next Week

Shows tasks due during the full calendar week immediately following the current week.

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
  "applicationVersion": "v3.4"
}
```

The application may update metadata when saving older JSON files.

No formal schema migration framework is required.

Older task records that do not contain AI productivity fields remain valid. Missing fields are normalized as:

- `estimatedHumanEffort`: `null`
- `hoursSpent`: `null`
- `percentageOfAIAssistance`: `0` (default assumption: no AI assistance was used)

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
- Displays only the calculated percentage in each task row
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

---

# Future Enhancements

Potential future improvements:

- Sorting or filtering by AI efficiency
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

- `estimatedHumanEffort` is absent
- `hoursSpent` is absent
- `percentageOfAIAssistance` resolves to `0`

Such tasks display `0%` rather than blank.


## v2.8 Effort Field Consolidation

`hoursSpent` is now the single source of truth for actual effort.

Changes:

- The separate Actual Effort field was removed from the Add/Edit Task dialog.
- AI Efficiency now uses `hoursSpent` as the actual-effort value.
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

---
name: omnifocus
description: "Manage OmniFocus tasks, projects, folders, and tags via Omni Automation. Use when the user asks to interact with OmniFocus, including: (1) Adding tasks to inbox or a project, (2) Listing or searching tasks (inbox, available, flagged, due, overdue), (3) Completing, deleting, or modifying tasks (notes, due dates, defer dates, flags, tags), (4) Managing projects and folders, (5) Setting repeat rules on tasks, (6) Getting OmniFocus statistics and summaries, (7) GTD workflows and task reviews. Requires OmniFocus 3+ installed on macOS."
---

# OmniFocus

Control OmniFocus via Omni Automation JS, called from a Python wrapper.

## Requirements

- macOS with OmniFocus 3 or 4 installed
- OmniFocus must be running (or will auto-launch)
- Python 3 (system Python is fine)

## Quick Reference

```bash
# Run via the wrapper
python3 scripts/omnifocus.py <command> [args...]

# Shorthand (if of is executable)
scripts/of <command> [args...]
```

All commands return JSON to stdout. Errors print `{"error": "..."}` and exit 1.

## Commands

### Read

| Command | Args | Description |
|---------|------|-------------|
| `inbox` | | List inbox tasks |
| `folders` | | List all folders |
| `projects` | `[folder]` | List projects, optionally filtered by folder |
| `tasks` | `<project>` | List tasks in a project |
| `tags` | | List all tags |
| `today` | | Tasks due today or overdue |
| `flagged` | | Flagged incomplete tasks |
| `search` | `<query>` | Search tasks by name or note |
| `info` | `<taskId>` | Full task details by ID |
| `summary` | | Database counts (projects, tasks, flagged, due, available, inbox) |
| `project-tree` | `<name> [limit]` | Recursive subtask tree with depth |
| `project-search` | `<term> [limit]` | Find projects by partial name |
| `folder` | `<name> [limit]` | Detail a folder: child folders + projects |
| `root` | `[limit]` | Top-level folders and unfiled projects |
| `tag-summary` | `<name> [limit]` | Tasks for a tag, grouped by folder/project |
| `tag-family` | `<name> [limit]` | Tasks across a tag and all its children |
| `available` | `[limit]` | All available (actionable) tasks |
| `review` | `[limit]` | All projects with total + available task counts |

### Create

| Command | Args | Description |
|---------|------|-------------|
| `add` | `<name> [project]` | Add task to inbox or project |
| `newproject` | `<name> [folder]` | Create project |
| `newfolder` | `<name>` | Create top-level folder |
| `newtag` | `<name>` | Create or get tag |

### Modify

| Command | Args | Description |
|---------|------|-------------|
| `complete` | `<taskId>` | Mark complete |
| `uncomplete` | `<taskId>` | Mark incomplete |
| `delete` | `<taskId>` | Permanently delete |
| `rename` | `<taskId> <name>` | Rename task |
| `note` | `<taskId> <text>` | Append to note |
| `setnote` | `<taskId> <text>` | Replace note |
| `defer` | `<taskId> <date>` | Set defer date (YYYY-MM-DD) |
| `due` | `<taskId> <date>` | Set due date (YYYY-MM-DD) |
| `flag` | `<taskId> [true\|false]` | Set flagged (default true) |
| `tag` | `<taskId> <tag>` | Add tag (creates if needed) |
| `untag` | `<taskId> <tag>` | Remove tag |
| `move` | `<taskId> <project>` | Move to project |

### Repeat

```bash
# repeat <taskId> <method> <interval> <unit>
python3 scripts/omnifocus.py repeat abc123 fixed 1 weeks
python3 scripts/omnifocus.py repeat abc123 due-after-completion 2 days
python3 scripts/omnifocus.py repeat abc123 defer-after-completion 1 months
python3 scripts/omnifocus.py unrepeat abc123
```

Methods: `fixed`, `due-after-completion`, `defer-after-completion`
Units: `days`, `weeks`, `months`, `years`

## Output Format

All commands return JSON. Task records include:

```json
{
  "id": "abc123",
  "name": "Task name",
  "note": "Notes here",
  "flagged": false,
  "completed": false,
  "available": true,
  "deferDate": "2026-04-01",
  "dueDate": "2026-04-10",
  "effectiveDueDate": "2026-04-10",
  "completionDate": null,
  "estimatedMinutes": 30,
  "project": "Project Name",
  "folder": "Folder Name",
  "tags": ["tag1", "tag2"],
  "repeat": {"method": "fixed", "recurrence": "FREQ=WEEKLY;INTERVAL=1"}
}
```

Write commands return `{"success": true, "task": {...}}`.

## Examples

```bash
# Add task to inbox
python3 scripts/omnifocus.py add "Buy groceries"

# Add task to specific project
python3 scripts/omnifocus.py add "Review docs" "Work Projects"

# Get today's tasks
python3 scripts/omnifocus.py today

# Search name and notes
python3 scripts/omnifocus.py search "quarterly report"

# Set due date and flag
python3 scripts/omnifocus.py due abc123 2026-04-10
python3 scripts/omnifocus.py flag abc123 true

# Add tags
python3 scripts/omnifocus.py tag abc123 "urgent"

# Create recurring task
python3 scripts/omnifocus.py add "Weekly review" "Habits"
python3 scripts/omnifocus.py repeat xyz789 fixed 1 weeks

# Hierarchical views
python3 scripts/omnifocus.py summary
python3 scripts/omnifocus.py root
python3 scripts/omnifocus.py folder "Personal"
python3 scripts/omnifocus.py project-tree "Work" 50
python3 scripts/omnifocus.py tag-family "review"
```

## Technical Details

Uses Omni Automation JS evaluated inside OmniFocus via AppleScript
`evaluate javascript`. This is more reliable than JXA for OmniFocus-specific
APIs — no cross-process type-conversion bugs.

Key implementation patterns:
- `filter()` not `byName()` — `byName()` returns an ObjectSpecifier proxy
  whose child collections fail to enumerate
- `asArray()` helper — `Array.isArray()` returns false for OmniJS collection types
- `safe()` wrapper — prevents crashes on null/undefined properties
- `id.primaryKey` — stable canonical ID in Omni Automation
- `effectiveDueDate` — respects project-cascaded due dates
- `task.available` — respects defer dates, sequential ordering, project status
- Dates output as ISO 8601 YYYY-MM-DD via `Date.toISOString()`

**First run:** OmniFocus may prompt to allow automation access. Enable in
System Settings > Privacy & Security > Automation.

## Notes

- Task IDs are OmniFocus internal primary keys (returned in all task responses)
- Dates use ISO 8601: YYYY-MM-DD
- Project and tag names are case-sensitive
- Tags are created automatically when using `tag` command
- `available` respects defer dates, sequential project ordering, and on-hold status

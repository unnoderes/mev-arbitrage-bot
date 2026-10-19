# Issue Tracker: Linear

This project uses **Linear** as its issue tracker. Skills like `to-tickets`, `triage`, and `to-spec` interact with Linear through the Linear MCP tools.

## Workflow

### Creating issues
- Use `linear__save_issue` MCP tool
- Required: `team` parameter (team name or ID)
- Required: `title` (unless using a template)
- Optional: `description`, `priority`, `labels`, `assignee`, etc.

### Listing issues
- Use `linear__list_issues` MCP tool
- Filter by: team, state, assignee, label, priority, project, etc.
- Search by title/description with `query` parameter
- Default ordering: `updatedAt` (can change to `createdAt`)

### Updating issues
- Use `linear__save_issue` with `id` parameter
- Can update: title, description, state, assignee, labels, priority, etc.

### Querying issue details
- Use `linear__get_issue` with issue ID or identifier (e.g., `LIN-123`)
- Returns full details including attachments, git branch, relations

## Labels
- List available labels: `linear__list_issue_labels`
- Create new labels: `linear__save_issue_label`

## States/Statuses
- List available statuses: `linear__list_issue_statuses`
- Specify by team

## Users
- List workspace users: `linear__list_users`
- Use "me" for current user in assignee/creator filters

## Projects & Cycles
- List projects: `linear__list_projects`
- List cycles: `linear__list_cycles`

## PRs as a request surface

**Disabled.** External PRs do not automatically enter the triage queue.

To enable: set this flag to `enabled`, then configure the webhook or polling mechanism.

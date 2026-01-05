# Feature Specification: GitHub Issues Sync (vk-issues-sync)

**Feature Branch**: `001-vk-issues-sync`  
**Created**: 2026-01-05  
**Status**: Draft  
**Input**: Unidirectional GitHub Issues → Vibe Kanban Tasks synchronization with incremental updates and conflict resolution UI

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Initial Issues Import (Priority: P0)

As a developer, I want to import all open GitHub Issues from a repository into Vibe Kanban Tasks so that I can start orchestrating AI agents on my existing backlog without manual task creation.

**Why this priority**: Foundation feature - without initial import, no sync exists. Delivers immediate value by populating Vibe Kanban with actionable tasks.

**Independent Test**: Can be fully tested by configuring a GitHub repo URL + PAT, triggering sync, and verifying all open Issues appear as Tasks in Vibe Kanban with correct title, description, and metadata.

**Acceptance Scenarios**:

1. **Given** a Vibe Kanban project with no GitHub sync configured, **When** user provides GitHub repo URL and valid PAT, **Then** system fetches all open Issues and creates corresponding Tasks with status "Todo"
2. **Given** a repo with 500 open Issues, **When** sync completes, **Then** all 500 Tasks exist in Vibe Kanban with matching Issue numbers, titles, descriptions, and GitHub links
3. **Given** a repo with no open Issues, **When** sync runs, **Then** system completes successfully with zero Tasks created and logs "No open Issues found"
4. **Given** an invalid PAT or repo URL, **When** sync attempts, **Then** system displays clear error message indicating authentication failure or repo not found

---

### User Story 2 - Incremental Sync of Updates (Priority: P1)

As a developer, I want Vibe Kanban to automatically detect when GitHub Issues are updated (title/description changes) so that my local Tasks stay current with the source of truth without manual intervention.

**Why this priority**: Keeps local environment aligned with GitHub reality. Prevents agents from working on stale information.

**Independent Test**: Can be tested by modifying an Issue title/description on GitHub, waiting for sync interval, and verifying Task reflects the changes in Vibe Kanban.

**Acceptance Scenarios**:

1. **Given** a Task linked to GitHub Issue #42, **When** Issue #42's title changes on GitHub, **Then** after next sync cycle (≤5 min), Task title updates to match
2. **Given** a Task with description "Original text", **When** linked GitHub Issue description changes to "Updated text", **Then** Task description updates after sync
3. **Given** 10 Issues with pending updates, **When** sync runs, **Then** only changed Tasks are updated (not all Tasks re-processed)
4. **Given** sync interval is 5 minutes idle, **When** user manually triggers sync, **Then** updates happen immediately (30-second active polling)

---

### User Story 3 - State Mapping (Open/Closed → Todo/Done) (Priority: P2)

As a developer, I want GitHub Issue state changes (Open→Closed, Closed→Reopened) to automatically update corresponding Task status so that completed work is reflected without manual status updates.

**Why this priority**: Automates workflow closure. Reduces manual overhead of marking Tasks done when Issues are closed upstream.

**Independent Test**: Can be tested by closing an open Issue on GitHub, waiting for sync, and verifying Task status changes from "Todo" to "Done" in Vibe Kanban.

**Acceptance Scenarios**:

1. **Given** a Task with status "Todo" linked to open Issue #123, **When** Issue #123 is closed on GitHub, **Then** Task status updates to "Done" after sync
2. **Given** a Task with status "Done" linked to closed Issue #456, **When** Issue #456 is reopened on GitHub, **Then** Task status reverts to "Todo" after sync
3. **Given** an Issue closed with reason "completed", **When** sync processes it, **Then** Task marked "Done" with timestamp of GitHub closure
4. **Given** an Issue closed as "not planned", **When** sync processes it, **Then** Task marked "Done" with note indicating closure reason

---

### User Story 4 - Conflict Resolution UI (Priority: P3)

As a developer, I want to be notified when a Task has been modified locally AND the linked GitHub Issue has been updated remotely so that I can choose which version to keep (or merge) instead of losing work.

**Why this priority**: Safety feature preventing data loss. Less critical than P0-P2 because conflicts are rare if GitHub is true source of truth.

**Independent Test**: Can be tested by modifying a Task description locally, then updating the same Issue on GitHub, triggering sync, and verifying a conflict modal appears with diff view and resolution options.

**Acceptance Scenarios**:

1. **Given** Task description is "Local changes", **When** linked GitHub Issue description updates to "Remote changes" before local Task syncs, **Then** system detects conflict and displays modal with three-way diff
2. **Given** conflict modal is open, **When** user selects "Keep Local", **Then** Task retains local changes and GitHub Issue remains unchanged (no write-back since unidirectional)
3. **Given** conflict modal is open, **When** user selects "Accept GitHub", **Then** Task overwrites local changes with GitHub Issue content
4. **Given** conflict modal is open, **When** user selects "Merge", **Then** system presents editable text field with both versions for manual merge, then saves result to Task
5. **Given** multiple conflicts detected, **When** sync completes, **Then** all conflicts are queued and user can resolve them one-by-one or batch-accept GitHub version

---

### User Story 5 - Manual Sync Trigger (Priority: P2)

As a developer, I want a manual "Sync Now" button so that I can pull latest GitHub Issues immediately without waiting for the automatic polling interval.

**Why this priority**: Improves UX by giving control. Important for time-sensitive work but not blocking MVP.

**Independent Test**: Can be tested by clicking "Sync Now" button in Settings and verifying sync executes immediately (within 30 seconds) and UI shows sync status progress.

**Acceptance Scenarios**:

1. **Given** user is in Settings → GitHub Integration, **When** user clicks "Sync Now", **Then** sync initiates within 30 seconds and progress indicator appears
2. **Given** sync is already running, **When** user clicks "Sync Now", **Then** button is disabled with message "Sync in progress..."
3. **Given** sync completes successfully, **When** user views sync status, **Then** "Last synced: [timestamp]" updates to current time
4. **Given** sync fails due to network error, **When** user clicks "Sync Now" again, **Then** system retries with exponential backoff and logs error details

---

### Edge Cases

- **Rate limiting**: What happens when GitHub API rate limit (5000 req/hour) is exceeded?
  - System pauses sync, logs warning, and resumes after rate limit window resets (displayed in UI: "Next sync in 42 minutes")
  
- **Large repos (>1000 Issues)**: How does sync handle pagination?
  - Uses GraphQL cursor-based pagination to fetch all Issues across multiple requests, with progress indicator showing "Syncing 342/1247 Issues..."
  
- **Orphaned Tasks**: What happens if a GitHub Issue is deleted?
  - Task remains in Vibe Kanban with warning badge "⚠️ Issue deleted on GitHub" but is not auto-deleted (user decides)
  
- **Token expiration**: What happens when PAT expires or is revoked?
  - Sync fails with clear error "GitHub authentication failed. Please update token in Settings" and disables auto-sync until resolved
  
- **Concurrent modifications**: What if user modifies Task while sync is updating it?
  - Sync detects in-flight local changes, aborts update for that specific Task, and flags for conflict resolution on next cycle
  
- **Network failures**: What happens during sync if connection drops?
  - Exponential backoff (1s, 2s, 4s, 8s, 16s, max 30s) with max 5 retries, then logs failure and waits for next scheduled sync
  
- **First-time setup with 0 Issues**: What happens if repo has no open Issues?
  - Sync completes successfully, displays "No Issues to import" message, and scheduling continues normally

## Requirements *(mandatory)*

### Functional Requirements

#### Synchronization Core

- **FR-001**: System MUST fetch all open GitHub Issues from configured repository on initial sync
- **FR-002**: System MUST create corresponding Vibe Kanban Task for each fetched Issue with fields: title, description, GitHub Issue number, GitHub Issue URL
- **FR-003**: System MUST use GraphQL API with cursor-based pagination to handle repositories with >100 Issues
- **FR-004**: System MUST sync incrementally by comparing `updated_at` timestamps between GitHub Issues and local Task metadata
- **FR-005**: System MUST update only Tasks whose linked GitHub Issues have `updated_at` > `last_synced_at`
- **FR-006**: System MUST map GitHub Issue states: `open` → Task status `Todo`, `closed` → Task status `Done`
- **FR-007**: System MUST preserve GitHub Issue metadata: labels (as tags), assignees, milestone, creation date, closure date

#### Authentication & Configuration

- **FR-008**: System MUST store GitHub Personal Access Token (PAT) in SQLite database with AES-256 encryption using machine-specific key
- **FR-009**: System MUST validate PAT has minimum required scopes: `repo`, `read:org`
- **FR-010**: System MUST allow user to configure one GitHub repository URL per Vibe Kanban project via Settings UI
- **FR-011**: System MUST validate repository URL format: `https://github.com/{owner}/{repo}` or `{owner}/{repo}`
- **FR-012**: System MUST test PAT + repo access on first setup and display clear error if authentication fails or repo is inaccessible

#### Sync Scheduling & Triggers

- **FR-013**: System MUST automatically sync on schedule: every 5 minutes when idle, every 30 seconds when changes detected
- **FR-014**: System MUST provide manual "Sync Now" button in Settings that triggers immediate sync
- **FR-015**: System MUST disable "Sync Now" button while sync is in progress with message "Sync in progress..."
- **FR-016**: System MUST display sync status: "Last synced: [timestamp]", "Next sync in: [countdown]", or "Syncing... [progress]"

#### Conflict Resolution

- **FR-017**: System MUST detect conflicts when Task `updated_at` > `last_synced_at` AND GitHub Issue `updated_at` > `last_synced_at`
- **FR-018**: System MUST display conflict resolution modal with three-way diff: local version, remote version, and base (last synced) version
- **FR-019**: System MUST offer resolution options: "Keep Local", "Accept GitHub", "Merge Manually"
- **FR-020**: System MUST queue multiple conflicts and allow batch resolution (e.g., "Accept GitHub for All")
- **FR-021**: System MUST log all conflict resolutions with timestamps, chosen action, and user ID for audit trail

#### Error Handling & Resilience

- **FR-022**: System MUST implement exponential backoff on API failures: 1s, 2s, 4s, 8s, 16s, max 30s between retries
- **FR-023**: System MUST retry failed sync operations maximum 5 times before logging failure and waiting for next scheduled sync
- **FR-024**: System MUST detect GitHub API rate limit (HTTP 429) and pause sync until rate limit window resets
- **FR-025**: System MUST display user-friendly error messages for: invalid PAT, repo not found, network failure, rate limit exceeded
- **FR-026**: System MUST handle PAT expiration by disabling auto-sync and prompting user to update token in Settings

#### Observability & Logging

- **FR-027**: System MUST log structured events for: sync started, sync completed, Issues fetched, Tasks created/updated, conflicts detected, errors
- **FR-028**: System MUST emit metrics: `sync_duration_ms`, `issues_imported_count`, `tasks_updated_count`, `sync_errors_total`, `conflicts_detected_count`
- **FR-029**: System MUST store sync history (last 100 syncs) with: timestamp, duration, Issues processed, success/failure status, error messages

#### Data Integrity

- **FR-030**: System MUST ensure Task creation and metadata updates are atomic (SQLite transactions)
- **FR-031**: System MUST prevent duplicate Tasks by checking existing `github_issue_number` before creating new Task
- **FR-032**: System MUST mark Tasks as "synced" only after successful database commit
- **FR-033**: System MUST handle orphaned Tasks (linked Issue deleted on GitHub) by flagging with warning badge "⚠️ Issue deleted on GitHub"

### Key Entities

- **GitHubSyncConfig**: Represents sync configuration per Vibe Kanban project
  - Attributes: `project_id` (FK), `repo_owner`, `repo_name`, `encrypted_pat`, `last_sync_at`, `next_sync_at`, `sync_enabled`, `sync_interval_seconds`
  - Relationships: One-to-one with VK Project

- **TaskGitHubMetadata**: Represents linkage between Vibe Kanban Task and GitHub Issue
  - Attributes: `task_id` (FK), `github_issue_id`, `github_issue_number`, `github_issue_url`, `last_synced_at`, `remote_updated_at`, `sync_conflict_status`
  - Relationships: One-to-one with Task (extends Task model)

- **SyncHistory**: Audit log of sync operations
  - Attributes: `sync_id`, `project_id` (FK), `started_at`, `completed_at`, `duration_ms`, `issues_fetched`, `tasks_created`, `tasks_updated`, `conflicts_detected`, `status` (success/failure), `error_message`
  - Relationships: Many-to-one with VK Project

- **ConflictResolution**: Records conflict resolution decisions
  - Attributes: `conflict_id`, `task_id` (FK), `detected_at`, `resolved_at`, `resolution_action` (keep_local/accept_github/merge), `resolved_by_user_id`, `local_content_snapshot`, `remote_content_snapshot`
  - Relationships: Many-to-one with Task

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: User can configure GitHub sync and import all open Issues from a 100-Issue repository in under 60 seconds
- **SC-002**: Incremental sync detects and updates changed Issues in under 10 seconds for repositories with <1000 Issues
- **SC-003**: System handles sync for repositories with 5000+ Issues without timeouts or memory issues (pagination working correctly)
- **SC-004**: 95% of sync operations complete successfully on first attempt (no transient failures requiring retry)
- **SC-005**: Conflict detection triggers within 30 seconds of detecting divergent local/remote changes
- **SC-006**: Users can resolve conflicts in under 2 minutes via clear UI with diff visualization
- **SC-007**: System recovers from GitHub API rate limiting automatically without user intervention
- **SC-008**: Zero data loss: all Issues present on GitHub are represented as Tasks after sync completes
- **SC-009**: User receives actionable error messages for authentication failures within 5 seconds of PAT validation
- **SC-010**: Sync history logs enable debugging of sync failures with timestamps, error messages, and retry counts

## Assumptions

1. **Single-user usage**: Shared PAT is acceptable because Nuno is the only user (simplifies auth model, no per-user OAuth)
2. **PAT expiration**: User will manually renew PAT when it expires (no auto-renewal via refresh tokens)
3. **No write-back to GitHub**: Unidirectional sync means local Task changes never update GitHub Issues (GitHub is always source of truth)
4. **SQLite encryption**: AES-256 with machine-specific key (via OS keyring or hardware ID) is sufficient security for local PAT storage
5. **Polling acceptable**: 5-minute idle polling is acceptable latency (webhook support deferred to P4 for remote deployments)
6. **Issue-only scope**: PRs, Projects, Milestones, Discussions are out of scope for MVP (can be added later)
7. **Label/assignee mapping**: GitHub labels → Vibe Kanban tags (if tag system exists); assignees stored as metadata but not used for task assignment
8. **Network reliability**: Internet connection is generally stable; exponential backoff handles transient failures
9. **Rate limit buffer**: Normal usage (1 sync every 5 min) stays well under GitHub's 5000 req/hour limit for authenticated requests
10. **Conflict rarity**: Because GitHub is source of truth, local Task edits should be rare; conflict resolution is safety net, not primary workflow

## Open Questions

*None remaining - all clarifications resolved during specification phase.*

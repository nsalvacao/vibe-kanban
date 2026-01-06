# Tasks: GitHub Issues Sync (vk-issues-sync)

**Input**: Design documents from `/specs/001-vk-issues-sync/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/github-sync.yaml

**Tests**: Following TDD workflow from `conductor/workflow.md` - tests written BEFORE implementation

**Organization**: Tasks grouped by user story (P0→P1→P2→P3) to enable independent implementation and testing

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: User story this task belongs to (US1=P0, US2=P1, US3=P2, US4=P3, US5=P2)
- Include exact file paths in descriptions

## Path Conventions

Based on plan.md structure:
- **Backend**: `crates/github-sync/`, `crates/db/`, `crates/server/`
- **Frontend**: `frontend/src/components/github-sync/`, `frontend/src/hooks/`, `frontend/src/stores/`
- **Tests**: `crates/github-sync/tests/`, `frontend/src/**/*.test.ts`, `tests/e2e/github-sync/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and GitHub sync crate structure

- [ ] T001 Create `crates/github-sync/` directory structure per plan.md
- [ ] T002 Initialize `crates/github-sync/Cargo.toml` with dependencies (octocrab, ring, tokio-cron-scheduler, machine-uid)
- [ ] T003 Add `crates/github-sync` to workspace `Cargo.toml` members list
- [ ] T004 [P] Create `crates/github-sync/src/lib.rs` with public API stub (start_sync_scheduler, manual_sync functions)
- [ ] T005 [P] Create `crates/github-sync/src/error.rs` with custom error types (GitHubSyncError enum)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### Database Schema & Migrations

- [ ] T006 Create migration `crates/db/migrations/20260105000001_github_sync_config.sql` with github_sync_config table schema
- [ ] T007 Create migration `crates/db/migrations/20260105000002_task_github_metadata.sql` with task_github_metadata table schema
- [ ] T008 Create migration `crates/db/migrations/20260105000003_sync_history.sql` with sync_history table schema
- [ ] T009 Create migration `crates/db/migrations/20260105000004_conflict_resolution.sql` with conflict_resolution table schema
- [ ] T010 Create migration `crates/db/migrations/20260105000005_add_github_issue_url_to_tasks.sql` to extend tasks table
- [ ] T011 Run `pnpm run prepare-db` to apply all new migrations

### Rust Data Models

- [ ] T012 [P] Create `crates/db/src/models/github_sync_config.rs` with GitHubSyncConfig struct per data-model.md
- [ ] T013 [P] Create `crates/db/src/models/task_github_metadata.rs` with TaskGitHubMetadata struct per data-model.md
- [ ] T014 [P] Create `crates/db/src/models/sync_history.rs` with SyncHistory struct per data-model.md
- [ ] T015 [P] Create `crates/db/src/models/conflict_resolution.rs` with ConflictResolution struct per data-model.md
- [ ] T016 Update `crates/db/src/models/task.rs` to add optional `github_issue_url` field
- [ ] T017 Update `crates/db/src/models/mod.rs` to export all new GitHub sync models

### Core Crypto & GitHub Client

- [ ] T018 [P] Create `crates/github-sync/src/crypto.rs` with encrypt_pat/decrypt_pat functions using ring + machine-uid
- [ ] T019 [P] Write unit test for `crypto.rs` encrypt/decrypt roundtrip in `crates/github-sync/tests/crypto_test.rs`
- [ ] T020 Create `crates/github-sync/src/client.rs` with GitHubClient wrapper for octocrab GraphQL API
- [ ] T021 Create `crates/github-sync/tests/fixtures/github_responses/` directory with JSON mock responses
- [ ] T022 [P] Create mock GitHub API responses in `tests/fixtures/github_responses/issues_page1.json` per research.md

### Type Generation

- [ ] T023 Run `cd crates/server && cargo run --bin generate_types` to generate TypeScript types for new models
- [ ] T024 Verify `shared/types.ts` contains GitHubSyncConfig, TaskGitHubMetadata, SyncHistory types
- [ ] T025 Run `pnpm run generate-types:check` to validate type sync in CI

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 (P0) - Initial Issues Import 🎯 MVP

**Goal**: Import all open GitHub Issues from a repository into Vibe Kanban Tasks on first sync

**Independent Test**: Configure GitHub repo + PAT, trigger sync, verify all open Issues appear as Tasks with status "Todo"

### Tests for US1 (Write FIRST, ensure they FAIL)

- [ ] T026 [P] [US1] Write unit test for `client.rs::fetch_open_issues()` using wiremock in `crates/github-sync/tests/client_test.rs`
- [ ] T027 [P] [US1] Write unit test for `sync.rs::create_task_from_issue()` in `crates/github-sync/tests/sync_test.rs`
- [ ] T028 [US1] Write integration test for full import flow in `crates/github-sync/tests/integration_import_test.rs` (depends on T026, T027)

### Backend Implementation for US1

- [ ] T029 [US1] Implement `client.rs::fetch_open_issues()` GraphQL query with cursor pagination (make T026 pass)
- [ ] T030 [US1] Implement `sync.rs::create_task_from_issue()` to convert GitHub Issue to Task + metadata (make T027 pass)
- [ ] T031 [US1] Implement `sync.rs::import_all_issues()` orchestration function (make T028 pass)
- [ ] T032 [US1] Implement `lib.rs::manual_sync()` public API to trigger import
- [ ] T033 [US1] Add structured logging (tracing::info) for import progress (issues_fetched, tasks_created)

### API Routes for US1

- [ ] T034 [US1] Create `crates/server/src/routes/github_sync.rs` with POST `/api/github-sync/config/{projectId}` endpoint
- [ ] T035 [US1] Implement POST `/api/github-sync/validate-repo` endpoint to test PAT + repo access
- [ ] T036 [US1] Implement POST `/api/github-sync/sync/{projectId}/trigger` endpoint to trigger manual sync
- [ ] T037 [US1] Update `crates/server/src/main.rs` to mount github_sync routes under /api/github-sync

### Frontend for US1

- [ ] T038 [P] [US1] Create `frontend/src/components/github-sync/GitHubSettings.tsx` with form (repo owner/name, PAT input)
- [ ] T039 [P] [US1] Create `frontend/src/hooks/useGitHubSync.ts` with TanStack Query hooks (useValidateRepo, useCreateConfig, useTriggerSync)
- [ ] T040 [US1] Implement "Test Connection" button functionality in GitHubSettings (calls useValidateRepo)
- [ ] T041 [US1] Implement "Save Configuration" button functionality (calls useCreateConfig, then useTriggerSync)
- [ ] T042 [US1] Add Settings → GitHub Integration tab navigation in main Settings UI

### E2E Test for US1

- [ ] T043 [US1] Write Playwright test `tests/e2e/github-sync/setup-and-import.spec.ts` for full P0 flow (setup → sync → verify tasks)

**Checkpoint**: User Story 1 (P0) complete - initial import working end-to-end

---

## Phase 4: User Story 2 (P1) - Incremental Sync of Updates

**Goal**: Automatically detect and apply GitHub Issue updates (title/description changes) to existing Tasks

**Independent Test**: Modify Issue #42 title on GitHub, trigger sync, verify Task title updates

### Tests for US2 (Write FIRST, ensure they FAIL)

- [ ] T044 [P] [US2] Write unit test for `sync.rs::detect_updates()` timestamp comparison in `crates/github-sync/tests/sync_test.rs`
- [ ] T045 [P] [US2] Write unit test for `sync.rs::update_task_from_issue()` in `crates/github-sync/tests/sync_test.rs`
- [ ] T046 [US2] Write integration test for incremental sync in `crates/github-sync/tests/integration_incremental_test.rs`

### Backend Implementation for US2

- [ ] T047 [US2] Implement `sync.rs::detect_updates()` to compare `remote_updated_at` > `last_synced_at` (make T044 pass)
- [ ] T048 [US2] Implement `sync.rs::update_task_from_issue()` to apply title/description changes (make T045 pass)
- [ ] T049 [US2] Implement `sync.rs::incremental_sync()` orchestration (fetch Issues, detect updates, apply) (make T046 pass)
- [ ] T050 [US2] Update `lib.rs::manual_sync()` to call incremental_sync after import (handle both initial + updates)

### Scheduler for US2 (Automatic Polling)

- [ ] T051 [US2] Create `crates/github-sync/src/scheduler.rs` with SyncScheduler struct using tokio-cron-scheduler
- [ ] T052 [US2] Implement `scheduler.rs::start()` to schedule sync every 5 minutes (idle interval)
- [ ] T053 [US2] Implement `scheduler.rs::adjust_interval()` to switch to 30s active polling when changes detected
- [ ] T054 [US2] Add graceful shutdown logic in scheduler::drop()
- [ ] T055 [US2] Initialize scheduler on Vibe Kanban server startup in `crates/server/src/main.rs`

### API Routes for US2

- [ ] T056 [US2] Implement GET `/api/github-sync/sync/{projectId}/status` endpoint to return sync status (idle, syncing, last_sync_at)
- [ ] T057 [P] [US2] Implement GET `/api/github-sync/sync/{projectId}/history` endpoint to return last 100 syncs

### Frontend for US2

- [ ] T058 [P] [US2] Create `frontend/src/components/github-sync/SyncStatusIndicator.tsx` to display "Last synced: [timestamp]"
- [ ] T059 [US2] Update useGitHubSync hook with useSyncStatus query (polls every 5s using refetchInterval)
- [ ] T060 [US2] Add SyncStatusIndicator to Settings → GitHub Integration tab
- [ ] T061 [P] [US2] Create `frontend/src/components/github-sync/ManualSyncButton.tsx` with "Sync Now" button

### E2E Test for US2

- [ ] T062 [US2] Write Playwright test `tests/e2e/github-sync/incremental-sync.spec.ts` for update detection flow

**Checkpoint**: User Story 2 (P1) complete - incremental sync working automatically + on-demand

---

## Phase 5: User Story 3 (P2) - State Mapping (Open/Closed → Todo/Done)

**Goal**: Sync GitHub Issue state changes to Task status (Open→Todo, Closed→Done)

**Independent Test**: Close Issue #123 on GitHub, trigger sync, verify Task status changes to "Done"

### Tests for US3 (Write FIRST, ensure they FAIL)

- [ ] T063 [P] [US3] Write unit test for `sync.rs::map_issue_state_to_task_status()` in `crates/github-sync/tests/sync_test.rs`
- [ ] T064 [US3] Write integration test for state mapping in `crates/github-sync/tests/integration_state_test.rs`

### Backend Implementation for US3

- [ ] T065 [US3] Implement `sync.rs::map_issue_state_to_task_status()` enum mapping (OPEN→Todo, CLOSED→Done) (make T063 pass)
- [ ] T066 [US3] Update `sync.rs::update_task_from_issue()` to apply status changes (make T064 pass)
- [ ] T067 [US3] Add closure reason handling (completed vs not_planned) in sync logging

### E2E Test for US3

- [ ] T068 [US3] Write Playwright test `tests/e2e/github-sync/state-mapping.spec.ts` for close/reopen flow

**Checkpoint**: User Story 3 (P2) complete - state changes sync automatically

---

## Phase 6: User Story 5 (P2) - Manual Sync Trigger

**Goal**: Provide "Sync Now" button to trigger immediate sync without waiting for scheduled interval

**Independent Test**: Click "Sync Now" in Settings, verify sync executes within 30 seconds

### Frontend Implementation for US5

- [ ] T069 [US5] Implement ManualSyncButton onClick handler to call useTriggerSync mutation
- [ ] T070 [US5] Add loading spinner + disable button while sync in progress
- [ ] T071 [US5] Display success/error toast on sync completion

### E2E Test for US5

- [ ] T072 [US5] Add manual sync trigger test to `tests/e2e/github-sync/incremental-sync.spec.ts`

**Checkpoint**: User Story 5 (P2) complete - manual sync button working

---

## Phase 7: User Story 4 (P3) - Conflict Resolution UI

**Goal**: Detect conflicts (local + remote changes) and provide UI to resolve (Keep Local, Accept GitHub, Merge)

**Independent Test**: Edit Task locally + GitHub Issue remotely, trigger sync, verify conflict modal appears with diff view

### Tests for US4 (Write FIRST, ensure they FAIL)

- [ ] T073 [P] [US4] Write unit test for `conflict.rs::detect_conflict()` three-way diff logic in `crates/github-sync/tests/conflict_test.rs`
- [ ] T074 [P] [US4] Write unit test for `conflict.rs::generate_diff()` using similar crate in `crates/github-sync/tests/conflict_test.rs`
- [ ] T075 [US4] Write integration test for conflict detection + resolution in `crates/github-sync/tests/integration_conflict_test.rs`

### Backend Implementation for US4

- [ ] T076 [US4] Create `crates/github-sync/src/conflict.rs` with detect_conflict() function per research.md
- [ ] T077 [US4] Implement `conflict.rs::generate_diff()` using similar crate for line-by-line diffs (make T074 pass)
- [ ] T078 [US4] Update `sync.rs::incremental_sync()` to detect conflicts and create ConflictResolution records (make T075 pass)
- [ ] T079 [US4] Update TaskGitHubMetadata to mark sync_conflict_status='pending' when conflict detected

### API Routes for US4

- [ ] T080 [US4] Implement GET `/api/github-sync/conflicts/{projectId}` endpoint to list pending conflicts
- [ ] T081 [US4] Implement POST `/api/github-sync/conflicts/{conflictId}/resolve` endpoint to apply resolution

### Frontend for US4

- [ ] T082 [P] [US4] Create `frontend/src/stores/githubSyncStore.ts` Zustand store for conflict queue
- [ ] T083 [P] [US4] Create `frontend/src/components/github-sync/ConflictModal.tsx` with three-column diff view
- [ ] T084 [US4] Implement conflict modal resolution buttons (Keep Local, Accept GitHub, Merge)
- [ ] T085 [US4] Add conflict queue polling to useGitHubSync hook (fetch pending conflicts on sync completion)
- [ ] T086 [US4] Display conflict badge "⚠️ Sync Conflict" on Tasks with pending conflicts
- [ ] T087 [US4] Trigger ConflictModal when user clicks on Task with conflict

### E2E Test for US4

- [ ] T088 [US4] Write Playwright test `tests/e2e/github-sync/conflict-resolution.spec.ts` for full conflict flow

**Checkpoint**: User Story 4 (P3) complete - conflict detection + resolution working

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Error handling, observability, performance optimization, documentation

### Error Handling & Resilience

- [ ] T089 [P] Implement exponential backoff in `client.rs::fetch_with_retry()` per research.md
- [ ] T090 [P] Implement rate limit detection in `client.rs` (HTTP 429, parse X-RateLimit-Reset header)
- [ ] T091 Update GitHubSyncConfig to store sync_error field and pause scheduler on rate limit
- [ ] T092 Add user-friendly error messages in frontend for: invalid PAT, repo not found, rate limit exceeded

### Observability

- [ ] T093 [P] Add structured logging (tracing::info/warn/error) to all sync operations (started, completed, errors)
- [ ] T094 [P] Emit metrics in SyncHistory: sync_duration_ms, issues_fetched, tasks_created/updated, conflicts_detected
- [ ] T095 [P] Optional: Add PostHog events for "github_sync_completed" (if POSTHOG_API_KEY configured)

### Performance Optimization

- [ ] T096 Verify SQLite transactions are used for all Task creation/updates (atomic operations)
- [ ] T097 Add database index validation: ensure all FKs and frequently queried fields have indexes
- [ ] T098 Test sync performance with 500+ Issues repo per quickstart.md benchmarks

### Documentation

- [ ] T099 [P] Update README.md with GitHub sync feature section (setup instructions, configuration)
- [ ] T100 [P] Create user guide in `docs/github-sync.md` with screenshots (setup, sync, conflicts)
- [ ] T101 Update AGENTS.md with GitHub sync crate information

### Final Integration

- [ ] T102 Run full test suite: `cargo test --workspace && pnpm test && npx playwright test`
- [ ] T103 Verify type generation: `pnpm run generate-types:check` passes in CI
- [ ] T104 Test local build: `./local-build.sh && cd npx-cli && node bin/cli.js` works

**Final Checkpoint**: All user stories (P0-P3) complete + polish done - ready for production

---

## Dependencies & Execution Order

### User Story Completion Order

```
Phase 1 (Setup) → Phase 2 (Foundation) → Phase 3 (US1-P0) 🎯 MVP
                                        ↓
                        Phase 4 (US2-P1) ← Can start after US1 complete
                                        ↓
                        Phase 5 (US3-P2) ← Can start after US2 (needs incremental sync)
                                        ↓
                        Phase 6 (US5-P2) ← Can start after US2 (uses manual sync)
                                        ↓
                        Phase 7 (US4-P3) ← Can start after US2 (needs conflict detection)
                                        ↓
                        Phase 8 (Polish) ← Final pass after all stories
```

### Parallelization Opportunities

**Within Phase 2 (Foundation)**:
- T012-T017 (Rust models) can run in parallel - different files
- T018-T019 (crypto) can run parallel to T020-T022 (client)

**Within Phase 3 (US1)**:
- T026-T027 (unit tests) can run in parallel
- T038-T039 (frontend) can run parallel to T034-T037 (backend routes)

**Within Phase 4 (US2)**:
- T044-T045 (unit tests) can run in parallel
- T056-T057 (API routes) can run parallel to T058-T061 (frontend)

**Within Phase 7 (US4)**:
- T073-T074 (unit tests) can run in parallel
- T082-T083 (frontend stores + modal) can run in parallel

**Phase 8 (Polish)**:
- T089-T095 (error handling + observability) are all parallelizable
- T099-T101 (documentation) are all parallelizable

---

## Implementation Strategy

### MVP Scope (Minimum Viable Product)

**MVP = Phase 1 + Phase 2 + Phase 3 (US1 only)**

This delivers:
- ✅ Initial import of GitHub Issues → Tasks
- ✅ PAT encryption + storage
- ✅ Manual sync trigger
- ✅ Settings UI for configuration
- ✅ Basic error handling

**MVP Validation**: Can import Issues from a real repo and see them as Tasks in Vibe Kanban.

### Incremental Delivery

After MVP, deliver in priority order:
1. **Phase 4 (US2-P1)**: Automatic polling + incremental updates (highest value after MVP)
2. **Phase 5 (US3-P2) + Phase 6 (US5-P2)**: State mapping + manual trigger (parallel-izable)
3. **Phase 7 (US4-P3)**: Conflict resolution (safety net, less common scenario)
4. **Phase 8 (Polish)**: Production-readiness (error handling, observability, docs)

### TDD Workflow Per Task

Per `conductor/workflow.md`:
1. Write test (RED phase) - ensure it fails
2. Run test suite: `cargo test` or `pnpm test`
3. Implement code (GREEN phase) - make test pass
4. Run test suite again - verify pass
5. Refactor if needed - tests still pass
6. Commit with git notes: task summary + files changed

---

## Task Summary

**Total Tasks**: 104
- **Phase 1 (Setup)**: 5 tasks
- **Phase 2 (Foundation)**: 20 tasks (BLOCKING - must complete first)
- **Phase 3 (US1-P0)**: 18 tasks 🎯 MVP
- **Phase 4 (US2-P1)**: 19 tasks
- **Phase 5 (US3-P2)**: 6 tasks
- **Phase 6 (US5-P2)**: 4 tasks
- **Phase 7 (US4-P3)**: 17 tasks
- **Phase 8 (Polish)**: 15 tasks

**Parallelization**: 41 tasks marked [P] (39% parallelizable within phases)

**MVP Tasks**: 43 tasks (Phase 1 + 2 + 3)

**Estimated Timeline** (1 developer, TDD workflow):
- MVP (43 tasks): ~3-4 weeks
- Full Feature (104 tasks): ~6-8 weeks

---

## Next Steps

1. **Review tasks.md** with stakeholder (Nuno)
2. **Run `/speckit.analyze`** to validate consistency (spec ↔ plan ↔ tasks)
3. **Begin implementation** with Phase 1 (Setup) following TDD workflow
4. **Track progress** using git notes per `conductor/workflow.md`
5. **Phase checkpoints** after US1, US2, US3+US5, US4 with manual verification

**Ready to implement** - All design artifacts complete, tasks actionable and testable.

# Implementation Plan: GitHub Issues Sync (vk-issues-sync)

**Branch**: `001-vk-issues-sync` | **Date**: 2026-01-05 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-vk-issues-sync/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

**Primary Requirement**: Unidirectional synchronization of GitHub Issues into Vibe Kanban Tasks, enabling developers to orchestrate AI agents on existing backlogs without manual task creation.

**Technical Approach**:
- **Backend**: New Rust crate `crates/github-sync` using `octocrab` for GitHub GraphQL API integration
- **Storage**: Extend existing SQLite schema with two new tables (`github_sync_config`, `task_github_metadata`) and add optional fields to Task model
- **Sync Engine**: Polling-based scheduler (5min idle, 30s active) with exponential backoff and transactional updates
- **Frontend**: React Settings panel for GitHub configuration, sync status indicator, and conflict resolution modal
- **Security**: AES-256 PAT encryption in SQLite using `ring` crate with machine-specific salt
- **Testing**: Mocked GitHub API responses for unit tests, integration tests with fixture repos, E2E Playwright tests for sync flow

## Technical Context

**Language/Version**: Rust 1.80+ (stable, per `rust-toolchain.toml`), TypeScript 5.x  
**Primary Dependencies**:
- **Backend**: `octocrab = "0.40"` (GitHub API), `ring = "0.17"` (encryption), `tokio-cron-scheduler = "0.13"` (polling)
- **Frontend**: React 18, Zustand (state), TanStack Query (API calls), Radix UI (modals)

**Storage**: SQLite via SQLx (offline mode) - extend with 2 new tables + Task model migration  
**Testing**: `cargo test` (backend unit/integration), `pnpm test:unit` (Vitest), `npx playwright test` (E2E)  
**Target Platform**: Local (npx) + Remote (systemctl/Docker) - Linux/macOS/WSL2  
**Project Type**: Monorepo web application (`crates/` backend + `frontend/` React)  
**Performance Goals**: 
- Import 100 Issues in <60s (SC-001)
- Incremental sync <10s for <1000 Issues (SC-002)
- 95% sync success rate (SC-004)

**Constraints**: 
- No external cloud dependencies (local-first, Constitution III)
- GitHub API rate limit: 5000 req/hour (authenticated)
- SQLite single-writer limitation (sync must serialize DB writes)
- PAT storage security: encrypted at-rest, machine-bound key

**Scale/Scope**: 
- Support repos up to 5000 Issues (SC-003)
- Max 10 concurrent projects with GitHub sync enabled
- Sync history retention: last 100 syncs per project

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Principle I: Multi-Agent Orchestration First
**Status**: ✅ **PASS**  
**Justification**: Feature directly enables agents to work on real GitHub backlogs without manual task creation. Synced Tasks enter existing worktree/agent execution flow unchanged.

### Principle II: Rust Backend + React Frontend Architecture
**Status**: ✅ **PASS**  
**Justification**: 
- Backend: New Rust crate `crates/github-sync` (modular, follows workspace pattern)
- Frontend: React components in `frontend/src/components/github-sync/`
- Type safety: ts-rs for `GitHubSyncStatus`, `SyncConfig`, `ConflictResolution` types
- No stack deviations required

### Principle III: Local-First with Remote Deployment Support
**Status**: ✅ **PASS**  
**Justification**:
- Encrypted PAT stored locally in SQLite (no cloud auth services)
- Sync works offline after initial import (Tasks persist locally)
- Polling strategy works for both local (`npx`) and remote (systemctl) deployments
- No external services required (GitHub API only, opt-in)

### Principle IV: Git Worktree Isolation (NON-NEGOTIABLE)
**Status**: ✅ **PASS**  
**Justification**: Feature creates Tasks that enter existing worktree isolation flow. Sync does not touch worktree management logic. No risk of breaking isolation model.

### Principle V: Code Review & Dev Server Integration
**Status**: ✅ **PASS**  
**Justification**: Tasks include `github_issue_url` field → Review UI can display GitHub Issue link for context. No changes to review workflow needed.

### Principle VI: Testing & Type Safety
**Status**: ✅ **PASS**  
**Justification**:
- All new Rust structs derive `TS` for type generation
- `pnpm run generate-types` will export `GitHubSyncConfig`, `TaskGitHubMetadata`, etc.
- Backend: Unit tests with mocked `octocrab` responses + integration tests with fixture repos
- Frontend: Vitest for Settings components + Playwright E2E for full sync flow
- CI: `generate-types:check` enforces sync

**Overall Constitution Compliance**: ✅ **APPROVED** - No violations, no complexity justification needed.

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

## Project Structure

### Documentation (this feature)

```text
specs/001-vk-issues-sync/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (octocrab patterns, encryption, polling)
├── data-model.md        # Phase 1 output (DB schema + Rust models)
├── quickstart.md        # Phase 1 output (dev setup + manual testing)
├── contracts/           # Phase 1 output (API endpoints OpenAPI schema)
│   └── github-sync.yaml
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
# Backend: New crate + DB migrations
crates/
├── github-sync/         # NEW: Sync engine crate
│   ├── Cargo.toml       # Dependencies: octocrab, tokio-cron-scheduler, ring
│   ├── src/
│   │   ├── lib.rs       # Public API: start_sync_scheduler, manual_sync
│   │   ├── client.rs    # GitHub API wrapper (octocrab)
│   │   ├── sync.rs      # Core sync logic (fetch, diff, update)
│   │   ├── scheduler.rs # Cron-based polling (5min idle, 30s active)
│   │   ├── crypto.rs    # PAT encryption/decryption (ring + machine salt)
│   │   └── conflict.rs  # Conflict detection logic
│   └── tests/
│       ├── mock_github.rs  # Mocked API responses
│       └── sync_test.rs    # Integration tests
│
├── db/
│   ├── migrations/
│   │   ├── 20260105_github_sync_config.sql      # NEW: Config table
│   │   ├── 20260105_task_github_metadata.sql    # NEW: Metadata table
│   │   └── 20260105_add_github_fields_to_task.sql # Task model extension
│   └── src/models/
│       ├── github_sync_config.rs  # NEW: SyncConfig model
│       ├── task_github_metadata.rs # NEW: Metadata model
│       ├── sync_history.rs        # NEW: Audit log model
│       └── task.rs                # MODIFIED: Add optional github_issue_url field
│
└── server/
    └── src/
        └── routes/
            └── github_sync.rs  # NEW: API endpoints (/api/github-sync/*)

# Frontend: Settings UI + Sync components
frontend/
└── src/
    ├── components/
    │   └── github-sync/        # NEW: GitHub sync components
    │       ├── GitHubSettings.tsx       # Settings panel
    │       ├── SyncStatusIndicator.tsx  # Sync progress/status
    │       ├── ConflictModal.tsx        # Three-way diff UI
    │       └── ManualSyncButton.tsx     # "Sync Now" trigger
    ├── hooks/
    │   └── useGitHubSync.ts    # NEW: TanStack Query hooks
    └── stores/
        └── githubSyncStore.ts  # NEW: Zustand store for sync state

# Tests
tests/e2e/
└── github-sync/
    ├── setup-and-import.spec.ts      # P0: Initial import flow
    ├── incremental-sync.spec.ts      # P1: Update detection
    └── conflict-resolution.spec.ts   # P3: Conflict modal flow
```

**Structure Decision**: Follows existing Vibe Kanban monorepo pattern:
- New modular crate (`crates/github-sync`) maintains non-invasive design (can be disabled easily)
- DB migrations extend schema without modifying core Task logic
- Frontend components isolate GitHub-specific UI to dedicated directory
- Aligns with Constitution Principle II (Rust backend + React frontend architecture)

## Complexity Tracking

**No complexity justification required** - Constitution Check passed with zero violations.

## Phase 0: Research

### Research Topics

1. **Octocrab GitHub API Integration**
   - GraphQL vs REST API tradeoffs for Issue fetching
   - Cursor-based pagination patterns for repos >100 Issues
   - Rate limit detection and backoff strategies
   - Mocking patterns for unit tests

2. **PAT Encryption Strategy**
   - `ring` crate AES-256-GCM encryption best practices
   - Machine-specific salt generation (hardware ID vs OS keyring)
   - SQLite BLOB storage for encrypted PATs
   - Key rotation considerations

3. **Tokio Cron Scheduler**
   - Dynamic interval adjustment (5min → 30s when active)
   - Graceful shutdown on application exit
   - Single-instance enforcement (prevent concurrent syncs)
   - Error recovery patterns

4. **Conflict Detection Logic**
   - Three-way diff algorithm (base, local, remote)
   - Timestamp-based staleness detection
   - SQLite transaction isolation levels for concurrent reads/writes

5. **Frontend State Management**
   - TanStack Query polling patterns for sync status
   - Zustand store design for conflict queue
   - Optimistic UI updates during sync

### Research Output

✅ **COMPLETE** - See [research.md](research.md) for detailed findings, decisions, and rationale.

**Key Decisions**:
- GraphQL API with `octocrab` (cursor pagination, field selection)
- AES-256-GCM with `machine-uid` salt (simplest secure solution per user requirement)
- `tokio-cron-scheduler` for dynamic intervals (5min → 30s transition)
- Timestamp-based three-way diff (minimal storage, clear conflict detection)
- TanStack Query + Zustand hybrid (server state vs client state separation)

---

## Phase 1: Design & Contracts

### Data Model

✅ **COMPLETE** - See [data-model.md](data-model.md)

**Artifacts**:
- 4 new tables: `github_sync_config`, `task_github_metadata`, `sync_history`, `conflict_resolution`
- Optional `tasks` table extension: `github_issue_url` field
- Rust models with `#[derive(TS)]` for type generation
- Validation rules and state transitions documented

### API Contracts

✅ **COMPLETE** - See [contracts/github-sync.yaml](contracts/github-sync.yaml)

**Endpoints**:
- `GET/POST/PATCH/DELETE /api/github-sync/config/{projectId}` - Configuration CRUD
- `POST /api/github-sync/sync/{projectId}/trigger` - Manual sync
- `GET /api/github-sync/sync/{projectId}/status` - Current status
- `GET /api/github-sync/sync/{projectId}/history` - Last 100 syncs
- `GET /api/github-sync/conflicts/{projectId}` - Pending conflicts
- `POST /api/github-sync/conflicts/{conflictId}/resolve` - Resolve conflict
- `POST /api/github-sync/validate-repo` - Test connection

### Quickstart Guide

✅ **COMPLETE** - See [quickstart.md](quickstart.md)

**Includes**:
- Dev environment setup (dependencies, migrations, type gen)
- 5 manual testing workflows (P0-P3 user stories)
- API call examples with expected responses
- Debugging tips (logs, database inspection, mocked API)
- Performance benchmarks aligned with Success Criteria

### Agent Context Update

✅ **COMPLETE** - Claude agent context updated with:
- Language: Rust 1.80+ stable + TypeScript 5.x
- Database: SQLite via SQLx (2 new tables + Task model extension)
- Project type: Monorepo web application

---

## Phase 2: Tasks Generation

**Status**: ⏸️ **PENDING** - Run `/speckit.tasks` to generate task breakdown

This phase is handled by the `/speckit.tasks` command, which will:
1. Parse this implementation plan
2. Extract P0-P3 user stories from spec.md
3. Generate atomic tasks with acceptance criteria
4. Create `tasks.md` with TDD workflow checkpoints

**Expected Output**: `specs/001-vk-issues-sync/tasks.md`

---

## Constitution Re-Check (Post-Design)

**Status**: ✅ **COMPLIANT** - No design changes required re-evaluation

### Design Validation Against Constitution

**Principle I (Multi-Agent Orchestration)**: ✅ Design unchanged - Tasks still enter existing worktree flow

**Principle II (Rust + React Stack)**: ✅ Confirmed:
- New crate `crates/github-sync` (Rust workspace pattern)
- Frontend components in `frontend/src/components/github-sync/` (React)
- ts-rs types generated for all new models

**Principle III (Local-First)**: ✅ Confirmed:
- PAT encrypted locally with machine-bound key (no cloud auth)
- Sync works offline after initial import (SQLite local storage)
- No mandatory external services (GitHub API is opt-in)

**Principle IV (Worktree Isolation)**: ✅ Design does not touch worktree logic - Tasks created via sync use existing Task model

**Principle V (Code Review Integration)**: ✅ `github_issue_url` field added to Task model → Review UI can link to GitHub

**Principle VI (Testing & Type Safety)**: ✅ Confirmed:
- All models derive `TS` (GitHubSyncConfig, TaskGitHubMetadata, SyncHistory)
- Unit tests with mocked octocrab (wiremock)
- Integration tests with fixture repos
- E2E Playwright tests for sync flow

**Overall Compliance**: ✅ **NO VIOLATIONS** - Design fully aligns with Constitution

---

## Implementation Readiness

**Phase 0 (Research)**: ✅ Complete  
**Phase 1 (Design)**: ✅ Complete  
**Phase 2 (Tasks)**: ⏸️ Ready to generate  
**Constitution Check**: ✅ Compliant

**Next Command**: `/speckit.tasks` - Generate atomic task breakdown for implementation

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |

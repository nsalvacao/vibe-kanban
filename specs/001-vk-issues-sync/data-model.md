# Data Model: GitHub Issues Sync

**Feature**: 001-vk-issues-sync  
**Date**: 2026-01-05  
**Status**: Ready for Implementation

## Entity Relationship Diagram

```
┌─────────────────────┐
│      Project        │
│                     │
│  id: UUID (PK)      │
│  name: String       │
│  ...                │
└──────────┬──────────┘
           │
           │ 1:1
           │
┌──────────▼──────────────────────┐
│  GitHubSyncConfig               │
│                                 │
│  id: UUID (PK)                  │
│  project_id: UUID (FK) UNIQUE   │◄───── One config per project
│  repo_owner: String             │
│  repo_name: String              │
│  encrypted_pat: BLOB            │       Encrypted PAT (AES-256-GCM)
│  last_sync_at: DateTime?        │
│  next_sync_at: DateTime?        │
│  sync_enabled: BOOLEAN          │
│  sync_interval_seconds: INT     │       300 (5min) or 30 (active)
│  sync_error: String?            │       Last error message
│  created_at: DateTime           │
│  updated_at: DateTime           │
└─────────────────────────────────┘


┌─────────────────────┐
│      Task           │
│                     │
│  id: UUID (PK)      │
│  project_id: UUID   │
│  title: String      │
│  description: String?│
│  status: TaskStatus │
│  created_at: DateTime│
│  updated_at: DateTime│
└──────────┬──────────┘
           │
           │ 1:0..1 (optional)
           │
┌──────────▼──────────────────────┐
│  TaskGitHubMetadata             │
│                                 │
│  id: UUID (PK)                  │
│  task_id: UUID (FK) UNIQUE      │◄───── One-to-one with Task
│  github_issue_id: String        │       GitHub node ID (e.g., "I_kwDO...")
│  github_issue_number: INT       │       Human-readable Issue #
│  github_issue_url: String       │       Full URL for UI linking
│  last_synced_at: DateTime       │       Timestamp of last sync
│  remote_updated_at: DateTime    │       GitHub Issue's updatedAt
│  last_synced_content: TEXT      │       Snapshot of title+desc at last sync
│  sync_conflict_status: TEXT?    │       "pending" | "resolved" | NULL
│  created_at: DateTime           │
│  updated_at: DateTime           │
└─────────────────────────────────┘


┌─────────────────────────────────┐
│  SyncHistory                    │
│                                 │
│  id: UUID (PK)                  │
│  project_id: UUID (FK)          │
│  sync_config_id: UUID (FK)      │
│  started_at: DateTime           │
│  completed_at: DateTime?        │       NULL if sync in progress
│  duration_ms: INT?              │
│  issues_fetched: INT            │
│  tasks_created: INT             │
│  tasks_updated: INT             │
│  conflicts_detected: INT        │
│  status: TEXT                   │       "success" | "failure" | "in_progress"
│  error_message: TEXT?           │
└─────────────────────────────────┘


┌─────────────────────────────────┐
│  ConflictResolution             │
│                                 │
│  id: UUID (PK)                  │
│  task_id: UUID (FK)             │
│  metadata_id: UUID (FK)         │
│  detected_at: DateTime          │
│  resolved_at: DateTime?         │       NULL until user resolves
│  resolution_action: TEXT?       │       "keep_local" | "accept_github" | "merge"
│  local_content_snapshot: TEXT   │       Task content at detection
│  remote_content_snapshot: TEXT  │       GitHub Issue content at detection
│  merged_content: TEXT?          │       User's merged result (if "merge")
└─────────────────────────────────┘
```

---

## SQL Schema

### Table: `github_sync_config`

```sql
CREATE TABLE github_sync_config (
    id TEXT PRIMARY KEY NOT NULL,  -- UUID v4
    project_id TEXT NOT NULL UNIQUE,  -- FK to projects(id)
    repo_owner TEXT NOT NULL,
    repo_name TEXT NOT NULL,
    encrypted_pat BLOB NOT NULL,  -- AES-256-GCM encrypted PAT
    last_sync_at TEXT,  -- ISO 8601 datetime
    next_sync_at TEXT,  -- ISO 8601 datetime
    sync_enabled INTEGER NOT NULL DEFAULT 1,  -- Boolean (1 = enabled)
    sync_interval_seconds INTEGER NOT NULL DEFAULT 300,  -- 5 minutes
    sync_error TEXT,  -- Last error message (NULL if no error)
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    
    FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE
);

CREATE INDEX idx_github_sync_config_project_id ON github_sync_config(project_id);
CREATE INDEX idx_github_sync_config_next_sync ON github_sync_config(next_sync_at) 
    WHERE sync_enabled = 1;  -- Partial index for scheduler queries
```

---

### Table: `task_github_metadata`

```sql
CREATE TABLE task_github_metadata (
    id TEXT PRIMARY KEY NOT NULL,  -- UUID v4
    task_id TEXT NOT NULL UNIQUE,  -- FK to tasks(id), one-to-one
    github_issue_id TEXT NOT NULL,  -- GitHub GraphQL node ID (e.g., "I_kwDOABcD...")
    github_issue_number INTEGER NOT NULL,  -- Human-readable Issue # (e.g., 42)
    github_issue_url TEXT NOT NULL,  -- Full URL (e.g., "https://github.com/owner/repo/issues/42")
    last_synced_at TEXT NOT NULL,  -- ISO 8601 datetime
    remote_updated_at TEXT NOT NULL,  -- GitHub Issue's updatedAt timestamp
    last_synced_content TEXT NOT NULL,  -- JSON: {"title": "...", "description": "..."}
    sync_conflict_status TEXT,  -- "pending" | "resolved" | NULL
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    
    FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE CASCADE
);

CREATE UNIQUE INDEX idx_task_github_metadata_task_id ON task_github_metadata(task_id);
CREATE INDEX idx_task_github_metadata_issue_number ON task_github_metadata(github_issue_number);
CREATE INDEX idx_task_github_metadata_conflict_status ON task_github_metadata(sync_conflict_status)
    WHERE sync_conflict_status = 'pending';  -- Partial index for conflict resolution UI
```

---

### Table: `sync_history`

```sql
CREATE TABLE sync_history (
    id TEXT PRIMARY KEY NOT NULL,  -- UUID v4
    project_id TEXT NOT NULL,  -- FK to projects(id)
    sync_config_id TEXT NOT NULL,  -- FK to github_sync_config(id)
    started_at TEXT NOT NULL,  -- ISO 8601 datetime
    completed_at TEXT,  -- ISO 8601 datetime, NULL if in progress
    duration_ms INTEGER,  -- Milliseconds, NULL if in progress
    issues_fetched INTEGER NOT NULL DEFAULT 0,
    tasks_created INTEGER NOT NULL DEFAULT 0,
    tasks_updated INTEGER NOT NULL DEFAULT 0,
    conflicts_detected INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL,  -- "success" | "failure" | "in_progress"
    error_message TEXT,  -- NULL if status = "success"
    
    FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE,
    FOREIGN KEY (sync_config_id) REFERENCES github_sync_config(id) ON DELETE CASCADE
);

CREATE INDEX idx_sync_history_project_id ON sync_history(project_id);
CREATE INDEX idx_sync_history_started_at ON sync_history(started_at DESC);  -- For "last 100 syncs" queries
```

---

### Table: `conflict_resolution`

```sql
CREATE TABLE conflict_resolution (
    id TEXT PRIMARY KEY NOT NULL,  -- UUID v4
    task_id TEXT NOT NULL,  -- FK to tasks(id)
    metadata_id TEXT NOT NULL,  -- FK to task_github_metadata(id)
    detected_at TEXT NOT NULL,  -- ISO 8601 datetime
    resolved_at TEXT,  -- ISO 8601 datetime, NULL until resolved
    resolution_action TEXT,  -- "keep_local" | "accept_github" | "merge" (NULL until resolved)
    local_content_snapshot TEXT NOT NULL,  -- JSON: {"title": "...", "description": "..."}
    remote_content_snapshot TEXT NOT NULL,  -- JSON: {"title": "...", "description": "..."}
    merged_content TEXT,  -- JSON: {"title": "...", "description": "..."} (only if resolution_action = "merge")
    
    FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE CASCADE,
    FOREIGN KEY (metadata_id) REFERENCES task_github_metadata(id) ON DELETE CASCADE
);

CREATE INDEX idx_conflict_resolution_task_id ON conflict_resolution(task_id);
CREATE INDEX idx_conflict_resolution_resolved_at ON conflict_resolution(resolved_at)
    WHERE resolved_at IS NULL;  -- Partial index for unresolved conflicts
```

---

### Migration: Add `github_issue_url` to `tasks` table (optional denormalization)

**Note**: This is optional - `github_issue_url` can be retrieved via JOIN with `task_github_metadata`. However, adding it to `tasks` table simplifies UI queries (no JOIN needed for displaying Issue link).

```sql
-- Migration: 20260105_add_github_issue_url_to_tasks.sql
ALTER TABLE tasks ADD COLUMN github_issue_url TEXT;

CREATE INDEX idx_tasks_github_issue_url ON tasks(github_issue_url)
    WHERE github_issue_url IS NOT NULL;  -- Partial index for GitHub-synced tasks
```

---

## Rust Model Structs

### `crates/db/src/models/github_sync_config.rs`

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::{FromRow, SqlitePool};
use ts_rs::TS;
use uuid::Uuid;

#[derive(Debug, Clone, FromRow, Serialize, Deserialize, TS)]
pub struct GitHubSyncConfig {
    pub id: Uuid,
    pub project_id: Uuid,
    pub repo_owner: String,
    pub repo_name: String,
    #[serde(skip)]  // Never serialize encrypted PAT to frontend
    pub encrypted_pat: Vec<u8>,
    pub last_sync_at: Option<DateTime<Utc>>,
    pub next_sync_at: Option<DateTime<Utc>>,
    pub sync_enabled: bool,
    pub sync_interval_seconds: i32,  // 300 or 30
    pub sync_error: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Deserialize, TS)]
pub struct CreateGitHubSyncConfig {
    pub project_id: Uuid,
    pub repo_owner: String,
    pub repo_name: String,
    pub pat: String,  // Plaintext PAT (encrypted before storage)
}

#[derive(Debug, Clone, Deserialize, TS)]
pub struct UpdateGitHubSyncConfig {
    pub repo_owner: Option<String>,
    pub repo_name: Option<String>,
    pub pat: Option<String>,  // If provided, re-encrypt
    pub sync_enabled: Option<bool>,
    pub sync_interval_seconds: Option<i32>,
}

impl GitHubSyncConfig {
    pub async fn create(
        pool: &SqlitePool,
        create: CreateGitHubSyncConfig,
    ) -> sqlx::Result<Self> {
        let id = Uuid::new_v4();
        let encrypted_pat = encrypt_pat(&create.pat)?;  // From crates/github-sync/src/crypto.rs
        
        sqlx::query_as::<_, Self>(
            r#"
            INSERT INTO github_sync_config 
            (id, project_id, repo_owner, repo_name, encrypted_pat, sync_enabled, sync_interval_seconds)
            VALUES (?, ?, ?, ?, ?, 1, 300)
            RETURNING *
            "#
        )
        .bind(id)
        .bind(create.project_id)
        .bind(create.repo_owner)
        .bind(create.repo_name)
        .bind(encrypted_pat)
        .fetch_one(pool)
        .await
    }
    
    pub async fn find_by_project_id(
        pool: &SqlitePool,
        project_id: Uuid,
    ) -> sqlx::Result<Option<Self>> {
        sqlx::query_as::<_, Self>(
            "SELECT * FROM github_sync_config WHERE project_id = ?"
        )
        .bind(project_id)
        .fetch_optional(pool)
        .await
    }
    
    pub fn decrypt_pat(&self) -> Result<String, CryptoError> {
        decrypt_pat(&self.encrypted_pat)  // From crates/github-sync/src/crypto.rs
    }
}
```

---

### `crates/db/src/models/task_github_metadata.rs`

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::{FromRow, SqlitePool};
use ts_rs::TS;
use uuid::Uuid;

#[derive(Debug, Clone, FromRow, Serialize, Deserialize, TS)]
pub struct TaskGitHubMetadata {
    pub id: Uuid,
    pub task_id: Uuid,
    pub github_issue_id: String,  // Node ID (e.g., "I_kwDOABcD...")
    pub github_issue_number: i32,
    pub github_issue_url: String,
    pub last_synced_at: DateTime<Utc>,
    pub remote_updated_at: DateTime<Utc>,
    pub last_synced_content: String,  // JSON string
    pub sync_conflict_status: Option<String>,  // "pending" | "resolved" | NULL
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Deserialize, TS)]
pub struct CreateTaskGitHubMetadata {
    pub task_id: Uuid,
    pub github_issue_id: String,
    pub github_issue_number: i32,
    pub github_issue_url: String,
    pub remote_updated_at: DateTime<Utc>,
    pub last_synced_content: String,  // JSON
}

impl TaskGitHubMetadata {
    pub async fn create(
        pool: &SqlitePool,
        create: CreateTaskGitHubMetadata,
    ) -> sqlx::Result<Self> {
        let id = Uuid::new_v4();
        let now = Utc::now();
        
        sqlx::query_as::<_, Self>(
            r#"
            INSERT INTO task_github_metadata 
            (id, task_id, github_issue_id, github_issue_number, github_issue_url, 
             last_synced_at, remote_updated_at, last_synced_content)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?)
            RETURNING *
            "#
        )
        .bind(id)
        .bind(create.task_id)
        .bind(create.github_issue_id)
        .bind(create.github_issue_number)
        .bind(create.github_issue_url)
        .bind(now)
        .bind(create.remote_updated_at)
        .bind(create.last_synced_content)
        .fetch_one(pool)
        .await
    }
    
    pub async fn find_by_task_id(
        pool: &SqlitePool,
        task_id: Uuid,
    ) -> sqlx::Result<Option<Self>> {
        sqlx::query_as::<_, Self>(
            "SELECT * FROM task_github_metadata WHERE task_id = ?"
        )
        .bind(task_id)
        .fetch_optional(pool)
        .await
    }
    
    pub async fn mark_conflict(&mut self, pool: &SqlitePool) -> sqlx::Result<()> {
        self.sync_conflict_status = Some("pending".to_string());
        
        sqlx::query(
            "UPDATE task_github_metadata SET sync_conflict_status = 'pending', updated_at = ? WHERE id = ?"
        )
        .bind(Utc::now())
        .bind(self.id)
        .execute(pool)
        .await?;
        
        Ok(())
    }
}
```

---

### `crates/db/src/models/sync_history.rs`

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::{FromRow, SqlitePool};
use ts_rs::TS;
use uuid::Uuid;

#[derive(Debug, Clone, FromRow, Serialize, Deserialize, TS)]
pub struct SyncHistory {
    pub id: Uuid,
    pub project_id: Uuid,
    pub sync_config_id: Uuid,
    pub started_at: DateTime<Utc>,
    pub completed_at: Option<DateTime<Utc>>,
    pub duration_ms: Option<i32>,
    pub issues_fetched: i32,
    pub tasks_created: i32,
    pub tasks_updated: i32,
    pub conflicts_detected: i32,
    pub status: String,  // "success" | "failure" | "in_progress"
    pub error_message: Option<String>,
}

#[derive(Debug, Clone, Serialize, TS)]
pub struct SyncHistorySummary {
    pub last_100_syncs: Vec<SyncHistory>,
    pub total_issues_imported: i32,
    pub total_conflicts: i32,
    pub avg_duration_ms: f64,
    pub success_rate: f64,  // Percentage (0-100)
}

impl SyncHistory {
    pub async fn start_sync(
        pool: &SqlitePool,
        project_id: Uuid,
        sync_config_id: Uuid,
    ) -> sqlx::Result<Self> {
        let id = Uuid::new_v4();
        let now = Utc::now();
        
        sqlx::query_as::<_, Self>(
            r#"
            INSERT INTO sync_history 
            (id, project_id, sync_config_id, started_at, status)
            VALUES (?, ?, ?, ?, 'in_progress')
            RETURNING *
            "#
        )
        .bind(id)
        .bind(project_id)
        .bind(sync_config_id)
        .bind(now)
        .fetch_one(pool)
        .await
    }
    
    pub async fn complete_sync(
        &mut self,
        pool: &SqlitePool,
        result: SyncResult,
    ) -> sqlx::Result<()> {
        let now = Utc::now();
        let duration_ms = (now - self.started_at).num_milliseconds() as i32;
        
        self.completed_at = Some(now);
        self.duration_ms = Some(duration_ms);
        self.status = result.status;
        self.issues_fetched = result.issues_fetched;
        self.tasks_created = result.tasks_created;
        self.tasks_updated = result.tasks_updated;
        self.conflicts_detected = result.conflicts_detected;
        self.error_message = result.error_message;
        
        sqlx::query(
            r#"
            UPDATE sync_history 
            SET completed_at = ?, duration_ms = ?, status = ?, 
                issues_fetched = ?, tasks_created = ?, tasks_updated = ?, 
                conflicts_detected = ?, error_message = ?
            WHERE id = ?
            "#
        )
        .bind(now)
        .bind(duration_ms)
        .bind(&self.status)
        .bind(self.issues_fetched)
        .bind(self.tasks_created)
        .bind(self.tasks_updated)
        .bind(self.conflicts_detected)
        .bind(&self.error_message)
        .bind(self.id)
        .execute(pool)
        .await?;
        
        Ok(())
    }
    
    pub async fn get_last_100(
        pool: &SqlitePool,
        project_id: Uuid,
    ) -> sqlx::Result<Vec<Self>> {
        sqlx::query_as::<_, Self>(
            r#"
            SELECT * FROM sync_history 
            WHERE project_id = ?
            ORDER BY started_at DESC
            LIMIT 100
            "#
        )
        .bind(project_id)
        .fetch_all(pool)
        .await
    }
}

#[derive(Debug, Clone)]
pub struct SyncResult {
    pub status: String,  // "success" | "failure"
    pub issues_fetched: i32,
    pub tasks_created: i32,
    pub tasks_updated: i32,
    pub conflicts_detected: i32,
    pub error_message: Option<String>,
}
```

---

## Validation Rules

### `GitHubSyncConfig`

- `repo_owner` and `repo_name` must be non-empty strings (validated in API layer)
- `encrypted_pat` must decrypt successfully before creating config (test decryption on create)
- `sync_interval_seconds` must be `>= 30` and `<= 3600` (30s to 1 hour)
- `project_id` must exist in `projects` table (enforced by FK constraint)
- Only one config per project (enforced by `UNIQUE(project_id)` constraint)

### `TaskGitHubMetadata`

- `github_issue_number` must be positive integer
- `github_issue_url` must match pattern `https://github.com/{owner}/{repo}/issues/{number}`
- `last_synced_content` must be valid JSON: `{"title": "...", "description": "..."}`
- `sync_conflict_status` must be one of: `"pending"`, `"resolved"`, or `NULL`
- `task_id` must exist in `tasks` table (enforced by FK constraint)
- One metadata per task (enforced by `UNIQUE(task_id)` constraint)

### `SyncHistory`

- `status` must be one of: `"success"`, `"failure"`, `"in_progress"`
- `duration_ms` must be `NULL` while status is `"in_progress"`
- `completed_at` must be `NULL` while status is `"in_progress"`
- All count fields (`issues_fetched`, `tasks_created`, etc.) must be `>= 0`

### `ConflictResolution`

- `resolution_action` must be one of: `"keep_local"`, `"accept_github"`, `"merge"`, or `NULL`
- `merged_content` must be `NULL` unless `resolution_action = "merge"`
- `resolved_at` must be `NULL` until `resolution_action` is set
- Content snapshots must be valid JSON: `{"title": "...", "description": "..."}`

---

## State Transitions

### Sync Status States

```
[Not Configured]
    │
    │ User configures GitHub repo + PAT
    ▼
[Idle] ◄──────────────────┐
    │                     │
    │ Timer triggers      │ Sync completes successfully
    ▼                     │
[Syncing] ────────────────┘
    │
    │ Error occurs
    ▼
[Error] ──────────────────► User fixes (e.g., updates PAT)
                            │
                            └──► [Idle]
```

### Conflict Resolution States

```
[No Conflict]
    │
    │ Both local and remote modified since last sync
    ▼
[Pending] ◄────────────────┐
    │                      │
    │ User opens modal     │ User clicks "Resolve Later"
    ▼                      │
[Reviewing] ───────────────┘
    │
    │ User selects action
    ▼
[Resolved]
    │
    │ API applies resolution
    ▼
[Synced]
```

---

## TypeScript Type Generation

After creating Rust models, run:

```bash
cd crates/server
cargo run --bin generate_types
pnpm run generate-types:check  # CI validation
```

**Expected generated types** (`shared/types.ts`):

```typescript
export interface GitHubSyncConfig {
  id: string;
  project_id: string;
  repo_owner: string;
  repo_name: string;
  // encrypted_pat omitted (marked with #[serde(skip)])
  last_sync_at: Date | null;
  next_sync_at: Date | null;
  sync_enabled: boolean;
  sync_interval_seconds: number;
  sync_error: string | null;
  created_at: Date;
  updated_at: Date;
}

export interface TaskGitHubMetadata {
  id: string;
  task_id: string;
  github_issue_id: string;
  github_issue_number: number;
  github_issue_url: string;
  last_synced_at: Date;
  remote_updated_at: Date;
  last_synced_content: string;  // JSON string
  sync_conflict_status: string | null;
  created_at: Date;
  updated_at: Date;
}

export interface SyncHistory {
  id: string;
  project_id: string;
  sync_config_id: string;
  started_at: Date;
  completed_at: Date | null;
  duration_ms: number | null;
  issues_fetched: number;
  tasks_created: number;
  tasks_updated: number;
  conflicts_detected: number;
  status: string;
  error_message: string | null;
}
```

---

**Data Model Phase Complete** - Ready to generate API contracts and quickstart guide.

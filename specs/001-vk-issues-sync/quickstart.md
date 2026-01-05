# Quickstart Guide: GitHub Issues Sync

**Feature**: 001-vk-issues-sync  
**Last Updated**: 2026-01-05

## Prerequisites

Before starting development on this feature:

- [x] Rust 1.80+ installed (`rustc --version`)
- [x] Node.js 18+ and pnpm 8+ installed (`node --version`, `pnpm --version`)
- [x] GitHub account with a test repository containing Issues
- [x] GitHub Personal Access Token (PAT) with `repo` scope

---

## Development Environment Setup

### 1. Clone and Install Dependencies

```bash
cd /mnt/d/GitHub/vibe-kanban

# Install all workspace dependencies
pnpm install

# Verify Rust dependencies resolve
cargo check --workspace
```

### 2. Add New Dependencies

#### Backend (`crates/github-sync/Cargo.toml`)

```bash
# Create new crate
mkdir -p crates/github-sync
cd crates/github-sync

# Initialize Cargo.toml
cat > Cargo.toml << 'EOF'
[package]
name = "github-sync"
version = "0.1.0"
edition = "2021"

[dependencies]
# Existing workspace dependencies
tokio = { workspace = true }
axum = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
sqlx = { workspace = true, features = ["sqlite", "runtime-tokio", "uuid", "chrono"] }
chrono = { version = "0.4", features = ["serde"] }
uuid = { version = "1.0", features = ["v4", "serde"] }
ts-rs = "10.0"

# New dependencies for this feature
octocrab = "0.40"
tokio-cron-scheduler = "0.13"
ring = "0.17"
machine-uid = "0.5"
thiserror = "2.0"
tracing = { version = "0.1", features = ["log"] }

[dev-dependencies]
wiremock = "0.6"
tokio-test = "0.4"
EOF

# Add to workspace
cd ../..
# Edit Cargo.toml to add "crates/github-sync" to workspace members
```

#### Update Root `Cargo.toml`

```toml
[workspace]
members = [
    # ... existing members ...
    "crates/github-sync",  # ADD THIS LINE
]
```

### 3. Run Database Migrations

```bash
# Create migration files
sqlx migrate add github_sync_config
sqlx migrate add task_github_metadata
sqlx migrate add sync_history
sqlx migrate add conflict_resolution
sqlx migrate add add_github_issue_url_to_tasks

# Copy SQL from data-model.md into migration files
# Then run migrations
pnpm run prepare-db
```

### 4. Generate TypeScript Types

```bash
# After creating Rust models with #[derive(TS)]
cd crates/server
cargo run --bin generate_types

# Verify types generated
cat ../../shared/types.ts | grep -A 10 "GitHubSyncConfig"
```

### 5. Start Development Servers

```bash
# Terminal 1: Backend (auto-reload on changes)
pnpm run backend:dev:watch

# Terminal 2: Frontend (Vite HMR)
pnpm run frontend:dev

# Access app at http://localhost:3000
```

---

## Manual Testing Workflows

### Test 1: Initial Configuration (P0)

**Goal**: Set up GitHub sync for the first time

1. Open Vibe Kanban → Settings → GitHub Integration
2. Fill form:
   - Repository: `{your-username}/{test-repo}`
   - PAT: `ghp_xxxxxxxxxxxxxxxxxxxx` (with `repo` scope)
3. Click "Test Connection"
   - **Expected**: Green checkmark + "Repository accessible (42 open Issues)"
4. Click "Save Configuration"
   - **Expected**: Configuration saved, sync scheduled for next 5 minutes
5. Wait up to 5 minutes (or click "Sync Now")
6. Navigate to Tasks page
   - **Expected**: All 42 open Issues appear as Tasks with status "Todo"

**API Calls**:
```bash
# Test connection
curl -X POST http://localhost:3000/api/github-sync/validate-repo \
  -H "Content-Type: application/json" \
  -d '{
    "repo_owner": "your-username",
    "repo_name": "test-repo",
    "pat": "ghp_xxxxxxxxxxxxxxxxxxxx"
  }'

# Create configuration
curl -X POST http://localhost:3000/api/github-sync/config/{project-id} \
  -H "Content-Type: application/json" \
  -d '{
    "repo_owner": "your-username",
    "repo_name": "test-repo",
    "pat": "ghp_xxxxxxxxxxxxxxxxxxxx"
  }'

# Trigger manual sync
curl -X POST http://localhost:3000/api/github-sync/sync/{project-id}/trigger
```

---

### Test 2: Incremental Sync (P1)

**Goal**: Verify sync detects and applies Issue updates

**Setup**:
1. Complete Test 1 (initial import done)
2. On GitHub, edit Issue #1:
   - Change title: "Original Title" → "Updated Title"
   - Update description
3. Note the current time

**Test**:
1. In Vibe Kanban, click "Sync Now" (Settings → GitHub Integration)
2. Wait for sync to complete (~10 seconds)
3. Navigate to Task corresponding to Issue #1
   - **Expected**: Task title is "Updated Title"
   - **Expected**: Task description matches GitHub Issue
   - **Expected**: Task `updated_at` timestamp reflects sync time

**API Calls**:
```bash
# Check sync status
curl http://localhost:3000/api/github-sync/sync/{project-id}/status

# Expected response:
{
  "status": "idle",
  "last_sync_at": "2026-01-05T20:15:00Z",
  "next_sync_at": "2026-01-05T20:20:00Z",
  "current_sync": null,
  "last_error": null
}

# View sync history
curl http://localhost:3000/api/github-sync/sync/{project-id}/history

# Expected: Last entry shows tasks_updated = 1, conflicts_detected = 0
```

---

### Test 3: State Mapping (P2)

**Goal**: Verify Issue state changes sync to Task status

**Setup**:
1. Complete Test 1 (initial import done)
2. On GitHub, close Issue #2
3. Note the issue number

**Test**:
1. In Vibe Kanban, click "Sync Now"
2. Wait for sync to complete
3. Navigate to Task corresponding to Issue #2
   - **Expected**: Task status changed from "Todo" → "Done"
   - **Expected**: Task `github_issue_url` link still works (opens GitHub)

**Reverse Test** (Issue reopened):
1. On GitHub, reopen Issue #2
2. In Vibe Kanban, click "Sync Now"
3. Navigate to same Task
   - **Expected**: Task status changed back to "Todo"

---

### Test 4: Conflict Resolution (P3)

**Goal**: Verify conflict detection and resolution UI

**Setup**:
1. Complete Test 1 (initial import done)
2. In Vibe Kanban, edit Task corresponding to Issue #3:
   - Change description to "Local changes made"
   - **Do NOT click Save yet**
3. On GitHub, edit Issue #3:
   - Change description to "Remote changes made"
4. In Vibe Kanban, now click Save on Task
5. Trigger sync (click "Sync Now")

**Test**:
1. After sync completes, navigate to Tasks page
   - **Expected**: Badge appears on Task #3: "⚠️ Sync Conflict"
2. Click on Task #3
   - **Expected**: Conflict resolution modal opens
   - **Expected**: Three columns displayed:
     - **Local**: "Local changes made"
     - **Base** (last synced): [original description]
     - **Remote**: "Remote changes made"
3. Select resolution option: "Accept GitHub"
4. Click "Resolve"
   - **Expected**: Modal closes
   - **Expected**: Task description now shows "Remote changes made"
   - **Expected**: Conflict badge disappears

**API Calls**:
```bash
# Get pending conflicts
curl http://localhost:3000/api/github-sync/conflicts/{project-id}

# Expected response:
{
  "conflicts": [
    {
      "conflict_id": "uuid",
      "task_id": "uuid",
      "detected_at": "2026-01-05T20:30:00Z",
      "local_content": {"title": "...", "description": "Local changes made"},
      "remote_content": {"title": "...", "description": "Remote changes made"},
      "base_content": {"title": "...", "description": "Original"}
    }
  ],
  "total_pending": 1
}

# Resolve conflict
curl -X POST http://localhost:3000/api/github-sync/conflicts/{conflict-id}/resolve \
  -H "Content-Type: application/json" \
  -d '{
    "resolution_action": "accept_github"
  }'
```

---

### Test 5: Error Handling (Rate Limiting)

**Goal**: Verify graceful handling of GitHub API rate limits

**Setup** (requires script to exhaust rate limit):
```bash
# Exhaust rate limit (run this in a separate terminal)
for i in {1..5000}; do
  curl -H "Authorization: token ghp_xxxxxxxxxxxxxxxxxxxx" \
    https://api.github.com/users/octocat
done
```

**Test**:
1. Trigger sync while rate limited
   - **Expected**: Sync fails with error message
   - **Expected**: UI displays: "GitHub API rate limit exceeded. Next sync at [timestamp]"
   - **Expected**: Scheduler pauses until rate limit resets
2. Check sync status
   - **Expected**: `sync_error` field shows rate limit error
   - **Expected**: `next_sync_at` is set to rate limit reset time (from `X-RateLimit-Reset` header)

---

## Debugging Tips

### View Encrypted PAT (for verification)

```rust
// In crates/github-sync/src/crypto.rs
#[test]
fn test_encrypt_decrypt_roundtrip() {
    let original_pat = "ghp_test1234567890";
    let encrypted = encrypt_pat(original_pat).unwrap();
    let decrypted = decrypt_pat(&encrypted).unwrap();
    assert_eq!(original_pat, decrypted);
}
```

### View Sync Logs

```bash
# Backend logs (structured tracing)
tail -f logs/backend.log | grep github_sync

# Expected entries:
# 2026-01-05T20:00:00Z INFO github_sync::scheduler: Sync started project_id=uuid
# 2026-01-05T20:00:05Z INFO github_sync::sync: Fetched 42 Issues
# 2026-01-05T20:00:10Z INFO github_sync::sync: Created 42 Tasks, Updated 0, Conflicts 0
# 2026-01-05T20:00:10Z INFO github_sync::scheduler: Sync completed duration_ms=10234
```

### Inspect Database State

```bash
# View sync config
sqlite3 dev_assets/vibe-kanban.db \
  "SELECT * FROM github_sync_config WHERE project_id = 'uuid';"

# View metadata for a Task
sqlite3 dev_assets/vibe-kanban.db \
  "SELECT * FROM task_github_metadata WHERE task_id = 'uuid';"

# View sync history
sqlite3 dev_assets/vibe-kanban.db \
  "SELECT started_at, status, issues_fetched, tasks_created, conflicts_detected 
   FROM sync_history 
   ORDER BY started_at DESC 
   LIMIT 10;"
```

### Mock GitHub API (for unit tests)

```rust
// In crates/github-sync/tests/mock_github.rs
use wiremock::{MockServer, Mock, ResponseTemplate};
use wiremock::matchers::{method, path};

#[tokio::test]
async fn test_sync_with_mocked_api() {
    let mock_server = MockServer::start().await;
    
    // Mock GraphQL endpoint
    Mock::given(method("POST"))
        .and(path("/graphql"))
        .respond_with(ResponseTemplate::new(200).set_body_json(json!({
            "data": {
                "repository": {
                    "issues": {
                        "nodes": [
                            {
                                "id": "I_kwDO...",
                                "number": 1,
                                "title": "Test Issue",
                                "body": "Test body",
                                "state": "OPEN",
                                "updatedAt": "2026-01-05T20:00:00Z"
                            }
                        ],
                        "pageInfo": {
                            "hasNextPage": false,
                            "endCursor": null
                        }
                    }
                }
            }
        })))
        .mount(&mock_server)
        .await;
    
    // Run sync with mock server URL
    // ... test assertions ...
}
```

---

## Performance Benchmarks

### Expected Metrics (Local Development)

| Scenario | Issues Count | Expected Time | Metric |
|----------|--------------|---------------|--------|
| **Initial Import** | 100 | <60s | SC-001 |
| **Initial Import** | 500 | <180s | (extrapolated) |
| **Incremental Sync** | 10 changed | <10s | SC-002 |
| **Conflict Detection** | 1 conflict | <5s | SC-005 |
| **Conflict Resolution UI** | User resolves | <2 min | SC-006 |

### Measure Sync Duration

```bash
# Add timing logs
time curl -X POST http://localhost:3000/api/github-sync/sync/{project-id}/trigger

# Or check duration_ms in sync_history table
sqlite3 dev_assets/vibe-kanban.db \
  "SELECT AVG(duration_ms) as avg_duration_ms, MIN(duration_ms), MAX(duration_ms)
   FROM sync_history 
   WHERE status = 'success';"
```

---

## Troubleshooting

### Issue: "PAT decryption failed"

**Cause**: Machine ID changed (e.g., moved DB to different machine)

**Fix**: Delete and recreate configuration with new PAT

```bash
curl -X DELETE http://localhost:3000/api/github-sync/config/{project-id}
# Then re-create via UI
```

---

### Issue: "Sync stuck in 'in_progress'"

**Cause**: Backend crashed during sync, didn't mark sync complete

**Fix**: Manually update database

```sql
UPDATE sync_history 
SET status = 'failure', 
    error_message = 'Sync interrupted',
    completed_at = datetime('now')
WHERE status = 'in_progress';
```

---

### Issue: "Rate limit not resetting"

**Cause**: `X-RateLimit-Reset` header parsing failed

**Fix**: Check logs for rate limit errors, manually set `next_sync_at`:

```sql
UPDATE github_sync_config 
SET next_sync_at = datetime('now', '+1 hour'),
    sync_error = NULL
WHERE project_id = 'uuid';
```

---

## Next Steps

After completing manual testing:

1. **Phase 2**: Generate task breakdown with `/speckit.tasks`
2. **Implementation**: Follow TDD workflow from `conductor/workflow.md`
3. **CI Integration**: Add E2E tests to GitHub Actions workflow

---

**Quickstart Complete** - Ready to begin implementation with clear testing strategy.

# Research: GitHub Issues Sync

**Feature**: 001-vk-issues-sync  
**Date**: 2026-01-05  
**Status**: Complete

## 1. Octocrab GitHub API Integration

### Decision: Use GraphQL API with `octocrab::graphql`

**Rationale**:
- **Cursor-based pagination**: GraphQL `issues(first: 100, after: cursor)` handles large repos efficiently
- **Field selection**: Fetch only needed fields (id, number, title, body, state, updatedAt) → reduces bandwidth
- **Single request**: Can fetch Issues + labels + assignees in one query vs multiple REST calls
- **Rate limit efficiency**: 1 GraphQL query = 1 point, vs REST pagination = N points

**Implementation Pattern**:
```rust
use octocrab::Octocrab;

let query = r#"
  query($owner: String!, $repo: String!, $after: String) {
    repository(owner: $owner, name: $repo) {
      issues(first: 100, after: $after, states: OPEN) {
        pageInfo {
          hasNextPage
          endCursor
        }
        nodes {
          id
          number
          title
          body
          state
          updatedAt
          labels(first: 10) { nodes { name } }
        }
      }
    }
  }
"#;

let octocrab = Octocrab::builder()
    .personal_token(pat)
    .build()?;

let response: GraphQLResponse = octocrab
    .graphql(query)
    .build()?
    .send()
    .await?;
```

**Alternatives Considered**:
- **REST API** (`/repos/{owner}/{repo}/issues`): Simpler but requires pagination with Link headers, multiple requests for labels/assignees, less efficient for large repos
- **GitHub CLI (`gh api`)**: Would require shelling out, harder to test, not Rust-native

**Mocking Strategy**:
- Use `wiremock` crate to mock GitHub GraphQL endpoint
- Fixture files in `tests/fixtures/github_responses/*.json`
- Example: `tests/fixtures/github_responses/issues_page1.json`

---

## 2. PAT Encryption Strategy

### Decision: AES-256-GCM with machine-specific salt (hardware UUID)

**Rationale**:
- **AES-256-GCM**: Authenticated encryption (prevents tampering), `ring` crate provides battle-tested implementation
- **Machine-specific salt**: Derive encryption key from hardware UUID (via `machine-uid` crate) + constant app secret → PAT only decryptable on same machine
- **SQLite BLOB storage**: Store encrypted PAT as BLOB in `github_sync_config` table, never in plaintext

**Implementation Pattern**:
```rust
use ring::aead::{Aad, LessSafeKey, Nonce, UnboundKey, AES_256_GCM};
use ring::rand::{SecureRandom, SystemRandom};
use machine_uid;

const APP_SECRET: &[u8] = b"vibe-kanban-github-sync-v1"; // Compile-time constant

fn derive_key() -> Result<LessSafeKey> {
    let machine_id = machine_uid::get()?; // Hardware UUID
    let mut key_material = [0u8; 32];
    
    // HKDF to derive 256-bit key from machine_id + APP_SECRET
    let mut ctx = ring::hkdf::Hkdf::<ring::hmac::HMAC_SHA256>::new(
        Some(APP_SECRET),
        machine_id.as_bytes(),
    );
    ctx.expand(&[], &mut key_material)?;
    
    let unbound_key = UnboundKey::new(&AES_256_GCM, &key_material)?;
    Ok(LessSafeKey::new(unbound_key))
}

fn encrypt_pat(pat: &str) -> Result<Vec<u8>> {
    let key = derive_key()?;
    let rng = SystemRandom::new();
    
    let mut nonce_bytes = [0u8; 12];
    rng.fill(&mut nonce_bytes)?;
    let nonce = Nonce::assume_unique_for_key(nonce_bytes);
    
    let mut in_out = pat.as_bytes().to_vec();
    key.seal_in_place_append_tag(nonce, Aad::empty(), &mut in_out)?;
    
    // Prepend nonce to ciphertext (needed for decryption)
    let mut result = nonce_bytes.to_vec();
    result.extend_from_slice(&in_out);
    Ok(result)
}

fn decrypt_pat(encrypted: &[u8]) -> Result<String> {
    let key = derive_key()?;
    
    let (nonce_bytes, ciphertext) = encrypted.split_at(12);
    let nonce = Nonce::assume_unique_for_key(nonce_bytes.try_into()?);
    
    let mut in_out = ciphertext.to_vec();
    let plaintext = key.open_in_place(nonce, Aad::empty(), &mut in_out)?;
    
    Ok(String::from_utf8(plaintext.to_vec())?)
}
```

**Alternatives Considered**:
- **OS Keyring** (via `keyring` crate): More secure (OS-managed encryption) but complicates Docker deployments and cross-platform testing. Rejected because Nuno wants "simplest to implement" (user requirement).
- **Plaintext with file permissions** (`chmod 600`): Insufficient security, rejected.
- **Password-based encryption**: Requires user to enter password on every sync, breaks unattended operation. Rejected.

**Key Rotation**: Not implemented in MVP. PAT rotation handled manually by user (update in Settings UI triggers re-encryption with same key).

---

## 3. Tokio Cron Scheduler

### Decision: `tokio-cron-scheduler = "0.13"` with dynamic interval

**Rationale**:
- **Cron-like syntax**: Easy to configure intervals (`*/5 * * * *` = every 5 minutes)
- **Tokio-native**: Integrates seamlessly with Axum server runtime
- **Dynamic interval**: Can change schedule at runtime (5min idle → 30s active when changes detected)

**Implementation Pattern**:
```rust
use tokio_cron_scheduler::{JobScheduler, Job};
use std::sync::Arc;
use tokio::sync::RwLock;

pub struct SyncScheduler {
    scheduler: JobScheduler,
    current_interval: Arc<RwLock<SyncInterval>>,
}

#[derive(Clone, Copy)]
enum SyncInterval {
    Idle,   // 5 minutes
    Active, // 30 seconds
}

impl SyncScheduler {
    pub async fn start(pool: SqlitePool) -> Result<Self> {
        let scheduler = JobScheduler::new().await?;
        let interval = Arc::new(RwLock::new(SyncInterval::Idle));
        
        let interval_clone = interval.clone();
        let pool_clone = pool.clone();
        
        let job = Job::new_async("*/5 * * * *", move |_uuid, _lock| {
            let pool = pool_clone.clone();
            let interval = interval_clone.clone();
            Box::pin(async move {
                // Check current interval
                let current = *interval.read().await;
                
                // Run sync
                match run_sync(&pool).await {
                    Ok(changes_detected) => {
                        if changes_detected {
                            *interval.write().await = SyncInterval::Active;
                            // TODO: Reschedule to 30s (requires job replacement)
                        }
                    }
                    Err(e) => tracing::error!("Sync failed: {}", e),
                }
            })
        })?;
        
        scheduler.add(job).await?;
        scheduler.start().await?;
        
        Ok(Self { scheduler, current_interval: interval })
    }
    
    pub async fn manual_sync(&self, pool: &SqlitePool) -> Result<()> {
        run_sync(pool).await?;
        Ok(())
    }
}
```

**Alternatives Considered**:
- **`tokio::time::interval`**: Simpler but less flexible (can't easily change interval dynamically). Rejected because requirement for 5min idle → 30s active transition.
- **Custom scheduler**: Too much complexity for MVP. Rejected.

**Graceful Shutdown**:
```rust
impl Drop for SyncScheduler {
    fn drop(&mut self) {
        // Scheduler auto-stops on drop (tokio-cron-scheduler handles cleanup)
    }
}
```

**Single-Instance Enforcement**:
- Use SQLite `SELECT ... FOR UPDATE` lock on `github_sync_config` row during sync
- If lock fails (another sync running), skip this cycle

---

## 4. Conflict Detection Logic

### Decision: Timestamp-based three-way diff with base snapshot

**Rationale**:
- **Three timestamps**: `task.updated_at`, `metadata.last_synced_at`, `github_issue.updated_at`
- **Conflict condition**: `task.updated_at > last_synced_at` AND `github_issue.updated_at > last_synced_at`
- **Base snapshot**: Store `last_synced_content` (title + description) in `task_github_metadata` for diff comparison

**Detection Algorithm**:
```rust
#[derive(Debug)]
pub enum SyncDecision {
    NoChange,           // All timestamps equal
    SafeUpdate,         // Only GitHub changed
    LocalOnlyChange,    // Only Task changed (warn user but don't block)
    Conflict {          // Both changed
        local: String,
        remote: String,
        base: String,
    },
}

pub fn detect_conflict(
    task: &Task,
    metadata: &TaskGitHubMetadata,
    github_issue: &GitHubIssue,
) -> SyncDecision {
    let task_changed = task.updated_at > metadata.last_synced_at;
    let github_changed = github_issue.updated_at > metadata.last_synced_at;
    
    match (task_changed, github_changed) {
        (false, false) => SyncDecision::NoChange,
        (false, true) => SyncDecision::SafeUpdate,
        (true, false) => {
            // Local change only - log warning but allow (GitHub is source of truth)
            tracing::warn!("Task {} modified locally but not synced back (unidirectional)", task.id);
            SyncDecision::LocalOnlyChange
        }
        (true, true) => {
            // Conflict: both modified since last sync
            SyncDecision::Conflict {
                local: format!("{}\n\n{}", task.title, task.description.as_deref().unwrap_or("")),
                remote: format!("{}\n\n{}", github_issue.title, github_issue.body.as_deref().unwrap_or("")),
                base: metadata.last_synced_content.clone(),
            }
        }
    }
}
```

**Three-Way Diff UI**:
- Use `similar` crate (Rust port of Python's difflib) to generate line-by-line diff
- Frontend displays three columns: Local | Base | Remote
- Color coding: green (additions), red (deletions), yellow (modifications)

**Alternatives Considered**:
- **Content-hash comparison**: More accurate but requires storing hashes of every field. Rejected for MVP (overkill).
- **Last-write-wins**: Simpler but risks data loss. Rejected per user requirement ("prompt user").

---

## 5. Frontend State Management

### Decision: TanStack Query + Zustand hybrid

**Rationale**:
- **TanStack Query**: Handles server state (sync status polling, manual sync mutations)
- **Zustand**: Handles client state (conflict queue, resolution choices, UI modals)
- **Separation of concerns**: Server state (API) vs UI state (modals, selections)

**TanStack Query Pattern** (Sync Status Polling):
```tsx
import { useQuery, useMutation } from '@tanstack/react-query';

export function useGitHubSyncStatus(projectId: string) {
  return useQuery({
    queryKey: ['github-sync-status', projectId],
    queryFn: () => fetch(`/api/github-sync/status/${projectId}`).then(r => r.json()),
    refetchInterval: 5000, // Poll every 5s
    staleTime: 3000,
  });
}

export function useManualSync(projectId: string) {
  return useMutation({
    mutationFn: () => 
      fetch(`/api/github-sync/trigger/${projectId}`, { method: 'POST' }),
    onSuccess: () => {
      // Invalidate status query to trigger re-fetch
      queryClient.invalidateQueries(['github-sync-status', projectId]);
    },
  });
}
```

**Zustand Pattern** (Conflict Queue):
```tsx
import create from 'zustand';

interface ConflictState {
  conflicts: Conflict[];
  currentConflict: Conflict | null;
  resolution: ResolutionChoice | null;
  
  addConflict: (conflict: Conflict) => void;
  nextConflict: () => void;
  resolveConflict: (choice: ResolutionChoice) => void;
  clearQueue: () => void;
}

export const useConflictStore = create<ConflictState>((set) => ({
  conflicts: [],
  currentConflict: null,
  resolution: null,
  
  addConflict: (conflict) => 
    set((state) => ({ conflicts: [...state.conflicts, conflict] })),
  
  nextConflict: () => 
    set((state) => {
      const [next, ...rest] = state.conflicts;
      return { currentConflict: next || null, conflicts: rest };
    }),
  
  resolveConflict: (choice) => 
    set({ resolution: choice }),
  
  clearQueue: () => 
    set({ conflicts: [], currentConflict: null, resolution: null }),
}));
```

**Optimistic UI Updates**:
- When user clicks "Sync Now", immediately show spinner + disable button
- On API success, update status to "Syncing..." and start polling
- On API failure, revert UI state and show error toast

**Alternatives Considered**:
- **Redux**: Too verbose for this use case. Rejected.
- **React Context**: Doesn't handle async state well. Rejected.
- **Zustand-only**: Can't handle server state polling elegantly. Rejected.

---

## Technology Selection Summary

| Decision | Technology | Rationale |
|----------|-----------|-----------|
| **GitHub API** | `octocrab` + GraphQL | Cursor pagination, field selection, rate limit efficiency |
| **Encryption** | `ring` AES-256-GCM + `machine-uid` | Authenticated encryption, machine-bound keys, simplest secure solution |
| **Scheduler** | `tokio-cron-scheduler` | Dynamic intervals, Tokio-native, graceful shutdown |
| **Conflict Detection** | Timestamp-based three-way diff | Minimal storage overhead, clear conflict conditions |
| **Frontend State** | TanStack Query + Zustand | Server state (polling) + client state (modals) separation |

---

## Open Questions Resolved

1. **Q: How to handle GitHub API rate limiting?**  
   **A**: Detect HTTP 429, extract `X-RateLimit-Reset` header, pause scheduler until reset time. Log warning in UI: "Rate limit reached. Next sync at [timestamp]".

2. **Q: What if PAT expires during sync?**  
   **A**: Detect HTTP 401, disable auto-sync, store error state in `github_sync_config.sync_error`. Frontend displays modal: "GitHub authentication failed. Please update token in Settings."

3. **Q: How to test encryption without exposing PAT in tests?**  
   **A**: Generate random PAT strings in tests (`"ghp_test_1234..."`), encrypt/decrypt round-trip, assert plaintext matches. Never commit real PATs.

4. **Q: Should sync run on application startup?**  
   **A**: Yes, trigger immediate sync on startup if `last_sync_at` > 5 minutes ago. Provides fresh data when user opens app.

5. **Q: How to prevent sync from blocking Task CRUD operations?**  
   **A**: Use SQLite `DEFERRED` transactions (read-first, upgrade to write if needed). Sync reads `tasks` table, writes only to `task_github_metadata`. Minimal lock contention.

---

**Research Phase Complete** - Ready to proceed to Phase 1 (Data Model Design).

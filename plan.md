# Plan: Archive & Restore Git Worktrees

## Overview

When a user archives the last thread on a linked git worktree, we make a WIP
commit, persist the worktree metadata to sqlite, anchor the commit with a custom
git ref, and delete the worktree. When any thread referencing that worktree is
later unarchived, we restore the worktree (creating it from the WIP commit),
restore the branch, and `git reset HEAD~` to put things back how they were.

## Stacked branches

| Branch | Based on | Scope |
|---|---|---|
| `AI-112/delete-worktree-on-archive` | `main` | Already merged — deletes worktree on last archive |
| `AI-112/archive-worktree-wip` | above | Archive side: WIP commit, sqlite, git ref |
| `AI-112/restore-worktree` | above | Unarchive side: full restore logic |

---

## Branch 2: `AI-112/archive-worktree-wip`

### 2.1 Git crate changes (`crates/git/src/repository.rs`)

**Add `--allow-empty` to commit:**
- Add `allow_empty: bool` field to `CommitOptions`.
- In `RealGitRepository::commit`, when `options.allow_empty` is true, append
  `--allow-empty` to the git command args.
- In `FakeGitRepository::commit` (if it exists), handle the new field.

**Add `update_ref` method to `GitRepository` trait:**
```
fn update_ref(&self, ref_name: String, commit: String) -> BoxFuture<'_, Result<()>>;
```
Implementation: `git update-ref <ref_name> <commit>`

**Add `delete_ref` method to `GitRepository` trait:**
```
fn delete_ref(&self, ref_name: String) -> BoxFuture<'_, Result<()>>;
```
Implementation: `git update-ref -d <ref_name>`

**Add `stage_all_including_untracked` method to `GitRepository` trait:**
```
fn stage_all_including_untracked(&self) -> BoxFuture<'_, Result<()>>;
```
Implementation: `git add -A`

This is different from the existing `stage_paths` (which uses `update-index`
for specific paths) and `stage_all` on Repository (which filters by current
status). We need `git add -A` to capture untracked files.

### 2.2 Project git_store changes (`crates/project/src/git_store.rs`)

Add high-level `Repository` methods that wrap the new low-level trait methods:

- `pub fn update_ref(&mut self, ref_name: String, commit: String) -> oneshot::Receiver<Result<()>>`
- `pub fn delete_ref(&mut self, ref_name: String) -> oneshot::Receiver<Result<()>>`
- `pub fn stage_all_including_untracked(&mut self) -> oneshot::Receiver<Result<()>>`

Also update `Repository::commit` to pass through `allow_empty` from
`CommitOptions`.

### 2.3 Sqlite table (`crates/agent_ui/src/thread_metadata_store.rs`)

**New migration** added to `ThreadMetadataDb::MIGRATIONS`:

```sql
CREATE TABLE IF NOT EXISTS archived_git_worktrees(
    id INTEGER PRIMARY KEY,
    worktree_path TEXT NOT NULL,
    main_repo_path TEXT NOT NULL,
    branch_name TEXT NOT NULL,
    commit_hash TEXT,
    restored INTEGER NOT NULL DEFAULT 0
) STRICT;
```

`commit_hash` is NULL until the last thread triggers the actual WIP commit +
worktree deletion.

**Join table** to link archived threads to their worktree:

```sql
CREATE TABLE IF NOT EXISTS archived_thread_worktrees(
    session_id TEXT NOT NULL,
    archived_worktree_id INTEGER NOT NULL REFERENCES archived_git_worktrees(id),
    PRIMARY KEY (session_id, archived_worktree_id)
) STRICT;
```

This replaces the earlier idea of adding a column to `sidebar_threads`
(which gets deleted on archive). The join table persists across the
thread's archive/unarchive lifecycle. On unarchive, delete the join
table row. On archive, insert a row.

**New struct:**

```rust
pub struct ArchivedGitWorktree {
    pub id: i64,
    pub worktree_path: PathBuf,
    pub branch_name: String,
    pub commit_hash: Option<String>,
    pub restored: bool,
}
```

**New db methods on `ThreadMetadataDb`:**

- `pub async fn find_or_create_archived_worktree(worktree_path: &str, main_repo_path: &str, branch_name: &str) -> Result<i64>`
  — Returns existing row ID if one exists for this path with `commit_hash = NULL`
    and `restored = false`, otherwise inserts a new row.
- `pub async fn update_archived_worktree_commit(id: i64, commit_hash: &str) -> Result<()>`
  — Sets `commit_hash` on the row.
- `pub async fn get_archived_worktree(id: i64) -> Result<Option<ArchivedGitWorktree>>`
- `pub async fn get_archived_worktree_by_path(worktree_path: &str) -> Result<Option<ArchivedGitWorktree>>`
  — `SELECT ... WHERE worktree_path = ? ORDER BY id DESC LIMIT 1`
- `pub async fn update_archived_worktree_restored(id: i64, worktree_path: &str, branch_name: &str) -> Result<()>`
  — Sets `restored = true` and updates path/branch (for collision fallback).
- `pub async fn delete_archived_worktree(id: i64) -> Result<()>`
- `pub async fn link_thread_to_archived_worktree(session_id: &str, archived_worktree_id: i64) -> Result<()>`
  — Inserts into `archived_thread_worktrees`.
- `pub async fn unlink_thread_from_archived_worktree(session_id: &str) -> Result<()>`
  — Deletes from `archived_thread_worktrees` by session_id.
- `pub async fn get_archived_worktree_for_thread(session_id: &str) -> Result<Option<ArchivedGitWorktree>>`
  — Joins `archived_thread_worktrees` with `archived_git_worktrees`.
- `pub async fn count_threads_for_archived_worktree(id: i64) -> Result<usize>`
  — `SELECT COUNT(*) FROM archived_thread_worktrees WHERE archived_worktree_id = ?`.

### 2.4 Store-level methods (`SidebarThreadMetadataStore`)

Expose the db methods through the store entity so sidebar code can use them:

- `pub fn find_or_create_archived_worktree(&self, ...) -> Task<Result<i64>>`
- `pub fn update_archived_worktree_commit(&self, ...) -> Task<Result<()>>`
- etc.

These spawn background tasks that call the db methods and don't go through
the batched channel (they need immediate results via Task, not eventual
consistency).

### 2.5 Sidebar archive flow changes (`crates/sidebar/src/sidebar.rs`)

**Every thread archived on a linked worktree** (in `archive_thread`):

1. Determine the thread's `folder_paths` (existing logic).
2. If `folder_paths.len() == 1`, resolve via `.git` file to check if it's a
   linked worktree (`resolve_git_worktree_to_main_repo`). This is now async,
   so needs restructuring — but we can check synchronously by looking at the
   thread entry's workspace repo snapshots: if the thread workspace's repo
   `is_linked_worktree()`, we know it's a linked worktree.
   
   **Simpler approach**: check if any repo in any workspace has
   `work_directory_abs_path == worktree_path` and `is_linked_worktree()`.
   If so, it's a linked worktree thread. Get the branch name from the
   repo's `snapshot().branch`.

3. Call `find_or_create_archived_worktree(worktree_path, main_repo_path, branch_name)`
   to get the row ID.
4. Call `link_thread_to_archived_worktree(session_id, row_id)` to insert
   into the join table. This persists the linkage even after the thread's
   `sidebar_threads` row is deleted by `store.delete`.

**Last thread archived** (in `maybe_delete_git_worktree_for_archived_thread`):

After confirming this is the last thread (existing check), before moving
to tempdir:

1. Find the linked worktree's repo entity (existing logic finds the main repo;
   we also need the linked worktree's own repo for staging + committing).
   The linked worktree's repo can be found by matching
   `work_directory_abs_path == worktree_path` across workspace repos.

2. **Stage all**: Call `repo.stage_all_including_untracked()` on the linked
   worktree's repo. Await the result.

3. **WIP commit**: Call `repo.commit("WIP".into(), None, CommitOptions { allow_empty: true, ..Default::default() }, ...)`.
   - If commit fails → show dismissable toast: "Could not archive worktree
     because committing failed. Delete the directory manually if needed."
   - Return early (do NOT delete the worktree).

4. **Get commit hash**: Call `repo.head_sha()` or use `revparse_batch(["HEAD"])`
   to get the WIP commit's hash. (head_sha is on the low-level GitRepository
   trait; we need a high-level method or use the existing `branches()` result.)

   **Note**: We don't have a `head_sha` method on the high-level `Repository`.
   Add one: `pub fn head_sha(&mut self) -> oneshot::Receiver<Result<Option<String>>>`.

5. **Update sqlite row**: Call `update_archived_worktree_commit(row_id, commit_hash)`.

6. **Create git ref**: Call `main_repo.update_ref(format!("refs/archived-worktrees/{row_id}"), commit_hash)` on the MAIN repo (the ref
   must be on the main repo since the worktree's repo is about to be deleted).

7. **Delete worktree**: Existing logic (move to tempdir, `remove_worktree`,
   background delete).

### 2.6 Error handling: WIP commit failure

If `stage_all_including_untracked` or `commit` fails:

- Log the error.
- Show a dismissable notification/toast to the user.
- Do NOT delete the worktree.
- The thread is still archived (removed from sidebar), but the worktree
  stays on disk. The sqlite row exists with `commit_hash = NULL`.
- On unarchive, the thread will find the row with NULL commit_hash. If the
  worktree still exists on disk, it just opens it (the thread's `work_dirs`
  point to it). If the worktree was manually deleted, the user gets a fresh
  worktree + warning.

### 2.7 Edge case: worktree already deleted before last archive

In `maybe_delete_git_worktree_for_archived_thread`, if
`resolve_git_worktree_to_main_repo` returns `None` (worktree dir doesn't
exist or isn't a git worktree), skip WIP commit and ref creation. The sqlite
row stays with `commit_hash = NULL`. On unarchive this gives the user a fresh
worktree + warning.

---

## Branch 3: `AI-112/restore-worktree`

### 3.1 Modify `activate_archived_thread` (`crates/sidebar/src/sidebar.rs`)

Currently this method:
1. Saves thread metadata to store (making it visible in sidebar)
2. Finds or opens a workspace matching `work_dirs`
3. Loads the thread in that workspace's agent panel

We add a new step between 1 and 2: **restore the git worktree if needed**.

```
fn activate_archived_thread(...) {
    // Save metadata (existing)
    store.save(metadata, cx);

    if let Some(path_list) = &session_info.work_dirs {
        if path_list.paths().len() == 1 {
            let worktree_path = path_list.paths()[0].clone();
            // Check if this is a linked worktree that needs restoration
            self.maybe_restore_git_worktree(worktree_path, session_info, agent, window, cx);
            return;
        }
    }

    // Fall through to existing logic for non-worktree threads
    ...
}
```

### 3.2 New method: `maybe_restore_git_worktree`

This is the core restore logic. It's async (needs db queries, git ops), so
it spawns a task.

```
fn maybe_restore_git_worktree(
    &mut self,
    worktree_path: PathBuf,
    session_id: acp::SessionId,
    session_info: AgentSessionInfo,
    agent: Agent,
    window: &mut Window,
    cx: &mut Context<Self>,
) {
    // 1. Query db for this thread's archived worktree via the join table
    //    SELECT w.* FROM archived_git_worktrees w
    //    JOIN archived_thread_worktrees tw ON tw.archived_worktree_id = w.id
    //    WHERE tw.session_id = ?
    
    // 2. Dispatch to restore logic (async)
    cx.spawn_in(window, async move |this, cx| {
        let row = db.get_archived_worktree_for_thread(&session_id).await?;
        
        // Delete the join table row now that we're unarchiving
        db.unlink_thread_from_archived_worktree(&session_id).await?;
        
        match row {
            None => {
                // No archived worktree info — just open normally
                // (worktree might still exist, or this isn't a worktree thread)
                this.update_in(cx, |this, window, cx| {
                    this.activate_archived_thread_in_workspace(
                        agent, session_info, window, cx
                    );
                })?;
            }
            Some(row) if row.commit_hash.is_none() => {
                // Worktree was archived but WIP commit failed or worktree
                // was never deleted. Try to open normally.
                if worktree_exists_on_disk(&worktree_path) {
                    this.update_in(cx, |this, window, cx| {
                        this.activate_archived_thread_in_workspace(
                            agent, session_info, window, cx
                        );
                    })?;
                } else {
                    // Worktree gone, no WIP commit — create fresh + warn
                    create_fresh_worktree_and_warn(...).await?;
                }
            }
            Some(row) => {
                // Full restore
                restore_archived_worktree(row, ...).await?;
            }
        }
        
        // Cleanup: if no more threads reference this row, delete it + ref
        if let Some(row) = &row {
            let remaining = db.count_threads_for_archived_worktree(row.id).await?;
            if remaining == 0 {
                main_repo.delete_ref(
                    format!("refs/archived-worktrees/{}", row.id)
                ).await?;
                db.delete_archived_worktree(row.id).await?;
            }
        }
    }).detach_and_log_err(cx);
}
```

### 3.3 Full restore logic (`restore_archived_worktree`)

This is the async function that handles all the edge cases.

**Step 1: Resolve worktree name**

```
let existing_worktrees = main_repo.worktrees().await?;
let worktree_exists = existing_worktrees.iter()
    .any(|wt| wt.path == row.worktree_path);

let final_worktree_path = if !worktree_exists {
    // Original name is free — use it
    row.worktree_path.clone()
} else if row.restored {
    // Another thread already restored this — reuse it
    // Skip worktree creation, jump to "associate thread"
    row.worktree_path.clone()  // (set a flag to skip creation)
} else {
    // Name collision with unrelated worktree — generate new name
    let new_name = generate_branch_name(&existing_branches, &mut rng)?;
    let new_path = main_repo.path_for_new_linked_worktree(&new_name, &setting)?;
    db.update_archived_worktree_restored(row.id, &new_path, &new_name).await?;
    new_path
};
```

**Step 2: Create worktree (if not already restored)**

```
if !already_restored {
    // git worktree add <path> <wip_commit_hash>
    main_repo.create_worktree(
        "detached".into(),  // branch name doesn't matter, detached HEAD
        final_worktree_path.clone(),
        Some(row.commit_hash.clone()),
    ).await?;
}
```

Wait — `create_worktree` takes a branch name and creates the worktree on that
branch. We want detached HEAD at the WIP commit. Looking at the low-level impl:

```rust
fn create_worktree(&self, branch_name: String, path: PathBuf, from_commit: Option<String>)
```

The `RealGitRepository::create_worktree` builds args as:
`git worktree add -b <branch_name> <path> [<commit>]`

This creates a NEW branch. We don't want that — we want detached HEAD.
We need to add support for `--detach` or pass the commit directly without `-b`.

**New: Add `detached` parameter to `create_worktree`** or add a separate
`create_worktree_detached(path, commit)` method.

Implementation: `git worktree add --detach <path> <commit>`

**Step 3: Reset WIP commit**

In the worktree's repo (which we'll find after creation):
```
worktree_repo.reset("HEAD~".into(), ResetMode::Mixed).await?;
```

This undoes the WIP commit, moving its changes to unstaged.

**Step 4: Resolve branch**

```
let branches = main_repo.branches().await?;
let wip_parent = row.commit_hash_parent;  // We need to store this or compute it

// Actually, we can compute the parent: it's what HEAD points to after
// git reset HEAD~. Or we can use revparse_batch(["HEAD~"]) before the
// reset. But we need to store or compute the WIP commit's parent hash.
//
// Simplest: after the reset, HEAD is at the parent. We can get its hash.

let current_head = worktree_repo.head_sha().await?;

let original_branch = &row.branch_name;
let existing_branch = branches.iter().find(|b| b.name == original_branch);

let final_branch_name = match existing_branch {
    Some(branch) if branch.commit_sha == current_head => {
        // Branch exists and points to the right commit — it's ours
        original_branch.clone()
    }
    Some(_) => {
        // Branch exists but points elsewhere — collision
        let new_name = generate_branch_name(&existing_branch_refs, &mut rng)?;
        db.update_archived_worktree_restored(
            row.id, &final_worktree_path, &new_name
        ).await?;
        new_name
    }
    None => {
        // Branch doesn't exist — recreate it
        original_branch.clone()
    }
};

// Create or checkout the branch
worktree_repo.create_branch(final_branch_name, None).await?;
// create_branch uses `git switch -c` which creates AND checks out
```

Wait, `create_branch` does `git switch -c <name> [<base>]`. If the branch
already exists (the "it's ours" case), `switch -c` would fail. For that case
we need `change_branch` instead.

```
if branch_exists_and_is_ours {
    worktree_repo.change_branch(final_branch_name).await?;
} else {
    worktree_repo.create_branch(final_branch_name, None).await?;
}
```

**Step 5: Update sqlite row**

```
db.update_archived_worktree_restored(
    row.id, &final_worktree_path, &final_branch_name
).await?;
```

**Step 6: Open the workspace and load the thread**

Reuse existing `open_workspace_and_activate_thread` or similar logic with
the `final_worktree_path`.

### 3.4 Cleanup after last unarchive

Cleanup is built into `maybe_restore_git_worktree` (see 3.2 above). After
unlinking the thread from the join table, we count remaining rows:

```sql
SELECT COUNT(*) FROM archived_thread_worktrees WHERE archived_worktree_id = ?
```

If 0, delete the git ref and the `archived_git_worktrees` row.

### 3.5 Fresh worktree creation (fallback)

When we can't restore (no row, no commit hash, ref gone, etc.):

1. Find the main repo for the thread's `work_dirs` path (use
   `resolve_git_worktree_to_main_repo` or the parent repo from the original
   worktree's known main repo path).
   
   **Problem**: If the worktree is deleted and the `.git` file is gone, we
   can't resolve to the main repo. We need another way.
   
   **Solution**: Store `main_repo_path` (the `original_repo_abs_path`) in the
   `archived_git_worktrees` table. That way even after the worktree is deleted,
   we know which main repo it belonged to.

   Update the table schema (already reflected in the final schema in 2.3):
   ```sql
   CREATE TABLE IF NOT EXISTS archived_git_worktrees(
       id INTEGER PRIMARY KEY,
       worktree_path TEXT NOT NULL,
       main_repo_path TEXT NOT NULL,
       branch_name TEXT NOT NULL,
       commit_hash TEXT,
       restored INTEGER NOT NULL DEFAULT 0
   ) STRICT;
   ```

2. Find the main repo entity by matching `work_directory_abs_path`.

3. Generate an adjective-noun branch name using
   `branch_names::generate_branch_name`.

4. Create worktree: `main_repo.create_worktree(branch_name, path, None)`.

5. Open workspace and load thread.

6. Show dismissable warning in the thread.

### 3.6 Dismissable warning UI

When restoration falls back to a fresh worktree, show a notification in
the thread. This could be:

- A toast notification via `workspace.show_notification(...)`.
- An inline banner in the agent panel / thread view.

The simplest is a workspace toast notification with a descriptive message:
"Unable to restore the original git worktree. You're now on a new worktree
split off from the project root."

### 3.7 Git crate additions for Branch 3

**Add `create_worktree_detached`** to `GitRepository` trait:
```
fn create_worktree_detached(&self, path: PathBuf, commit: String) -> BoxFuture<'_, Result<()>>;
```
Implementation: `git worktree add --detach <path> <commit>`

Or modify `create_worktree` to accept an `Option<String>` branch name where
`None` means detached.

**Add `head_sha`** to high-level `Repository`:
```
pub fn head_sha(&mut self) -> oneshot::Receiver<Result<Option<String>>>
```
Wraps the low-level `GitRepository::head_sha()`.

---

## Summary of all file changes

### Branch 2: `AI-112/archive-worktree-wip`

| File | Changes |
|---|---|
| `crates/git/src/repository.rs` | Add `allow_empty` to `CommitOptions`; add `update_ref`, `delete_ref`, `stage_all_including_untracked` to trait + `RealGitRepository` + `FakeGitRepository` |
| `crates/project/src/git_store.rs` | Add high-level `Repository` wrappers for new git methods; add `head_sha` method |
| `crates/agent_ui/src/thread_metadata_store.rs` | New migration for `archived_git_worktrees` table; new `ArchivedGitWorktree` struct; new db methods; expose through store |
| `crates/sidebar/src/sidebar.rs` | Modify `archive_thread` to find-or-create archived worktree row; modify `maybe_delete_git_worktree_for_archived_thread` to do WIP commit + ref creation before deletion |

### Branch 3: `AI-112/restore-worktree`

| File | Changes |
|---|---|
| `crates/git/src/repository.rs` | Add `create_worktree_detached` (or modify `create_worktree` for detached) |
| `crates/project/src/git_store.rs` | Wrap new detached worktree method |
| `crates/agent_ui/src/thread_metadata_store.rs` | Add query-by-path method; add ref count update methods; add cleanup method |
| `crates/sidebar/src/sidebar.rs` | New `maybe_restore_git_worktree` method; modify `activate_archived_thread` to call it; cleanup logic on unarchive |

---

## Resolved decisions

1. **`create_worktree` detached mode**: Make the branch name `Option<String>`
   where `None` means `--detach`.

2. **Thread-to-archived-worktree linkage**: Use a join table
   `archived_thread_worktrees(session_id, archived_worktree_id)`. Insert on
   archive, delete on unarchive. Count remaining rows to decide cleanup.

3. **AskPassDelegate for commit**: Add a `Repository::commit_no_verify`
   convenience method (or similar) that takes just a message + options,
   uses a no-op `AskPassDelegate`, and skips hooks. This keeps the WIP
   commit callsite clean.

4. **Finding the linked worktree's own repo entity**: Grab the repo entity
   in `maybe_delete_git_worktree_for_archived_thread` BEFORE workspace
   pruning happens. The method already runs before `store.delete` and
   before `prune_stale_worktree_workspaces`, so the repo is still available.

5. **Toast for WIP commit failure**: Use the multi-workspace window's
   notification system.

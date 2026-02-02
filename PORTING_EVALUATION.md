# Evaluation: Porting git-imerge from Git to Jujutsu (jj)

## Executive Summary

Porting `git-imerge` to jujutsu is a **medium-to-high complexity effort**. The project is a single 4,366-line Python module (`gitimerge.py`) that makes 50+ distinct `git` subprocess calls, uses Git's object model extensively for both state storage and commit manipulation, and relies on Git's index/working-copy model for conflict resolution. While jj uses Git's object database internally, it lacks Python bindings and several low-level plumbing equivalents, requiring significant architectural changes.

The most pragmatic path forward is a **hybrid approach**: operate in jj's colocated mode, use `jj` CLI for high-level operations (merge, rebase, conflict detection), and fall back to Git plumbing for low-level object manipulation (tree creation, commit-tree, hash-object). A full "native jj" port would require either rewriting in Rust to use `jj-lib`, or accepting a fragile dependency on jj's unstable CLI output format.

---

## 1. Scope of Git Dependencies

### 1.1 Git Commands Used (50+ invocations)

The `GitRepository` class (starting at `gitimerge.py:335`) wraps all Git interactions. Every operation goes through `subprocess` calls. The commands fall into several categories:

**Object creation and manipulation (hardest to port):**
- `git hash-object -t blob -w` (line 605) -- write blobs to object database
- `git hash-object -t commit -w` (line 1130) -- write raw commit objects
- `git commit-tree` (line 1058) -- create commits from tree + parents
- `git cat-file blob` (line 505) -- read raw blob content
- `git cat-file commit` (lines 836, 1103) -- read raw commit objects

**Merge operations (medium difficulty):**
- `git merge --no-commit` (line 671) -- initiate merge for user resolution
- `git checkout -f` + `git merge` (lines 655-656) -- automatic merge attempts
- `git merge --abort` via `git reset --merge` (line 910) -- abort failed merges
- `git -c rerere.enabled=false merge` (line 656) -- merge with rerere disabled

**Reference management (medium difficulty):**
- `git update-ref` (lines 612, 860, 868, 880) -- create/update/delete refs
- `git for-each-ref` (lines 383, 481, 494, 873) -- enumerate refs
- `git symbolic-ref` (lines 1026, 1036) -- manage HEAD

**History walking (low difficulty):**
- `git log --format=%H %P` (line 805) -- walk commits with parents
- `git merge-base --all` (line 914) -- find common ancestors
- `git rev-list --no-merges --count` (line 929) -- count commits
- `git rev-parse` (lines 355, 465, 800) -- resolve references

**Working tree / index (medium difficulty):**
- `git diff-files --quiet` (line 442) -- check for unstaged changes
- `git diff-index --cached --quiet` (line 451) -- check for uncommitted changes
- `git update-index -q --refresh` (line 471) -- refresh index
- `git add` (used by user during conflict resolution)
- `git commit --no-verify` (line 736) -- commit resolved merges
- `git reset --hard` (line 902) -- reset working tree
- `git checkout` (line 1040) -- switch branches

**Configuration (low difficulty):**
- `git config` (lines 397, 405, 411, 432) -- read/write `imerge.*` keys

### 1.2 State Storage Model

`git-imerge` stores all its state inside Git's own ref namespace:

```
refs/imerge/NAME/state     -- JSON blob containing merge metadata
refs/imerge/NAME/auto/I-J  -- automatically merged commit at position (I,J)
refs/imerge/NAME/manual/I-J -- manually merged commit at position (I,J)
```

This is clever because it makes imerge state:
- Pushable/fetchable via `git push origin refs/imerge/*`
- Garbage-collection safe (refs prevent GC of intermediate commits)
- Inspectable with standard git tools

The state JSON blob (written at line 602-617) contains: version, blockers list, tip1, tip2, goal, goalopts, manual flag, and branch name.

### 1.3 Core Algorithm

The incremental merge works on a 2D grid where:
- Row axis = commits on branch 1 (from merge base)
- Column axis = commits on branch 2 (from merge base)
- Each cell (i,j) = merge of commit i from branch 1 with commit j from branch 2

The algorithm uses binary search to find the "frontier" of conflicts, presents the smallest conflicting pairs to the user, and records results. The `MergeState`, `MergeFrontier`, `Block`, and `SubBlock` classes manage this grid.

---

## 2. Key Differences Between Git and jj

### 2.1 Working Copy Model

| Aspect | Git (current) | jj |
|--------|--------------|-----|
| Working copy | Dirty state separate from commits | **Is** a commit (`@`), auto-snapshotted |
| Index/staging | Explicit `git add` required | Does not exist |
| Conflict blocking | Merge stops; user must resolve | Conflicts committed as first-class data |
| Detached HEAD | Special mode | Normal operation; no "current branch" |

**Impact:** `git-imerge` relies heavily on:
1. Checking if the working tree is clean (`require_clean_work_tree`, lines 673-701)
2. Detecting merge-in-progress via `.git/MERGE_HEAD` (line 707)
3. Having users run `git add` + `git-imerge continue` for conflict resolution
4. Committing merge results via `git commit --no-verify`

In jj, the conflict resolution workflow would be fundamentally different. Conflicts are stored in commits, not in the working tree. The `git add` / `continue` loop would be replaced with editing the working copy commit and having jj detect when conflicts are resolved.

### 2.2 Merge Conflict Philosophy

`git-imerge`'s core value proposition is: "break a complex merge into pairwise conflicts so each one is easier to understand." This remains valuable in jj, but the mechanics change:

- In git: merge fails -> user resolves -> `git add` -> `imerge continue`
- In jj: merge creates a conflicted commit -> user edits -> conflict auto-resolves when file is saved

The `automerge()` method (line 648) does `checkout -f` + `merge` + check exit code. In jj, you'd do `jj new commit1 commit2` and then check if the resulting commit has conflicts (`jj log -r @ -T 'if(conflict, "true", "false")'`).

### 2.3 Reference vs Bookmark System

| Git refs | jj bookmarks |
|----------|-------------|
| Hierarchical (`refs/imerge/NAME/auto/1-2`) | Flat names, no hierarchy |
| Auto-advance on commit (for HEAD branch) | Never auto-advance |
| Stored in `.git/refs/` | Stored in `.jj/`, synced to `.git/refs` in colocated mode |

**Impact:** The entire `refs/imerge/` hierarchy would need rethinking. Options:
1. Use jj bookmarks with flat names like `imerge/NAME/auto/1-2` (bookmarks support `/` in names)
2. In colocated mode, continue using `git update-ref` directly
3. Store state outside the VCS entirely (e.g., in `.jj/imerge/` or a side file)

### 2.4 No Python Library for jj

Git has rich Python tooling (subprocess to `git`, `pygit2`, `gitpython`). jj has:
- `jj-lib` (Rust only, unstable API)
- CLI output with `-T` templates and `--no-graph` (unstable format)
- No Python bindings

---

## 3. Porting Strategies

### Strategy A: Hybrid (Colocated Mode) -- Recommended

**Approach:** Run in a jj colocated repo. Use `jj` for user-facing operations and conflict detection. Use `git` plumbing for low-level object manipulation.

**What changes:**
- Replace `git checkout -f` + `git merge` with `jj new commit1 commit2` for merge attempts
- Replace `git merge --abort` with `jj abandon @`
- Replace branch/HEAD management with `jj bookmark` commands
- Keep `git hash-object`, `git commit-tree`, `git cat-file` for object-level work
- Replace user-facing `git add` + `continue` workflow with jj's edit-and-save model
- Keep state storage in `refs/imerge/` (accessible via both git and jj)

**Pros:**
- Minimal rewrite (keep ~60% of the `GitRepository` class)
- Leverages Git's plumbing where jj has no equivalent
- Works today with existing jj versions

**Cons:**
- Dual dependency on both `git` and `jj` CLIs
- Potential synchronization issues between jj and git state
- Users must use a colocated repo

**Estimated scope:** ~1,500-2,000 lines changed in `gitimerge.py`, plus new `JjRepository` class or modified `GitRepository`.

### Strategy B: Pure jj CLI

**Approach:** Replace all git subprocess calls with jj CLI equivalents.

**What changes:**
- All 50+ git commands replaced with jj equivalents
- Object creation via `jj new`, `jj describe`, `jj commit` instead of `commit-tree`/`hash-object`
- State storage via bookmarks or external files
- History walking via `jj log` with revsets and templates
- Conflict detection via template expressions

**Blockers:**
- **No equivalent for `git commit-tree`**: jj cannot create a commit from an arbitrary tree object. The closest is `jj new parent1 parent2` which creates an empty merge commit, but you can't inject a specific tree.
- **No equivalent for `git hash-object`**: Cannot write arbitrary blobs to the object store.
- **No equivalent for `git cat-file commit`**: Cannot read raw commit bytes for the `reparent()` operation (lines 1103-1130) which manually rewrites commit parent pointers.
- **Unstable output format**: jj's CLI output is not stable; parsing it is fragile.

**Pros:**
- Clean single-tool dependency
- Takes advantage of jj's superior conflict model

**Cons:**
- Several operations have no jj equivalent (see blockers)
- Fragile dependency on CLI output parsing
- Significantly more work

**Estimated scope:** Full rewrite of `GitRepository` class (~700 lines), plus changes to `MergeState` persistence (~200 lines), conflict resolution workflow (~300 lines). Total: ~2,500-3,000 lines changed.

### Strategy C: Rewrite in Rust using jj-lib

**Approach:** Rewrite the tool in Rust, using `jj-lib` as a library.

**Pros:**
- Full access to jj's internal APIs
- Best integration with jj's conflict model
- Type-safe, high-performance

**Cons:**
- Complete rewrite from Python to Rust (~4,000 lines)
- `jj-lib` API is unstable and undocumented
- Massive effort; effectively a new project
- Loses Python's ease of modification

**Estimated scope:** 4,000-6,000 lines of new Rust code.

---

## 4. Detailed Change Analysis by Component

### 4.1 `GitRepository` class (lines 335-1130, ~800 lines)

| Method | Lines | jj Equivalent | Difficulty |
|--------|-------|---------------|------------|
| `git_dir()` | 352-358 | `jj workspace root` | Low |
| `check_imerge_name_format()` | 360-368 | Custom validation or keep git plumbing | Low |
| `iter_existing_imerge_names()` | 380-388 | `jj bookmark list` with filter, or keep `git for-each-ref` | Low |
| `set/get_default_imerge_name()` | 390-413 | jj config or custom file | Low |
| `unstaged_changes()` | 438-445 | Not applicable (no index in jj) | Conceptual change |
| `uncommitted_changes()` | 447-457 | Not applicable | Conceptual change |
| `require_clean_work_tree()` | 673-701 | Check if `@` has conflicts | Medium |
| `automerge()` | 648-666 | `jj new c1 c2` + check conflicts | Medium |
| `manualmerge()` | 668-671 | `jj new c1 c2` (conflicts stored in commit) | Medium |
| `simple_merge_in_progress()` | 703-712 | Check if `@` is a merge with conflicts | Medium |
| `commit_user_merge()` | 714-751 | `jj new` (finalize current working copy) | Medium |
| `rev_parse()` | 799-800 | `jj log -r REV -T commit_id --no-graph` | Low |
| `rev_list_with_parents()` | 802-808 | `jj log -r REVSET -T 'commit_id ++ " " ++ parents.map(|p| p.commit_id()).join(" ")'` | Medium |
| `get_author_info()` | 815-834 | `jj log -r REV -T` with author template | Low |
| `get_tree()` | 851-852 | No jj equivalent; use git in colocated | **High** |
| `commit_tree()` | 1050-1080 | No jj equivalent; use git in colocated | **High** |
| `reparent()` | 1093-1130 | `jj rebase` for simple cases; no raw equivalent | **High** |
| `update_ref()` | 856-868 | `jj bookmark set` / `jj bookmark move` | Medium |
| `delete_imerge_refs()` | 870-889 | `jj bookmark delete` for each | Medium |
| `abort_merge()` | 908-910 | `jj abandon @` or `jj undo` | Low |
| `compute_best_merge_base()` | 912-950 | Revset: `heads(::A & ::B)` | Low |
| `linear_ancestry()` | 958-999 | `jj log -r 'A::B'` with template | Medium |
| `get_head_refname()` | 1020-1033 | No "current branch" in jj; use `@` | Conceptual change |
| `checkout()` | 1039-1048 | `jj new REV` or `jj edit REV` | Low |

### 4.2 State Storage (`MergeState.save/read`, lines 2584-3338)

The `MergeState` class persists its grid of merge results as git refs. Each intermediate merge commit is saved as `refs/imerge/NAME/{auto,manual}/I-J`, and metadata is stored as a JSON blob at `refs/imerge/NAME/state`.

**Porting options:**
1. **Keep using git refs in colocated mode** -- simplest, but couples to git
2. **Use jj bookmarks** -- flat namespace, need naming convention like `imerge-NAME-auto-1-2`
3. **Use a JSON file** -- store in `.jj/imerge/NAME.json` with all commit hashes; simpler but not pushable
4. **Use jj's operation log** -- complex, not designed for this

Recommendation: Option 1 (keep git refs) for hybrid approach, Option 3 for pure jj approach.

### 4.3 Merge/Conflict Resolution Workflow (lines 648-751)

Current flow:
```
automerge() -> checkout -f commit1 -> merge commit2 -> check exit code
  success: return HEAD sha1
  failure: abort merge, raise AutomaticMergeFailed

manualmerge() -> merge --no-commit commit2
  user resolves conflicts
  git add <files>
  git-imerge continue -> commit_user_merge() -> git commit --no-verify
```

jj flow would be:
```
automerge() -> jj new commit1 commit2 -> check if @ has conflicts
  no conflicts: return @ commit id
  conflicts: jj abandon @ -> raise AutomaticMergeFailed

manualmerge() -> jj new commit1 commit2 (conflicts materialized in working copy)
  user edits files to resolve conflict markers
  jj-imerge continue -> check conflicts resolved -> jj new (move to next)
```

### 4.4 Command-Line Interface (lines 4008-4366)

The CLI uses `argparse` and dispatches to `cmd_*` functions. The interface itself is git-centric:
- `git-imerge merge BRANCH` -- would become `jj-imerge merge BRANCH`
- References to `git add` in user instructions would change
- `--branch` arguments reference git branches (would become jj bookmarks)

### 4.5 Test Suite (shell scripts in `t/`)

The test suite (`t/test-lib.sh` and 6 test scripts) uses raw git commands:
- `git init`, `git add`, `git commit`, `git branch -D`, `git rev-parse`, `git diff`
- Would need complete rewrite for jj equivalents
- ~300 lines total across all test files

### 4.6 Bash Completion (`completions/git-imerge`)

The completion script (305 lines) uses `git for-each-ref` to enumerate branches and imerge names. Would need rewrite using `jj bookmark list` and `jj log`.

---

## 5. Fundamental Design Challenges

### 5.1 Tree-Level Manipulation

The hardest porting challenge is `git-imerge`'s use of tree-level operations. The `commit_tree()` method (line 1050) creates commits from arbitrary tree objects with specified parents. This is used in simplification (converting the merge grid into a clean history) at lines 3076-3087.

jj has no equivalent operation. You cannot create a commit with a specific tree SHA -- you can only create commits by modifying the working copy. In colocated mode, you could still use `git commit-tree` directly, but in a pure jj port, you'd need to:
1. Create a working copy state matching the desired tree
2. Commit it with the right parents

This is significantly more complex and slower.

### 5.2 The Reparent Operation

The `reparent()` method (line 1093-1130) reads a raw commit object, modifies its parent lines, and writes it back with `git hash-object -t commit`. This is used by `cmd_reparent` and `reparent_recursively()`.

In jj, `jj rebase -r COMMIT -d NEW_PARENT` handles simple reparenting, but it doesn't give you the same low-level control (e.g., rewriting a commit to have 3 parents instead of 2, or changing parent order).

### 5.3 Conflict Detection vs Resolution

In git, `git-imerge` can detect conflicts by checking the exit code of `git merge`. In jj, you'd need to:
1. Create a merge commit: `jj new commit1 commit2`
2. Parse `jj log` output to check if the commit has conflicts
3. If no conflicts, record the commit hash and move on
4. If conflicts, abandon the commit

This is functionally equivalent but requires more subprocess calls and output parsing.

### 5.4 State Isolation

`git-imerge` does a lot of `checkout -f` + `merge` sequences that thrash the working tree. In jj, each `jj new commit1 commit2` modifies the working copy commit. This means the user's actual work could be disrupted.

Potential solutions:
- Use `jj workspace add` to create a separate workspace for imerge operations
- Use the `--at-operation` flag to avoid modifying the main workspace
- Work entirely at the object level (requires git plumbing in colocated mode)

---

## 6. Recommended Approach

### Phase 1: Hybrid Port (Colocated Mode)

1. Create a `JjGitRepository` class that extends or wraps `GitRepository`
2. Replace user-facing operations with jj equivalents:
   - `checkout` -> `jj new`
   - `merge` -> `jj new parent1 parent2`
   - `commit` -> `jj commit` / `jj describe`
   - Branch management -> `jj bookmark`
3. Keep git plumbing for object manipulation:
   - `git commit-tree`, `git hash-object`, `git cat-file` (unchanged)
4. Adapt conflict resolution workflow:
   - Replace `git add` + `continue` with jj's edit-in-place model
5. Update CLI help text and user instructions
6. Rewrite test suite for jj

### Phase 2: Reduce Git Dependency (Optional)

1. Replace `git for-each-ref` with `jj bookmark list`
2. Replace `git log` history walking with `jj log` + revsets
3. Replace `git merge-base` with revset expressions
4. Move state storage from `refs/imerge/` to jj-native storage

### Phase 3: Full Native Port (If jj-lib Stabilizes)

1. If/when jj-lib gets Python bindings or a stable API, rewrite object manipulation
2. Alternatively, rewrite in Rust as a jj extension

---

## 7. Effort Estimate Summary

| Component | Lines Affected | Difficulty |
|-----------|---------------|------------|
| `GitRepository` class | ~500 of 800 | Medium-High |
| State storage (`MergeState`) | ~200 | Medium |
| Conflict resolution workflow | ~300 | Medium |
| CLI and user instructions | ~100 | Low |
| Test suite | ~300 (full rewrite) | Medium |
| Bash completion | ~150 (full rewrite) | Low |
| **Total** | **~1,550 lines** | **Medium** |

This estimate is for the hybrid approach (Strategy A). A pure jj port (Strategy B) would roughly double this. A Rust rewrite (Strategy C) would be a ground-up project.

---

## 8. Risks and Open Questions

1. **jj CLI stability**: jj's output format is not stable. Machine-readable output via `-T` templates may change between versions.
2. **Colocated mode edge cases**: Mixing `git` plumbing with `jj` operations may cause synchronization issues. After running `git` commands directly, `jj` may need to reimport state.
3. **Performance**: Subprocess calls to `jj` may be slower than `git` plumbing (jj has more startup overhead due to auto-snapshotting).
4. **jj's conflict model**: `git-imerge`'s algorithm assumes conflicts are binary (succeed/fail). jj's first-class conflicts add nuance that the algorithm doesn't currently exploit.
5. **User expectations**: jj users may expect different interaction patterns (e.g., not needing `git add`).
6. **Feature interaction**: How does imerge's temporary workspace thrashing interact with jj's auto-snapshotting? Need to ensure jj doesn't create spurious operations for intermediate states.

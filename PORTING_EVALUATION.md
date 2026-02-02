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

---

## 9. Alternative: A jj-Native Approach to Pairwise Conflict Resolution

Rather than porting git-imerge's implementation, jj's design enables a fundamentally simpler approach to the same problem. The core user need is: **"I want to resolve one small conflict at a time, not a pile of unrelated conflicts simultaneously."** jj's first-class conflicts, automatic propagation, and flexible rebase make this achievable with far less machinery than git-imerge requires.

### 9.1 Why git-imerge Is So Complex

git-imerge's 4,366 lines exist largely because of Git's limitations:

1. **Git merges are blocking** -- a conflict stops execution, so the tool must carefully manage the `checkout -f` / `merge` / `abort` / `commit` dance
2. **Git has no conflict-in-commits** -- intermediate conflicted states can't be saved, so the tool needs a ref-based persistence layer (`refs/imerge/`)
3. **Git's working tree is shared state** -- the tool must check for clean state, save/restore HEAD, and manage the index
4. **Optimization is essential** -- because each merge attempt thrashes the working tree, git-imerge uses bisection to minimize the number of merge probes

In jj, **none of these constraints exist**:
- Merges never block (conflicts are stored in commits)
- Conflicted commits are normal commits that can be rebased and manipulated
- `jj new` creates a fresh commit without disrupting anything
- Conflict resolution in one commit automatically propagates to descendants

### 9.2 The Simplest jj Approach: Incremental Rebase

The simplest way to achieve pairwise conflict resolution in jj requires **no external tool at all**. It's a workflow pattern:

**Scenario:** You want to merge branch A (commits a1, a2, a3) with branch B (commits b1, b2, b3), but a direct merge produces a complex conflict.

**Step 1: Rebase one commit at a time**
```bash
# Instead of merging all of A onto tip of B at once:
#   jj new tip_a tip_b   <-- this produces the big complex conflict

# Rebase just a1 onto b1, then a2 onto that result, etc:
jj rebase -s a1 -d b1
# If a1' has conflicts, resolve them (edit files, jj squash)
# Then continue with a2, a3...
```

Since jj's rebase never stops, all commits are rebased at once. Some may have conflicts, some may not. The user works through them in order:
```bash
jj log                    # see which commits have conflicts
jj new <first_conflicted> # check it out
# resolve conflicts by editing files
jj squash                 # fold resolution back into the commit
# jj automatically re-rebases descendants -- some conflicts may auto-resolve
jj log                    # check what's left
```

**What this gives you:** Each conflict involves only one commit's worth of changes from branch A, making each conflict smaller and more focused than a big-bang merge.

**What this doesn't give you:** It doesn't decompose the *destination* side. If B has 20 commits and a1 conflicts with some combination of them, you still get the full conflict of a1 vs all-of-B in one shot.

### 9.3 The Full 2D Grid Approach in jj

For the full git-imerge-style experience (decomposing *both* sides), a script could build the merge grid using jj's native operations:

```
Grid structure (same as git-imerge):

      b0    b1    b2    b3
  a0  base  b1    b2    b3     <- row 0: just branch B commits
  a1  a1    (1,1) (1,2) (1,3)  <- row 1: a1 merged with each b
  a2  a2    (2,1) (2,2) (2,3)  <- row 2: a1+a2 merged with each b
  a3  a3    (3,1) (3,2) (3,3)  <- row 3: all of A merged with each b

Cell (i,j) = merge of cell(i-1,j) and cell(i,j-1)
             using cell(i-1,j-1) as the base
```

**In jj, building this grid is straightforward:**

```bash
# For each cell (i,j) in the grid:
jj new cell_i-1_j cell_i_j-1    # create merge commit with two parents
jj describe -m "imerge (i,j)"   # label it
# Check if @ has conflicts:
#   jj log -r @ -T 'conflict' --no-graph
# Record the change ID, move on to the next cell
```

The critical difference from git-imerge: **every cell can be created regardless of conflicts**. In git, `git merge` fails on conflict and you must abort before trying the next cell. In jj, `jj new parent1 parent2` always succeeds -- it creates a conflicted commit if the merge isn't clean.

This means you can build the **entire grid at once**, then present the user with the conflicted cells in order. When the user resolves one, jj's conflict propagation may auto-resolve others.

### 9.4 Conflict Propagation Changes Everything

This is the key insight that makes a jj-native tool fundamentally different from git-imerge.

**In git-imerge:** Resolving cell (1,1) gives you a commit SHA. Cells (2,1) and (1,2) must be computed separately by doing new merges. The resolution of (1,1) helps indirectly (it's an ancestor of the parents being merged), but the tool must explicitly compute each cell.

**In jj:** If the grid is built as a DAG where cell (i,j) is a child of cell (i-1,j) and cell (i,j-1), then resolving a conflict in cell (i,j) causes jj to **automatically re-rebase all descendant cells**. This means:

1. Build the grid as a DAG of merge commits
2. Find the smallest conflicted cell (closest to 0,0)
3. User resolves it
4. jj propagates the resolution -- descendants are automatically rebased
5. Check which cells still have conflicts
6. Repeat from step 2

In the best case, resolving cell (1,1) automatically resolves cells (2,1), (1,2), (2,2), etc., because the same conflict was cascading through the grid. In git-imerge, you'd have to recompute each of those cells explicitly.

### 9.5 What a Minimal jj-imerge Tool Would Look Like

Instead of 4,366 lines of Python managing git plumbing, a jj-native tool needs:

1. **Grid construction** (~50-100 lines): enumerate commits on both branches, create merge commits for each cell using `jj new parent1 parent2`
2. **Conflict detection** (~20 lines): query each cell for conflicts using `jj log -r <rev> -T 'conflict'`
3. **User workflow** (~30 lines): point the user at the next conflicted cell, wait for resolution
4. **Simplification/finish** (~50 lines): once all cells are clean, extract the final result from cell (n,m) and move a bookmark to it

**That's roughly 150-250 lines of shell script or Python**, compared to git-imerge's 4,366 lines. The reduction comes from:
- No working-tree management (jj handles it)
- No state persistence layer (commits exist in the DAG; use change IDs to track them)
- No conflict detection retry loop (jj merges never fail)
- No bisection algorithm needed (you can build the full grid cheaply since each cell is just `jj new`)
- No `automerge` / `manualmerge` distinction (all cells are created the same way)

### 9.6 Sketch of the Algorithm

```python
#!/usr/bin/env python3
"""jj-imerge: incremental merge for jujutsu"""

import subprocess, json, sys

def jj(*args):
    """Run a jj command and return stdout."""
    return subprocess.check_output(
        ['jj', '--color=never'] + list(args)
    ).decode().strip()

def jj_log(revset, template):
    """Query jj log with a template, return lines."""
    out = jj('log', '-r', revset, '-T', template, '--no-graph')
    return [l for l in out.splitlines() if l]

def has_conflict(rev):
    """Check if a revision has conflicts."""
    result = jj_log(rev, 'if(conflict, "true", "false")')
    return result and result[0] == "true"

def get_commit_id(rev):
    """Get the commit ID of a revision."""
    return jj_log(rev, 'commit_id')[0]

def build_grid(base, tip1, tip2):
    """Build the incremental merge grid.

    Returns a 2D array of change IDs, where grid[i][j] is the
    merge of commits 0..i from branch1 with commits 0..j from branch2.
    """
    # Get commit lists
    commits1 = jj_log(f'{base}..{tip1}', 'commit_id ++ "\\n"')
    commits2 = jj_log(f'{base}..{tip2}', 'commit_id ++ "\\n"')

    n, m = len(commits1), len(commits2)
    grid = [[None]*(m+1) for _ in range(n+1)]

    # Row 0 = base, b1, b2, ...
    grid[0][0] = get_commit_id(base)
    for j in range(1, m+1):
        grid[0][j] = commits2[j-1]

    # Column 0 = base, a1, a2, ...
    for i in range(1, n+1):
        grid[i][0] = commits1[i-1]

    # Fill the grid
    for i in range(1, n+1):
        for j in range(1, m+1):
            parent1 = grid[i][j-1]   # left neighbor
            parent2 = grid[i-1][j]   # top neighbor
            jj('new', parent1, parent2, '-m',
               f'imerge ({i},{j})')
            grid[i][j] = get_commit_id('@')

    return grid

def find_conflicts(grid):
    """Return list of (i, j) cells that have conflicts, sorted by i+j."""
    conflicts = []
    for i in range(1, len(grid)):
        for j in range(1, len(grid[0])):
            if has_conflict(grid[i][j]):
                conflicts.append((i, j))
    conflicts.sort(key=lambda x: x[0] + x[1])
    return conflicts

def resolve_loop(grid):
    """Present conflicted cells to user one at a time."""
    while True:
        conflicts = find_conflicts(grid)
        if not conflicts:
            print("All conflicts resolved!")
            return grid[-1][-1]  # final merge result

        i, j = conflicts[0]
        print(f"Conflict at ({i},{j}) -- {len(conflicts)} remaining")
        print(f"  This merges commit {i} from branch1")
        print(f"  with commit {j} from branch2")

        # Point user at the conflict
        jj('new', grid[i][j])
        print("Resolve the conflict, then run: jj squash")
        input("Press Enter when resolved...")

        # After resolution, jj auto-rebases descendants
        # Re-check the grid (change IDs are stable across rebases)
```

### 9.7 Important Caveats

**Grid construction modifies the DAG.** Building the full grid creates n*m merge commits. For large branches (e.g., 50 x 50 = 2,500 commits), this could be slow and cluttered. Mitigations:
- Only build the grid incrementally (row by row or along the diagonal)
- Use a separate jj workspace (`jj workspace add`) to isolate imerge commits
- Clean up intermediate commits after finishing

**Change IDs vs commit IDs.** The sketch above uses commit IDs, but change IDs would be more robust since they survive rewrites. When jj propagates a conflict resolution, the commit ID of descendant cells changes, but their change IDs stay the same.

**Conflict propagation is not guaranteed to cascade.** If cell (1,1) has a conflict because a1 modifies file X and b1 also modifies file X, resolving (1,1) helps descendant cells *that inherit the same conflict*. But cell (2,1) might have a *different* conflict (a2 modifies file Y which b1 also touches), which won't be resolved by fixing (1,1). The user would still need to resolve each unique conflict, but they won't see the *same* conflict repeated across multiple cells.

**The simplification step still matters.** After all conflicts are resolved, the grid contains many intermediate merge commits that shouldn't appear in the final history. You'd still need to extract the final result -- either by using cell (n,m) directly as a merge commit, or by using `jj rebase` to produce a clean rebase/merge result. This is simpler in jj than in git because `jj rebase -r` can reparent individual commits without low-level plumbing.

### 9.8 Comparison: git-imerge vs jj-native Approach

| Aspect | git-imerge | jj-native approach |
|--------|-----------|-------------------|
| Lines of code | 4,366 | ~150-250 |
| Grid construction | Merge + check exit code + abort on failure | `jj new p1 p2` (always succeeds) |
| Conflict detection | Exit code of `git merge` | Template query on commit |
| State persistence | `refs/imerge/` hierarchy | Commits in the DAG (change IDs) |
| Working tree management | `checkout -f`, `reset --hard`, index manipulation | Not needed |
| Bisection optimization | Yes (avoids unnecessary merges) | Optional (building all cells is cheap) |
| Conflict resolution propagation | Manual (recompute each cell) | Automatic (jj rebases descendants) |
| User workflow | `git add` + `git-imerge continue` | Edit files + `jj squash` |
| Cleanup/finish | `simplify()` with `commit-tree` plumbing | `jj rebase -r` to reparent final result |
| Dependencies | Python + git | Python (or shell) + jj |

### 9.9 Recommended Path Forward

For a user who just wants pairwise conflict resolution in jj:

1. **Start with the incremental rebase workflow** (section 9.2) -- requires no tooling at all. This handles 80% of cases where you're rebasing a feature branch onto an updated main.

2. **If that's not sufficient**, build a lightweight script (section 9.6) that constructs the 2D merge grid using `jj new`. This handles the case where both sides have significant changes and you want to decompose conflicts in both dimensions.

3. **Only if heavy use demands it**, invest in a full tool with: progress visualization (like git-imerge's `diagram` command), multiple simplification goals, session persistence, and bash completion.

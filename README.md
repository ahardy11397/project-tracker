# project-tracker

A project and task dependency tracker CLI written 100% in [Memelang](https://github.com/memelang) (`meme`).

Demonstrates graduate-level graph algorithms implemented in Canonical Structured English:
- **Directed Acyclic Graphs (DAGs)** with forward and reverse adjacency maps (`Map<String, List<String>>`).
- **Set-Time Cycle Detection** using reachability traversal over `Set<String>` to reject self-cycles, 2-cycles, and transitive cycles before tasks are persisted.
- **Topological Sorting** via Kahn's algorithm with deterministic lexicographical tie-breaking across parallel ready tasks.
- **Critical-Path Scheduling** via dynamic programming over topological order, supporting multiple chains sharing tasks and deterministic tie-breaking.
- **Transitive Blocking Analysis** traversing incomplete prerequisite tasks.
- **Null-Safe JSON Persistence** in `data/tasks.json` with `Optional<List<TaskRow>>` handling for missing or corrupt files.

Verified with 100% test pass rate across **all six Memelang backend targets**: Python, Rust, C++20, Go, C (C11), and TypeScript (Node.js).

---

## CLI Usage

Compile and run directly with `meme run main.meme -- ...` or build a native executable with `meme build main.meme -o tracker_bin`.

```bash
# Add a task with duration in integer hours and optional comma-separated dependencies
tracker add-task <id> <title> <duration-hours> [dep1,dep2,...]

# Mark an incomplete task as complete (completing an already completed task fails)
tracker complete <id>

# List incomplete tasks whose dependencies are all completed, sorted by id
tracker ready

# Output topological order of ALL tasks (work order)
tracker order

# Print the longest dependency chain by total hours and the total duration
tracker critical-path

# List all transitive incomplete tasks blocking an incomplete task, sorted by id
tracker blocked <id>

# Output completed / total count and completion percentage
tracker progress
```

### Examples

```bash
# Add tasks
./tracker_bin add-task spec "Architecture Spec" 4
./tracker_bin add-task db "Database Migration" 6 spec
./tracker_bin add-task backend "API Backend" 12 db
./tracker_bin add-task frontend "UI Components" 8 spec
./tracker_bin add-task deploy "Production Deploy" 4 backend,frontend

# View status
./tracker_bin ready
# Output: spec

./tracker_bin critical-path
# Output:
# spec -> db -> backend -> deploy
# Total hours: 26

./tracker_bin progress
# Output: 0/5 tasks complete (0%)

# Complete initial task
./tracker_bin complete spec
./tracker_bin ready
# Output: db, frontend
```

---

## Graph Model & Persistence

### In-Memory Graph Model
The dependency graph is represented as a Directed Acyclic Graph (DAG) with vertices representing tasks and directed edges representing prerequisites ($u \to v$ where $u \in \text{deps}(v)$):
- **Task Records**:
  ```meme
  record Task:
      id: String
      title: String
      duration: Int
      deps: List<String>
      completed: Bool
  ```
- **Adjacency**: `build_adjacency_map` produces `Map<String, List<String>>` (forward adjacency mapping each task to its dependents), and `build_dependencies_map` produces `Map<String, List<String>>` (reverse adjacency mapping each task to its direct prerequisites).
- **Visited Sets**: Cycle checks and reachability queries use `Set<String>` (`empty set of String`, `add ... to set`, `contains`) for fast lookup and cycle prevention.

### JSON Persistence
- Data is saved to `data/tasks.json`.
- Uses `TaskRow` with comma-separated dependency string serialization to adhere strictly to flat primitive record schemas in JSON readers:
  ```meme
  record TaskRow:
      id: String
      title: String
      duration: Int
      deps: String
      completed: Bool
  ```
- **Tolerance**: Data is loaded using Optional record readers (`read json records TaskRow from file path or else nothing`). Missing files, invalid JSON syntax, or schema mismatches cleanly return an empty task list (`empty List<Task>`) without throwing exceptions or generating tracebacks.

---

## Algorithm Notes

### 1. Set-Time Cycle Detection (`would_cycle`)
Before any task is added or updated, `would_cycle(tasks, id, deps)` checks whether introducing the new dependency edges would form a directed cycle.
- **Self-Cycle**: Checked immediately if `id` is present in `deps`.
- **Transitive Cycle**: A BFS traversal starting from each dependency in `deps` explores prerequisite chains using a `visited: Set<String>`. If `id` is encountered along any path, the addition is rejected with exit code 1, leaving the persisted dataset completely untouched.

### 2. Deterministic Topological Sort (`topological_order`)
Computes a valid execution sequence where every dependency appears strictly before its dependents:
- Implements Kahn's algorithm maintaining unsatisfied dependency counts (in-degrees).
- When multiple tasks have zero unsatisfied dependencies concurrently, ties are broken **lexicographically by task ID** (`sort candidates ascending`).
- Pairwise assertion tests verify that for all tasks $t$ and all $d \in t.\text{deps}$, $\text{index}(d) < \text{index}(t)$.

### 3. Critical Path Scheduling (`critical_path`)
Finds the longest dependency chain through the graph by cumulative task hours:
- Processes tasks along the topological order using dynamic programming.
- For each task $v$, the longest path ending at $v$ is determined by:
  $$\text{dist}(v) = \max_{u \in \text{deps}(v)} (\text{dist}(u) + \text{duration}(v))$$
  (or $\text{duration}(v)$ if $\text{deps}(v)$ is empty).
- **Shared Chains & Tie-Breaking**: When multiple paths achieve identical total hours, the tie-break is resolved lexicographically by the first differing task ID (`is_chain_smaller`).
- Scans all nodes to select the global maximum duration chain.

### 4. Transitive Blocked Analysis (`blocked_tasks`)
Computes the set of incomplete tasks that prevent an incomplete task $T$ from being worked on:
- Walks the ancestor prerequisite subgraph starting from $T$.
- Only incomplete tasks are recorded and recursively expanded. Completed ancestors are ignored because they are already satisfied.
- The resulting task IDs are returned in lexicographically sorted order.

---

## Test Suites

Tests are located in `tests/meme/*.meme` and run using Memelang's built-in test runner:

```bash
uv run --project /mnt/ssd/projects/meme meme test --target python
```

### Test Coverage (29 test cases across 8 suites)
1. `01_add_validation_test.meme` (4 cases):
   - `add_task_ok`: valid task creation with and without dependencies.
   - `duplicate_id_detection`: duplicate task ID rejection.
   - `unknown_dep_detection`: rejecting dependencies that do not exist.
   - `duplicate_dep_helper`: duplicate entries in dependency lists.
2. `02_cycle_rejection_test.meme` (5 cases):
   - `self_cycle_rejected`: self-dependency (`a -> a`).
   - `two_cycle_rejected`: mutual dependency (`a -> b -> a`).
   - `indirect_cycle_rejected`: transitive cycle (`a -> b -> c -> a`).
   - `dag_no_false_positive`: diamond DAG accepted without false positives.
   - `cycle_rejection_leaves_sheet_unchanged`: verification that rejected cycle additions leave persisted files untouched.
3. `03_topo_order_test.meme` (3 cases):
   - `topo_order_linear`: linear dependency chain ordering.
   - `topo_order_diamond_pairwise_validity`: pairwise check that all dependencies appear before dependents.
   - `topo_order_deterministic_tiebreak`: lexicographical tie-breaking for parallel independent tasks.
4. `04_ready_filtering_test.meme` (2 cases):
   - `ready_tasks_initial`: initial ready tasks filtering.
   - `ready_tasks_progression`: dynamic readiness updates as prerequisite tasks are marked completed.
5. `05_critical_path_test.meme` (3 cases):
   - `critical_path_diamond_dag_known_answer`: diamond DAG with branches of 12h vs 16h (verifying 16h path chosen).
   - `critical_path_tie_break`: equal-length / equal-duration paths tie-broken lexicographically.
   - `critical_path_multiple_shared_chains`: overlapping chains sharing nodes.
6. `06_blocked_test.meme` (3 cases):
   - `blocked_transitive_chain`: deep prerequisite chain blocker detection.
   - `blocked_completed_prereqs_excluded`: completed prerequisites omitted from blocker list.
   - `blocked_branching_dag`: multi-branch incomplete blocker sorting.
7. `07_progress_test.meme` (5 cases):
   - `progress_empty_list`: 0 tasks $\to$ 0/0, 0%.
   - `progress_none_completed`: 10 tasks, 0 done $\to$ 0/10, 0%.
   - `progress_partial_completed`: 10 tasks, 3 done $\to$ 3/10, 30%.
   - `progress_all_completed`: 10 tasks, 10 done $\to$ 10/10, 100%.
   - `progress_integer_division`: 3 tasks, 1 done $\to$ 1/3, 33%.
8. `08_persistence_tolerance_test.meme` (4 cases):
   - `roundtrip_task_records`: JSON serialization and deserialization roundtrip.
   - `missing_file_tolerance`: non-existent files load cleanly as empty lists.
   - `corrupt_json_tolerance`: invalid JSON loaded cleanly without crashing.
   - `schema_mismatch_tolerance`: schema mismatches load cleanly without crashing.

---

## Multi-Target Verification

Every test suite passes with **0 failures** across all six supported Memelang compilation targets:

| Target | Command | Result |
|---|---|---|
| **Python** | `uv run meme test --target python` | **29 passed, 0 failed** |
| **Rust** | `uv run meme test --target rust` | **29 passed, 0 failed** |
| **C++20** | `uv run meme test --target cpp` | **29 passed, 0 failed** |
| **Go** | `uv run meme test --target go` | **29 passed, 0 failed** |
| **C (C11)** | `uv run meme test --target c` | **29 passed, 0 failed** |
| **TypeScript** | `uv run meme test --target ts` | **29 passed, 0 failed** |

---

## End-to-End Demo (10-Task Release Checklist)

Compiled via `meme build main.meme -o tracker_bin`:

```text
=== 1. Initialize 10-Task Release Checklist ===
Added task spec
Added task db_migration
Added task api_services
Added task cache_warmup
Added task web_bundle
Added task cdn_deploy
Added task smoke_ui
Added task integrations
Added task audit_signoff
Added task dns_cutover

=== 2. Initial State ===
--- Progress ---
0/10 tasks complete (0%)
--- Ready Tasks ---
spec
--- Topological Work Order ---
spec
db_migration
api_services
cache_warmup
web_bundle
cdn_deploy
smoke_ui
integrations
audit_signoff
dns_cutover
--- Critical Path ---
spec -> db_migration -> api_services -> cache_warmup -> integrations -> audit_signoff -> dns_cutover
Total hours: 43
--- Tasks Blocking integrations ---
api_services
cache_warmup
cdn_deploy
db_migration
smoke_ui
spec
web_bundle

=== 3. Complete Initial Task (Unlocking Parallel Branches) ===
Completed task spec
--- Ready Tasks (both branches unlocked) ---
db_migration
web_bundle
--- Progress ---
1/10 tasks complete (10%)

=== 4. Advance Both Branches in Parallel ===
Completed task db_migration
Completed task web_bundle
--- Ready Tasks ---
api_services
cdn_deploy
--- Progress ---
3/10 tasks complete (30%)

=== 5. Complete Frontend Branch ===
Completed task cdn_deploy
Completed task smoke_ui
--- Ready Tasks ---
api_services
--- Tasks Blocking integrations (Frontend branch cleared, Backend branch remaining) ---
api_services
cache_warmup
--- Progress ---
5/10 tasks complete (50%)
```

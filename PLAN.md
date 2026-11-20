# PLAN — project-tracker: a Memelang graduate-level demo

Build a project/task dependency tracker CLI entirely in Memelang. Beta
demonstration program: DAGs, topological sort, and critical-path scheduling.

Language docs: /mnt/ssd/projects/meme/docs/WRITING_GUIDE.md and
docs/TUTORIAL.md (read both first). Compiler commands via
`uv run --project /mnt/ssd/projects/meme meme ...` from this repo.

## Ground rules (non-negotiable)

- Work ONLY in this repo. NO git commits.
- 100% Memelang; durations in integer HOURS.
- JSON persistence in `data/` via Optional record readers/writers; missing/
  corrupt file = clean Optional handling, never a traceback.
- Everything must parse with `--provider heuristic`.
- Test suites run in the FOREGROUND only.

## Domain and CLI (thin argv-router main + logic modules; unknown command →
usage + exit 1; every error = one stderr-style line + exit 1)

    tracker add-task <id> <title> <duration-hours> [dep1,dep2,...]
                      # ids are short slugs; deps comma-separated; unknown
                      # deps rejected; duplicate id rejected
    tracker complete <id>
    tracker ready                  # incomplete tasks whose deps are all done,
                                   # sorted by id
    tracker order                  # topological order of ALL tasks (work order)
    tracker critical-path          # longest dependency chain by total hours;
                                   # prints the chain + total hours
    tracker blocked <id>           # transitive blocking tasks of an incomplete
                                   # task, sorted
    tracker progress               # done/total counts + percent complete

Engine rules: cycles are rejected at add-time (`add-task` that would create
a dependency cycle fails with one stderr line + exit 1 and leaves the task
unadded); completing an already-complete task fails; critical path must
handle MULTIPLE chains sharing tasks (the chain through the graph with
maximum total hours; tie-break lexicographically by first differing task
id). Graph stored as `Map<String, List<String>>` adjacency + task records;
visited sets as `Set<String>`.

## Tests — tests/meme/*.meme, `meme test`

At minimum: add (ok, duplicate rejected, unknown dep rejected); cycle
rejection (2-cycle, self-dep) with sheet-unchanged assertion; topological
order validity (every dep before dependent — assert pairwise over the
order); ready-list filtering; critical path on a diamond DAG with known
answer; critical path tie-break; blocked transitive chain; progress math;
corrupt data tolerance. Fixed data, unique scratch files, cleaned up.

## Acceptance ("done means")

1. `meme test` python — 0 failed.
2. Same `--target rust cpp go c ts` — ALL SIX green.
3. `meme build main.meme -o tracker_bin` — end-to-end demo: a 10-task
   release checklist with two parallel branches, complete some tasks,
   show ready/order/critical-path/progress. Transcript in your summary.
4. README.md: CLI usage, graph model, algorithm notes, tests.
5. Final message: files, test count per target, demo transcript,
   deviations with reasons.

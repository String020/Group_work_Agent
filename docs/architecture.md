# ContextKit architecture

Version: proposed contract v1, 15 October 2026. Read with [requirements](requirements.md) and [agent rules](../AGENTS.md). This is an implementation plan, not evidence of a working implementation or team approval.

## 1. Decisions for team review

The product scope is agreed in the current brief. Every specific decision below is **Team-proposed (TP)**. Course constraints are identified separately as CC01-CC12 in the requirements. Record acceptance or amendments in a reviewed GitLab MR before implementation depends on a decision; do not invent reviewer names or approval dates.

| ID | Proposed v1 decision | Reason and review point |
| --- | --- | --- |
| D01 | Python 3.12 venv, standard library only; Bash 4.4+, Git and `find`; no Docker in MVP. | Small offline setup. Use `unittest`, `ast`, `cProfile` and `tracemalloc`; confirm unit-server tools. Keep comment-only `requirements.txt` until an approved dependency is needed. |
| D02 | Bash owns repository inventory and Git collection; fixed exclusions and resource limits. | Bash performs core work, rather than only launching Python. Review scope and path safety. |
| D03 | CLI and output schema v1; exact UTF-8 byte budget for the whole `context.md`. | Auditable without a model/tokenizer. Review interface, empty results, exit codes and publication behavior. |
| D04 | Static local-module graph; path-based nodes; bounded forward and reverse BFS. | Useful dependency context with explicit uncertainty. Review source-root/import cases. |
| D05 | Fixed keyword/symbol/distance/Git formula and greedy score-per-byte packing. | Explainable team-designed logic; no trained model or optimality claim. Freeze weights before evaluation. |
| D06 | No target execution; no persistent cache; fresh hashes and deterministic outputs; publish into a new directory. | Avoid stale context and accidental target edits. Review snapshot and error behavior. |
| D07 | Human-labelled cases, B0/B1 comparisons, held-out split, tests and profiling. | Measure context retrieval and workflow cost without claiming LLM task success. Review labels and targets independently. |
| D08 | Human design control, owner plus independent human checker, logged AI use; proposed three activities across three stages. | Address CC07's source disagreement and preserve authentic teamwork. Team chooses activities and responsibilities. |

## 2. Components and flow

```text
repository + task + byte budget + source roots
                     |
                     v
          Python CLI: validate inputs
                     |
                     v
     Bash scan_repo.sh: find/prune .py paths
        + collect Git HEAD/change status
                     |
        NUL records + explicit scan status
                     v
    Python scan.py: validate paths, decode, hash
                     |
                     v
        parse.py: AST symbols/imports/spans
                     |
                     v
   graph.py: resolve local modules; forward/reverse
                     |
                     v
     rank.py: candidates, features, explanations
                     |
                     v
  budget.py + render.py: exact-byte greedy selection
                     |
                     v
   output.py: recheck snapshot, validate, publish
                     |
        +------------+-------------+
        v            v             v
    context.md   manifest.json   analysis.json

 bundled tasks + human labels -> evaluate.py -> B0/B1 comparisons
 test.sh -> unit/integration/end-to-end checks
 profile.sh -> stage timings/cProfile + unprofiled repeated runs
```

Python controls the pipeline but must consume the Bash inventory; it cannot independently rediscover files as a fallback. The scanner's exclusion decisions and Git records affect every production run. Graph construction, feature calculations and packing are team-designed Python logic. Evaluation reuses candidate generation/rendering but never feeds reference labels into selection.

### Suggested repository layout

Only the three Markdown files in this delivery exist by virtue of this document. The other entries below are implementation targets.

```text
ContextKit/
|-- AGENTS.md
|-- README.md
|-- .gitignore
|-- requirements.txt             # no third-party packages in v1
|-- scripts/
|   |-- scan_repo.sh             # core discovery and Git collection
|   |-- demo.sh
|   |-- test.sh
|   `-- profile.sh
|-- src/contextkit/
|   |-- __init__.py
|   |-- __main__.py
|   |-- cli.py
|   |-- models.py                # dataclasses, versions, validation
|   |-- scan.py
|   |-- parse.py
|   |-- graph.py
|   |-- rank.py
|   |-- budget.py
|   |-- render.py
|   |-- output.py
|   `-- evaluate.py
|-- tests/
|   |-- fixtures/
|   |-- test_scan.py
|   |-- test_parse_graph.py
|   |-- test_rank_budget.py
|   `-- test_cli_evaluation.py
|-- scenario/
|   |-- repo/                   # committed, sanitised Python source
|   |-- tasks/                  # T01.txt, ...
|   `-- references.json         # human-reviewed cases and labels
|-- docs/
|   |-- requirements.md
|   |-- architecture.md
|   |-- scenario.md
|   `-- ai-use.md
`-- artifacts/                  # generated; ignored by Git
```

Do not depend on an archived `.git` directory, editable installations, an IDE, an API key or files outside the project. README must begin with actual setup/demo commands and expected output. Use `PYTHONPATH=src` for module execution; scripts resolve the project root relative to their own location. The CLI is exactly the one in requirements section 4, including required `--out`, `--budget-bytes`, and mutually exclusive task inputs.

## 3. Bash inventory and Git boundary

### Discovery

Run the trusted `scripts/scan_repo.sh` with the canonical target directory as a quoted positional argument, using a subprocess argument list and no `shell=True`. Bash uses `set -euo pipefail`, explicit handling of optional Git failures, and a private temporary directory cleaned by a trap. Python owns that workspace; no temporary files go in the target repository.

Bash performs a non-symlink-following `find` traversal, prunes excluded directory names and emits only regular `.py` file paths. Exclude directory basenames `.git`, `.venv`, `venv`, `__pycache__`, `.pytest_cache`, `.mypy_cache`, `.ruff_cache`, `node_modules`, `build`, `dist`, `artifacts` and `vendor` at any depth. Do not prune all dot-directories. This fixed list is v1 policy; `.gitignore` parsing and custom exclusion rules are outside MVP. Tests and migrations are eligible source files.

Use NUL-delimited paths throughout. Do not parse `ls`, split on whitespace or store NUL streams in Bash variables. Python validates records, normalises paths to repository-relative POSIX strings and sorts by that string. Spaces and UTF-8 names work. Skip filenames containing control characters (U+0000-U+001F or U+007F) or invalid UTF-8, with diagnostics. Reject path escape; reject symlink files/directories even if discovered during a race. Source roots cannot escape the target or be symlinks.

After exclusions, more than 10,000 eligible `.py` paths is fatal. Skip a file larger than 1 MiB with `file_too_large`. More than 64 MiB cumulative raw bytes read is fatal; do not silently keep only a prefix. Check size before and while reading. The target must be a reasonably stable local repository; these limits are not a sandbox against deliberately hostile inputs.

### Git collection

Bash checks that Git's top-level directory equals the target root; do not attach enclosing-repository metadata to a subdirectory input. Record HEAD if present and collect `git status --porcelain=v1 -z --untracked-files=all` plus staged/unstaged changed paths using NUL-safe Git output. Use `--no-optional-locks`, disable filesystem-monitor integration, and disable external diff/text conversion if using diff commands. Never fetch, checkout, stage, commit, run hooks or execute target tools.

Use structured status results to distinguish `git_unavailable`, `not_git_root`, `git_error` and a valid snapshot. An unborn HEAD is valid with `head: null`. Keep rename source/destination, deletion, staged, unstaged and untracked information. Only present parsed files receive a changed-file feature; a rename destination qualifies. Missing Git disables its feature with a warning; scanner failure does not fall back to Python.

### Internal scanner contract

The scanner writes `files.nul`, `git-status.nul`, `git-staged.nul`, `git-unstaged.nul`, `git-head.txt` and `git-state.txt` into the supplied private workspace. Paths in all path streams are relative to the target. `git-status.nul` retains Git porcelain-v1 NUL rename semantics; `git-staged.nul` and `git-unstaged.nul` contain name-only paths. `git-state.txt` is exactly one enum: `available`, `git_unavailable`, `not_git_root` or `git_error`. Unavailable Git produces empty Git path streams and an empty HEAD file. Scanner exit 0 means discovery completed, not that Git is available; a nonzero exit means fatal discovery failure.

`scan.py` alone converts these bytes into typed records, validates Git path fields and removes excluded/non-Python paths from change features while preserving summary counts. Captured stderr is bounded and sanitised before inclusion in diagnostics. Bash subprocess return codes must be checked through completion.

## 4. AST extraction and static import resolution

Read raw source bytes once, record SHA-256, detect source encoding with `tokenize.detect_encoding`, decode and normalise line endings to LF. Parse using Python 3.12 `ast.parse`; do not import target modules, use `importlib` on them, or execute them. A source encoding/syntax failure is a visible per-file skip. An unreadable source is fatal. Check the per-file 100,000 AST-node cap before downstream traversal; catch parse recursion/resource errors as per-file diagnostics where recoverable.

Record functions, async functions, classes, methods and nested definitions. Use qualified lexical names such as `Cache.get` and `outer.inner`, with AST kind, definition line and decorator-inclusive `start_line`, plus inclusive `end_line` from `end_lineno`. Assignments are not full symbol-definition analysis in v1. Record all static `Import` and `ImportFrom` nodes, including those inside functions, conditional branches and `TYPE_CHECKING`; they indicate possible static dependencies, not proven runtime execution.

### Module identities and roots

Nodes use source paths, such as `src/pkg/cache.py`, rather than module names, so duplicate module names cannot overwrite nodes. Build an import-name lookup from source roots (default `.`). Choose the longest matching supplied root per file; strip it, replace slashes with dots and remove `.py`. Map `pkg/__init__.py` to `pkg`; root `__init__.py` has no inferred package name. Invalid identifier components have no import-name mapping but remain analysable path nodes.

For `src` projects, explicitly pass `--source-root . --source-root src`. These roots change import lookup, not the Bash inventory. Multiple files mapping to the same module name are ambiguous: record the candidates and omit a guessed edge. No `sys.path` execution, installed-package inspection or automatic namespace-package emulation occurs. No synthetic graph nodes are created for missing files/directories.

### Resolution rules

1. `import a.b as alias`: resolve exact local `a.b`. An alias changes recorded binding, not module identity. Do not infer extra parent-package edges.
2. `from a.b import name`: add an edge to exact local `a.b` if uniquely mapped, and to `a.b.name` if that is also a uniquely mapped local module. Deduplicate edges. `name` might be an attribute; absence of a child module is not proof of an error.
3. Relative imports use the current module's package (`__init__.py` represents its own package). One dot keeps that package; each extra dot removes one component. An escape beyond the known package is `relative_out_of_scope`.
4. `from x import *` may establish the local `x` edge, but exported symbols are unknown. Do not expand wildcards or follow re-exports into symbol bindings.
5. Dynamic import calls, `eval`, runtime path changes and string references produce no dependency edge. The analysis states this limitation.

Each import record includes source path/span, written module, imported names/aliases, relative level, resolved targets, and resolution status: `resolved_local`, `partial_local`, `external_or_unresolved`, `ambiguous` or `relative_out_of_scope`. Do not claim a missing local mapping is definitely an installed external package. A syntax-skipped file has no graph node/edge target, but its skip is visible.

Construct `forward[A] = imported local paths` and `reverse[B] = paths importing B`, with sorted unique lists. Cycles are legal; traversal uses visited sets. This is a module import graph, not a symbol reference graph or a runtime call graph.

Small illustrative graph, not a measured project result:

```json
{
  "nodes": ["pkg/api.py", "pkg/cache.py"],
  "edges": [{"from": "pkg/api.py", "to": "pkg/cache.py"}],
  "reverse": {"pkg/api.py": [], "pkg/cache.py": ["pkg/api.py"]}
}
```

## 5. Candidates, ranking and packing

### Candidate boundaries

Partition each parsed file into its top-level function/async-function/class spans, including decorators, and the gaps between those spans. Each definition is one whole candidate; methods/nested functions are metadata within the enclosing candidate, not additional overlapping candidates. Partition gaps (imports, module statements, comments) into consecutive chunks of at most 60 source lines. Omit whitespace-only chunks. These ranges are disjoint; no candidate can duplicate another's source lines.

Large definitions are not split in v1. They may not fit a small budget, which is a reported limitation. Candidate ID is `path:start_line:end_line:kind`; it is stable for the same source snapshot, not across edits. Each carries the original raw-file hash and a hash of its LF-normalised UTF-8 snippet bytes.

### Features and score

Freeze this tokenizer as policy `rank-v1`: extract ASCII identifier-like words using `[A-Za-z_][A-Za-z_0-9]*`, retain the lowercase whole word and lowercase snake/camel-case components, deduplicate, then remove the fixed stop set `{a, an, the, to, of, in, on, for, and, or, is, be, with, please, python, py}`. Use the same function for task, path, snippet text and symbol names. Non-ASCII source is preserved, but v1 keyword retrieval is English/identifier-oriented; it has no multilingual semantic matching.

Let `Q` be the task token set. For candidate `c`:

- `K(c)` is the fraction of `Q` present in tokens of the path plus candidate source text (including comments/docstrings).
- `S(c)` is the fraction of `Q` present in tokens of qualified symbol names whose definition starts inside the candidate.
- Empty `Q` gives `K=S=0`.
- A module's lexical score is the maximum `4K+3S` of its candidates. Select at most five modules with positive lexical score as graph seeds; sort descending score, then path.
- Run multi-source BFS forward from seeds and separately on reverse adjacency, each to depth 2. `d(c)` is the minimum reachable distance for its file in either traversal. Do not mix forward/reverse hops within a traversal. Seed distance is 0. No reachable seed means no distance.
- `D(c)=1/(1+d(c))` if reachable, otherwise 0. Record seed, direction and a shortest path; ties prefer forward, then lexicographic path sequence.
- `G(c)=1` if its current path is changed in available Git metadata, otherwise 0.

```text
score(c) = 4*K(c) + 3*S(c) + 2*D(c) + 0.5*G(c)
```

Example only: `K=1/2`, `S=1/4`, `d=1`, `G=0` gives score `3.75`. Graph distance is relevance evidence, not proof that the snippet is required. A task with no lexical seeds can still select Git-changed candidates; with neither lexical matches nor Git signals, selection is empty. Report this explicitly.

Use rational arithmetic (`fractions.Fraction`) internally for score/density comparisons. Persist exact numerator/denominator pairs plus a six-decimal display score. Raw rank sorts by descending score, then path/start/end/kind. Packing uses a separate density order described below; analysis records both. Do not let rounded display values determine ties.

### Exact-byte budget

`render.py` is the only renderer used by selection, output and baselines. Header contains `# ContextKit context`, a Task heading and task text as an indented block, followed by a blank line. No selection-dependent summary appears in the header. Each snippet block includes an escaped path heading with original `start-end`, a Python fence, source and fixed separators. Choose a backtick fence longer than any backtick run in that snippet (minimum three). Escape Markdown metacharacters in paths. Specify LF everywhere.

Measure UTF-8 bytes of the rendered header and each full block, not source length, characters or estimated tokens. `used = len(header.encode('utf-8')) + sum(block_byte_costs)`. Header bytes are charged once; each candidate's block includes all its own separators, so costs are additive. Context bytes are unaffected by which metadata or diagnostics are written elsewhere.

```text
validate budget > 0 and header_bytes <= budget
remaining = budget - header_bytes
sort positive-score candidates by:
  descending score / block_byte_cost,
  descending score, then path, start_line, end_line, kind
for candidate in that order:
    if block_byte_cost <= remaining:
        select candidate; remaining -= block_byte_cost
    else:
        record over_budget; continue
render selected blocks in selection order
assert len(context_bytes) <= budget
```

Zero-score candidates are rejected as `no_signal`. Do not truncate a function/class or stop at the first oversized candidate. Greedy packing is deterministic and simple; it is not a globally optimal solution and may favour short snippets. For example, 500 header bytes and two blocks of 600 and 400 bytes use exactly 1,500 bytes. This arithmetic is illustrative, not a benchmark.

## 6. Stable v1 data contracts

Put dataclasses/validators in `models.py`; use explicit conversion to JSON primitives. Contracts are proposed for review and stable once accepted. Breaking changes require a new schema version and updated consumers/tests. Public JSON uses `schema_version: 1`; policy identifiers are `scan-v1`, `imports-v1`, `rank-v1`, `render-v1`. Use actual release version for `tool_version`, never a fabricated implementation version in this draft.

All paths are relative POSIX strings; spans are 1-based inclusive integers. SHA-256 values are 64 lowercase hexadecimal characters. JSON arrays use the documented semantic order, object keys are sorted on serialization, UTF-8 is emitted without ASCII-only escaping, indentation is two spaces and files end with LF. Scores use `{numerator, denominator}` integer objects (positive denominator) and display numbers. Missing optional facts use `null`, not invented values.

| Record | Required fields and invariant |
| --- | --- |
| `SourceFile` (internal) | `path`, `module_name` (nullable), `raw_sha256`, `encoding`, `line_count`, decoded LF text, extracted symbols/imports. Only successfully parsed files become graph nodes. |
| `Symbol` | `qualified_name`, `kind`, `definition_line`, `start_line`, `end_line`; decorators included in start and all spans inside file. |
| `Candidate` | `id`, `path`, `kind`, `start_line`, `end_line`, `file_sha256`, `snippet_sha256`, features, score, `block_bytes`; no overlapping spans in one file. |
| `Diagnostic` | `code`, `severity` (`warning` or `info`), `path` (nullable), bounded `detail`; sort by code/path/detail. No machine-specific absolute path, raw source or task text. |

### manifest.json

Top-level required fields:

- `schema_version`, `tool_version`, `policies` (the four policy IDs).
- `input`: `task_sha256` (UTF-8 task text as supplied, without stripping its contents), `source_roots` (normalised sorted list), `files` (sorted path/raw-SHA pairs for every file read, including parse skips), `inventory_sha256` (hash of sorted inventory paths joined with NUL), and `git` (`state`, nullable `head`, sorted change records).
- `budget`: `unit: "utf8_bytes"`, `limit`, `header_bytes`, `used`.
- `context_sha256`: hash of the exact final context bytes.
- `selected`: ordered `Candidate` summaries with span/hash, feature values, exact score/display score and block byte cost. Do not duplicate source text.
- `warnings`: sorted diagnostic codes, deduplicated; detailed records are in analysis.

`selected` order must match context blocks; `budget.used` must equal the actual file size and header plus selected block costs. File-level hashes describe original bytes; snippet hashes describe decoded LF text, so they need not match raw source byte slices with CRLF or non-UTF-8 encodings.

### analysis.json

Top-level fields: `schema_version`, `counts`, `diagnostics`, `imports`, `graph`, `seeds`, `candidates`, `empty_reason`. `counts` includes discovered, read, parsed, skipped, graph nodes/edges, candidate and selected counts. `graph` contains sorted `nodes`, `edges`, `forward`, `reverse`. Import records follow section 4. Each candidate records ID/span, raw rank, density order (nullable for no-signal candidates), K/S/D/G values, exact score/display score, seed/direction/distance/path or nulls, byte cost and outcome (`selected`, `over_budget` or `no_signal`).

`empty_reason` is null for a nonempty selection; otherwise one of `no_python_files`, `no_parsed_files`, `no_candidates`, `no_positive_scores` or `all_over_budget`, choosing the first applicable stage. Timings, timestamps, output directory and absolute repository location do not belong in canonical JSON. Diagnostic detail must avoid embedding OS paths/locale-specific exception strings that break repeatability.

## 7. Publication, reproducibility and logging

Validate that `--out` is a new path outside the canonical target, with no conflicting symlink. Write all three outputs into a private sibling staging directory. Reload and validate JSON, spans, hashes and byte totals. Re-run inventory/Git collection and rehash all previously read files before publication; any inventory, source or Git snapshot change fails with `snapshot_changed`. Recheck skipped-large-file sizes as well. This detects ordinary concurrent edits; it does not promise a transactional filesystem snapshot against adversarial races.

Only then rename the complete staging directory to the requested output path. If another process created it meanwhile, fail without overwriting. On failure remove only this run's staging files; never clean the user's existing output. A crash may leave a private staging directory, but must not publish a knowingly incomplete success result. No persistent index/cache in MVP.

Identical snapshots, policies, runtime, task, roots and budget must produce byte-identical three files across output locations. Pin the Python minor series to 3.12; record the actual patch version and tool versions in verification artifacts. No third-party runtime/test dependencies are planned; `requirements.txt` explicitly documents that fact. Future dependencies require an exact pin and justification in the proposal/report.

Logging is auxiliary: stdout gives a concise success/empty summary; stderr gives bounded diagnostics and error codes. Do not log raw task/source, secrets or fabricated metrics. Stage timing records are produced only for profiling under `artifacts/profile/`, with environment details; they are not part of deterministic production output.

## 8. Errors and observable behavior

| Situation | Behavior / CLI exit |
| --- | --- |
| Bad arguments/runtime, missing Bash/`find`, invalid roots/task, existing output, budget cannot fit header | Explain correction; exit 2 before final output. |
| Git unavailable/non-Git root/Git command failure | Diagnostic and neutral Git feature; continue. Valid unborn HEAD is null. |
| Syntax/decode/per-file AST cap/oversized file | Per-file diagnostic and skip; continue. |
| No files/matches/fitting candidates | Publish valid empty context and analysis reason; exit 0 when header fits. |
| Required scan failure, unreadable source, aggregate cap, changed snapshot | Exit 3; no final output. |
| Cannot create/validate/publish final output | Exit 4; no overwrite or reported success. |
| Unexpected internal defect | Exit 1 with concise report; no fabricated successful output. |

Fatal validation errors in generated artifacts use exit 4; source invariant failures detected before writing use exit 3. `--help`/`--version` do not scan. Task/source text is always data, including strings that look like shell commands or agent instructions. Target tests and imports are never executed by ContextKit.

## 9. Automated tests, baselines and profiling

### Automated Testing as a selected FIT2109 technique

`scripts/test.sh` activates/uses the documented venv, sets source path, runs `unittest` discovery and propagates its final exit status. All test inputs are local. Integration tests create temporary Git repositories locally without network access; use disposable fixture author configuration only inside those repositories, not the user's global Git identity.

Cover FR01-FR12 with purpose-based checks:

- Scanner exclusions, spaces/Unicode, unsupported names, symlinks, aggregate limits, Git renames/deletions/unborn HEAD and scanner failure.
- AST decorator spans, async/nested/class definitions, encoding and syntax failures; human-checked absolute/relative/ambiguous/wildcard import fixtures and cyclic graphs.
- A small hand-calculated K/S/distance/score example, BFS direction/ties, no seeds and density ordering.
- Whole-context exact fit, one-byte-short and multibyte Unicode budgets; oversized-symbol skipping and non-overlapping excerpts.
- End-to-end subprocess runs checking all three artifacts against source ranges/hashes, same-snapshot determinism, target side-effect sentinels, output refusal and simulated snapshot/publish failures.
- Evaluation metric arithmetic on tiny known sets and offline demo/test execution. Do not merely assert that a function returns its own computed value.

`scripts/demo.sh` copies the committed scenario into a private project-local workspace, creates any needed Git state there using deterministic fixture content, runs production selection and the two baselines, verifies outputs and prints the results. A source archive has no need to preserve nested `.git` data. Never modify the committed scenario while preparing the demo.

### Reference cases and evaluation

Use at least 12 cases: 8 development, 4 held out, covering symbol localisation, dependency context and reverse-dependency/impact context. All three families occur in held-out cases. Store source files/tasks and human-labelled required file/line ranges, rationale, case family and split. Keep actual label author/checker and review evidence; pending labels remain explicitly pending and cannot support a claimed evaluation result.

Illustrative reference structure (the ranges and IDs below are examples, not approved answers):

```json
{
  "schema_version": 1,
  "cases": [{
    "id": "T01",
    "task_file": "tasks/T01.txt",
    "family": "dependency_context",
    "split": "development",
    "required": [{"path": "pkg/cache.py", "start_line": 10, "end_line": 18}],
    "rationale": "The example change requires the cache validation logic.",
    "review": {"status": "pending", "author": null, "checker": null}
  }]
}
```

Use the identical scanned snapshot, candidates, renderer and budgets (2,000/4,000/8,000 bytes) for all methods. B0 assigns every candidate a constant positive score and packs in path/start/end/kind order. B1 uses `4K+3S` only and the same positive-score density packing as production. ContextKit uses the full score. All methods skip blocks that do not fit and continue. B0 is intentionally simple; B1 tests the extra value of graph/Git features. Do not require a tuned competitor, model integration or extra ablation framework for MVP.

Treat required line ranges as unions of `(path, line)` pairs, merging overlap before counting. Selected ranges use the same representation. Report:

```text
file recall = required files selected / required files
line recall = selected required lines / required lines
line precision = selected required lines / all selected source lines
byte utilisation = actual context bytes / byte budget
```

Primary cases must have nonempty reference sets. If selection is empty with a nonempty reference, recall and precision are 0. Empty-reference boundary cases are excluded from aggregate retrieval metrics and reported separately. Count only original source lines for precision, not Markdown/task lines. Compute macro means over cases and report per-case values, not just an aggregate.

Save policy/input/label hashes, selected IDs, budgets, raw per-case metrics and environment separately under `artifacts/evaluation/`. Proposed target: mean held-out line recall above B0 at 8,000 bytes; report B1, precision, runtime and all failures too. Freeze labels/split/policy before the held-out run; any later tuning requires a disclosed new evaluation round. Retrieval relevance does not establish LLM task success. Do not claim improvement until observed.

### Profiling as a selected FIT2109 technique

`scripts/profile.sh` generates a deterministic 500-file/2-MiB workload locally, records its construction parameters/hash, runs five unprofiled end-to-end repetitions, then captures Python `cProfile`/`pstats`, stage wall times and `tracemalloc` peak memory. Stage wall time around Bash subprocesses includes scanning/Git; cProfile does not describe Bash's internal CPU cost. Record Python patch/Bash/Git/OS, machine/load context, all repetitions and median. Python standard-library harnesses may measure subprocess wall time; no extra package manager is needed.

The proposed <=5-second median is unverified and awaits team review. Inspect the measured bottleneck, propose a small optimization only if justified, and compare before/after on identical inputs with correctness tests. Alternatively record why no optimization is warranted. Keep actual results and residual limits; do not optimise speculatively or cite profiler overhead as normal runtime.

## 10. Implementation order and human checkpoints

1. **Agree the contract:** review D01-D08; validate the workflow pain point and existing approach; assign genuine owners/checkers; decide how to address CC07. Team members write the AI delegation decisions themselves.
2. **Build the scan boundary:** implement Bash discovery/Git, typed records, exclusions, limits and scanner tests. Demonstrate that Bash output is required by the pipeline.
3. **Extract static facts:** implement encoding/hash/AST spans, source-root mapping, imports and reverse graph. Review hand-labelled examples before ranking.
4. **Complete the smallest vertical slice:** candidate ranges, fixed scoring, byte renderer/packing, three outputs and CLI. Validate a tiny end-to-end case, empty output and error exits.
5. **Add scenario/evidence:** human-review labels and splits; implement B0/B1 and metrics; add offline demo/test scripts and complete boundary tests. Freeze evaluation policy before held-out runs.
6. **Measure and reproduce:** profile, investigate the actual bottleneck, run the documented clean extraction on the unit server, and record observed results. Prepare README, scenario documentation, AI log, proposal/report/video separately.

Keep each step in a focused branch/MR with independent member review. Suggested ownership areas are scan/environment, AST/graph, and ranking/output/evaluation, with rotated checking; names and assignments remain the team's decision. All members need real traceable work and must understand the integrated flow. AI assistance is recorded when it actually occurs, not when this plan mentions it. Design approval, reference labels, interpretation of results and final submission responsibility stay with humans.

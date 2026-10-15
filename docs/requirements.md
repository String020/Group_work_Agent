# ContextKit requirements

Version: proposed contract v1, 15 October 2026. Status: ready for team review; implementation and performance are not verified by this document.

## 1. Purpose and authority

ContextKit helps a developer choose useful Python source context before handing a development task to a coding agent. Given a local Python repository, a task and a context budget, it produces `context.md`, `manifest.json` and `analysis.json`. It selects and explains source material; it does not solve the task or measure a model's ability to solve it.

The agreed product direction is an offline Python 3.12 + Bash CLI with AST extraction, module dependencies and reverse dependencies, explainable ranking, budgeted snippets, automated tests and evaluation. Specific v1 interfaces and algorithms below are proposed team choices, not requirements imposed by FIT2109. See [architecture](architecture.md) for D01-D08 and [agent rules](../AGENTS.md) for the development workflow.

### Source register

The following supplied files were read locally. Page references are PDF page numbers, not website updates. Their contents are the course evidence for this draft; no claim is made about later staff clarifications.

| Source | Locator | What it establishes |
| --- | --- | --- |
| S1 `FIT2109_GroupProject_Specs.pdf` | pp. 1-3 | Tool scope, substantive Bash, two other techniques, Python/server setup, demo/tests, offline scenario and repository layout |
| S1 | pp. 4-5 | AI plan/log/reflection, proposal/final deliverables, GitLab and integrity |
| S2 `FIT2109_GroupProject_Rubric.pdf` | p. 1, C1-C5; p. 2 notes | Assessment evidence, reproducibility, independent AI checking and repository practices |
| S3 `FIT2109_Proposal_Template.docx` | sections 1-6 and Appendix A-D | Proposal structure, baseline, dependency justification, AI plan and acknowledgement |

Use these status labels throughout:

- **Course-confirmed (CC):** directly supported by S1/S2/S3. A disagreement between sources remains visible.
- **Product-agreed (PA):** specified in the team's current ContextKit brief; not claimed to be a course mandate.
- **Team-proposed (TP):** a concrete implementation or acceptance choice awaiting team review. It becomes binding on implementation when the team accepts the relevant design decision.

## 2. Course constraints and required evidence

| ID | Course-confirmed constraint | Evidence to retain |
| --- | --- | --- |
| CC01 | A team of three builds a focused tool for a real developer workflow, with defined inputs, useful output, team-designed logic, an end-to-end scenario and improvement over a simple baseline. S1 p. 1; S2 C1-C3. | Validated pain point, scenario cases, baseline procedure and measured comparison. ContextKit's pain point still needs team evidence. |
| CC02 | Bash performs meaningful core work; a launcher/setup script alone does not count. At least two other FIT2109 techniques have clear roles. S1 pp. 1-2; S2 C2. | Scanner/Git implementation and tests; Automated Testing and Profiling artifacts. These two technique choices are PA, while the minimum of two is CC. |
| CC03 | Team logic uses Bash and Python 3; the server supplies Python 3.12. Other tool languages require prior written staff approval on Ed and an approval link in README. S1 p. 2. | Python 3.12 runtime check; no additional language in this MVP. |
| CC04 | Choose venv or Docker. Non-Docker uses Python 3.12 venv, exactly pinned dependencies in `requirements.txt`, and no other package manager. Justify reused third-party packages and verify boundary/error behavior. S1 p. 2. | Proposed venv route, standard-library inventory, dependency policy and clean-server reproduction receipt. Docker is not selected. If adopted later, the course's uncached build limit of 3 minutes applies. |
| CC05 | README starts with setup/demo commands and expected output. Supply `scripts/demo.sh`, `scripts/test.sh`, `src/`, `tests/`, `scenario/`, `.gitignore`, `docs/scenario.md` and `docs/ai-use.md`. S1 pp. 2-4. | Committed runnable scripts, all scenario inputs, instructions and logs. The three files in this delivery do not replace those future deliverables. |
| CC06 | Scenario runs offline apart from dependency installation, without manual fixes. Do not rely on fixed ports, privileges or files outside the project. Test a clean extraction on the unit server before submission. S1 p. 3. | Bundled fixtures, tasks and labels; actual clean-extraction commands, versions and results. A local run does not establish server compatibility. |
| CC07 | **Source discrepancy:** S1 p. 4 requires at least two planned steps and two real AI activities across different stages; S2 C4 requires at least three real activities at different stages; S3 Appendix A asks for at least three stages. S2 C1 still says two planned steps. | Proposed conservative plan: at least three genuine activities across three distinct stages, with the six planning questions for each. Record the discrepancy and any later staff clarification. This is a team response, not a claim that the sources agree. |
| CC08 | Every member participates in the human-owned AI plan, owns an AI-assisted step or gives a reasoned opt-out, and has a checking role. Each activity is checked by a member other than its owner(s). Log date, task, owner, checker, tool, instruction, output, checks, decision and residual risk. Reflect on plan versus practice. Proposal/report writing assistance must be acknowledged but does not count as a qualifying activity. S1 p. 4; S2 C4; S3 Appendix A/D. | `docs/ai-use.md`, actual prompts/output summaries, verification references and independent human decisions. No fabricated acceptance/review. |
| CC09 | Use group Monash GitLab throughout with regular genuine commits, branches and MRs under each member's own Monash credentials/email. MRs are reviewed before merge; main remains working. S1 p. 5; S2 C5 and p. 2. | Traceable contributions, MR review and FeedbackFruits completion. Commit quantity alone is not the criterion. |
| CC10 | Core logic must be original to this project; do not reuse FIT2109 A1/A2 or other assessed-work code. Do not submit secrets, personal/production data, destructive operations or production changes. S1 p. 5. | Source attribution, safe synthetic/sanitised fixtures and review of archive contents. |
| CC11 | Proposal: up to 4 A4 main pages plus 4 appendix pages; final report: up to 10 plus 10. Both use 12 pt Times New Roman, single spacing and templates. Final includes repository ZIP, report PDF and video of at most 5 minutes, with every member visible and explaining their contribution; no AI-generated/altered footage or voices. S1 pp. 4-5; S3. | Separately prepared submissions; these Markdown engineering documents are not the proposal/report. Source deadlines are Friday Week 11 and Friday Week 14; verify actual submission times separately. |
| CC12 | Existing-tool/current-workflow evidence and a baseline measure support the proposal. The rubric reserves HD for a meaningful team-discovered scenario; use of the provided sample can earn up to D. S2 C1 and p. 2; S3 sections 1-5. | Team's own scenario and validation evidence; a cited representative mainstream approach. Do not invent research, users, interviews or mark predictions. |

## 3. MVP scope and exclusions

**PA in scope:** substantive Bash repository inventory and Git change collection; static Python AST imports, symbols and inclusive line spans; directed module graph and reverse adjacency; explainable task keywords, symbols and dependency distance; bounded selection of exact source excerpts; three outputs; offline demo, tests, profiling and baseline evaluation.

**PA outside MVP:** real LLM integration/API calls, automatic code editing, task execution or repair, embeddings, vector databases, semantic-search models, non-Python source analysis, dynamic imports/call graphs, runtime tracing, IDE/web interfaces, plugins, persistent caches and optimal-knapsack claims. Reading Git metadata does not require connecting to GitLab. GitLab is the team's development platform.

**TP supported environment:** Linux unit server, Bash 4.4+, Python exactly in the 3.12 minor series, Git and `find`. Git absence or non-Git input degrades gracefully; Bash, Python and `find` are required. Windows Git Bash is a convenience target requiring separate checks, not a promised replacement for unit-server verification. D01-D03 cover these choices.

## 4. Public interface and outputs (TP, D02/D03/D06)

After activating the Python 3.12 venv, run from the ContextKit project root with its source directory on `PYTHONPATH`:

```bash
PYTHONPATH=src python3.12 -m contextkit select \
  --repo scenario/repo \
  --task-file scenario/tasks/T01.txt \
  --budget-bytes 8000 \
  --out artifacts/T01
```

| Argument | Contract |
| --- | --- |
| `select` | The MVP production subcommand. |
| `--repo PATH` | Required existing readable directory; source repository remains read-only. |
| `--task TEXT` or `--task-file PATH` | Exactly one; nonempty after whitespace stripping. Task file is UTF-8; use its full text as data. Maximum 65,536 UTF-8 bytes. |
| `--budget-bytes N` | Required positive integer. Maximum size of the **entire UTF-8 `context.md`**, including task, headings, fences and separators. It is not a token limit. |
| `--out PATH` | Required new output directory outside the target repository; an existing path, including symlink, is rejected. No overwrite flag in v1. |
| `--source-root RELPATH` | Optional repeatable existing directories inside `--repo`; default `.`. Used for import mapping, not to bypass scanning. For a `src` layout use `--source-root . --source-root src`; longest matching root wins per file. |
| `--help`, `--version` | Explain flags, budget unit and contract/tool version without scanning. |

Exit codes: `0` = completed output, possibly with reported warnings/empty selection; `2` = invalid arguments, unsupported runtime, missing required tools or task/header cannot fit; `3` = fatal scan/read/resource/snapshot failure; `4` = output publication failure; `1` = unexpected internal error, reported without pretending success. Fatal errors leave no final output directory. Detailed behavior is in architecture section 8.

Successful output contains exactly these public files:

| File | Required contents |
| --- | --- |
| `context.md` | Task and selected excerpts in selection order, each labelled with repository-relative path and original inclusive line range. Rendered code preserves decoded source lines with LF endings. |
| `manifest.json` | Schema version, tool/policy versions, task/input fingerprints, Git availability/HEAD, budget limit/unit/used bytes, ordered selection IDs/ranges/hashes/scores/costs and warning codes. No absolute machine paths or timestamps. |
| `analysis.json` | Scan/parse counts, skips and diagnostics; nodes/edges/reverse adjacency; import classifications; all candidate scores, seeds, distance paths/reasons, selection outcomes and rejection reasons. No source text duplication or timing fields. |

stdout reports status, selected count and used/allowed bytes, plus output path; stderr contains concise warning/error messages. Raw task/source text is not logged. JSON is UTF-8 with deterministic ordering. Profiling and evaluation artifacts live separately under `artifacts/`, not inside the three-file production output.

## 5. Functional requirements

All rows implement PA scope through TP v1 contracts. The verification column names tests/evidence to implement; it does not record tests already passed.

| ID | Required behavior | Acceptance check |
| --- | --- | --- |
| FR01 | Validate runtime, arguments, repository, roots, task, output location and minimum renderable context before scanning. | Valid text/file forms work; mutually exclusive/missing/invalid inputs exit 2 with actionable error and no final output. |
| FR02 | Bash owns `.py` discovery, fixed exclusion pruning and Git collection; Python consumes its NUL records. No Python filesystem-discovery fallback. | Scanner fixture with spaces, Unicode, hidden directories, exclusions and symlinks; integration proves scanner records determine indexed paths. Bash failure is fatal. |
| FR03 | Collect HEAD and staged/unstaged/untracked change status through Bash without modifying target Git state. Non-Git/Git-unavailable cases emit diagnostics and neutral Git features. | Temporary Git fixture including unborn HEAD, rename, deletion and staged/unstaged change; non-Git directory still generates outputs. |
| FR04 | Decode source using Python source encoding rules, parse with Python 3.12 `ast`, and extract imports plus functions, async functions, classes and nested/method qualified names with decorator-inclusive ranges. | Human-checked fixtures assert names, import locations and exact 1-based inclusive spans. Syntax/encoding errors are visible skips. |
| FR05 | Build module edges `importer -> imported local module` and derived reverse adjacency, retaining unresolved/ambiguous imports as records rather than invented edges. | Absolute/relative imports, packages, `src` roots, cycles, missing modules, duplicate module names and conditional/function-local imports have expected records. |
| FR06 | Construct non-overlapping snippet candidates and explain keyword, symbol, graph and Git features using architecture D04/D05. | Exact candidate ranges, frozen score examples, seed selection, forward/reverse BFS distances, tie-breaks and no-seed behavior are asserted. |
| FR07 | Greedily select positive-score snippets by score per exact rendered byte, skip non-fitting snippets and continue; never cut a symbol to fill remaining space. | Exact-fit/one-byte-short cases and large-symbol skip; final `context.md` byte size never exceeds limit. No claim of optimal selection. |
| FR08 | Generate mutually consistent outputs with original source hashes and deterministic schema v1 fields. Publish only after all files validate and indexed inputs are rechecked. | Reload JSON, match excerpts/ranges/hashes and budget totals; identical snapshots produce byte-identical three files. Changed input fails instead of publishing stale ranges. |
| FR09 | Report partial analysis, omissions, missing matches and graph limitations clearly. Empty repository/no positive scores/all snippets too large are successful empty selections when the header fits. | Expected empty context plus reason codes; skipped files never appear as parsed selections. |
| FR10 | `scripts/demo.sh` runs the bundled realistic scenario, production selection and baseline comparison offline; `scripts/test.sh` runs the tool's automated tests and propagates failures. | Both run from a clean extraction without editing paths or using `.git` from the submission archive. Demo creates any Git fixture locally. |
| FR11 | Evaluate against human-labelled reference files/line ranges and two frozen baselines at the same budgets using the same renderer and candidate pool. | Per-case and aggregate metrics, input/policy hashes and raw results can be regenerated. No labels inferred from ContextKit rankings. |
| FR12 | Profiling measures end-to-end and stage costs and supports an evidence-based performance investigation. | `scripts/profile.sh` saves cProfile results, stage timings, workload description, runtime/environment and repeated unprofiled wall times; report identifies an observed bottleneck and an optimization/retest or justified decision not to optimize. |

## 6. Non-functional requirements

| ID | Requirement and status | Verification |
| --- | --- | --- |
| NFR01 | PA offline execution: no online API, model download, telemetry or network dependency during selection, demo, tests, evaluation or profiling. TP stdlib-only v1 also allows setup without third-party downloads. | Network-disabled rerun with all scenario inputs bundled. |
| NFR02 | TP deterministic outputs for identical tool/policy/runtime, task, source bytes, roots, Git snapshot and budget; no absolute paths/time/timings in canonical outputs. | Repeat runs in different extraction locations and output directories compare all three files. Performance artifacts are explicitly non-deterministic. |
| NFR03 | TP read-only target and bounded traversal: no execution/import of target modules, shell interpolation of inputs, symlink traversal or path escape. | Sentinel target side effects remain absent; traversal/quoting/symlink fixtures pass. |
| NFR04 | TP limits: at most 10,000 discovered eligible `.py` paths, at most 1 MiB per source file, at most 64 MiB total source bytes read and at most 100,000 AST nodes in any one file. Too-large single files are skipped; aggregate limits fail explicitly. | Boundary tests and diagnostics; do not silently truncate inventory or graph. These are engineering limits, not a security sandbox for hostile code. |
| NFR05 | TP reproducible environment: require Python 3.12; record actual patch/Bash/Git/OS in verification artifacts. Use venv and pip only; v1 uses standard library, including `unittest`, `cProfile` and `tracemalloc`. Keep a comment-only `requirements.txt` stating no third-party dependencies. Any later package must use an exact version pin and justification. | Clean-server setup follows README; script fails clearly under unsupported Python. No invented Python patch compatibility claim. |
| NFR06 | TP maintainability: separate scanner, parsing, graph, ranking, budgeting, output and evaluation; contract changes require review. | Focused modules, tests and MR review against D01-D08. |
| NFR07 | TP resource target pending measurement: on a documented 500-file/2-MiB fixture, target median unprofiled end-to-end time <= 5 seconds over 5 runs on the unit server. | Record actual machine/load and all runs. This is an unverified proposed target, not a measured guarantee or course time limit. Revise only through documented team review. |
| NFR08 | PA honest evidence: context relevance metrics are proxies for selection quality; no claim of higher LLM task success, universal relevance or complete dependency resolution. | README/report distinguish measured retrieval results, limitations and unrun checks. |

## 7. Evaluation and release acceptance (TP, D07)

Use at least 12 human-reviewed task cases spanning symbol localisation, dependency context and reverse-dependency/impact context, plus separate error/boundary fixtures. A task is a realistic developer request with a rationale for each reference file and required inclusive line range; a reference answer is not a generated code solution. Use 8 development cases and 4 held-out evaluation cases, with all three families represented in the held-out set. Freeze labels, split, ranking policy and budgets before the held-out run. Keep author/checker identities and review evidence; do not fabricate them.

Baselines: B0 takes candidates in path/start-line order; B1 uses keyword + symbol ranking without graph/Git features. Both use the same candidates, byte accounting, whole-snippet rule and skip-and-continue behavior as ContextKit. Evaluate budgets of 2,000, 4,000 and 8,000 bytes; primary comparison is at 8,000. Report file recall, line recall, line precision, byte utilisation, runtime, and per-case omissions. Definitions and edge cases are in architecture section 9.

The proposed improvement target is higher mean held-out line recall than B0 at the primary budget, while also reporting B1 and precision/runtime tradeoffs. It is a target to test, not a promised result. Do not require every case to improve. If it is not met, report the failure, investigate on development cases and have the team decide whether to revise the approach; never relabel answers or tune silently on held-out cases. The course's improvement requirement remains outstanding until supported by real evidence.

Release checks:

1. Team reviews D01-D08, acceptance targets, scenario validity and the AI plan; records actual owners/checkers and decisions.
2. FR01-FR09 invariants pass meaningful automated unit and end-to-end tests, including failure cases. Every selection matches a source range and the full context stays within budget.
3. Demo, tests, baselines and profiling run offline; every required artifact is available. Claimed measurements come from completed runs.
4. Clean ZIP extraction on the unit server follows README without manual fixes or external local files; missing `.git` is handled. Record exact environment and results.
5. CC01-CC12 evidence and other submission artifacts are reviewed separately. Passing tool tests alone does not establish course submission readiness.

## 8. Edge cases and expected outcomes

| Case | v1 response |
| --- | --- |
| No Python files, no positive scores or no candidate fits | Successful three-file output with task/header and explicit empty-selection reason, if header fits. |
| Budget smaller than rendered header/task | Exit 2, report required minimum; no partial outputs. |
| Syntax error, unsupported source encoding, file >1 MiB or per-file AST cap exceeded | Skip that file, record code/path/detail, continue; no AST/graph node for it. |
| Unreadable source, failed required traversal, aggregate resource cap exceeded or source changed during run | Exit 3; do not silently claim complete analysis. |
| File path has spaces/Unicode | Supported with NUL-safe transport and repository-relative JSON paths. |
| Symlinks or paths with control characters/non-UTF-8 names | Symlinks are not followed/selected; unsupported names are skipped with warnings. Paths escaping the root are rejected. |
| Missing Git, non-Git root or enclosing Git repository with a different root | Disable Git features with diagnostic; still scan Python files. |
| Unborn Git HEAD | Record null HEAD; collect available status, no history assumed. |
| Deleted/renamed/untracked file | Status retained; only existing scanned/parsed paths can be selected. Rename destination gets changed-file feature. |
| Cyclic imports or multiple possible module targets | Visited-set traversal; cycles allowed. Ambiguous imports have no guessed edge. |
| Dynamic import, wildcard import, re-export or conditional import | Record the supported static facts and uncertainty as architecture section 4 specifies; never claim a runtime call graph. |
| Large class/function cannot fit | Skip whole candidate with `over_budget`; continue to other candidates. Method extraction does not imply independent method snippets in v1. |
| Existing/unwritable output directory or publish error | Existing path exits 2; publication error exits 4; never overwrite a previous run. |

## 9. Team review still required

Approve or amend D01-D08 in a reviewed MR before implementation depends on them. Assign real responsibilities and an independent checker for each AI activity. Validate the pain point using the team's own evidence, select the scenario, freeze reference answers and confirm the AI source discrepancy. None of those approvals, research results, tests or measurements are asserted by this draft.

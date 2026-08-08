# HANDOVER: verification-and-validation pipeline work

This file is a working handover for a fresh agent session continuing the verification-and-validation work on branch `claude/verification-validation-coding-36ikp1`.
It is a branch-scoped working artifact, not product documentation: delete this file and its `docs/documentation-audiences.json` inventory entry before this branch merges to `main`.

## How to start a fresh session

1. Check out the branch: `git fetch origin claude/verification-validation-coding-36ikp1 && git checkout claude/verification-validation-coding-36ikp1`.
2. Read `AGENTS.md` in full - it is the always-loaded operating contract for this repo.
3. Load the `firstmate-coding-guidelines` skill before editing any tracked material - it owns knowledge placement, the one-owner rule, and the style rules that every change below must follow.
4. Read `docs/code-verification.md` - it is the deliverable already landed and the conceptual base for every remaining task.
5. Read this file top to bottom, then execute the remaining tasks in order.

## Context: what this work is about

The captain's goal is trustworthy verification and validation of agent-built code: when a spec is created and coding completed, verification should approach 100% sensitivity (real defects get caught) and specificity (a pass verdict is genuinely a pass), and the captain must never be lied to about outcomes.

The framework, stated fully in `docs/code-verification.md`, in one paragraph:
the problem splits into reporting integrity (is the completion claim true - fully solvable by accepting only machine-checked artifacts, never agent narrative) and verification quality (do the checks catch defects against the spec - measurable via mutation testing but never 100%).
Evidence ranks from deterministic execution outside the agent, to independent adversarial review with fresh context, to same-agent re-check, to worthless self-report.
Sensitivity is raised by executable acceptance criteria, test authorship independent of implementation, mutation testing, property-based tests, and real-application smoke runs.

The captain explicitly rejected ceremonial or infinite layering.
A verification layer earns its place only if the implementer cannot influence its evidence, its blind spots are uncorrelated with existing layers, and its failure actually blocks something.
Layers failing that test (self-review prompts, agent confidence scores, unenforced checklists, coverage-percentage targets, second reviews that read the first review) are negative value and must not be built.

## State already landed on the branch

Commit `b6194ea` (plus this handover commit) contains:

- `docs/code-verification.md` - new operator reference, "Verifying agent-built work".
- `docs/documentation-audiences.json` - the page registered as `operator-current` (and this handover registered as `agent-runtime`).
- `README.md` - one line added to the Documentation index.

`bin/fm-doc-audience-check.sh` passes on this state.
No scripts, tests, hooks, or skills have been changed yet.

## Repo conventions every remaining task must follow

- One full sentence per line in tracked Markdown; never wrap multiple sentences onto one physical line; plain dash `-`, never an em dash.
- Never add an agent name as a commit co-author.
- Every tracked Markdown file must be registered in `docs/documentation-audiences.json` (surfaces are ordered case-sensitively by path) and `bin/fm-doc-audience-check.sh` must pass; the check only sees git-tracked files, so `git add` before running it.
- `bin/*.sh` must pass shellcheck; run `bin/fm-lint.sh` before treating any script change as done.
- Colocate tests in `tests/` named `<subject>.test.sh`, extending the existing runner pattern; tests must exercise behavior through an executable or public interface, never assert implementation-source bytes.
- Contracts are stated in full exactly once; every other mention is a one-line cross-reference (grep before writing).
- Push with `git push -u origin claude/verification-validation-coding-36ikp1`; never push to another branch; do not open a PR unless the captain asks.

## Remaining tasks, in order

### Task 1 - add a "How this pipeline fails" section to `docs/code-verification.md`

The captain asked how the pipeline breaks and approved capturing the answer.
Append a section covering the eight failure modes, each in two or three sentences with its defense:

1. The implementer can touch the gate (Goodhart on tests, CI config, and hooks) - defended by Task 2.
2. Verifying the wrong artifact (green bound to a stale commit instead of the exact head SHA).
3. Flaky tests destroying both sensitivity and specificity via rerun-until-green and alarm fatigue.
4. Correlated blind spots between "independent" agents sharing a base model, plus reviewer sycophancy, large-diff sensitivity collapse, and prompt injection.
5. Evidence theater - narrative reports can fabricate output; only harness-executed or forge-fetched output is beyond fabrication.
6. Human-layer decay - automation complacency, gates disabled under pressure and never re-enabled, verification debt compounding from one weakened test.
7. The spec is the ceiling - conformance to the written spec, not intent; non-functional defect classes (performance, security, races, real-data behavior) need dedicated checks.
8. Economics - the pipeline actually run under deadline pressure is the real pipeline; incidents come from changes judged too small to verify.

Close the section with the three-part real-vs-ceremony test from the context section above and the delete-list of ceremonial layers.
Acceptance: section reads in the existing document voice, follows the style rules, adds no restated contracts, and `bin/fm-doc-audience-check.sh` still passes.

### Task 2 - gate-lock: keep implementer crewmates out of the verification surface

Goal: make it structurally impossible, not merely instructed, for a ship-task crewmate to edit the files that define "pass" - the project's test files, CI workflow config, and hook config - closing failure mode 1.

Design constraints:

- Study the existing PreToolUse guard family first and follow its pattern exactly: `bin/fm-arm-pretool-check.sh` with `bin/fm-arm-command-policy.mjs`, `bin/fm-cd-pretool-check.sh` with `bin/fm-cd-command-policy.mjs`, and their docs `docs/arm-pretool-check.md`, `docs/cd-guard.md`, `docs/subagent-guard.md`.
  Do not invent a parallel enforcement mechanism.
- Study how those guards are armed into a crewmate's harness hook configuration (search `bin/fm-spawn.sh` and `bin/fm-brief.sh` for the arming path) before deciding where the new guard plugs in.
- The guard must be deny-with-clear-message on writes (Edit, Write, and Bash write operations) targeting the protected paths, and completely inert when not armed.
- Protected-path defaults: `tests/`, `test/`, `.github/workflows/`, and harness hook config files; make the set overridable per task or per project rather than hardcoded-only, since project layouts differ.
- The mechanism must be opt-in at spawn or brief level; changing default behavior for all existing tasks is out of scope without the captain's approval.
- A legitimate test change then flows as its own separately-reviewed change; the guard's denial message should say exactly that.

Deliverables: the guard script plus policy file following existing naming, a colocated `tests/<subject>.test.sh`, a maintainer doc page registered in the audience inventory, and shellcheck plus `bin/fm-lint.sh` green.
Acceptance: with the guard armed, an edit to a protected path is denied with the routing message; unprotected paths are unaffected; unarmed behavior is byte-identical to today; the colocated test proves all three through the executable interface.

### Task 3 - seeded-bug drill: measure pipeline sensitivity instead of assuming it

Goal: an occasional calibration that seeds N plausible single-line bugs into a project one at a time, checks whether the project's test suite catches each, and reports a sensitivity number with the misses - turning "are the tests good?" into a measurement.

Design constraints:

- Hard rule 1 applies: firstmate never writes to a project, so the drill must run as a scout task inside its own disposable worktree, never in the primary checkout and never driven directly by firstmate.
- Implement as a generated brief variant in `bin/fm-brief.sh` (the `--herdr-lab` flag is the precedent for a guarded generated variant) or as an agent skill that scaffolds the scout brief; study both and pick the one that fits the existing scaffold architecture, documenting why.
- The generated drill contract must require: one mutation applied at a time with the full test command run per mutation; every mutation reverted before the next; the worktree left with zero residue; no pushes, no PRs, no branches on the remote, ever.
- The report (`data/<id>/report.md`) must record, per mutation: the exact diff, the exact test command, caught or missed, and the final catch rate; misses are the actionable output.
- Mutation classes to specify in the brief: flipped comparison operators, off-by-one boundary changes, deleted conditional branches, swapped logical operators, and removed error handling - plausible defects, not syntax errors.

Acceptance: scaffolding the drill brief produces the complete contract above with placeholders for project and test command; the safety lines (scout-only, no-push, revert-per-mutation, residue-free) are non-optional parts of the generated text; a colocated test proves the generated brief contains the safety contract via the scaffold's executable interface.

### Task 4 (optional - confirm with the captain before building)

Extend the ship-brief scaffold in `bin/fm-brief.sh` so acceptance criteria and evidence requirements travel with the worker: a placeholder demanding concrete testable acceptance criteria at scaffold time, and a completion-report requirement of exact command plus captured output for every claim.
Check the one-owner rule carefully - the brief scaffold and `docs/code-verification.md` must not restate each other; the brief carries the requirement, the doc carries the rationale, and each may cross-reference the other in one line.

## Verification of the handover work itself

Before ending any session on this branch, run and record:

```sh
git add -A && bin/fm-doc-audience-check.sh
bin/fm-lint.sh
bin/fm-test-run.sh
```

Then review the complete branch diff against `main` (not just the latest commit) for style, one-owner violations, and audience placement, commit with a plain descriptive message and no agent co-author, and push to the designated branch.

## Open decisions for the captain

- Task 2: hook-enforced guard as specified, or a cheaper brief-level constraint first with the hook as follow-up.
- Task 3: brief variant in `fm-brief.sh` versus a scaffolding skill, if the implementer finds the architecture argues for the skill.
- Task 4: whether to build it at all.
- Whether any of this work should merge to `main` via a PR, and when.

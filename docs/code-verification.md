# Verifying agent-built work

This page explains how to structure specs, checks, and review so that a completion claim about agent-built code is trustworthy.
It is operator guidance for working with the crew; the delivery-path contract, merge authority, and the never-merge-a-red-PR boundary remain owned by `AGENTS.md` section 7.

The goal is often stated as "near-100% sensitivity and specificity" - every real defect gets caught, and every pass verdict is genuine.
That goal hides two different problems, and only one of them is fully solvable.

## Two separate problems

**Reporting integrity** - when a worker says "done, tests pass", is that statement true?
**Verification quality** - even when the tests genuinely pass, do those tests actually catch defects against the spec?

Reporting integrity can be driven to effectively 100% by making claims machine-checkable.
Verification quality can never reach 100% - testing shows the presence of defects, never their absence - but it can be measured and pushed high.
Conflating the two leads to distrust of accurate reports and misplaced trust in weak test suites, so treat them separately.

## The evidence hierarchy

An agent's self-report is the weakest form of evidence that exists.
From strongest to weakest:

1. **Deterministic execution outside the agent** - CI status read from the forge API, a test command run by a harness hook, a guarded script's exit code.
   The model cannot talk its way past these.
2. **Independent adversarial verification** - a different agent with fresh context, instructed to refute the claim rather than confirm it.
3. **Same-agent re-check** - better than nothing, but the agent that wrote a defect tends to re-read the code with the same blind spot.
4. **Self-report** - "I ran the tests and they pass" carries almost no weight on its own.

Structure every workflow so the agent's words carry no weight and its artifacts carry all of it.

## Reporting integrity: accept artifacts, not claims

- **Define "done" as a machine fact.**
  A PR is ready when the forge reports green checks on the recorded head commit, not when a worker says checks are green.
  Firstmate applies this stance already: `bin/fm-pr-check.sh` records the PR and head identity and arms a poll against the forge, and `bin/fm-crew-state.sh` reconciles a worker's actual run state instead of trusting its last status line.
- **Gate turn completion on execution, not narrative.**
  Where the harness supports hooks, run the test suite at the turn boundary and block completion while it fails.
  The harness executes hooks deterministically, so "done" becomes structurally impossible while tests are red.
  The `no-mistakes` delivery path plays this role for ship tasks: its pipeline owns review, tests, and CI, and its gates cannot be summarized away.
- **Demand reproducible evidence in reports.**
  A completion report must include the exact command run and its actual output, or a screenshot for visual work, such that the reader could re-run it in one paste.
  "I verified the endpoint works" is not evidence; the command and its captured output are.
- **Use adversarial review with fresh context.**
  When a separate review deliverable is authorized, brief the reviewer to assume the work is broken, hunt for the failure case, and attempt to refute each claim in the implementation report.
  Do not show the reviewer the implementer's self-assessment first, and do not reuse the implementer's session - inherited context inherits its assumptions.

The common failure mode is not malicious lying.
Agents pattern-match toward completion: they run a subset of tests, misread output optimistically, or claim success on the happy path.
Externalized deterministic gates make that optimism irrelevant.

## Verification quality: make the spec executable

- **Turn the spec into acceptance criteria before implementation.**
  Every requirement must be falsifiable - concrete inputs, expected outputs, and observable behavior.
  A requirement like "handle errors gracefully" has undefined sensitivity because nobody can check it.
  A cheap ritual: before coding starts, have the agent restate the spec as a checklist of testable claims and surface every ambiguity as a question.
- **Separate test authorship from implementation.**
  The biggest sensitivity failure is letting the implementing agent write both the code and the tests that judge it - it grades its own homework and tests what it built rather than what was specified.
  Either have a separate agent derive tests from the spec before implementation, or have an independent reviewer audit the tests against the spec alone, without reading the implementation, asking which specified behavior has no test.
- **Measure sensitivity with mutation testing.**
  Inject small plausible defects - a flipped comparison, an off-by-one boundary, a deleted branch - and confirm the suite fails on each.
  Surviving mutants are defects the verification would miss, which turns "are the tests good?" from a feeling into a number.
  Dedicated tools exist for most ecosystems, and an agent can run a manual round on demand.
- **Prefer property-based tests where the spec has invariants.**
  One property such as "output is always sorted" or "round-trip is identity" covers input classes that example tests never enumerate.
- **Run the real thing.**
  Unit tests can stay green while the application fails to start.
  An end-to-end smoke check - actually launching, actually exercising the changed path - catches the integration class of defects that unit suites are structurally blind to, so make it part of the gate rather than a courtesy.

## Per-task flow

For a ship task, the layers compose as: spec with executable acceptance criteria, independent test derivation, implementation, deterministic gates including the full suite and a smoke run, adversarial review where authorized, green checks on the recorded PR head, and a final human spot-check of the diff plus one re-run of the evidence command.
Each layer has different blind spots, which is the point - sensitivity compounds across independent layers.
Write the acceptance criteria into the task instructions when scaffolding with `bin/fm-brief.sh`, so the definition of done travels with the worker instead of living only in conversation.

## Honest limits

- **Specificity** - a pass verdict being genuinely a pass - is achievable at effectively 100%, because "pass" is defined by machine-checked facts.
- **Sensitivity** - a real defect being caught - stays below 100% no matter the process; mutation testing tells you where it actually stands, and independent test authorship, adversarial review, and real-application runs are what raise it.
- **Not being lied to** is fully solvable: it is a property of the workflow, not of the model, and it holds exactly when no claim is accepted without its artifact.

# Daily Work Directive — Abelastro Engineering

**Owner:** Abel Wondimu · **Version:** 1.0 · **Created:** 2026-10-23
**Canonical copy:** `Abelastro/Abelastro/DAILY-WORK-DIRECTIVE.md`
**Purpose:** one shippable unit of work per day, chosen in advance, so an agent can start without guessing.

---

## 0. Session-start protocol (the agent reads this first)

Run these four steps before touching code.

```bash
cd /workspace/githubProjects/profile-readme
git pull --rebase origin main
grep -n '^- \[ \]' DAILY-WORK-DIRECTIVE.md | head -1   # first unchecked day
```

1. Open the repo named on that day's block and pull `main` (`master` for `tomato-agent-chatbot`).
2. Do only that day's task. If it is already half-finished, finish it; do not start tomorrow's.
3. Run the repo's test command and typecheck. A red suite stops the day. Fix or revert; never push red.
4. Commit, push, then tick the checkbox here with the date, the commit SHA, and one honest line about the outcome. Push this file too.

If a day is blocked (missing token scope, missing API key, upstream breakage), write `BLOCKED: <reason>` under the checkbox, leave it unchecked, and take the next unchecked day from a different repo. Do not fake completion.

**One day = one PR-sized change.** If a task balloons, ship the smallest correct slice and move the rest to the next day's note. Small, green, pushed beats large, half-done, local.

---

## 1. Quality bar (non-negotiable)

- Every behavior change gets a test that fails without the change. No exceptions.
- `README.md` changes when public behavior changes. New rule, new endpoint, new flag: document it in the same commit.
- Conventional commits: `feat(scope):`, `fix(scope):`, `test(scope):`, `docs(scope):`, `chore(scope):`, `refactor(scope):`.
- Feature and release days update `CHANGELOG.md` under `## [Unreleased]`.
- The profile README test count stays accurate. Recount with the command in section 2, do not estimate.
- Never commit `.env`, keys, tokens, or generated SQLite files. Never force-push `main`.
- Do not add `.github/workflows/*` to a commit until the fine-grained token has **Workflows: Read and write**. The push will be rejected and the retry wastes the day. CI files live in `ci-files/` until then.

---

## 2. Environment and commands

`python` is not installed. Use `python3` (3.11.2). Node is v22.23.2, npm 10.9.9.

| Repo | Stack | Test command | Branch |
|---|---|---|---|
| `pr-quality-engine` | TypeScript | `npx vitest run && npx tsc --noEmit` | `main` |
| `llm-eval-platform` | Python | `python3 -m pytest -q` | `main` |
| `agent-orchestrator` | Python | `PYTHONPATH=src python3 -m pytest -q` | `main` |
| `typesafe-rag-client` | TypeScript | `npx vitest run && npx tsc --noEmit` | `main` |
| `test-engineering-examples` | Python + TS | `python3 -m pytest -q` then `npx vitest run` | `main` |
| `rag-knowledge-assistant` | Python | `python3 -m pytest -q` | `main` |
| `cv-quality-inspection` | Python | `python3 -m pytest -q` | `main` |
| `ai-research-agent` | Python | `python3 -m pytest -q` | `main` |
| `tomato-agent-chatbot` | Python | `python3 -m pytest tests/ -q` | `master` |

**Push command (the only form that works with this token):**

```bash
AUTH=$(printf 'x-access-token:%s' "$GITHUB_TOKEN" | base64 -w0)
git -c http.extraHeader="Authorization: Basic $AUTH" push
```

Do not embed the token in the remote URL (the security layer redacts it and git fails with `bad/illegal format`). Do not use `Authorization: Bearer` (git returns `invalid credentials`).

**Commit with workflows excluded when the token lacks the scope:**

```bash
git add -A -- . ':(exclude).github'
```

Passing a second explicit exclude on the same command (for example `node_modules`) makes git abort. Use the repo's `.gitignore` instead.

**Recount tests before updating the profile badge:**

```bash
cd /workspace/githubProjects/pr-quality-engine && npx vitest run 2>&1 | grep -E "Tests +[0-9]+ passed"
cd ../llm-eval-platform && python3 -m pytest -q 2>&1 | tail -1
```

---

## 3. Why this order

The job is reviewing AI-generated code and judging whether tests prove the requirements. Two repos speak to that directly: `pr-quality-engine` (automated review of tests and diffs) and `llm-eval-platform` (judging model output quality). Those come first, get the most polish, and stay release-ready. `agent-orchestrator` is the systems-depth story. The remaining repos get maintenance passes so they do not rot, and the profile stays current. That weighting is deliberate; do not reshuffle weeks without a reason.

---

# Week 1 — `pr-quality-engine` (flagship)

### - [ ] Day 1 — Python test-smell rules
- Build: `pythonNoAssertionRule`, `pythonBroadExceptRule`, `pythonSleepRule` in `src/rules/testSmells.ts` (or a new `src/rules/pythonTestSmells.ts` exported through the barrel).
- Tests: one positive and one negative case each in `tests/testSmells.test.ts`, using `addedFileDiff("tests/test_x.py", [...])`.
- Done when: `npx vitest run` green, `tsc --noEmit` clean, rule table in README updated.
- Commit: `feat(rules): add Python test-smell detection`

### - [ ] Day 2 — Per-rule severity overrides
- Build: `rules: { "<rule-id>": "off" | "info" | "minor" | "major" | "critical" }` in `EngineConfig`; apply in the engine before rendering. Unknown rule ids warn, do not crash.
- Tests: override downgrades, `"off"` removes the finding, unknown id produces a warning.
- Done when: config is documented in README with a JSON example.
- Commit: `feat(config): support per-rule severity overrides`

### - [ ] Day 3 — Baseline suppression
- Build: `--baseline <path>` and `.prqe-baseline.json`. A finding is suppressed when rule id, file, and normalized evidence match a baseline entry. `--write-baseline` emits the file.
- Tests: suppress exact match, keep changed evidence, write/read round-trip.
- Done when: README explains the adopt-existing-repo workflow.
- Commit: `feat(cli): add baseline suppression for legacy findings`

### - [ ] Day 4 — SARIF correctness pass
- Build: stable `partialFingerprints`, `helpUri` per rule pointing at `docs/rules.md#<rule-id>`, `level` derived from severity.
- Tests: snapshot of a two-finding SARIF document; fingerprint stability across runs.
- Done when: output validates against the SARIF 2.1.0 schema shape used in tests.
- Commit: `feat(report): improve SARIF fingerprints and rule help links`

### - [ ] Day 5 — Composite GitHub Action
- Build: `action.yml` (inputs: `config`, `baseline`, `fail-on`; outputs: `findings`, `exit-code`) plus `examples/github-workflow.yml`.
- Tests: a shell-level smoke test that runs the built CLI on `fixtures/sample.diff` and asserts exit codes 0 and 1.
- Done when: `action.yml` is valid YAML and the example is copy-paste runnable. Save the workflow file to `ci-files/prqe-action-example.yml`; do not commit it under `.github/`.
- Commit: `feat(action): add composite GitHub Action`

### - [ ] Day 6 — Generated rule reference
- Build: `scripts/gen-rule-docs.ts` that reads the rule registry and writes `docs/rules.md` with id, severity, description, and a good/bad example per rule.
- Tests: generator is deterministic; a test asserts the committed `docs/rules.md` matches regenerated output.
- Done when: `docs/rules.md` covers all rules and is linked from README.
- Commit: `docs(rules): generate rule reference from registry`

### - [ ] Day 7 — Release v0.2.0
- Build: bump `package.json` to `0.2.0`, move `[Unreleased]` to `[0.2.0]` with today's date, add a short upgrade note.
- Verify: full suite green, `npm pack --dry-run` shows only intended files.
- Done when: tag `v0.2.0` pushed and GitHub release notes pasted from the changelog.
- Commit: `chore(release): v0.2.0`

---

# Week 2 — `llm-eval-platform`

### - [ ] Day 8 — Judge response cache
- Build: content-addressed cache keyed by `sha256(model + prompt + params)` in `src/llm_eval/judges/`, persisted through the SQLite store. `--no-cache` flag on the CLI.
- Tests: second identical call hits cache; changed prompt misses; cache disabled path.
- Done when: README documents the cache and its invalidation rule.
- Commit: `feat(judges): add content-addressed response cache`

### - [ ] Day 9 — Dataset versioning
- Build: compute a stable hash of dataset rows; store it on each run; refuse to compare runs whose dataset hashes differ unless `--allow-dataset-drift`.
- Tests: hash stability across key order, mismatch refusal, override flag.
- Done when: run metadata includes `dataset_hash`.
- Commit: `feat(datasets): version datasets by content hash`

### - [ ] Day 10 — HTML report
- Build: `src/llm_eval/report/html.py` rendering metrics table, per-case pass/fail, and deltas versus a baseline run. Single self-contained file, no CDN.
- Tests: snapshot the rendered HTML for a two-case fixture; assert escaping of `<script>` in model output.
- Done when: `llm-eval report --format html` writes a file that opens offline.
- Commit: `feat(report): add self-contained HTML report`

### - [ ] Day 11 — Pairwise comparison
- Build: `compare_pairwise(run_a, run_b)` with win/loss/tie counts and a per-metric breakdown. Judge optional; deterministic metric fallback for tests.
- Tests: A beats B, B beats A, tie, empty set.
- Done when: CLI `llm-eval compare --pairwise` prints a summary table.
- Commit: `feat(compare): add pairwise run comparison`

### - [ ] Day 12 — CI quality gate command
- Build: `llm-eval gate --baseline <run> --max-regression 0.02 --min-score 0.8` with exit codes 0 (pass), 1 (regression), 2 (config error).
- Tests: pass, regression, missing baseline, bad threshold.
- Done when: README shows a GitHub Actions step using it (file saved to `ci-files/`).
- Commit: `feat(cli): add quality gate command with exit codes`

### - [ ] Day 13 — Provider abstraction
- Build: `providers/base.py` protocol, `openai.py`, `fake.py` (deterministic responses for tests). Route judges and generators through it.
- Tests: fake provider drives a full eval run with zero network; provider selection by config.
- Done when: no test imports `openai` directly.
- Commit: `refactor(providers): abstract model providers behind a protocol`

### - [ ] Day 14 — Release v0.2.0
- Build: bump version, changelog entry, tag, release notes.
- Verify: `python3 -m pytest -q` green from a clean checkout.
- Commit: `chore(release): v0.2.0`

---

# Week 3 — `agent-orchestrator`

### - [ ] Day 15 — Mermaid DAG export
- Build: `workflow.to_mermaid()` emitting `graph TD` with edges and node labels; CLI `agent-orchestrator graph --mermaid`.
- Tests: linear, fan-out, fan-in, cycle-rejection message.
- Done when: `docs/architecture.md` embeds a generated diagram.
- Commit: `feat(dag): export workflows as Mermaid graphs`

### - [ ] Day 16 — Retry backoff strategies
- Build: `fixed`, `exponential`, `exponential_jitter` in `retry.py`, configured per node with `max_attempts` and `base_delay`.
- Tests: attempt counts, delay sequence with a fake clock, jitter bounds, non-retryable error bypass.
- Done when: README documents the strategies.
- Commit: `feat(retry): add configurable backoff strategies`

### - [ ] Day 17 — Human-in-the-loop timeout
- Build: approval nodes accept `timeout_seconds` and `on_timeout: "fail" | "auto_approve" | "escalate"`. Escalation records an event and moves to a configured fallback node.
- Tests: timeout fail, auto-approve, escalation path.
- Done when: state survives a simulated process restart.
- Commit: `feat(hitl): add approval timeouts and escalation`

### - [ ] Day 18 — Checkpoint retention
- Build: keep last N checkpoints per run plus the newest per node; `compact()` prunes older ones. Configurable `retention: {max_per_run, max_age_days}`.
- Tests: pruning keeps newest, age-based pruning, no-op under limit.
- Done when: store size stops growing on a 100-iteration loop test.
- Commit: `feat(store): add checkpoint retention and compaction`

### - [ ] Day 19 — OpenTelemetry export
- Build: optional `otel` extra; emit spans for node start/end, retries, and HITL waits. No hard dependency when disabled.
- Tests: an in-memory span exporter asserts span names and parent/child links.
- Done when: import without the extra still works.
- Commit: `feat(tracing): export spans to OpenTelemetry`

### - [ ] Day 20 — Workflow validation CLI
- Build: `agent-orchestrator validate workflow.json` checking unknown node types, unreachable nodes, missing dependencies, duplicate ids, and cycle detection.
- Tests: one fixture per failure mode plus a valid workflow.
- Done when: exit code 0/1 and human-readable errors.
- Commit: `feat(cli): add workflow validation command`

### - [ ] Day 21 — Release v0.2.0
- Build: bump version, changelog, tag, release notes.
- Verify: `PYTHONPATH=src python3 -m pytest -q` green.
- Commit: `chore(release): v0.2.0`

---

# Week 4 — cross-repo quality and profile

### - [ ] Day 22 — Mutation testing demo
- Build: `test-engineering-examples/python/` mutation setup with `mutmut` (or `cosmic-ray`) on the calculator module and a short write-up of survived mutants.
- Tests: the mutation command runs; document the score.
- Done when: README section explains why a green suite is not proof.
- Commit: `test(python): add mutation testing demonstration`

### - [ ] Day 23 — Property-based tests
- Build: `hypothesis` tests for the Python calculator and `fast-check` tests for the TypeScript utilities.
- Tests: at least three properties (commutativity, identity, round-trip).
- Done when: both suites green and shrink failures are discussed in the README.
- Commit: `test(properties): add property-based test examples`

### - [ ] Day 24 — Ten review scenarios
- Build: `code-review-playbook/scenarios/` with ten real-shaped diffs, each with the review comments a strong reviewer would leave and the reasoning.
- Tests: a script validates every scenario has a diff, a review, and a verdict.
- Done when: index table links all ten.
- Commit: `docs(scenarios): add ten worked review scenarios`

### - [ ] Day 25 — Streaming in the RAG client
- Build: `streamQuery()` with `AbortSignal` and async iteration; chunk events include sources.
- Tests: abort mid-stream, ordered chunks, error propagation.
- Done when: README example shows cancellation.
- Commit: `feat(client): add streaming with abort support`

### - [ ] Day 26 — Golden-set evaluation runner
- Build: `rag-knowledge-assistant/evals/` with 20 question/answer/source rows and a runner that computes retrieval precision, grounding, and hallucination rate against thresholds.
- Tests: runner on a fake retriever; threshold failure returns non-zero.
- Done when: `python3 -m evals.run` prints a table.
- Commit: `feat(evals): add golden-set evaluation harness`

### - [ ] Day 27 — Synthetic defect generator
- Build: `cv-quality-inspection/src/utils/synthetic.py` generating scratches, dents, and discoloration on a blank canvas with seeded randomness.
- Tests: deterministic under seed, shapes within bounds, defect count matches request.
- Done when: example script writes three sample images to `data/samples/`.
- Commit: `feat(synthetic): add seeded synthetic defect generator`

### - [ ] Day 28 — Trace and replay
- Build: `ai-research-agent` records each tool call (name, params, result, duration) to JSON and can replay a trace without re-running tools.
- Tests: record/replay round-trip, replay ignores live tool errors.
- Done when: CLI `--replay trace.json` produces the same answer.
- Commit: `feat(agent): add trace recording and replay`

### - [ ] Day 29 — Dependency hygiene
- Build: Dependabot config per repo (npm and pip manifests) and a `.pre-commit-config.yaml` with ruff/black for Python and prettier/eslint for TypeScript where applicable.
- Tests: configs are valid YAML/TOML; pre-commit runs clean on current code.
- Done when: configs are committed to at least the four flagship repos (workflow files stay in `ci-files/`).
- Commit: `chore: add Dependabot and pre-commit configuration`

### - [ ] Day 30 — Retro and next cycle
- Build: update the profile README test count, refresh featured cards, write a 10-line retro here (what shipped, what slipped, what to change), and draft Days 31–60 from the backlog in section 5.
- Done when: retro is committed and the next cycle's first week is filled in.
- Commit: `docs: day-30 retro and cycle-two plan`

---

## 4. Daily loop, in order

1. Pull the target repo.
2. Branch only if the change is risky; otherwise commit straight to `main` for one-day tasks.
3. Write the failing test first when the behavior is testable. Then the code.
4. Run the full suite, not just the new file.
5. Update README/CHANGELOG if public behavior moved.
6. Commit with a conventional message.
7. Push with the Basic-auth form.
8. Tick the day here with date, SHA, and a truthful note. Push this file.
9. Stop. Do not start the next day.

---

## 5. After Day 30 — the sustainable loop

Run a five-day cadence per week and let the backlog refill itself.

| Weekday | Theme | Output |
|---|---|---|
| Monday | Feature | One user-visible capability with tests |
| Tuesday | Hardening | Edge cases, error paths, fuzz or property tests |
| Wednesday | Docs | README, examples, one worked scenario |
| Thursday | Refactor | Remove duplication, tighten types, no behavior change |
| Friday | Release and retro | Changelog, tag, profile refresh, next week's list |

Backlog sources, in priority order: failing or flaky tests, TODOs in code, issues opened by Dependabot, gaps between README claims and reality, and one "reviewer's eye" pass where you read your own diff as if rejecting it.

Track four numbers monthly and write them in the retro: total tests, CI green rate, open issues, and days since the last release per repo. If a repo has no release in 60 days, it drops out of the featured list until it earns its place back.

---

## 6. Guardrails

- A day is done when it is pushed, not when it compiles.
- If tests fail for reasons outside the day's scope, fix the smallest thing that restores green and note it.
- If a task needs a paid API key, implement the deterministic fake path and gate the live path behind an env var. Never let a missing key block a test.
- If the token lacks a scope, save the artifact to `ci-files/` and continue. Do not burn the day retrying.
- Keep the profile honest. A smaller true number beats a bigger invented one.

---

## 7. Progress log

Append one row per completed day. Newest at the bottom.

| Day | Date | Repo | Commit | Outcome |
|---|---|---|---|---|
| — | — | — | — | Directive created |

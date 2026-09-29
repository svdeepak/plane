# Coding Agent in the Build Pipeline — Progress & Recall (Tasks 18–20 + follow-ups)

**Last updated:** 2026-09-29
**Repo (fork):** https://github.com/svdeepak/plane  (parent: makeplane/plane, default branch `preview`, AGPL-3.0)
**Local clone:** `~/Desktop/plane-ci`  (shallow, branch `ci/coding-agent-workflow`)
**Working notes also in-repo:** `AGENT_CI_NOTES.md`

This file is the single place to pick this work back up. It records what was built, what
works, what broke and why, exact commands, and the open follow-up ideas.

---

## 0. TL;DR — where things stand

- **Task 18 (invoke agent on fixed prompt): ✅ DONE and demonstrated live.** The agent
  created `docs/ci-agent-demo.md` on a CI runner and pushed it to branch
  `agent/demo-1790593655`.
- **Task 19 (pass/fail build from analysis prompt): ✅ BUILT, ⏳ not yet run live.**
  Workflow + prompt exist and lint clean; needs a demo run to show PASS and FAIL.
- **Task 20 (when to use this): ✅ ANSWERED** (captured below and in `AGENT_CI_NOTES.md`),
  and several cautions were confirmed *empirically* during task-18 runs.
- **Follow-up ideas (log-watch on deploy; auto-debug on failure): discussed, NOT built.**

---

## 1. Setup that had to happen (and the order it bit us)

To get a coding agent running in GitHub Actions, all of these were required:

1. **Fork the repo** — `gh repo fork makeplane/plane`.
2. **Auth = Claude subscription OAuth token** (chosen over a metered API key and over
   OpenRouter). Generate locally:
   ```bash
   claude setup-token          # prints an sk-ant-oat01-... token
   ```
   - Gotcha: it does **not** auto-open a browser in all terminals — it prints an
     authorize URL you paste into a browser. After approving, the **token prints back in
     the terminal**, not the browser (the browser just says "you can close this window").
   - Store it in the fork: **Settings → Secrets and variables → Actions →** new secret
     named exactly `CLAUDE_CODE_OAUTH_TOKEN`.
   - Note: a subscription normally uses interactive `claude login` and you never see a
     token; CI can't do interactive login, so `setup-token` mints one explicitly. Same
     subscription, no extra cost. These tokens are long-lived but **do expire** — re-mint
     if CI auth starts failing.
3. **Enable Actions on the fork** (forks disable Actions by default).
4. **`workflow_dispatch` workflows only appear in the Actions UI once they're on the
   default branch (`preview`).** We opened PR #1 and merged it into `preview` to register
   them. (Files can live on a feature branch, but the UI won't list them until they're on
   `preview`.)
5. **`id-token: write` permission** — the official `claude-code-action` needs it to fetch
   an OIDC token (error otherwise: *"Unable to get ACTIONS_ID_TOKEN_REQUEST_URL"*).
6. **Install the "Claude" GitHub App** on the fork (`https://github.com/apps/claude`) —
   the marketplace action requires it (error otherwise: *"Claude Code is not installed on
   this repository"*). This is separate from the local `claude` CLI and from the token.
7. **(For auto-PR) "Allow GitHub Actions to create and approve pull requests"** — a repo
   setting under Settings → Actions → General. Currently **OFF**, so the auto-PR step
   fails with *"GitHub Actions is not permitted to create or approve pull requests"*. The
   branch still pushes; only the PR-open step fails.

---

## 2. Task 18 — invoke a coding agent against a fixed prompt

**Files:**
- `.github/workflows/coding-agent-demo.yml` — the workflow.
- (Header comments still mention the marketplace action — see note below; the body now
  uses the headless CLI.)

**What it does now:** manual (`workflow_dispatch`) trigger with an optional `prompt`
input. If the input is blank it uses a fixed default prompt. It then:
1. Resolves the prompt (input passed via **env var**, not string interpolation — see the
   injection bug below).
2. Installs the Claude Code CLI (`npm install -g @anthropic-ai/claude-code`).
3. Runs it **headless**: `claude -p "$PROMPT" --output-format json --max-turns 15
   --allowedTools "Read,Edit,Write,Glob,Grep" --permission-mode acceptEdits`.
4. Prints the full diff, writes it to the run summary, and (if changed) pushes a branch
   `agent/demo-<ts>` and tries to open a PR.

**IMPORTANT — why it uses the headless CLI, not the marketplace action:**
The first successful run used `anthropics/claude-code-action@v1` and it worked (edited
`CONTRIBUTING.md`, +11 lines). But **subsequent `workflow_dispatch` runs no-opped** — the
action set up, detected agent mode, then exited in ~12s with no work, no cost, no change.
The marketplace action is built around **PR/issue events**; for pure `workflow_dispatch`
file-generation it short-circuits. Switching to the **headless `claude -p` CLI** made
execution reliable (it always runs the agent). Trade-off: the CLI won't auto-open a PR
itself, so the workflow does the branch-push + `gh pr create` manually.

**Proof it worked (headless run, run id 36413719243):**
- `?? docs/ci-agent-demo.md` created → committed `1 file changed, 7 insertions(+)` →
  pushed branch `agent/demo-1790593655`.
- Only the final auto-PR step failed (the repo setting in §1.7).

**Exact content the agent wrote** to `docs/ci-agent-demo.md`:
```markdown
# Running the coding agent in CI

- This repository can run an AI coding assistant ("Claude Code") automatically as part of its GitHub Actions workflows, instead of only running it on someone's laptop.
- A person triggers it manually from the GitHub Actions tab (there's a button to run it on demand), so it never runs unexpectedly on every code change.
- The workflow gives the assistant a fixed, pre-written instruction (like "add some documentation"), so it always does a known, low-risk task rather than something unpredictable.
- Once the assistant finishes, its changes are shown as a diff in the workflow summary and opened as a normal pull request, so a human can review everything before it's merged.
- A separate workflow can also use the assistant purely as a checker — it reads a code change, decides pass or fail, and can stop a build from going green if it finds a problem like a leaked secret.
```

**The fixed default prompt (used when input is blank):**
> "Add a short 'Running the coding agent in CI' section near the top of CONTRIBUTING.md
> (create the file if it does not exist) that explains, in 4-6 bullet points readable by a
> non-engineer, how this repository invokes an AI coding agent from GitHub Actions. Do not
> modify any source code. Keep the change to documentation only."

---

## 3. Task 19 — pass or fail the build from an analysis prompt

**Files:**
- `.github/workflows/coding-agent-analysis-gate.yml` — the gate workflow.
- `.ci/analysis-prompt.md` — the scoped analysis prompt (secret/PII leak check).

**Mechanism (the core idea):**
1. Compute the diff to analyse (PR base…HEAD, or last commit on manual runs).
2. Run Claude Code **headless** with the analysis prompt + diff:
   `claude -p "$PROMPT" --output-format json --max-turns 8 --allowedTools "Read,Grep"`.
3. The prompt forces a strict verdict as the final line:
   `{"verdict":"PASS"|"FAIL","reason":"...","findings":[...]}`.
4. Extract the verdict with `jq`/`grep`, then a shell step maps it to an **exit code**:
   `PASS → exit 0`, `FAIL → exit 1`, anything else → exit 1 (fail-closed). Agent/parse
   failure exits 3 (distinct from an analysis FAIL).
5. Verdict + findings are written to `$GITHUB_STEP_SUMMARY` so it's reviewable.

This mirrors the repo's own `check-version.yml`, which already fails CI with `exit 1` when
a condition isn't met — here an **agent** decides the condition.

**Design choices that keep it trustworthy:**
- Fail-closed on error/invalid verdict.
- "High-confidence only" — the prompt says PASS when in doubt, to avoid blocking
  legitimate work on false positives.
- Distinct exit codes separate an *analysis FAIL* from an *agent/infra failure*.
- Starts as `workflow_dispatch`; to make it a real merge gate, uncomment the
  `pull_request:` trigger and mark the job a required status check in branch protection.

**STATUS: built + lint-clean, but NOT yet run live.** Next step is to demo it both ways:
run once on a clean diff (expect PASS/green) and once on a diff containing a fake secret
(expect FAIL/red). Also note: this workflow uses the headless CLI so it does **not** need
the GitHub App or `id-token` — only the `CLAUDE_CODE_OAUTH_TOKEN` secret.

---

## 4. Task 20 — when can we use a coding agent this way?

**Good fits:**
- Fuzzy review that resists regex (does the PR description match the code? doc coverage,
  tone/clarity of user-facing strings, obviously risky patterns a linter misses).
- Advisory / augmentation (agent comments and suggests; humans + deterministic checks
  decide).
- Generation off the critical path (release notes, changelogs, migration guides, test
  scaffolding, issue triage/labeling).
- Bounded, well-specified transformations with a clear rubric.
- Triage before human effort (pre-screen dependency bumps, summarize large diffs).

**Poor fits / anti-patterns:**
- Anything a deterministic tool already does (compile, tests, eslint, tsc, license/secret
  scanners) — don't ask an LLM to do what a compiler does correctly and free.
- Sole hard-gate on merge/deploy.
- Security-critical *final* authority (agent flags; scanner + human confirm).
- High-frequency runs where cost/latency dominate.
- Vague, unspecified prompts (non-reproducible verdicts).

**Rule of thumb:** use the agent as a **judgment layer that augments deterministic checks,
defaults to advisory, and only hard-gates when the decision is inherently fuzzy, the prompt
is tightly scoped, and there's a human override.** If a linter, compiler, or test can
answer the question, use that instead.

**Cautions we confirmed empirically during task 18 (not just theory):**
- **Non-determinism is real:** the *same* prompt produced +11 lines on one run and *no
  changes* on the next. Do not assume repeatable output.
- **Cost is real:** one agent run reported `total_cost_usd ≈ 0.12` and used
  `claude-haiku-4-5` + `claude-sonnet-5`. Per-push runs would add up fast.
- **Tooling assumptions bite:** the marketplace action silently no-opped on
  `workflow_dispatch`. Verify the agent actually *did* work (check turns/cost/diff), don't
  trust a green check alone.
- **LLMs "fix" tests by weakening them:** relevant to the auto-debug idea below — an agent
  can make a red check green by hiding the problem.

---

## 5. Follow-up ideas discussed (NOT built yet)

The user asked about two extensions. Assessment given; nothing implemented.

### 5a. LLM watches logs after a successful deployment  (LOW risk — recommended)
- Trigger: `on: deployment_status` (state = success) or a post-deploy step.
- Fetch recent deploy/runtime logs (CloudWatch / platform API / `kubectl logs`).
- `claude -p`: summarize errors, flag anomalies vs a healthy baseline, rate severity.
- Output: PR comment / Slack / issue. Optionally a **severe-only** smoke-gate using the
  same JSON-verdict → exit-code mechanism as task 19.
- Caveats: logs are huge → window/sample (same token-budget problem as llms.txt); keep it
  **advisory**, do not replace real alerting (Datadog/CloudWatch alarms).

### 5b. LLM auto-debugs on CI failure and pushes a fix  (HIGH risk — shape carefully)
- Trigger: `on: workflow_run` with `conclusion == failure`.
- Gather failing logs + breaking diff + relevant source; `claude -p` to diagnose + fix.
- **CRITICAL — what happens with the fix:**
  - ✅ **Open a PR** with the fix, re-run CI on it, human reviews/merges. (Do this.)
  - ⚠️ Push directly to the branch — unreviewed AI code; avoid.
  - ❌ Push and self-merge / auto-deploy — no human; can cascade bad fixes. Don't.
- Failure modes to guard: **fix loops** (agent commit re-triggers failure → another fix;
  cap with max-iterations and don't re-trigger on agent commits), **plausible-but-wrong
  fixes**, **masking vs fixing** (weakening a test), **runaway cost**.
- **Firm line:** automatic *diagnosis & proposal* = good; automatic *merge/deploy of AI
  fixes* = no.

**Suggested order when revisiting:** build 5a first (safe quick win), then 5b as
failure-triage-**to-PR** with a loop cap.

---

## 6. Current repo state (facts, as of last update)

- **Workflows registered & active on `preview`:**
  - `Coding Agent — Fixed Prompt Demo` (`.github/workflows/coding-agent-demo.yml`)
  - `Coding Agent — Analysis Gate` (`.github/workflows/coding-agent-analysis-gate.yml`)
- **Files present on branch `ci/coding-agent-workflow`:** both workflows,
  `.ci/analysis-prompt.md`, `AGENT_CI_NOTES.md`. (Also merged to `preview` via PR #1,
  plus later commits.)
- **Agent-produced branch:** `agent/demo-1790593655` containing `docs/ci-agent-demo.md`
  (no PR opened — blocked by the repo setting in §1.7).
- **Recent demo runs:** mix of success/failure — successes = real agent work; failures =
  the injection bug (fixed) and the auto-PR permission (repo setting, still off).
- **Secret set:** `CLAUDE_CODE_OAUTH_TOKEN` (added by user).
- **GitHub App:** Claude app installed on the fork.

---

## 7. Bugs found & fixes applied (so we don't repeat them)

1. **OIDC error** with the marketplace action → added `id-token: write` to `permissions:`.
2. **"Claude Code is not installed"** → installed the Claude GitHub App on the fork.
3. **Workflows not visible in Actions UI** → merge them to the default branch `preview`.
4. **`workflow_dispatch` no-op** with the marketplace action → switched to headless
   `claude -p` CLI.
5. **Shell injection / `exit 127` ("the: command not found")** in the "Resolve prompt"
   step: the `prompt` input was interpolated directly into a shell string, so quotes in
   the input broke it. **Fix:** pass the input via an `env:` var (`INPUT_PROMPT`) and
   reference `$INPUT_PROMPT` — the shell then treats it as data, not code. (General rule:
   never `"${{ github.event.inputs.X }}"` directly inside a shell line.)
6. **Auto-PR fails** ("GitHub Actions is not permitted to create or approve pull
   requests") → enable the repo setting in §1.7, or open the PR manually.

---

## 8. Immediate next steps when revisiting

1. **Run task 19 live** to show PASS (clean diff) and FAIL (diff with a fake secret).
   ```bash
   gh workflow run "Coding Agent — Analysis Gate" --repo svdeepak/plane --ref ci/coding-agent-workflow
   ```
2. **Enable auto-PR** (Settings → Actions → General → allow Actions to create PRs) if you
   want the demo to open PRs automatically.
3. **Tidy the demo workflow header comment** — it still describes the marketplace action;
   update it to reflect the headless-CLI implementation.
4. **Optionally push the headless-CLI demo to `preview`** so `preview` is the canonical
   version (currently the latest demo edits are on `ci/coding-agent-workflow`).
5. **Decide on follow-ups (§5).** Recommended: build 5a (log-watch), then 5b
   (failure-triage-to-PR with a loop cap).

---

## 9. Useful commands (copy-paste)

```bash
# Trigger the fixed-prompt demo (blank prompt = fixed default)
gh workflow run "Coding Agent — Fixed Prompt Demo" --repo svdeepak/plane --ref ci/coding-agent-workflow

# Trigger with an explicit prompt (note: passed safely via -f)
gh workflow run "Coding Agent — Fixed Prompt Demo" --repo svdeepak/plane \
  --ref ci/coding-agent-workflow -f prompt="Create a NEW file docs/foo.md with ..."

# Watch the latest run
RUN=$(gh run list --repo svdeepak/plane --workflow "Coding Agent — Fixed Prompt Demo" --limit 1 --json databaseId --jq '.[0].databaseId')
gh run view "$RUN" --repo svdeepak/plane
gh run view "$RUN" --repo svdeepak/plane --log-failed   # if it failed

# See what an agent branch contains
gh api "repos/svdeepak/plane/contents/docs/ci-agent-demo.md?ref=agent/demo-1790593655" --jq '.content' | base64 -d

# Re-mint the OAuth token if CI auth fails
claude setup-token
```

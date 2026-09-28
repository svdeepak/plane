# Coding Agent in the Build Pipeline (Tasks 18–20)

This fork adds two GitHub Actions workflows that stand up a **coding agent (Claude Code)**
inside CI. It's a learning/demo setup, kept intentionally safe: manual triggers only,
least-privilege permissions, and no secrets committed to the repo.

## Files

| File | Task | Purpose |
|------|------|---------|
| `.github/workflows/coding-agent-demo.yml` | 18 | Invoke Claude Code against a **fixed prompt** (agent mode). |
| `.github/workflows/coding-agent-analysis-gate.yml` | 19 | Run an **analysis prompt** and **pass/fail the build** from the agent's verdict. |
| `.ci/analysis-prompt.md` | 19 | The scoped analysis prompt (secret/PII leak check) that emits a strict JSON verdict. |

## Auth — subscription OAuth token (no metered API key)

Both workflows authenticate with a **Claude subscription OAuth token**, so there is no
pay-per-token API key and no per-run cost against a metered key.

1. Locally, run:
   ```bash
   claude setup-token
   ```
   This authenticates against your Claude subscription and prints a long-lived token.
2. In the fork on GitHub: **Settings → Secrets and variables → Actions → New repository secret**
   - Name: `CLAUDE_CODE_OAUTH_TOKEN`
   - Value: the token from step 1
3. Nothing else to configure. The workflows read the secret via
   `${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}`.

> The token is still a secret (a CI runner is stateless and needs *some* credential),
> but it draws on your existing subscription rather than a billed API key. Confirm your
> plan permits automated use.

## Task 18 — Fixed-prompt demo

`coding-agent-demo.yml` runs on **manual dispatch**. It sends Claude Code a fixed,
low-risk prompt (add a short CI section to `CONTRIBUTING.md`) and shows the resulting
diff. Tool access is restricted (`Read,Edit,Write,Glob,Grep`) and turns are capped.

Run it: **Actions → "Coding Agent — Fixed Prompt Demo" → Run workflow.**

## Task 19 — Pass/fail the build from an analysis prompt

`coding-agent-analysis-gate.yml` is the gating mechanism:

1. Computes the diff to analyse (PR base…HEAD, or last commit on manual runs).
2. Runs Claude Code **headlessly** (`claude -p … --output-format json`) with the prompt
   in `.ci/analysis-prompt.md`.
3. Extracts the strict verdict object — `{"verdict":"PASS"|"FAIL", …}` — from the output.
4. A shell step maps the verdict to an **exit code**: `PASS → exit 0`, `FAIL → exit 1`.
   The non-zero exit is what fails the CI check.

This mirrors the repo's own `check-version.yml`, which already fails CI with `exit 1`
when a condition isn't met — here an **agent** decides the condition.

Design choices that keep the gate trustworthy:
- **Fail-closed:** if the agent errors or returns no valid verdict, the build fails.
- **High-confidence only:** the prompt tells the agent to `PASS` when in doubt, so
  legitimate work isn't blocked by false positives.
- **Reviewable:** the verdict and findings are written to the job summary.
- **Distinct failure modes:** agent/parse failure exits `3`, an analysis `FAIL` exits `1`.

To promote it to a real merge gate: uncomment the `pull_request:` trigger and mark the
job as a required status check in branch protection.

## Task 20 — When to use a coding agent this way (summary)

**Good fits:** fuzzy review that resists regex (does the PR description match the code?),
advisory comments, generation off the critical path (release notes, changelogs), bounded
transformations, triage before human effort.

**Poor fits / anti-patterns:** anything a deterministic tool already does (compile, tests,
eslint, license/secret scanners); sole hard-gate on merge/deploy; security-critical final
authority; high-frequency runs where cost/latency dominate; vague, unspecified prompts.

**Rule of thumb:** use the agent as a **judgment layer that augments deterministic checks,
defaults to advisory, and only hard-gates when the decision is inherently fuzzy, the prompt
is tightly scoped, and there's a human override.** If a linter, compiler, or test can answer
the question, use that instead.

# Running the coding agent in CI

- This repository can run an AI coding assistant ("Claude Code") automatically as part of its GitHub Actions workflows, instead of only running it on someone's laptop.
- A person triggers it manually from the GitHub Actions tab (there's a button to run it on demand), so it never runs unexpectedly on every code change.
- The workflow gives the assistant a fixed, pre-written instruction (like "add some documentation"), so it always does a known, low-risk task rather than something unpredictable.
- Once the assistant finishes, its changes are shown as a diff in the workflow summary and opened as a normal pull request, so a human can review everything before it's merged.
- A separate workflow can also use the assistant purely as a checker — it reads a code change, decides pass or fail, and can stop a build from going green if it finds a problem like a leaked secret.

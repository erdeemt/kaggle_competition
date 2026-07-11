---
name: ship-experiment
description: Finish the current experiment - commit remaining work, push the experiment branch, and open a PR against main WITHOUT merging it. Use when the user says the experiment is done, wants to push results, or wants to open a PR for the current work.
---

# Ship Experiment

Finalize the current experiment branch per the project workflow in CLAUDE.md.

## Steps

1. **Verify you are on an experiment branch** (`erd_exp/*` or `bet_exp/*`). If on `main`, STOP — never commit experiment work to main. Offer to move the work to a new experiment branch with `/new-experiment` instead.

2. **Commit any remaining work** on the branch with a clear message summarizing the experiment. Review `git status` and `git diff` first; don't blindly `git add -A` files that look unrelated or accidental (large data files, secrets, OS junk like `.DS_Store` / `Thumbs.db`).

3. **Push the branch:**
   ```
   git push -u origin <branch_name>
   ```

4. **Open a PR against main** with `gh pr create`. The PR body should include:
   - **Goal:** what the experiment tried
   - **Approach:** what was done / changed
   - **Results:** metrics, findings, or outcome (ask the user if unknown)
   - **Files:** where the experiment lives in the shared file structure

5. **DO NOT merge the PR.** Do not enable auto-merge. Do not delete the branch. The PR stays open as the record of the experiment. Report the PR URL to the user and stop.

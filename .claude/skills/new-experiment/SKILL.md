---
name: new-experiment
description: Start a new experiment branch with the correct owner prefix (erd_exp/ on Erdem's Windows machine, bet_exp/ on Betül's Mac), branched from the latest main. Use when the user wants to start a new experiment, new work item, or a new branch for this project.
---

# New Experiment

Create a new experiment branch following the project workflow in CLAUDE.md.

Argument: the experiment name. If not given, ask the user for a short name, then convert it to lowercase snake_case.

## Steps

1. **Determine the prefix from the OS platform:**
   - Windows (win32) → `erd_exp/` (Erdem's machine)
   - macOS (darwin) → `bet_exp/` (Betül's machine)
   - Anything else → ask the user which prefix to use.

2. **Check for uncommitted changes.** If the working tree is dirty, stop and ask the user whether to commit them on the current branch or stash them first. Never carry uncommitted changes silently onto the new branch.

3. **Sync main:**
   ```
   git checkout main
   git pull origin main
   ```

4. **Create the branch:**
   ```
   git checkout -b <prefix>/<experiment_name>
   ```
   Example: `erd_exp/baseline_model`

5. Tell the user the branch is ready and remind them: experiment files must be added **into the existing file structure from main** (extend the skeleton, don't create a parallel layout), and the experiment will end with `/ship-experiment` (push + PR, no merge).

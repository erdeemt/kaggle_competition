# Project Workflow Rules

This repository is a shared project between two collaborators. Claude Code MUST follow these rules on every machine.

## Collaborators & machine detection

| Person | Machine | Branch prefix |
|--------|---------|---------------|
| Erdem  | Windows (win32) | `erd_exp/` |
| Betül  | macOS (darwin)  | `bet_exp/` |

Detect whose machine you are on by the OS platform:
- **Windows** → you are on Erdem's machine → use the `erd_exp/` prefix.
- **macOS** → you are on Betül's machine → use the `bet_exp/` prefix.

If the platform is ambiguous, fall back to `git config user.name` and ask the user.

## Branching model — "experiments"

- `main` holds only the **shared file structure** (skeleton). Erdem maintains it.
- Every piece of work is an **experiment**. One experiment = one branch, named
  `<prefix>/<experiment_name>` (e.g. `erd_exp/baseline_cnn`, `bet_exp/feature_engineering`).
- Experiment names: short, lowercase, snake_case, descriptive of the work.

### Starting an experiment
1. `git checkout main` and `git pull origin main` — always branch from the **latest** main.
2. `git checkout -b <prefix>/<experiment_name>`.
3. Add the experiment's files **into the existing main file structure** — do not invent a parallel layout; extend the skeleton that main defines.

### Finishing an experiment
1. Commit the work on the experiment branch (never on `main`).
2. Push the branch: `git push -u origin <prefix>/<experiment_name>`.
3. Open a PR against `main` with `gh pr create`, summarizing the experiment and its results.
4. **NEVER merge the PR. NEVER merge anything into `main`.** PRs stay open as the record of each experiment. Do not delete experiment branches either.

## Hard rules (do not violate)

- ❌ No commits directly on `main` (exception: Erdem explicitly asking to update the shared file structure).
- ❌ No merging PRs, no `git merge` into `main`, no squash/rebase onto `main`.
- ❌ No deleting or force-pushing experiment branches.
- ✅ Always pull latest `main` before creating a new experiment branch.
- ✅ Always open a PR at the end of an experiment (`/ship-experiment` skill does this).

## Skills

- `/new-experiment <name>` — creates a correctly-prefixed experiment branch from latest main.
- `/ship-experiment` — commits, pushes the current experiment branch, and opens a PR (without merging).

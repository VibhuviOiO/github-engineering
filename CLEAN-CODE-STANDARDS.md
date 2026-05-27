# Clean Code Standards

These standards apply to shared reusable workflows and actions in this repository.

## Design rules

- Keep cross-cutting actions generic: deploy, notify, approval, artifact upload, release, and SecOps utilities should not contain stack-specific branching.
- Keep stack-specific logic isolated by technology: Python, Java, and Node.js actions should live in technology-specific folders or reusable workflows.
- Prefer inputs over duplication: environment, deployment target, artifact names, and notification targets should be parameters.
- Keep actions small and composable: one action should represent one concern.

## Workflow rules

- Reusable workflows should stay thin and orchestrate actions rather than embed long shell logic.
- Job names should be short and UI-friendly.
- Shared workflows should expose a stable input interface and avoid leaking caller-specific assumptions.
- Prefer one clear source of truth for trigger logic: caller workflows decide when to run, shared workflows decide how to run.

## Repository rules

- Place generic CI actions in `.github/actions/ci/`.
- Place stack-specific CI actions under technology folders such as `.github/actions/ci/python/`.
- Place security actions in `.github/actions/secops/`.
- Place notifications and operational helpers in `.github/actions/utils/`.
- Do not keep duplicate actions in multiple folders.

## Naming rules

- Use action folder names that describe one capability.
- Use workflow names that describe one technology or one delivery path.
- Use environment names as inputs, not as separate deploy action folders.
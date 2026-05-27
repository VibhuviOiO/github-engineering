# Scaffolding

This repository is intended to grow as a shared engineering actions library.

## Current layout

- `.github/actions/ci/`: generic CI actions
- `.github/actions/ci/install/python/`: Python-specific install actions
- `.github/actions/ci/quality/python/`: Python-specific quality actions
- `.github/actions/ci/test/python/`: Python-specific test actions
- `.github/actions/secops/`: security and compliance actions
- `.github/actions/utils/`: notifications and operational helpers
- `.github/workflows/`: reusable workflows built from those actions

## Recommended next folders

- `.github/actions/ci/install/java/`
- `.github/actions/ci/install/node/`
- `.github/actions/ci/quality/java/`
- `.github/actions/ci/quality/node/`
- `.github/actions/ci/test/java/`
- `.github/actions/ci/test/node/`
- `.github/workflows/python/`
- `.github/workflows/java/`
- `.github/workflows/node/`

## Suggested starter actions per stack

Python:

- `pip-install`
- `pytest-run`
- `ruff-check`

Java:

- `maven-build`
- `gradle-build`
- `junit-run`

Node:

- `npm-install`
- `npm-test`
- `npm-build`

## Extension pattern

1. Add or reuse a small action under the correct folder.
2. Compose it into one reusable workflow for that technology.
3. Keep deploy, notify, approval, and SecOps actions generic unless a real stack-specific need appears.
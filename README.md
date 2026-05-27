# Github engineering

Shared reusable GitHub Actions content for application repositories.

See `CLEAN-CODE-STANDARDS.md` for repository conventions and `SCAFFOLDING.md` for the recommended growth structure.

Folder layout under `.github/actions`:

- `ci/`: main CI actions such as build, deploy, artifact upload, release versioning, approval, and capability-based stack folders
- `ci/install/python/`: Python-specific install actions such as `pip-install`
- `ci/quality/python/`: Python-specific quality actions
- `ci/test/python/`: Python-specific test actions
- `secops/`: security and DevSecOps actions such as dependency scan, secret scan, SAST, SBOM, image scan, and IaC scan
- `utils/`: notification and operational utility actions such as notify, rollback notify, and incident notify

Recommended repository structure for multiple tech stacks:

- Keep cross-technology actions generic and reusable: deploy, notify, approval, artifact upload, SBOM, image scan, IaC scan.
- Keep technology-specific actions separate by technology: Python, Java, and Node.js build/test/package actions should live in their own action folders or reusable workflows.
- Prefer capability-first stack folders under `ci/`, for example `ci/install/python/`, `ci/quality/python/`, `ci/test/python/`, and the equivalent Java and Node paths later.
- Treat environments as inputs, not folder names. Deployment logic should use one deploy action with environment parameters rather than one folder per environment.

Practical rule:

- Technology-specific concerns belong to technology-specific actions or workflows.
- Environment-specific concerns belong to workflow inputs and deployment parameters.

Current shared FastAPI pipeline use cases:

- `fast-api-ci.yml`: one reusable workflow with core CI always on and optional jobs for artifact upload, dependency scan, SBOM, secret scan, SAST, image scan, IaC scan, release, approval, deploy, and notifications

Caller-controlled feature flags:

- Consumers decide repository triggers in their own workflow file.
- Consumers enable only the optional jobs they want by setting boolean inputs on the single reusable workflow call.
- Keep install, quality, build, and test as the always-on path. Turn the rest on only when the team is ready.

Example caller overrides:

- Disable only SBOM: `run-sbom: false`
- Keep security checks but disable deployment: `run-deploy: false`
- Enable deploy only on `develop`: `run-deploy: ${{ github.event_name == 'push' && github.ref == 'refs/heads/develop' }}`

Notification utility actions with dummy targets:

- `notify-slack`: channel `#engineering-alerts`, dummy webhook URL
- `notify-pagerduty`: service `PDEV123`, dummy integration key
- `notify-email`: recipient `engineering-alerts@example.com`, SMTP host `smtp.example.com`
- `notify-generic`: endpoint `https://example.internal/hooks/notify`
# Security and AppSec configs

### A small, practical toolbox for adding security checks to everyday software delivery.

This directory collects copyable configuration for the controls that tend to matter
early in a project: finding leaked credentials, reviewing dependencies, scanning
source trees and containers, and keeping runtime images small and non-root.

The files are intentionally boring. They use familiar formats, explicit versions,
and narrow defaults so an engineer can understand and adapt them during a normal
pull request review.

## Start here

| Need | Use | Where it belongs in a project |
| --- | --- | --- |
| Scan commits for credentials | [Reusable secret scanning workflow](actions/secret-scan.yaml) | Called from `.github/workflows/security.yaml` |
| Add a general CI security gate | [Reusable security baseline](actions/security-baseline.yaml) | Called from `.github/workflows/security.yaml` |
| Scan source code with Semgrep CLI | [Reusable Semgrep workflow](actions/semgrep.yaml) | Called from `.github/workflows/security.yaml` |
| Keep actions and npm packages current | [Dependabot config](https://github.com/Felipemguerra/configs/blob/master/dependabot/dependabot.yml) | `.github/dependabot.yml` |
| Check secrets before they reach CI | [Pre-commit hooks](pre-commit/.pre-commit-config.yaml) | `.pre-commit-config.yaml` |
| Build a smaller Node image as a non-root user | [Node Dockerfile](docker/node.Dockerfile) | Project `Dockerfile` |
| Keep sensitive files out of image builds | [Docker ignore file](docker/.dockerignore) | Project `.dockerignore` |
| Tune local secret-detection rules | [Gitleaks config](gitleaks/config.toml) | Project Gitleaks config path |

## Reusable GitHub workflows

The workflows in `actions/` are reusable workflows. A consuming repository owns
the trigger and calls the central workflows with a normal job-level `uses` entry.
For example, this runs all three scans for pull requests targeting `master`:

```yaml
name: Security scans

on:
  pull_request:
      branches:
        - master
      types: [opened, reopened]

permissions:
	contents: read
	security-events: write
	pull-requests: write

jobs:
	baseline:
		uses: Felipemguerra/configs/.github/workflows/security-baseline.yaml@<reviewed-ref>

	semgrep:
		uses: Felipemguerra/configs/.github/workflows/semgrep.yaml@<reviewed-ref>

	secret-scan:
		uses: Felipemguerra/configs/.github/workflows/secret-scan.yaml@<reviewed-ref>
		with:
			configs-ref: <reviewed-ref>
```

Replace `<reviewed-ref>` with an immutable commit SHA or a reviewed release tag
after publishing the central workflow. Do not use a floating branch for production
consumers. The caller's `contents: read` permission allows checkout and scanning;
`security-events: write` is used for SARIF uploads, and `pull-requests: write` is
used by Dependency Review. The optional `GITLEAKS_LICENSE` secret is inherited only
by the secret-scan call and is needed for some organization accounts.

The secret workflow checks out `gitleaks/config.toml` from this repository into the
caller run. Set `configs-ref` to the same reviewed ref as the workflow to keep the
rules and workflow synchronized. Pull request, push, schedule, and manual triggers
belong in each consuming repository; reusable workflows do not define those events.
The Semgrep workflow uses Semgrep's native CLI and uploads SARIF results to GitHub
code scanning. Its default registry ruleset is `p/default`; callers can provide a
different registry ruleset or a configuration path through the `config` input.

## Copyable configuration

The workflows can still be copied when a repository needs to customize them locally.
For a GitHub repository using local copies, the smallest useful starting point is the
secret scan and security baseline:

```sh
mkdir -p .github/workflows
cp actions/secret-scan.yaml .github/workflows/secret-scan.yaml
cp actions/security-baseline.yaml .github/workflows/security-baseline.yaml
cp -R gitleaks .
```

Then add the desired repository triggers, commit the files, and open a pull request.
The baseline runs dependency review for pull requests, scans the repository with
Trivy, and uploads SARIF results to GitHub code scanning. A copied secret workflow
should use the local `gitleaks/config.toml` path.

For Dependabot, copy the file to the exact location GitHub expects:

```sh
mkdir -p .github
cp dependabot/dependabot.yml .github/dependabot.yml
```

## Container baseline

The [Node Dockerfile](docker/node.Dockerfile) uses a build stage for development
dependencies and a smaller runtime stage. It also runs the application as the
built-in `node` user rather than root. The example assumes:

- `npm ci` installs dependencies from a lockfile.
- `npm run build` creates `dist/`.
- `dist/server.js` starts the application.
- The application listens on port `3000`.

Change those application-specific details when needed, but preserve the multi-stage
build and non-root runtime unless there is a documented reason not to. At runtime,
add platform-specific restrictions such as a read-only root filesystem, dropped
Linux capabilities, resource limits, and a network policy where supported.

## Important operating notes

- Treat action and tool versions as dependencies. Review and update them through the
	same process used for application dependencies.
- A secret finding is an incident signal. Revoke and rotate the credential even if
	the offending commit is later removed from the current branch.
- Prefer a narrow, reviewed exception to a broad allowlist. An allowlist should
	explain why a finding is safe and who owns the decision.
- These are starting points, not a complete security program. Add language-specific
	testing, license policy, IaC scanning, image signing, and deployment controls to
	match the system's threat model.

## Repository layout

```text
actions/       GitHub Actions workflows
dependabot/    Dependency update configuration
docker/        Container build defaults
gitleaks/      Secret-detection rules
pre-commit/    Local developer hooks
```

The examples are designed to be copied, reviewed, and changed. They should make the
secure path easier to take without pretending that one configuration fits every
application.

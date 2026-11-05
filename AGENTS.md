# AGENTS.md — nabu

Guide for coding agents and developers working in this repository.

## Workspace

Nabu is three repositories checked out side by side: `nabu-core` (Go backend), `nabu-web` (SPA) and
`nabu` (this one: public docs and the reusable deploy workflow). As with Hammurapi, the charts
`nabu-core`/`nabu-web` and the deploy script `bin/nabu-deploy` live in `installer`, the
repository of the stand. The specification is `hammurapi-specs/specs/NAB/CMN/FTR.NAB.CMN-0001/`.

- Deviations from a spec are written into the spec (version bump, "Принятые решения").
- Do not commit, tag or push unless asked.
- English for code, comments, docs, CI, scripts, logs, test names, and the messages and tool descriptions that services return. Russian stays in test data that exercises Russian text and the specifications in `hammurapi-specs`.

## Contents

```text
README.md, nabu.jpg      overview with the banner
docs/                    deployment, configuration (every variable of nabu-core), connecting Hammurapi
.github/workflows/deploy-component.yml   reusable deploy called by nabu-core and nabu-web releases
.github/workflows/checks.yml             actionlint and the payload of the deploy
.github/actionlint.yaml                  checks of the deploy are called through check (SC2317/SC2329)
```

A tag of this repository does not touch the machine: it is only a version of `deploy-component.yml`.
Every change that adds a setting or behaviour visible to operators updates `docs/`
(configuration.md lists every environment variable).

## Checks before a commit

A change is ready when every job of `.github/workflows/checks.yml` passes locally at the versions the workflow pins, plus the
matching lines below. Read the workflow for the exact commands. Install a missing tool outside the repository
at the pinned version. Name any job you could not run in your report, with the reason.

- `deploy-component.yml`: its post-deploy checks describe the charts of `installer`; compare them with the routes of the chart version being released, and tag this repository before bumping `DEPLOY_WORKFLOW_REF`.
- Always, after the checks pass: run the `code-review` skill on the diff to be committed. Fix each finding, or name it in your report with the reason it stays. Ask the user when you doubt a finding.

## Rules

- Configuration and secrets reach the machine as one JSON on the stdin of SSH, built by `jq` from
  `$ENV` — never in arguments, files or logs. `checks.yml` compiles the payload; keep it passing.
- The workflow runs `/opt/hammurapi-infra/current/bin/nabu-deploy`; the chart version is the
  `CHART_VERSION` of the caller (a release of `installer`).
- The workflow is YAML: edit it by hand; do not run generic formatters. `actionlint` must pass with
  shellcheck 0.9 (the runner) and newer; pin actions by SHA.
- Public hosts are `nabu.<domain>` and `nabu-api.<domain>`; they are in `PUBLIC_EXTRA_HOSTS` of the
  stand certificate (`installer`).

## Releases

Charts or `nabu-deploy` changed → tag `installer`. The workflow changed → tag this repository.
Then bump `DEPLOY_WORKFLOW_REF` and `CHART_VERSION` in `nabu-core` and `nabu-web`
(`deploy/sync-ref.sh`) and tag core and web.

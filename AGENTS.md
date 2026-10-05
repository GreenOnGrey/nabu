# AGENTS.md — nabu

Guide for coding agents and developers working in this repository.

## Workspace

Nabu is three repositories checked out side by side: `nabu-core` (Go backend), `nabu-web` (SPA) and
`nabu` (this one: public docs and the reusable deploy workflow). As with Hammurapi, the charts
`nabu-core`/`nabu-web` and the deploy script `bin/nabu-deploy` live in `hammurapi-infra`, the
repository of the stand. The specification is `hammurapi-specs/specs/NAB/CMN/FTR.NAB.CMN-0001/`.

- Deviations from a spec are written into the spec (version bump, "Принятые решения").
- Do not commit, tag or push unless asked.

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

## Rules

- Configuration and secrets reach the machine as one JSON on the stdin of SSH, built by `jq` from
  `$ENV` — never in arguments, files or logs. `checks.yml` compiles the payload; keep it passing.
- The workflow runs `/opt/hammurapi-infra/current/bin/nabu-deploy`; the chart version is the
  `CHART_VERSION` of the caller (a release of `hammurapi-infra`).
- The workflow is YAML: edit it by hand; do not run generic formatters. `actionlint` must pass with
  shellcheck 0.9 (the runner) and newer; pin actions by SHA.
- Public hosts are `nabu.<domain>` and `nabu-api.<domain>`; they are in `PUBLIC_EXTRA_HOSTS` of the
  stand certificate (`hammurapi-infra`).

## Releases

Charts or `nabu-deploy` changed → tag `hammurapi-infra`. The workflow changed → tag this repository.
Then bump `DEPLOY_WORKFLOW_REF` and `CHART_VERSION` in `nabu-core` and `nabu-web`
(`deploy/sync-ref.sh`) and tag core and web.

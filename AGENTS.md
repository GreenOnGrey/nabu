# AGENTS.md — nabu

Guide for coding agents and developers working in this repository.

## Workspace

Nabu is three repositories checked out side by side: `nabu-core` (Go backend), `nabu-web` (SPA) and
`nabu` (this one: docs, charts, `bin/nabu-deploy`, the reusable deploy workflow). Nabu is deployed to
the Hammurapi stand prepared by `hammurapi-infra`; the specification is
`hammurapi-specs/specs/NAB/CMN/FTR.NAB.CMN-0001/`.

- Deviations from a spec are written into the spec (version bump, "Принятые решения").
- Do not commit, tag or push unless asked.

## Contents

```text
README.md, nabu.jpg      overview with the banner
docs/                    deployment, configuration (every variable of nabu-core), connecting Hammurapi
charts/nabu-core/        api, worker, agent, relay, migrate hook, cleaner, RBAC for sandboxes, NetworkPolicies
charts/nabu-web/         the SPA
bin/nabu-deploy          runs on the stand machine; sources /opt/hammurapi-infra/current/lib/common.sh
config/versions.env      Helm version and checksum used by the workflows
.github/workflows/deploy-component.yml   reusable deploy called by nabu-core and nabu-web releases
.github/workflows/release.yml            tag vX.Y.Z: charts to oci://ghcr.io/<org>/charts, bin to /opt/nabu/releases/<tag>
.github/workflows/checks.yml             actionlint, shellcheck, helm lint
```

Every change that adds a setting or behaviour visible to operators updates `docs/`
(configuration.md lists every environment variable).

## Rules

- Configuration and secrets reach the machine as one JSON on the stdin of SSH, built by `jq` from
  `$ENV` — never in arguments, files or logs. `nabu-deploy` keeps them in memory.
- `SECRETS_KEY` must not change silently: `nabu-deploy` checks its fingerprint (exit 24) unless the run
  passes `allow_key_rotation`.
- The database is created through the primary pod of the stand's CloudNativePG cluster with the
  password on stdin; do not pass passwords in `psql -c`.
- The charts, workflows and scripts are YAML and shell: edit them by hand; do not run generic
  formatters. `shellcheck` and `actionlint` must pass; pin actions by SHA.
- Public hosts are `nabu.<domain>` and `nabu-api.<domain>`; they are in `PUBLIC_EXTRA_HOSTS` of the
  stand certificate (`hammurapi-infra`).

## Releases

Tag this repository → bump `DEPLOY_WORKFLOW_REF` and `CHART_VERSION` in `nabu-core` and `nabu-web`
(`deploy/sync-ref.sh`) → tag core and web. A release of core or web runs `deploy-component.yml` of the
pinned tag, which calls `/opt/nabu/current/bin/nabu-deploy`.

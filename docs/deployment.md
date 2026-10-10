# Deployment

Nabu runs on the Hammurapi stand: the same minikube machine, the namespaces `nabu`,
`nabu-sandboxes` and `nabu-agents` (a pod per user's agent), the stand's Postgres (CloudNativePG), Kafka (Strimzi) and S3 (SeaweedFS) with a
separate database, bucket and topic prefix. It is deployed exactly like Hammurapi: the charts
`nabu-core`, `nabu-web` and the script `bin/nabu-deploy` live in `hammurapi-infra` next to those of
Hammurapi, and this repository holds only the docs and the reusable deploy workflow. A tag of this
repository does not touch the machine.

```text
hammurapi-infra tag vX.Y.Z → deploy-prod.yml: charts hammurapi-*, nabu-* (version X.Y.Z) to
                             oci://ghcr.io/<org>/charts; the stand with bin/nabu-deploy in
                             /opt/hammurapi-infra/current
nabu tag vX.Y.Z            → only a version of deploy-component.yml for core and web
nabu-core / nabu-web tag   → release.yml there: image, SBOM, cosign signature, Trivy,
                             then deploy-component.yml of this repo (DEPLOY_WORKFLOW_REF)
deploy-component.yml       → configuration check, signature check, SSH (or IAP) to the machine,
                             JSON with configuration and secrets on stdin of
                             /opt/hammurapi-infra/current/bin/nabu-deploy, checks through the public domains
```

`nabu-core/deploy/versions.env` and `nabu-web/deploy/versions.env` pin `DEPLOY_WORKFLOW_REF` (a tag
of this repository) and `CHART_VERSION` (a release of `hammurapi-infra`). The order of a change:
charts or `nabu-deploy` → tag `hammurapi-infra`; the workflow → tag this repository; then bump
`deploy/versions.env` in core and web (`deploy/sync-ref.sh`) and tag them.

## Before the first release

1. **The stand.** `hammurapi-infra` is installed (`/opt/hammurapi-infra/current`) with the public
   domain. Its certificate must include `nabu.<domain>` and `nabu-api.<domain>`: they are in
   `PUBLIC_EXTRA_HOSTS` of `hammurapi-infra/config/defaults.env` from the infra release that added Nabu;
   `nabu-deploy` also adds them if they are missing.
2. **GitHub OAuth App for sign-in** (organization → Settings → Developer settings → OAuth Apps):
   - Homepage URL: `https://nabu.<domain>`
   - Authorization callback URL: `https://nabu-api.<domain>/auth/callback`
   - The App needs no permissions beyond sign-in; Nabu asks for `read:user user:email read:org` and
     lets in only members of `NABU_GITHUB_ALLOWED_ORG` (by default the owner of the repository that
     runs the workflow, i.e. `GreenOnGrey`).
3. **Organization variables and secrets** (organization → Settings → Secrets and variables → Actions).

### Variables

| Variable | Purpose | Default |
| --- | --- | --- |
| `NABU_BOOTSTRAP_ADMINS` | Emails of the first global administrators (comma-separated) | — (required) |
| `NABU_AUTH_PROVIDER` | `github` or `oidc` | `github` |
| `NABU_GITHUB_CLIENT_ID` | Client ID of the OAuth App above | — (required for `github`) |
| `NABU_GITHUB_ALLOWED_ORG` | Only members of this organization may sign in | the repository owner |
| `NABU_OIDC_ISSUER`, `NABU_OIDC_CLIENT_ID`, `NABU_OIDC_PROVIDER_NAME` | OIDC sign-in | — |
| `NABU_DEFAULT_LANGUAGE`, `NABU_DEFAULT_TIMEZONE` | Defaults of new users | `en`, `Europe/Moscow` |
| `NABU_WHISPER_URL` | Speech recognition for voice messages; empty — voice input is off | — |
| `DEPLOY_TRANSPORT`, `PROD_HOST`, `PROD_USER`, `GCP_*` | Access to the machine — the same as for Hammurapi | `ssh` |

### Secrets

| Secret | Purpose |
| --- | --- |
| `NABU_SECRETS_KEY` | 32-byte key (`openssl rand -base64 32`) that encrypts LLM keys, tokens and signs JWTs. **Never change it**: `nabu-deploy` stores its fingerprint and refuses another key unless the run asks for `allow_key_rotation` |
| `NABU_GITHUB_CLIENT_SECRET` | Secret of the OAuth App (for `github`) |
| `NABU_OIDC_CLIENT_SECRET` | Secret of the OIDC client (for `oidc`) |
| `NABU_GHCR_PULL_TOKEN` | Pulls images and charts from ghcr.io; falls back to `HAMMURAPI_GHCR_PULL_TOKEN` |
| `NABU_BOOTSTRAP_LLM_API_KEY` | Optional: the first LLM connection (DeepSeek); later — Admin → Models |
| `NABU_TELEGRAM_BOT_TOKEN` | Optional: the Telegram bot; the webhook secret is generated and kept |
| `NABU_SKILLS_GIT_TOKEN` | Optional: reads skill repositories of the catalog |
| `NABU_DEPLOY_CALLBACK_SECRET` | Optional: signs the deploy result callback (deploy contract of Hammurapi) |
| `PROD_SSH_KEY`, `PROD_KNOWN_HOSTS` | SSH to the machine — the same as for Hammurapi |

Secrets travel as one JSON on the stdin of SSH and land in the Secret `nabu-secrets`; they never
appear in command arguments, files or logs.

## What `nabu-deploy` does

1. Checks the stand (`hammurapi-infra.sh status`), takes a lock.
2. Creates the namespaces `nabu`, `nabu-sandboxes` and `nabu-agents` (with the `restricted` pod
   security level), the pull secret `ghcr-pull` in each.
3. Writes `nabu-config` (ConfigMap) and `nabu-secrets` (Secret); keeps `AGENT_SERVICE_TOKEN` and
   `TELEGRAM_WEBHOOK_SECRET` across releases; checks the fingerprint of `SECRETS_KEY`.
4. Creates the role and the database `nabu` in the stand's Postgres (through the primary pod, the
   password on stdin) and `nabu-postgres-conn`; copies `kafka-conn`; creates the bucket `nabu` and
   `nabu-s3-conn`.
5. Adds `nabu.<domain>` and `nabu-api.<domain>` to the stand certificate if needed.
6. `helm upgrade --install` of `nabu-core` or `nabu-web` from `oci://ghcr.io/greenongrey/charts`
   (`--atomic`, migrations in a pre-upgrade hook), then waits for the rollout.
7. Prints the JSON result: `webUrl`, `apiUrl`, `revision`, `firstInstall`.

Exit codes: 17 a deploy is running; 20 the stand is not ready; 21 web without core; 22 namespaces;
23 configuration and secrets; 24 `SECRETS_KEY` changed; 25 Helm; 26 database; 27 pods not ready.

After the deploy the workflow checks `https://nabu-api.<domain>/api/v1/config`, the JWKS, the redirect
of `/auth/login`, that `/metrics` is closed, the relay endpoint and the site.

## First sign-in

Open `https://nabu.<domain>`, sign in with GitHub as one of `NABU_BOOTSTRAP_ADMINS` and set up
**Admin → Models** (or rely on `NABU_BOOTSTRAP_LLM_API_KEY`), the catalog and the clients. To connect
Hammurapi see [hammurapi.md](hammurapi.md).

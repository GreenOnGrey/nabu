# Configuration

`nabu-core` is one binary; the mode is the first argument (`api`, `worker`, `agent`, `relay`,
`sandbox`, `migrate`, `cleaner`). Everything is set by environment variables. On the stand the chart
and `bin/nabu-deploy` set them; this page is the reference. Durations are Go durations with `d` for
days (`15m`, `365d`); sizes accept `KB`, `MB`, `GB`.

## URLs and listeners

| Variable | Purpose | Default |
| --- | --- | --- |
| `PUBLIC_WEB_URL` | The site (`https://nabu.<domain>`) | `http://localhost:5173` |
| `PUBLIC_API_URL` | The API (`https://nabu-api.<domain>`); also the issuer of Nabu's JWTs | `http://localhost:8080` |
| `INTERNAL_URL` | The internal server of `api` as the agent operator and sandboxes reach it | `http://localhost:8081` |
| `COOKIE_DOMAIN` | Domain of the session cookies shared by the site and the API | — |
| `CORS_ALLOWED_ORIGINS` | Origins allowed to call the API with cookies | — |
| `HTTP_ADDR`, `INTERNAL_ADDR`, `SERVICE_ADDR` | Public API, internal API (MCP, sandbox sync), health and metrics | `:8080`, `:8081`, `:9100` |
| `RELAY_ADDR` | Listener of `relay` | `:8085` |
| `RELAY_INTERNAL_URL` | `relay` as `agent` reaches it | `http://localhost:8085` |
| `RELAY_SANDBOX_WS_URL` | `relay` as sandboxes reach it | from `RELAY_INTERNAL_URL` |
| `RELAY_PUBLIC_WS_URL` | `relay` as external workspaces (runners of clients) reach it | `wss://<PUBLIC_API_URL>/v1/workspaces/connect` |

## Sign-in

| Variable | Purpose | Default |
| --- | --- | --- |
| `AUTH_PROVIDER` | `github` or `oidc` | `github` |
| `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET` | OAuth App; callback `<PUBLIC_API_URL>/auth/callback` | — (required for `github`) |
| `GITHUB_ALLOWED_ORG` | Only members of this organization may sign in | — |
| `GITHUB_BASE_URL`, `GITHUB_API_URL` | GitHub Enterprise | `https://github.com`, `https://api.github.com` |
| `OIDC_ISSUER`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET` | OIDC provider (PKCE; a verified email is required) | — (required for `oidc`) |
| `OIDC_SCOPES`, `OIDC_PROVIDER_NAME` | Scopes and the name on the sign-in button | `openid email profile`, `Keycloak` |
| `BOOTSTRAP_ADMINS` | Emails that become global administrators on the first sign-in | — |
| `DEFAULT_LANGUAGE`, `DEFAULT_TIMEZONE` | Defaults of new users (`en`, `ru`, `de`, `es`, `zh`) | `en`, `Europe/Moscow` |

## Secrets and storage

| Variable | Purpose | Default |
| --- | --- | --- |
| `SECRETS_KEY` | 32 bytes, base64 or hex: encrypts LLM keys and tokens, signs JWTs | — (required) |
| `DATABASE_URL` | Postgres | — (required) |
| `KAFKA_BROKERS` | Comma-separated brokers | — (required except `migrate`, `relay`, `cleaner`) |
| `S3_ENDPOINT`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_BUCKET`, `S3_USE_SSL` | Object storage: attachments, spaces, session snapshots | bucket `nabu` |

## Agent

| Variable | Purpose | Default |
| --- | --- | --- |
| `AGENT_ADDR` | The agent operator as `api` and `worker` reach it | `http://localhost:8090` |
| `AGENT_SERVICE_TOKEN` | Shared token of `api`, `worker` and `agent` | — (required for `worker`, `agent`) |
| `AGENT_IDLE_TIMEOUT` | An idle session is saved to S3 and closed | `15m` |
| `TURN_TIMEOUT` | Longest turn of the agent | `30m` |
| `INBOUND_WORKERS` | Turns processed in parallel by one `worker` | `4` |
| `RUN_WORKSPACE_WAIT` | How long a run of a service agent waits for its external workspace | `120s` |
| `BOOTSTRAP_LLM_API_KEY`, `BOOTSTRAP_LLM_BASE_URL`, `BOOTSTRAP_LLM_TYPE` | The first LLM connection, created once when there are none | —, `https://api.deepseek.com`, `deepseek` |
| `WHISPER_URL` | Speech recognition for voice input | — (voice off) |

Agent operator only (`agent`):

| Variable | Purpose | Default |
| --- | --- | --- |
| `AGENT_LISTEN_ADDR` | Listener of the operator | `:8090` |
| `AGENT_MAX_SESSIONS`, `AGENT_MAX_RUNS` | Pi processes in total and, of them, runs of service agents | `100`, `10` |
| `AGENT_WORKDIR` | Directories of sessions | `/work` |
| `PI_BINARY`, `PI_EXTENSION_DIR` | Pi and the `nabu-workspace` extension (set in the release image) | `/usr/local/bin/pi`, `/opt/nabu/pi-extensions/nabu-workspace` |
| `PI_EXTRA_ENV` | Extra environment of Pi, space-separated `KEY=VALUE` (e.g. a proxy) | — |

## Spaces (personal sandboxes)

| Variable | Purpose | Default |
| --- | --- | --- |
| `SANDBOX_EXECUTOR` | `k8s` — a pod per user; `none` — no sandboxes | `none` |
| `SANDBOX_NAMESPACE`, `SANDBOX_IMAGE` | Namespace and image of sandbox pods | `nabu-sandboxes`, — |
| `SANDBOX_IDLE_TIMEOUT` | An idle sandbox is synced and stopped | `30m` |
| `SANDBOX_CPU`, `SANDBOX_MEMORY` | Limits of a sandbox pod | `1`, `1Gi` |
| `SPACE_QUOTA` | Size of a user's space | `5GB` |
| `SANDBOX_EGRESS_ALLOW` | CIDRs sandboxes may connect to | — |
| `SANDBOX_EXCLUDE` | Extra patterns not synced to S3 (besides `node_modules/`, `.venv/`, `__pycache__/`, `*.tmp`) | — |

## Tasks, channels, retention

| Variable | Purpose | Default |
| --- | --- | --- |
| `TASK_TICK`, `TASK_RUN_TIMEOUT` | Scheduler tick and the longest run of a task | `15s`, `30m` |
| `TASK_MIN_INTERVAL`, `TASK_MAX_ACTIVE`, `TASK_MAX_FAILURES` | Limits of recurring tasks per user | `15m`, `20`, `3` |
| `TASK_CATCHUP_WINDOW` | After downtime only the last missed run within this window is executed | `1h` |
| `TELEGRAM_BOT_TOKEN`, `TELEGRAM_WEBHOOK_SECRET` | Telegram channel (webhook `/hooks/v1/telegram/<secret>`) | — |
| `TELEGRAM_API_URL` | Bot API | `https://api.telegram.org` |
| `AUDIT_RETENTION` | Audit partitions older than this are dropped | `365d` |
| `TOPIC_ARCHIVE_AFTER` | Idle topic conversations are archived | `14d` |
| `UPLOAD_MAX_BYTES` | Largest attachment | `50MB` |

## Observability

| Variable | Purpose | Default |
| --- | --- | --- |
| `LOG_LEVEL` | `debug`, `info`, `warn`, `error` | `info` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Traces | — |

Metrics are served on `SERVICE_ADDR` at `/metrics`; health at `/healthz` and `/readyz`.

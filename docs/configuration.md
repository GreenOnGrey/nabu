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

Agent pods (`worker`; `api` reads `AGENT_EXECUTOR`). The agent of every user and group agent runs
in its own pod — see [isolation.md](isolation.md):

| Variable | Meaning | Default |
| --- | --- | --- |
| `AGENT_EXECUTOR` | `k8s` — a pod per owner; `local` — every session in the operator at `AGENT_ADDR` (development) | `local` |
| `AGENT_NAMESPACE` | Namespace of agent pods | `nabu-agents` |
| `AGENT_IMAGE` | Image of an agent pod | — (required with `k8s`) |
| `AGENT_POD_CPU_REQUEST`, `AGENT_POD_CPU` | CPU request and limit of a pod | `100m`, `1` |
| `AGENT_POD_MEMORY_REQUEST`, `AGENT_POD_MEMORY` | Memory request and limit of a pod | `256Mi`, `1Gi` |
| `AGENT_POD_WORK` | Size of the working directory of a pod | `1GB` |
| `AGENT_POD_MAX_SESSIONS` | Sessions in a pod | `8` |
| `AGENT_POD_IDLE_TIMEOUT` | A pod without turns is stopped | `15m` |
| `AGENT_POD_MIN_IDLE` | An idle pod may give its place away after this time | `60s` |
| `AGENT_PODS_MAX` | Ceiling of pods; `0` — from the quota of the namespace, otherwise learned from the cluster | `0` |
| `AGENT_START_PARALLEL` | Pods starting at the same time | `10` |
| `AGENT_START_TIMEOUT` | A placed pod must become ready within this time | `60s` |
| `AGENT_SCHEDULE_TIMEOUT` | A pod the cluster cannot place for this long sets the ceiling; raise it to the time a node takes to appear when nodes scale automatically | `15s` |
| `AGENT_QUEUE_TIMEOUT` | Longest wait of a turn for a pod | `10m` |
| `AGENT_WARM_WINDOW` | Activity window of the warm reserve | `3h` |
| `AGENT_WARM_SHARE` | Share of the ceiling kept warm for the most active owners | `0.3` |
| `AGENT_WARM_MAX` | Limit of the warm reserve; `0` — none | `0` |

Agent operator only (`agent`):

| Variable | Purpose | Default |
| --- | --- | --- |
| `AGENT_LISTEN_ADDR` | Listener of the operator | `:8090` |
| `AGENT_MODE` | `pool` — runs of service agents and checks only; `owner` — the pod of one owner (set by `worker`); `all` — everything in one operator | `all` |
| `AGENT_OWNER`, `AGENT_GENERATION`, `AGENT_JWT_PUBLIC_KEY` | Set by `worker` in the pod of an owner (`AGENT_MODE=owner`): whose pod it is, the number of this start and the public key its tokens are checked with. None is a secret | — |
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
| `TELEGRAM_BOT_TOKEN`, `TELEGRAM_WEBHOOK_SECRET` | The first configuration of the Telegram channel: copied into the channel once, later the channel is managed in **Admin → Channels** | — |
| `TELEGRAM_API_URL` | Bot API | `https://api.telegram.org` |
| `TG_TABLE_MAX_COLS` | Tables wider than this go to Telegram as a code block | `8` |
| `KEY_PEPPER` | HMAC key of the personal Telegram keys; empty — derived from `SECRETS_KEY` | — |
| `EMAIL_REPLY_LIMIT_PER_HOUR` | Letters of the bot to one address per hour | `20` |
| `EMAIL_ATTACH_MAX` | Total size of files attached to an answer; the rest go as links | `10MB` |
| `EMAIL_INBOUND_MAX` | Total size of attachments accepted from a letter | `25MB` |
| `CONFIRMATION_TTL` | How long a held action waits for the confirmation of the user | `24h` |
| `PURGE_SCHEDULE` | Cron of the purge of archived accounts (time zone `DEFAULT_TIMEZONE`) | `30 3 * * *` |
| `AUDIT_RETENTION` | Audit partitions older than this are dropped | `365d` |
| `TOPIC_ARCHIVE_AFTER` | Idle topic conversations are archived | `14d` |
| `UPLOAD_MAX_BYTES` | Largest attachment | `50MB` |

## Observability

| Variable | Purpose | Default |
| --- | --- | --- |
| `LOG_LEVEL` | `debug`, `info`, `warn`, `error` | `info` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Traces | — |

Metrics are served on `SERVICE_ADDR` at `/metrics`; health at `/healthz` and `/readyz`. Channels and
accounts add `nabu_channel_inbound_total`, `nabu_channel_outbound_total`,
`nabu_email_auth_rejected_total`, `nabu_imap_connected`, `nabu_vkteams_poll_lag_seconds`,
`nabu_pending_confirmations`, `nabu_users{status}` and `nabu_purge_deleted_total`. Alert when
`nabu_imap_connected` is 0 for 5 minutes while the mail channel is enabled and when rejected letters grow.

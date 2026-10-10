# Connecting Hammurapi

Hammurapi uses Nabu in two ways (`FTR.HMR.CMN-0006`):

- **the chat** in Hammurapi is a window to the user's personal agent in Nabu — the same main
  conversation, topics and memory; Hammurapi calls `/client/v1/me/*` with `Nabu-On-Behalf-Of: <email>`;
- **the scenarios of the cycle** (issue analysis, gate generation, conformance check, code generation,
  review updates, rollback reverts) run on service agents of Nabu that Hammurapi starts through
  `/client/v1/agents/{name}/runs`. Code generation runs with the Hammurapi runner as an external
  workspace connected to `relay`.

The personal agent, in turn, calls the MCP of Hammurapi on behalf of the user (reading, creating
issues).

## 1. The client in Nabu

**Admin → Clients → New client**:

| Field | Value |
| --- | --- |
| Name | `hammurapi` |
| URL | `https://hammurapi.<domain>` |
| Delegation | on — the chat acts for users by email |
| Import | on — for the transfer of the agent settings |
| Service agents | the agents Hammurapi may start (after the transfer: `hammurapi-issue-analysis`, `hammurapi-gate-generation`, `hammurapi-conformance-check`, `hammurapi-codegen`, `hammurapi-review-update`, `hammurapi-rollback-revert`) |

The client secret is shown once. Put it into the organization secrets used by the Hammurapi deploy:

| Variable / secret | Value |
| --- | --- |
| `HAMMURAPI_NABU_URL` (variable) | `https://nabu-api.<domain>` |
| `HAMMURAPI_NABU_CLIENT_ID` (variable) | the client id |
| `HAMMURAPI_NABU_CLIENT_SECRET` (secret) | the client secret |

Then release (or redeploy) `hammurapi-core`. Users of Hammurapi and Nabu are matched by email, so both
should sign in through the same GitHub organization or the same OIDC provider.

## 2. Transfer of the agent settings

In Hammurapi, **Admin → Nabu → Transfer to Nabu** moves the LLM connections with keys, the models of
scenarios, the MCP servers and the skills source of Hammurapi into Nabu
(`POST /client/v1/import/hammurapi-agent`) and creates the service agents `hammurapi-<scenario>`.
Hammurapi binds its scenarios to them; a repeated transfer updates the same objects. Allow the new
agents to the client (step 1) if they are not allowed yet, and check the binding table in
**Admin → Nabu**.

When every scenario runs in Nabu, set the organization variable `HAMMURAPI_AGENT_OPERATOR=false`:
the next release of `hammurapi-core` is deployed without the built-in agent operator.

## 3. Hammurapi as a tool of personal agents

**Admin → Catalog → New item**, type MCP:

| Field | Value |
| --- | --- |
| Name | `hammurapi` |
| URL | `https://hammurapi-api.<domain>/mcp/nabu` |
| Access | personal, kind `delegation`, audience `hammurapi` |
| Read only | off — otherwise the agent sees only reading tools and cannot create issues |

Nabu calls it with its own JWT (audience `hammurapi`, issuer `https://nabu-api.<domain>`) and
`Nabu-On-Behalf-Of: <email>`. Hammurapi checks the signature by the JWKS of Nabu and applies the
rights of that user; a user unknown to Hammurapi is created without roles. Users enable the item in
**Connections**.

## Network

On the stand both run in one cluster. Service agents reach the MCP of the Hammurapi worker
(`worker-mcp.hammurapi.svc:8083`, allowed from the namespace `nabu`) and the internal server of
Hammurapi `api` (`api-internal.hammurapi.svc:8081`) with per-run tokens. The Hammurapi runner connects
to `wss://nabu-api.<domain>/v1/workspaces/connect`; it needs no inbound port.

# Connecting Hammurapi

Hammurapi works only with Nabu (`FTR.HMR.CMN-0006`, `FTR.HMR.CMN-0007`): it has no agent of its own and
does not start without the client of Nabu. It uses Nabu in two ways:

- **the chat** in Hammurapi is a window to the user's personal agent in Nabu — the same main
  conversation, topics, memory and files with their previews; Hammurapi calls `/client/v1/me/*` with
  `Nabu-On-Behalf-Of: <email>`;
- **the scenarios of the cycle** (issue analysis, gate generation, conformance check, code generation,
  review updates, rollback reverts) run on service agents of Nabu that Hammurapi starts through
  `/client/v1/agents/{name}/runs`. The code scenarios run in a temporary sandbox of Nabu: Hammurapi
  uploads the working copy of the branch (`PUT /client/v1/uploads/{id}`), the agent changes the files,
  Hammurapi takes the changes (`GET /client/v1/runs/{id}/changes`), commits them and opens the PR.
  The sandbox has no git credentials.

The personal agent, in turn, calls the MCP of Hammurapi on behalf of the user (reading, creating
issues).

## 1. The client in Nabu

**Admin → Clients → New client**:

| Field | Value |
| --- | --- |
| Name | `hammurapi` |
| URL | `https://hammurapi.<domain>` |
| Delegation | on — the chat acts for users by email |
| Service agents | the agents Hammurapi may start, one per scenario or shared: issue analysis, gate generation, conformance check, code generation, review update, rollback revert |

The client secret is shown once. Put it into the organization secrets used by the Hammurapi deploy:

| Variable / secret | Value |
| --- | --- |
| `HAMMURAPI_NABU_URL` (variable) | `https://nabu-api.<domain>` |
| `HAMMURAPI_NABU_CLIENT_ID` (variable) | the client id |
| `HAMMURAPI_NABU_CLIENT_SECRET` (secret) | the client secret |
| `NABU_WEB_URL` (variable, optional) | `https://nabu.<domain>`: a code task links to the service agents of Nabu |

Then release (or redeploy) `hammurapi-core`. Users of Hammurapi and Nabu are matched by email, so both
should sign in through the same GitHub organization or the same OIDC provider.

## 2. Service agents of the scenarios

Create the service agents in **Admin → Service agents** and allow them to the client (step 1). In
Hammurapi, **Admin → Nabu** binds each scenario to an agent; an unbound scenario does not run.

The agents of the code scenarios (code generation, review update, rollback revert) need the
workspace **Nabu sandbox**, the profile `code` (Go, Python, JavaScript) and the proxy
**Package registries** (`packages`): **Admin → Sandbox images → Proxies** sets the registries and
their tokens; the sandbox reaches only the addresses of its proxies through the egress gateway.

## 3. Hammurapi as a tool of personal agents

**Admin → Catalog → New item**, type MCP:

| Field | Value |
| --- | --- |
| Name | `hammurapi` |
| URL | `https://hammurapi-api.<domain>/mcp/nabu` |
| Access | personal, kind `delegation`, audience `hammurapi` |
| Read only | off — otherwise the agent sees only reading tools and cannot create issues |

The **Read only** switch is in the form for both access modes. With it on, Nabu offers the agent the
tools Hammurapi marks as read-only and allows calls only to those marked so at the last **Check** of
the item: check the item again after an upgrade of Hammurapi that changes its tools.

Nabu calls it with its own JWT (audience `hammurapi`, issuer `https://nabu-api.<domain>`) and
`Nabu-On-Behalf-Of: <email>`. Hammurapi checks the signature by the JWKS of Nabu and applies the
rights of that user; a user unknown to Hammurapi is created without roles. Users enable the item in
**Connections**.

## Network

On the stand both run in one cluster. Service agents reach the MCP of the Hammurapi worker
(`worker-mcp.hammurapi.svc:8083`, allowed from the namespace `nabu`) with per-run tokens. Temporary
sandboxes reach the internet only through `nabu-egress:3128` and only the addresses of their proxies.

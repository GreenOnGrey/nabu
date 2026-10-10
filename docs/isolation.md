# Isolation of personal agents

How Nabu keeps the data and the actions of one user apart from another. Isolation is layered: every
layer ties data and actions to one user and does not trust its neighbours. What stays shared is
listed in [Limits](#limits).

## Data

- **Conversations, memory, tasks and files** are stored with the id of their owner, and every query
  filters by it. The id comes from the browser session or from a signed token, never from request
  parameters.
- **Files of the space** live in S3 under `spaces/<user id>/`, attachments under `attachments/<user id>/`,
  session snapshots under `sessions/<conversation id>.jsonl`.
- **Group agents** own their data through a separate technical account, so they see neither the
  memory nor the files of the members.

## The agent pod

- **A pod per owner.** The agent of every user and of every group agent runs in a pod of its own in
  the namespace `nabu-agents`: its conversations, topics and runs of scheduled tasks, and nothing of
  anybody else. `worker` creates the pod with the first message and deletes it after
  `AGENT_POD_IDLE_TIMEOUT` without turns.
- **The pod is disposable.** A snapshot of the session goes to S3 after every turn, so a deleted or
  lost pod loses nothing but a turn in flight.
- **No secret and no rights in the pod.** Not root, a read-only root file system, all capabilities
  dropped, no Kubernetes service account token, limits of CPU, memory and disk. The environment
  holds only the id of the owner, the number of this start of the pod and the public key of Nabu.
- **A token per pod.** `worker` signs every request with a five-minute token for this owner and this
  start of the pod. The pod refuses any other token, so a request sent to an address that now
  belongs to another pod never runs there. No shared secret opens all pods.
- **Network policy.** Incoming connections come only from `worker`. Outgoing ones go only to DNS,
  the internal API of Nabu, `relay`, port 443 on the internet (model providers) and the addresses in
  `agents.egressAllow`. Agent pods do not see each other.
- **Service agents** run in a shared pool, the Deployment `agent`: their runs are short and keep no
  data of users between runs.

## The agent session

- **One Pi process per conversation** inside the pod of its owner. It has its own directory
  `sessions/<id>` and an environment built from scratch: nothing of the operator's environment is
  inherited.
- **Pi has no files or shell of its own.** Its tools `read`, `write`, `edit`, `bash`, `grep`, `find`
  and `ls` are routed to the sandbox of the user through `relay`. Commands run in the sandbox, not in
  the agent pod. A session without a workspace gets none of these tools.
- **Sessions do not hold credentials of external systems.** They get short-lived Nabu tokens issued
  for one user and one conversation. The key of the LLM connection is passed in the environment of
  the Pi process.

## The sandbox

- **A separate pod per user** in the namespace `nabu-sandboxes`: not root, a read-only root file
  system, all capabilities dropped, no Kubernetes service account token, limits of CPU, memory and disk.
- **Network policy.** No incoming connections. Outgoing ones go only to DNS, `relay`, the internal
  API of Nabu and the addresses listed in `SANDBOX_EGRESS_ALLOW`.
- **Access by token.** `relay` accepts a call only when its token was issued for the workspace of
  this very user (`sandbox-<user id>`); a token of another user is refused.

## External tools (MCP)

- Every call goes through the proxy in `api`. The proxy checks the token of the session and adds the
  credentials itself: the personal OAuth token of the user, a delegation JWT for the user's email, or
  the platform key.
- A personal item works only for the user who connected it. A revoked access stops working with the
  next call.
- In products such as Hammurapi the rights are applied by the product itself, by the email of the user.

## Channels and access

- A message from Telegram, VK Teams or mail reaches the agent only when the sender is matched to
  one account (a personal key, the email of the account, an authentic letter) and the channel is
  available to that user.
- Blocked and archived users are stopped at the entry, in the built-in tools and in the MCP proxy.

## Limits

- **Shared LLM connections.** A model key belongs to a connection, not to a user, and it is passed
  to the pod of every user of that model; users are told apart only in the usage report. The model
  cannot run code in the agent pod, because the shell is routed to the sandbox, but a vulnerability
  in Pi or in an extension would expose the key.
- **Internet on port 443.** The network policy of agent pods cannot tell a model provider from any
  other site.
- **The pool of service agents.** Runs of all service agents share the pod `agent`.
- **Without pods of owners.** With `agents.enabled: false` (`AGENT_EXECUTOR=local`) every session
  runs in the shared operator under one system user, separated only by directories and
  environments. This mode is for development.
- **Platform MCP items.** Items in the platform mode use one key for everybody: what a user can see
  there is decided by the external system, not by Nabu.
- **Audit and administrators.** A platform administrator sees the audit of tool calls of every user;
  the arguments are masked and truncated.

# Channels and accounts

How employees reach their agents outside the web application, and what happens to an account when
an employee leaves (`FTR.NAB.CMN-0002`). Everything here is set in the administration; the secrets
of channels are stored encrypted with `SECRETS_KEY` and are never shown again.

## Channels

**Admin → Channels** lists the channels of the instance:

| Channel | How Nabu knows the user | Notes |
| --- | --- | --- |
| Web | Sign-in | Always available, cannot be switched off |
| A product (service client) | Delegation by email (`Nabu-On-Behalf-Of`) | Listed while the service client exists |
| Mail | The authentic sender address | Threads become topics |
| VK Teams | The email of the VK Teams account | No linking |
| Telegram | A personal key sent to the bot once | Rich messages, drafts in private chats |

A channel is available to a user when it is **enabled** and either **for all users** or enabled in
the card of the user (**Admin → Users → Card**). A change reaches every pod within a minute. A
message through a closed channel gets «the channel is not available» and never reaches the agent; a
task whose channel is closed delivers its result to the web with a mark in the run history.

### Telegram

1. Create a bot with BotFather, set its token in **Channels → Telegram → Configure** (or pass
   `TELEGRAM_BOT_TOKEN` at the first deploy) and switch the channel on: like mail and VK Teams it
   is off by default. Nabu registers the webhook itself.
2. A user presses **Connections → Telegram → Get key**, copies `NB-XXXX-XXXX-XXXX` — it is shown
   once and stored as an HMAC — and sends it to the bot as the first message.
3. One Telegram account per user: a new link replaces the old one, which gets a notice. Five wrong
   keys block the Telegram account for an hour. Switching the channel off keeps the link.

Answers are rich messages of the Bot API (headings, lists, code, native tables); when the method is
refused Nabu sends HTML. Private chats show the answer as it is written (`sendMessageDraft`), groups
get «typing» and one final message.

### VK Teams

Set the bot API address of the corporate server (`https://<server>/bot/v1`) and the bot token.
One worker polls the events (a Postgres advisory lock; another instance takes over within 15
seconds and continues from the stored event). The platform lets a bot answer only users who added
it to contacts or wrote first, so tasks deliver to VK Teams only after the first message.

### Mail

The bot has a mailbox at Google Workspace, Yandex 360 or VK WorkMail. Choose the provider — IMAP,
SMTP and the `authserv-id` are filled — then set the mailbox, its aliases, the corporate domains and
the sign-in secret:

- **Yandex 360, VK WorkMail** — an app password of the mailbox.
- **Google Workspace** — the JSON key of a service account with domain-wide delegation limited to the
  scope `https://mail.google.com/`. The delegation gives access to any mailbox of the domain: use a
  separate service account and keep the key as a top secret.

Rules, in this order:

1. Automatic mail (`Auto-Submitted`, `Precedence: bulk|list|junk`, mailing lists, `noreply@…`) is
   ignored.
2. Only authentic letters of corporate domains are accepted: the top `Authentication-Results`
   header whose `authserv-id` is the one of the provider (or ends with `.<authserv-id>`) must say
   `dmarc=pass` or `dkim=pass` for the sender domain, and the domain must be in the list. Everything
   else is rejected and written to the log (**Channels → Mail** shows recent rejections).
3. A new thread becomes a topic titled by the subject; answers in the thread continue it.
   Attachments (up to `EMAIL_INBOUND_MAX`) land in the space of the user.
4. The bot answers **by mail only when it is the only recipient** (one address in `To`, the mailbox
   or its alias, and an empty `Cc`): to the sender only, in the same thread, with files up to
   `EMAIL_ATTACH_MAX` attached and the rest as links. Otherwise the answer stays in the topic with an
   unread mark. At most `EMAIL_REPLY_LIMIT_PER_HOUR` letters go to one address per hour.
5. The text of letters is data for the agent, not instructions. In mail topics a tool that changes
   data is not performed: the user confirms it in the web chat (**Execute** / **Reject**, 24 hours),
   then the result comes with the next letter.
6. A corporate sender without an account or with a closed channel gets one short «the channel is not
   available» letter a day, only when the bot is the only recipient. External domains never get an
   answer.

One worker holds the IMAP connection with `IDLE` (an advisory lock), reconnects with a growing
delay and reconciles the inbox every 5 minutes. Handled letters get the flag `$NabuProcessed` and
move to `Nabu/Processed`.

### Group chats

Switch **Groups** on for Telegram or VK Teams. A user with access to the channel adds the bot to a
group and becomes the owner of a **group agent**: a separate agent of the chat with its own memory
and space and platform connections only — no personal access, memory or files of the members. It
sees and stores only messages that mention it or reply to it, answers only members with a Nabu
account and access to the channel, and signs its answers «(group)». Its cost is a separate line of
the usage report. When the bot is removed from the group its data are kept for the archive
retention and return if the same owner adds it again. **Admin → Group agents** transfers or
disables agents; the owner changes the name, the tone, the model and the skills through
`PATCH /api/v1/group-agents/{id}`.

## Archiving and restoring accounts

When an employee leaves, archive the account instead of deleting it: **Admin → Users → Archive by
list**, or the API for an HR system.

- Archiving closes sign-in and every channel at once, ends sessions, pauses tasks, removes the
  sandbox, personal accesses and the Telegram link, and hands group agents of the owner to an
  administrator or disables them. Conversations, memory, files and tasks are kept.
- Restoring is explicit only. An archived employee who signs in sees «the account is waiting to be
  restored» and administrators get a request; when the provider gave the employee a new ID, the
  administrator confirms linking it. Tasks come back paused; Telegram and personal accesses are
  connected again by the user.
- **Admin → Settings** sets the retention (180 days by default); before saving Nabu shows how many
  accounts the next purge deletes. After the retention the account and its data are deleted for
  good (`PURGE_SCHEDULE`); the audit stays. A later sign-in with the same email creates a new account.

### API

Give a service client the rights `users:archive` and `users:restore` (**Admin → Clients**; they are
separate on purpose) and call with its token:

```sh
curl -X POST https://nabu-api.<domain>/client/v1/users:archive -H "Authorization: Bearer $TOKEN" \
  -d '{"emails":["ivanov@company.ru"],"groupAgents":"transfer","initiator":"hr-system"}'
curl -X POST https://nabu-api.<domain>/client/v1/users:restore -H "Authorization: Bearer $TOKEN" \
  -d '{"emails":["ivanov@company.ru"]}'
```

The answer lists a result per email: `archived`, `restored`, `already_archived`, `already_active`,
`not_found`, `purged`. Repeating a call is safe. After a restore through the API the identity of the
next sign-in with the email is linked to the account. A delegated request for an archived user
answers `403 user_archived`, for a user whose product channel is closed — `403 channel_unavailable`.

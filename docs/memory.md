# Sources of the memory of communications

The collection of communications (FTR.NAB.CMN-0003) reads a source through its MCP server in the catalog.
An item of the catalog becomes a source in **Admin → Memory of communications → Sources**: choose the
adapter of its server and mark it. Only a personal item with an adapter can be a source.

| Adapter | Server | Sources |
| --- | --- | --- |
| `gmail` | The official Gmail MCP of Google | Gmail |
| `mail` | `imap-mcp` of the stand (any mail with OAuth for IMAP) | The mailbox, named by the item |
| `vkworkspace` | The official MCP of VK WorkSpace, `https://ai.workspace.vk.ru/mcp` | VK Teams, VK WorkMail |

## VK WorkSpace (FTR.NAB.ADM-0001)

1. **Admin → Catalog → Add MCP**: URL `https://ai.workspace.vk.ru/mcp`, access **personal**, personal access
   **OAuth MCP (automatic)**. On save Nabu finds the authorization server of VK WorkSpace and registers its client
   with the callback of the item; the card shows the server and the client. Nothing is entered by hand.
2. Connect the item yourself in **Connections** and press **Check** in the card: the tools of the server appear.
   Allow the tools the agent may use; the collection needs `messenger-get-current-user`,
   `messenger-list-chats`, `messenger-list-messages` for VK Teams and `mail-list-folders`, `mail-search`,
   `mail-read-thread` for the mail. A source whose tools are not allowed is not offered to users.
3. **Admin → Memory of communications → Sources**: adapter `vkworkspace`, mark it as a source.
4. Users connect VK WorkSpace in **Connections** and switch on «VK Teams» and «VK WorkMail» in the
   settings of the collection.

What is collected:

- VK Teams: private chats and groups. A day of a group gets a note only when the user wrote or was mentioned;
  channels, chats with bots and muted groups are skipped — a user returns a muted group in the settings of the
  collection. The discussions of a day join the note of its chat. A run reads at most `MEMORY_VK_CHAT_LIMIT`
  messages of a chat.
- VK WorkMail: the own folders without spam, drafts and trash; newsletters and shared mailboxes are skipped.
- Reading marks nothing as read; the collector gets only the read tools.

**Re-register the client** in the card when VK WorkSpace forgot it: the connections of users are reset, they
connect again, and their collection pauses until then. If the discovery fails, the card names the reason
and offers to fill OAuth in by hand.

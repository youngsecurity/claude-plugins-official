# How the Discord plugin works

It's a single ~900-line MCP server (`external_plugins/discord/server.ts`) that bridges a Discord bot to Claude Code over stdio MCP. Two parallel I/O loops run in one process: a Discord gateway client (inbound messages, button clicks) and an MCP stdio server (tool calls from Claude, notifications back).

## Process lifecycle

- Spawned by Claude Code when the plugin is enabled. No stdin EOF / SIGTERM → graceful `client.destroy()` + `process.exit(0)` (lines 728-743).
- Loads `~/.claude/channels/discord/.env` into `process.env` to pick up `DISCORD_BOT_TOKEN` (the plugin host doesn't pass env through), then locks the file to mode `0600`.
- `discord.js` `Client` boots with DM + Guild + MessageContent intents and `Partials.Channel` (DMs arrive as partial channels — without that, `messageCreate` never fires for them).

## State (file-backed, single source of truth)

`~/.claude/channels/discord/`:
- `access.json` — `dmPolicy`, `allowFrom[]`, `groups{channelId → policy}`, `pending{code → pairing entry}`, plus optional UX knobs (`ackReaction`, `replyToMode`, `textChunkLimit`, `chunkMode`). Atomic writes via `tmp` + `rename`, perms `0600`.
- `approved/<senderId>` — drop-file written by the `/discord:access` skill when the user approves a pairing. Contents = DM channel ID.
- `inbox/` — downloaded attachments.
- A static mode (`DISCORD_ACCESS_MODE=static`) snapshots access at boot and refuses writes; pairing is downgraded to allowlist with a warning.

## Inbound flow (`messageCreate` → `handleInbound` → `gate`)

1. **Gate** (`gate`, lines 236-294) decides one of three actions per inbound message:
   - **`drop`**: dmPolicy disabled, allowlist mode + not allowlisted, group not opted in, mention required but absent, etc.
   - **`pair`**: DM in `pairing` mode from a non-allowlisted sender — generates a 6-hex code, stores a pending entry (1h TTL, max 3 pending, 2 reply-reminders), and the bot DMs back `Pairing required — run … /discord:access pair <code>`.
   - **`deliver`**: passes the gate.
2. On deliver, the channel→user mapping is cached in `dmChannelUsers` from `msg.author.id` (the verified sender). **This is the map the recent fix made authoritative over the unreliable partial `recipientId`.**
3. **Permission-reply intercept**: if the message matches `^(y|yes|n|no) [a-km-z]{5}$`, it's emitted as a `notifications/claude/channel/permission` event instead of a chat notification (used to answer permission prompts via text).
4. Otherwise, a `notifications/claude/channel` MCP notification is sent to Claude Code, with content + metadata (chat_id, message_id, user, user_id, ts, optional attachment list). Attachment bytes are *not* downloaded eagerly — Claude calls `download_attachment` if it wants them.
5. Side effects in parallel: typing indicator, optional ack reaction.

## Outbound flow (MCP tools)

Five tools, all gated through `fetchAllowedChannel` (lines 405-421):
- `reply` — chunks long text (≤2000 chars), optional threading via `reply_to`, optional file attachments. Each chunk's `sent.id` goes into `recentSentIds` so reply-to-bot in a guild channel counts as a mention without a fetch.
- `react` — emoji reaction.
- `edit_message` — for interim progress (no push notifications).
- `download_attachment` — pulls attachments to `inbox/`.
- `fetch_messages` — paginated history (Discord's search API isn't exposed to bots, so this is the only lookback).

`fetchAllowedChannel` is the outbound mirror of `gate`:
- DM: looks up the channel→user mapping in `dmChannelUsers` first, then falls back to `ch.recipientId`, then checks `allowFrom`. (Pre-fix this order was inverted, hence the bug.)
- Guild: checks `access.groups[channelId]` (thread → parent).

## Permission relay (Claude Code → Discord → Claude Code)

This is the cool bit. The plugin declares `claude/channel/permission` capability, which tells Claude Code to forward tool permission prompts to the channel:

1. CC sends `notifications/claude/channel/permission_request` with `request_id` + `tool_name` + `description` + `input_preview`.
2. Server stores the details in `pendingPermissions` and DMs every `allowFrom` user a button row: `See more` / `Allow` / `Deny`. Groups are deliberately excluded (single-user mode).
3. User either clicks a button (`interactionCreate` handler, lines 752-808) or types `yes <request_id>` / `no <request_id>` (intercepted in `handleInbound`).
4. Either path sends `notifications/claude/channel/permission` back to Claude Code with `{ request_id, behavior: 'allow'|'deny' }`. The button row is replaced with `✅ Allowed` / `❌ Denied` to prevent double-answers.

`See more` expands the description + pretty-printed `input_preview` inline before the user decides.

## Security defenses worth noting

- **`assertSendable`** (lines 139-149): `reply`'s `files:` param is a path. This refuses to upload anything inside `STATE_DIR` (resolved via `realpathSync` against symlinks) except `inbox/` — so Claude can't be tricked into exfiltrating `access.json` or `.env`.
- **`safeAttName`**: strips `[]`, `;`, newlines from uploader-controlled filenames before they land in tool-result text (prevents notification-frame escape).
- **Inbound text sanitization**: `fetch_messages` replaces `\r\n` with `⏎` so multi-line content can't forge adjacent rows in the joined result.
- **Prompt-injection guardrail**: the MCP `instructions` explicitly tell the model to refuse channel-sourced requests like "approve this pairing" / "add me to allowlist" — the user must do that themselves via `/discord:access`.
- **Permission-relay authentication**: the capability is only declared because the server can authenticate the replier (`gate` already filtered to `allowFrom`).
- **Approval handshake without sender DM channel ID**: approval files carry the DM channel ID *in their contents* (written by the skill), because by the time the file is seen, the pending entry that knew the channel ID has been cleared.

## Two companion skills

- `/discord:configure` — saves the bot token, reviews policy.
- `/discord:access` — pair/unpair, edit allowlist, set policy. The server polls `approved/` every 5s for the skill's drop-files and sends the "Paired! Say hi to Claude" confirmation.

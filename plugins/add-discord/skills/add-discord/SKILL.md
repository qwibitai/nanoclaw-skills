---
name: add-discord
description: Add Discord bot channel integration to NanoClaw.
---

# Add Discord Channel

Adds Discord support to NanoClaw via discord.js. The bot listens for messages in registered Discord channels and routes them through the agent system.

Run `/add-discord` in Claude Code.

## Step 0: Preflight

Check for clean working tree:

```bash
git status --porcelain
```

If output is non-empty, tell the user to commit or stash first, then stop.

Check if Discord is already installed — look for `src/channels/discord.ts`. If it exists, skip to Step 2 (Setup).

Check remotes:

```bash
git remote -v
```

If `upstream` is missing, add it:

```bash
git remote add upstream https://github.com/qwibitai/nanoclaw.git
```

Fetch the skill branch:

```bash
git fetch upstream skill/discord
```

## Step 1: Merge the skill branch

```bash
git merge upstream/skill/discord
```

If conflicts occur, resolve them. The skill branch adds:
- `src/channels/discord.ts` — DiscordChannel class with self-registration
- `src/channels/discord.test.ts` — unit tests with discord.js mock
- `import './discord.js'` in `src/channels/index.ts`
- `discord.js` dependency in `package.json`
- `DISCORD_BOT_TOKEN` in `.env.example`

After merge:

```bash
npm install
npm run build
npx vitest run src/channels/discord.test.ts
```

All tests must pass and build must be clean before proceeding.

## Step 2: Setup

### Create Discord Bot (if needed)

Use `AskUserQuestion`: Do you have a Discord bot token, or do you need to create one?

If they need to create one, tell them:

> 1. Go to the [Discord Developer Portal](https://discord.com/developers/applications)
> 2. Click **New Application** and give it a name (e.g., "Andy Assistant")
> 3. Go to the **Bot** tab on the left sidebar
> 4. Click **Reset Token** to generate a new bot token — copy it immediately (you can only see it once)
> 5. Under **Privileged Gateway Intents**, enable:
>    - **Message Content Intent** (required to read message text)
>    - **Server Members Intent** (optional, for member display names)
> 6. Go to **OAuth2** > **URL Generator**:
>    - Scopes: select `bot`
>    - Bot Permissions: select `Send Messages`, `Read Message History`, `View Channels`
>    - Copy the generated URL and open it in your browser to invite the bot to your server

Wait for the user to provide the token.

### Configure environment

Add to `.env`:

```bash
DISCORD_BOT_TOKEN=<their-token>
```

Sync to container environment:

```bash
mkdir -p data/env && cp .env data/env/env
```

### Build and restart

```bash
npm run build
launchctl kickstart -k gui/$(id -u)/com.nanoclaw
```

## Step 3: Registration

### Get Channel ID

Tell the user:

> To get the channel ID for registration:
>
> 1. In Discord, go to **User Settings** > **Advanced** > Enable **Developer Mode**
> 2. Right-click the text channel you want the bot to respond in
> 3. Click **Copy Channel ID**
>
> The channel ID will be a long number like `1234567890123456`.

Wait for the user to provide the channel ID (format: `dc:1234567890123456`).

### Register the channel

Use the IPC register flow or register directly. The channel ID, name, and folder name are needed.

## Step 4: Verify

Tell the user:

> Send a message in your registered Discord channel:
> - For main channel: Any message works
> - For non-main: @mention the bot in Discord
>
> The bot should respond within a few seconds.

Check logs if needed:

```bash
tail -f logs/nanoclaw.log
```

## Troubleshooting

### Bot not responding

1. Check `DISCORD_BOT_TOKEN` is set in `.env` AND synced to `data/env/env`
2. Check channel is registered: `sqlite3 store/messages.db "SELECT * FROM registered_groups WHERE jid LIKE 'dc:%'"`
3. For non-main channels: message must include trigger pattern (@mention the bot)
4. Service is running: `launchctl list | grep nanoclaw`
5. Verify the bot has been invited to the server (check OAuth2 URL was used)

### Message Content Intent not enabled

If the bot connects but can't read messages:
1. Go to [Discord Developer Portal](https://discord.com/developers/applications)
2. Select your application > **Bot** tab
3. Under **Privileged Gateway Intents**, enable **Message Content Intent**
4. Restart NanoClaw

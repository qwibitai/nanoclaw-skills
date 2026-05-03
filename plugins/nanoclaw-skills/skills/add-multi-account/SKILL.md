---
name: add-multi-account
description: Add a second (or third) account for any OneCLI OAuth provider to an agent group. Each extra account becomes its own MCP server instance routed through a different OneCLI agent identity so the gateway resolves to the correct OAuth connection. Works with Gmail, Google Calendar, Google Drive, or any provider with multiple connected accounts.
---

# Add Multi-Account Provider

This skill wires additional accounts for any OneCLI OAuth provider into an agent group. The gateway disambiguates which OAuth connection to use based on which OneCLI agent identity the MCP server proxies through — so each extra account gets its own MCP server instance with a different `HTTPS_PROXY` token.

**Prerequisite:** The base integration (account 1) must already be working — e.g. via `/add-gmail` for Gmail. OneCLI must be running and the extra account(s) must already be connected in the OneCLI web UI (`onecli apps list` shows `connected`).

## Per-provider reference

| Provider | MCP command | Base stub dir | OAuth path env var | Credentials env var | Tool prefix |
|---|---|---|---|---|---|
| `gmail` | `gmail-mcp` | `~/.gmail-mcp` | `GMAIL_OAUTH_PATH` | `GMAIL_CREDENTIALS_PATH` | `mcp__gmail` |
| `google-calendar` | `google-calendar-mcp` | `~/.calendar-mcp` | `GOOGLE_OAUTH_CREDENTIALS` | `GOOGLE_CALENDAR_MCP_TOKEN_PATH` | `mcp__calendar` |
| `google-drive` | `google-drive-mcp` | `~/.gdrive-mcp` | `GOOGLE_DRIVE_OAUTH_CREDENTIALS` | `GOOGLE_DRIVE_MCP_TOKEN_PATH` | `mcp__drive` |

---

## Phase 1: Pre-flight

Ask the user which provider and how many additional accounts they want to add using `AskUserQuestion`.

Then confirm those extra accounts are connected in OneCLI:

```bash
onecli apps list
```

If only one account shows for the provider: nothing to do. If two or more show as `connected`: continue.

## Phase 2: Get agent access tokens

List all OneCLI agents:

```bash
onecli agents list --fields name,identifier,accessToken
```

- **Account 1** → existing NanoClaw agent(s) (e.g. "Nano", "Terminal Agent") — already configured, no changes needed
- **Account 2** → **Default Agent** (built-in and unused by default — reuse it as the proxy identity)
- **Account 3+** → create one proxy agent per additional account:
  ```bash
  onecli agents create --name "<Provider> Account 3 Proxy" --identifier "<provider>-acct-3-proxy"
  ```

Note the `accessToken` for each agent you'll use for accounts 2, 3, …

## Phase 3: Wire agents to connections (OneCLI web UI)

Open `http://127.0.0.1:10254` → **Agents**.

**For each NanoClaw agent that uses this provider (e.g. "Nano"):**
1. Click **Manage Access**
2. Switch to **Selective** mode
3. Re-check all secrets this agent needs (e.g. Anthropic API key) — switching to selective loses auto-access to everything else
4. Check **account 1's** connection (e.g. `work@company.com`)
5. Save

**For the Default Agent (account 2):**
1. Click **Manage Access**
2. Switch to **Selective** mode
3. Check **account 2's** connection only (e.g. `personal@gmail.com`)
4. Save — no other secrets needed, this agent is proxy-only

**For each account 3+ proxy agent:** repeat, selecting the correct connection.

No gateway restart needed — connections are resolved per-request.

## Phase 4: Create credential stub dirs

Account 1 uses the existing base stub dir. For each additional account N, copy the stub:

```bash
# Adjust BASE to match your provider's stub dir (see reference table above)
BASE=~/.gmail-mcp
N=2   # increment for each additional account

NEW="${BASE}-${N}"
mkdir -p "$NEW"
cp "$BASE/gcp-oauth.keys.json" "$NEW/gcp-oauth.keys.json" 2>/dev/null || true
cp "$BASE/credentials.json" "$NEW/credentials.json"
chmod 600 "$NEW"/*.json
```

The stub files use `"access_token": "onecli-managed"` placeholders — the OneCLI gateway substitutes the real token at request time. Copying from the base dir gives you the right structure.

Confirm the new dir is under an allowed root in `~/.config/nanoclaw/mount-allowlist.json` (the parent dir such as `/Users/<you>` is usually already listed).

## Phase 5: Update container.json

Find the agent group's `container.json` at `groups/<folder>/container.json`.

For each additional account N, add a new entry under `mcpServers`:

```jsonc
"<provider>-N": {
  "command": "<mcp-command>",
  "args": [],
  "env": {
    "<OAUTH_PATH_VAR>": "/workspace/extra/.<provider>-mcp-N/gcp-oauth.keys.json",
    "<CREDENTIALS_VAR>": "/workspace/extra/.<provider>-mcp-N/credentials.json",
    "HTTPS_PROXY": "http://x:<acct-N-agent-token>@host.docker.internal:10255",
    "HTTP_PROXY": "http://x:<acct-N-agent-token>@host.docker.internal:10255",
    "https_proxy": "http://x:<acct-N-agent-token>@host.docker.internal:10255",
    "http_proxy": "http://x:<acct-N-agent-token>@host.docker.internal:10255"
  }
}
```

Gmail account 2 example:

```jsonc
"gmail-2": {
  "command": "gmail-mcp",
  "args": [],
  "env": {
    "GMAIL_OAUTH_PATH": "/workspace/extra/.gmail-mcp-2/gcp-oauth.keys.json",
    "GMAIL_CREDENTIALS_PATH": "/workspace/extra/.gmail-mcp-2/credentials.json",
    "HTTPS_PROXY": "http://x:<default-agent-token>@host.docker.internal:10255",
    "HTTP_PROXY": "http://x:<default-agent-token>@host.docker.internal:10255",
    "https_proxy": "http://x:<default-agent-token>@host.docker.internal:10255",
    "http_proxy": "http://x:<default-agent-token>@host.docker.internal:10255"
  }
}
```

Also add to `additionalMounts` for each account N:

```jsonc
{
  "hostPath": "/Users/<you>/.<provider>-mcp-N",
  "containerPath": ".<provider>-mcp-N",
  "readonly": false
}
```

## Phase 6: Widen the tool allowlist

Open `container/agent-runner/src/providers/claude.ts` and find the `TOOL_ALLOWLIST` array.

Replace the base provider entry with a wildcard that covers all account suffixes:

```typescript
// Before (covers only the base account):
'mcp__gmail__*',

// After (covers gmail, gmail-2, gmail-3, …):
'mcp__gmail*__*',
```

Repeat for each provider you're adding accounts to.

## Phase 7: Build and restart

```bash
pnpm run build
```

Then restart the NanoClaw service:

```bash
# macOS — find the service name dynamically (it includes a hash suffix)
launchctl kickstart -k gui/$(id -u)/$(launchctl list | grep nanoclaw | awk '{print $3}')

# Linux
systemctl --user restart nanoclaw
```

## Phase 8: Verify

Ask the agent to use a tool that touches multiple accounts. For Gmail:

> List labels on both Gmail accounts

The agent should call `mcp__gmail__*` tools for account 1 and `mcp__gmail-2__*` tools for account 2, returning results from each.

## Troubleshooting

**"Multiple connections exist for this provider"** — The NanoClaw agent (e.g. Nano) is still in `all` mode. Redo Phase 3 for that agent: switch to selective, assign account 1's connection, re-check the Anthropic secret.

**Account 2 returns 401** — The `HTTPS_PROXY` token in `container.json` is wrong. Re-run `onecli agents list --fields name,accessToken` and update the Default Agent's token.

**New MCP server not appearing** — The stub dir may not be in the mount allowlist. Check `~/.config/nanoclaw/mount-allowlist.json`; the parent dir (e.g. `/Users/<you>`) must be listed.

**Selective mode lost Anthropic access** — Open Manage Access for the Nano agent, re-check the Anthropic secret, save.

**Container starts but both accounts hit the same inbox** — The Default Agent was not switched to selective mode, or it has both connections assigned. Open Manage Access for Default Agent and ensure only account 2's connection is checked.

## Removal

1. In OneCLI web UI → revert all modified agents back to **All** mode via Manage Access
2. Remove the `<provider>-N` MCP entries and their mounts from `container.json`
3. Revert `mcp__<provider>*__*` back to `mcp__<provider>__*` in TOOL_ALLOWLIST
4. Run `pnpm run build` and restart the service
5. Delete stub dirs: `rm -rf ~/.<provider>-mcp-2 ~/.<provider>-mcp-3 ...`

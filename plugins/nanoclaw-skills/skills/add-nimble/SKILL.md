---
name: add-nimble
description: Add Nimble web search to NanoClaw via Nimble's hosted MCP — live web search, page extraction, and structured-data agents for the container agent, with cited sources. No local MCP server or extra dependency; one hosted endpoint and an API key.
---

# Add Nimble Web Search Integration

Adds [Nimble](https://nimbleway.com) web intelligence to NanoClaw through Nimble's hosted
MCP endpoint. The container agent gets live web tools over one `http` MCP server — no local
MCP process, no new dependency in the image. Mirrors the `/add-parallel` shape.

## What This Adds

- **Quick Web Search** (`nimble_search`) — fast live search with configurable content
  richness, locale/country targeting, and domain/date filters
- **Page Extraction** (`nimble_extract`) — clean content from a specific URL
- **Site & Data Tools** (`nimble_map`, `nimble_crawl_*`, `nimble_agents_*`) — site mapping,
  multi-page crawls, and pre-built structured-data agents (permission-gated in the guidance)

## Prerequisites

User must have:

1. A Nimble API key from the Nimble Platform (https://app.nimbleway.com)
2. NanoClaw already set up and running (container image built at least once)

## Implementation Steps

Run all steps automatically. Only pause for user input when explicitly needed.

Steps target current NanoClaw `main`. Where an anchor differs on an older fork, the
**older forks** note under that step gives the fallback.

### 1. Get a Nimble API Key

Use `AskUserQuestion: Do you have a Nimble API key, or should I help you get one?`

**If they have one:** collect it now.

**If they need one:** tell them:

> 1. Sign in to the Nimble Platform at https://app.nimbleway.com (create an account at
>    https://nimbleway.com if you don't have one)
> 2. Open the API settings and copy your API key
> 3. Paste the key here

Wait for the API key.

### 2. Add the API Key to `.env`

```bash
# Create .env if missing
[ -f .env ] || touch .env

# Add or update NIMBLE_API_KEY
if ! grep -q "^NIMBLE_API_KEY=" .env; then
    echo "NIMBLE_API_KEY=${API_KEY_FROM_USER}" >> .env
    echo "✓ Added NIMBLE_API_KEY to .env"
else
    sed -i.bak "s/^NIMBLE_API_KEY=.*/NIMBLE_API_KEY=${API_KEY_FROM_USER}/" .env && rm -f .env.bak
    echo "✓ Updated NIMBLE_API_KEY in .env"
fi
```

Verify (prints the count only, never the value):

```bash
grep -c "^NIMBLE_API_KEY=" .env
```

### 3. Forward the Key Into the Container

Edit `src/container-runner.ts`. Add the import next to the other relative imports (near
`import { log } from './log.js';`):

```typescript
import { readEnvFile } from './env.js';
```

Find the env line inside `buildContainerArgs`:

```typescript
  args.push('-e', `TZ=${TIMEZONE}`);
```

and add directly after it:

```typescript
  // Forward the Nimble API key (read from .env at spawn) into the container.
  const nimbleApiKey = readEnvFile(['NIMBLE_API_KEY']).NIMBLE_API_KEY;
  if (nimbleApiKey) {
    args.push('-e', `NIMBLE_API_KEY=${nimbleApiKey}`);
  }
```

`readEnvFile` reads `.env` at spawn time and never loads the key into the host process
env, so it can't leak to unrelated children — the same pattern the Claude provider uses.

**Older forks:** if `src/container-runner.ts` instead has an allowlist like
`const allowedVars = ['CLAUDE_CODE_OAUTH_TOKEN', 'ANTHROPIC_API_KEY'];`, just add
`'NIMBLE_API_KEY'` to that array and skip this step's import + block.

### 4. Register the Hosted MCP Server in the Agent Runner

Edit `container/agent-runner/src/index.ts`. Find where the `mcpServers` map is finished —
the loop that merges `container.json` servers:

```typescript
  for (const [name, serverConfig] of Object.entries(config.mcpServers)) {
    mcpServers[name] = serverConfig;
    log(`Additional MCP server: ${name} (${serverConfig.command})`);
  }
```

and add directly after it:

```typescript
  // Nimble hosted web-search MCP — registered only when the key was forwarded,
  // so keyless containers boot unchanged. The runner types mcpServers as stdio
  // configs but passes entries opaquely to the agent SDK, which accepts an
  // { type: 'http', url, headers } server — hence the localized cast.
  if (process.env.NIMBLE_API_KEY) {
    (mcpServers as Record<string, unknown>)['nimble'] = {
      type: 'http',  // REQUIRED for hosted MCP servers
      url: 'https://mcp.nimbleway.com/mcp',
      headers: { Authorization: `Bearer ${process.env.NIMBLE_API_KEY}` },
    };
    log('Nimble web search MCP configured');
  }
```

The endpoint must be `/mcp` (Streamable HTTP) — never `/sse`, which redirect-loops.

**Tool allowlisting:** no edit needed on current NanoClaw — the Claude provider derives
`mcp__<server>__*` allow patterns from the registered server names
(see `mcpAllowPattern` in `container/agent-runner/src/providers/claude.ts`), so registering
`nimble` grants `mcp__nimble__*` automatically.

**Older forks:** if the agent runner instead has a literal `allowedTools: [...]` array, add
`'mcp__nimble__*'` to it. If `mcpServers` is typed loosely (`Record<string, any>`), drop
the `as Record<string, unknown>` cast.

### 5. Add Usage Guidance to the Shared Agent Instructions

Append to `container/CLAUDE.md` (the shared base every group's CLAUDE.md is composed from):

```markdown

## Web search (Nimble)

When Nimble tools (`mcp__nimble__*`) are available, they are your primary way to reach
the live web:

- **`nimble_search`** — web search. Use freely whenever current or verifiable information
  helps: news, prices, schedules, releases, facts you're not certain of. Prefer a search
  over a guess.
- **`nimble_extract`** — fetch one specific URL as clean content. Use when the user
  shares a link or a search result needs its full page.
- **Heavier tools** (`nimble_crawl_*`, `nimble_map`, `nimble_agents_*`,
  `nimble_extract_async`) crawl whole sites, run structured-data agents, or work
  asynchronously. Ask the user before starting one. For async jobs don't block: schedule
  a check that polls `nimble_task_results` (or `nimble_crawl_status`) and report back
  when done.

**Always cite sources.** When an answer uses web results, include the source URLs so the
user can verify. If both Nimble tools and the built-in `WebSearch` are available, prefer
the Nimble tools.
```

**Older forks:** if `container/CLAUDE.md` doesn't exist and `groups/main/CLAUDE.md` is a
hand-edited file (no "Composed at spawn" header), add the same section there instead.

### 6. Build and Restart

Only the host code needs compiling. The agent runner (`/app/src`) and the shared
`container/CLAUDE.md` are mounted into the container at spawn, so steps 4 and 5 need no
image rebuild.

```bash
pnpm run build          # older forks: npm run build
```

Build must be clean before proceeding. Then restart the service so the host picks up step 3
and the next container spawn picks up steps 4 and 5:

```bash
# macOS
source setup/lib/install-slug.sh 2>/dev/null && \
  launchctl kickstart -k "gui/$(id -u)/$(launchd_label)" || \
  launchctl kickstart -k "gui/$(id -u)/com.nanoclaw"

# Linux: systemctl --user restart nanoclaw
```

### 7. Test the Integration

Tell the user to test:

> Send a message to your assistant: `@[YourAssistantName] what's the latest Node.js LTS
> version? check the web and cite your source`
>
> The assistant should call `mcp__nimble__nimble_search` and answer with source URLs.

The reply with cited sources is the verification. If you also want to see the
registration line, note that container stderr reaches `logs/nanoclaw.log` at **debug
level only** — restart the service with `LOG_LEVEL=debug` (the repo's `/debug` skill
shows how), then:

```bash
grep "Nimble web search MCP configured" logs/nanoclaw.log | tail -1
```

## Security Notes

- The API key lives in `.env` (gitignored) and is forwarded into the agent container's
  environment at spawn — the same model `/add-parallel` uses. Be aware this differs from
  NanoClaw's OneCLI vault model (`docs/SECURITY.md`: credentials injected at the gateway,
  never present in the container): with this skill the agent process **can** read
  `NIMBLE_API_KEY`, and agents ingest untrusted web content by design. Use a dedicated
  key for this integration and rotate it from the Nimble dashboard if in doubt. If
  OneCLI's generic secret injection gains support for third-party MCP hosts, this skill
  can switch to a placeholder header rewritten at the proxy — the registration shape
  would not change.
- No step commits, logs, or echoes the key; the hosted endpoint is authenticated with
  `Authorization: Bearer` over HTTPS, and no key material lands in any tracked file.

## Troubleshooting

**Container hangs or times out at boot:**

- Check that `type: 'http'` is present in the `nimble` entry — untyped hosted entries hang
  the SDK
- Confirm the URL ends in `/mcp`. The `/sse` path is a redirect loop and must never be used

**Nimble tools don't appear to the agent:**

- Verify the key was forwarded: `.env` has `NIMBLE_API_KEY=` and step 3's block is present
  in `src/container-runner.ts`
- Verify registration: `grep -n "Nimble web search MCP configured" container/agent-runner/src/index.ts`
- Run `pnpm run build` and restart, then send a new message — the agent runner is mounted
  read-only at `/app/src`, so a running container keeps the old source until it respawns
- To watch the registration line at runtime: container stderr is logged at **debug level
  only**, so restart with `LOG_LEVEL=debug` (see the repo's `/debug` skill) and grep
  `logs/nanoclaw.log` for `Nimble web search MCP configured` — if it's absent under debug
  logging, the key never reached the container

**Auth errors (401/403) from Nimble:**

- The key is wrong or was rotated. Verify it directly against the endpoint:

  ```bash
  curl -sS -X POST https://mcp.nimbleway.com/mcp \
    -H "Authorization: Bearer $(grep '^NIMBLE_API_KEY=' .env | cut -d= -f2-)" \
    -H 'content-type: application/json' \
    -H 'accept: application/json, text/event-stream' \
    -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"smoke","version":"0.0.1"}}}'
  ```

  A healthy key returns a `serverInfo` block naming the `Nimble` server.

**Egress-locked installs:** the opt-in lockdown (`NANOCLAW_EGRESS_LOCKDOWN=true`) places
containers on an internal network whose only route out is the OneCLI gateway. This
skill's hosted-MCP calls have not been verified under lockdown and may be blocked —
re-run the step 7 test after enabling it before relying on the integration.

## Uninstalling

Follow `REMOVE.md` in this skill's directory — it reverses every change this skill makes
(both reach-ins, the CLAUDE.md section, and the `.env` line), then rebuilds and restarts.

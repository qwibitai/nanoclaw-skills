---
name: add-nimble
description: Add Nimble web search to NanoClaw via Nimble's hosted MCP — live web search, page extraction, and structured-data agents for the container agent, with cited sources. No local MCP server or extra dependency; one hosted endpoint, with the API key held in the OneCLI vault and never shown to the agent.
---

# Add Nimble Web Search Integration

Adds [Nimble](https://nimbleway.com) web intelligence to NanoClaw through Nimble's hosted
MCP endpoint. On current NanoClaw nothing in the source tree changes: the server registers
through `ncl groups config add-mcp-server`, the API key goes straight into the OneCLI
vault (dashboard form or an operator-run host command — **never pasted into agent chat**),
and the usage guidance lands in the group's standing-instructions file.

## What This Adds

- **Quick Web Search** (`nimble_search`) — fast live search with configurable content
  richness, locale/country targeting, and domain/date filters
- **Page Extraction** (`nimble_extract`) — clean content from a specific URL
- **Site & Data Tools** (`nimble_map`, `nimble_crawl_*`, `nimble_agents_*`) — site mapping,
  multi-page crawls, and pre-built structured-data agents (permission-gated in the guidance)

## Prerequisites

1. A Nimble account (https://app.nimbleway.com — the user creates/copies their API key
   there, into the OneCLI dashboard, never into this chat)
2. NanoClaw set up and running, with the OneCLI gateway (standard install)

## Implementation Steps

Run all steps automatically. Only pause for user or operator input when explicitly needed.

Steps target current NanoClaw `main` (v2.2+), where remote Streamable-HTTP MCP servers are
a built-in config type. **On an older fork (v2.1.x, no `ncl groups config add-mcp-server`
command), use the [Older forks](#older-forks-v21x) section instead.**

### 1. Put the Nimble API Key in the OneCLI Vault

The key must never appear in this conversation. It travels user → OneCLI vault directly,
one of two ways:

**Preferred — prefilled dashboard form.** Resolve the OneCLI dashboard URL the user's
browser can reach:

```bash
docker inspect onecli --format '{{range .Config.Env}}{{println .}}{{end}}' | grep '^APP_URL='
```

If the value is a loopback or container-bridge address (`127.0.0.1`, `172.17.0.1`,
`host.docker.internal`), ask the operator which URL they open the OneCLI dashboard at,
suggesting `http://127.0.0.1:10254` as the default. Then gate the deeplink —
`curl -fs <dashboard-url>/connections/custom` must return HTTP 200 — and send the user:

> 1. Sign in at https://app.nimbleway.com and copy your API key
> 2. Open this link, paste the key into the prefilled form, and save:
>    `<dashboard-url>/connections/secrets?create=generic&host=mcp.nimbleway.com&name=Nimble&header=Authorization&format=Bearer%20%7Bvalue%7D`

**Fallback — operator-run host command** (older OneCLI without the prefill route, or no
reachable dashboard): ask an operator to run, on the host, with the key saved in a file
that only they touch (and delete after):

```bash
onecli secrets create --name "Nimble" --type generic \
  --host-pattern "mcp.nimbleway.com" \
  --header-name "Authorization" --value-format "Bearer {value}" \
  --file <key-file>
```

Either way, verify registration (names only — never values):

```bash
onecli secrets list | grep -i nimble
```

### 2. Register the Hosted MCP Server

Register the server on each target agent group (`ncl groups list` shows group ids):

```bash
ncl groups config add-mcp-server --id GROUP_ID --name nimble \
  --url https://mcp.nimbleway.com/mcp
```

- The URL must end in `/mcp` (Streamable HTTP). NanoClaw rejects the `sse` transport, and
  Nimble's `/sse` path is a redirect loop — never use it.
- Do **not** pass `--headers` with any credential: NanoClaw's own URL validation directs
  auth to OneCLI, and the gateway injects the `Authorization` header per request.
- Tool allowlisting needs no edit: NanoClaw derives the `mcp__nimble__*` allow pattern from
  the registered server name.

### 3. Add Usage Guidance to the Group's Standing Instructions

Write the block below into `groups/GROUP_FOLDER/instructions.prepend.md` for each selected
group — replace an existing `<!-- nimble-web-search:start -->` …
`<!-- nimble-web-search:end -->` block in place, append otherwise. Do **not** write into
`groups/GROUP_FOLDER/CLAUDE.md` (regenerated at spawn; appended blocks are lost) and do
**not** edit the tracked `container/CLAUDE.md`.

```markdown
<!-- nimble-web-search:start -->
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
<!-- nimble-web-search:end -->
```

### 4. Restart and Test

```bash
ncl groups restart --id GROUP_ID
```

Confirm the registration landed:

```bash
ncl groups config get --id GROUP_ID   # the mcpServers map should show "nimble"
```

Then tell the user to test:

> Send a message to your assistant: `@[YourAssistantName] what's the latest Node.js LTS
> version? check the web and cite your source`
>
> The assistant should call `mcp__nimble__nimble_search` and answer with source URLs.

The reply with cited sources is the verification.

## Security Notes

- The API key never appears in agent chat, agent transcripts, `.env`, `container.json`,
  or the container. It enters the OneCLI vault through the dashboard form (or an
  operator-run `--file` command on the host), and the gateway injects the `Authorization`
  header on the wire for `mcp.nimbleway.com` — matching NanoClaw's credential model
  (`docs/SECURITY.md`).
- The hosted endpoint is HTTPS-only, and NanoClaw itself rejects URLs carrying
  credentials.

## Troubleshooting

**Nimble tools don't appear to the agent:**

- `ncl groups config get --id GROUP_ID` — the `mcpServers` map must contain `nimble` with
  `type: "http"` and the `/mcp` URL
- A restart is required after config changes: `ncl groups restart --id GROUP_ID`
- The container logs `Additional MCP server: nimble (HTTP)` at boot; the repo's `/debug`
  skill shows how to surface container logs (`LOG_LEVEL=debug`)

**Auth errors (401/403) from Nimble:**

- The vault entry is missing, wrong, or was rotated: `onecli secrets list`, then redo
  step 1. Endpoint reachability can be checked without any key — an unauthenticated
  `POST https://mcp.nimbleway.com/mcp` returns an auth error, not a connection failure.

**Agent ignores the usage guidance:**

- Check `groups/GROUP_FOLDER/instructions.prepend.md` contains the `nimble-web-search`
  block and restart the group. A session that already ran keeps reasoning from its
  history; `/clear` starts a clean one.

**Container hangs or times out at boot:**

- Confirm the URL ends in `/mcp`. The `/sse` path is a redirect loop and NanoClaw rejects
  the `sse` transport in any case.

## Older forks (v2.1.x)

On forks without `ncl groups config add-mcp-server` (the container config's MCP type was
stdio-only), the integration is two small source edits plus an `.env` key. This is the
path that was container-tested end-to-end on v2.1.53. The key still must not pass through
agent chat: an **operator** adds it on the host.

**A. Key into `.env`** (gitignored) — operator-run on the host:

```bash
[ -f .env ] || touch .env
grep -q "^NIMBLE_API_KEY=" .env || echo "NIMBLE_API_KEY=<key>" >> .env
grep -c "^NIMBLE_API_KEY=" .env   # prints a count only, never the value
```

**B. Forward the key into the container** — in `src/container-runner.ts`, add next to the
other relative imports:

```typescript
import { readEnvFile } from './env.js';
```

and directly after the `TZ` env-args line (`args.push('-e', ...)`) in `buildContainerArgs`:

```typescript
// Forward the Nimble API key (read from .env at spawn) into the container.
const nimbleApiKey = readEnvFile(['NIMBLE_API_KEY']).NIMBLE_API_KEY;
if (nimbleApiKey) {
  args.push('-e', `NIMBLE_API_KEY=${nimbleApiKey}`);
}
```

(On forks that instead have an `allowedVars` allowlist array, just add `'NIMBLE_API_KEY'`
to it and skip the import + block.)

**C. Register the server** — in `container/agent-runner/src/index.ts`, directly after the
`config.mcpServers` merge loop:

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

(If the fork has a literal `allowedTools: [...]` array, also add `'mcp__nimble__*'` to it;
if `mcpServers` is typed loosely, drop the cast.)

**D. Guidance** — append step 3's `## Web search (Nimble)` section (without the marker
comments) to `container/CLAUDE.md`, the shared base these forks compose from (or to a
hand-edited `groups/main/CLAUDE.md` if the fork has no composed base).

**E. Build and restart** — only the host code compiles; the agent runner and
`container/CLAUDE.md` are mounted into the container at spawn, so no image rebuild:

```bash
pnpm run build
# macOS: launchctl kickstart -k "gui/$(id -u)/com.nanoclaw"   (or the install's label)
# Linux: systemctl --user restart nanoclaw
```

Then run step 4's functional test. To watch the registration line
(`Nimble web search MCP configured`), note container stderr reaches `logs/nanoclaw.log`
at **debug level only** — restart with `LOG_LEVEL=debug` per the repo's `/debug` skill.

**v2.1.x security note:** on this legacy path the key is forwarded into the container's
environment (the same model `/add-parallel` uses) — the agent process can read it, and
agents ingest untrusted web content by design. Use a dedicated key and rotate it from the
Nimble dashboard if in doubt. Current NanoClaw removes this tradeoff entirely.

## Uninstalling

Follow `REMOVE.md` in this skill's directory — it reverses every change this skill makes
on both current and older forks.

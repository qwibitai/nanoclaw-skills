---
name: add-nimble
description: Add bounded Nimble web search to NanoClaw via Nimble's hosted MCP — live search and page extraction with cited sources. On current NanoClaw, the API key stays in OneCLI and never reaches the agent.
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
- A deliberately bounded tool surface: site crawls, maps, async extract, and structured-data
  agents are not exposed to this group

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

### 2. Grant the Secret to Each Target Group's Agent

NanoClaw agents in `selective` mode receive **only** the secrets explicitly granted to
them. Grant the Nimble secret to each target group's agent, then read the grants back.
This applies to both lanes of step 1 (dashboard-created secrets included).

NanoClaw pins its bundled OneCLI CLI at 2.2.5 (`versions.json`), whose
`set-secrets` **replaces** the agent's whole list — never append blind. Use the
canonical safe merge (read → merge → set → read back) NanoClaw's own skills ship:

```bash
# <agentGroupId> is the `agentGroupId` field in groups/GROUP_FOLDER/container.json;
# NIMBLE_SECRET_ID is the Nimble entry's id from `onecli secrets list`.
AGENT_ID=$(onecli agents list | jq -r '.data[] | select(.identifier=="<agentGroupId>") | .id')
CURRENT=$(onecli agents secrets --id "$AGENT_ID" | jq -r '[.data[]] | join(",")')
MERGED=$(printf '%s' "$CURRENT,NIMBLE_SECRET_ID" | tr ',' '\n' | sort -u | paste -sd ',' -)
onecli agents set-secrets --id "$AGENT_ID" --secret-ids "$MERGED"
onecli agents secrets --id "$AGENT_ID"   # read back: the list must include NIMBLE_SECRET_ID
```

**Different OneCLI version?** A standalone or newer OneCLI may expose a different
grants surface. If `onecli agents secrets` / `set-secrets` are absent, stop and have an
operator apply the equivalent grant from that version's own `onecli agents --help` —
this skill does not guess command syntax it cannot verify.

Without this grant, a selective-mode install completes every other step and then gets
401/403 from Nimble — the gateway only injects granted secrets.

### 3. Register the Hosted MCP Server

Register the server on each target agent group (`ncl groups list` shows group ids):

```bash
ncl groups config add-mcp-server --id GROUP_ID --name nimble \
  --url https://mcp.nimbleway.com/mcp \
  --enabled-tools '["nimble_search","nimble_extract"]'
```

- The URL must end in `/mcp` (Streamable HTTP). NanoClaw rejects the `sse` transport, and
  Nimble's `/sse` path is a redirect loop — never use it.
- Do **not** pass `--headers` with any credential: NanoClaw's own URL validation directs
  auth to OneCLI, and the gateway injects the `Authorization` header per request.
- The explicit allowlist is enforced by the provider adapter. It exposes only
  `mcp__nimble__nimble_search` and `mcp__nimble__nimble_extract`; registering the hosted
  server never grants its crawl, map, async, or agent tools to this group.

### 4. Select External Web Search for the Group

Disable the provider's built-in web-search tool for each target group. This generic,
per-group policy leaves every group that does not opt out on its provider default:

```bash
ncl groups config update --id GROUP_ID --web-search-mode disabled
ncl groups config update --id GROUP_ID --builtin-tool-mode mcp-only
ncl groups config update --id GROUP_ID --response-delivery-mode terminal
```

These commands require a NanoClaw release that includes the typed `web_search_mode`,
`builtin_tool_mode`, and `response_delivery_mode` settings. If any flag is absent from
`ncl groups config update --help`, stop and ask the operator to update NanoClaw; do not
patch a provider globally or add a Nimble-specific runtime condition.

For the Codex provider, NanoClaw maps this setting to Codex's official
`web_search = "disabled"` configuration and disables its provider-native shell, browser,
direct-fetch, and other base tools; only the configured MCP tools remain model-visible.
Providers that cannot enforce `mcp-only` must reject the group config instead of silently
ignoring it. Terminal delivery disables the group's mid-turn
message/reaction tools, so an acknowledgment cannot be mistaken for the completed answer;
the normal final-result path remains unchanged. Groups that do not opt in keep the normal
conversation behavior.

### 5. Add Usage Guidance to the Group's Standing Instructions

Write the block below into `groups/GROUP_FOLDER/instructions.prepend.md` for each selected
group — replace an existing `<!-- nimble-web-search:start -->` …
`<!-- nimble-web-search:end -->` block in place, append otherwise. Do **not** write into
`groups/GROUP_FOLDER/CLAUDE.md` (regenerated at spawn; appended blocks are lost) and do
**not** edit the tracked `container/CLAUDE.md`.

```markdown
<!-- nimble-web-search:start -->
## Web search (Nimble)

When Nimble tools are available, they are your primary way to reach
the live web:

- **`nimble_search`** — web search. Use freely whenever current or verifiable information
  helps: news, prices, schedules, releases, facts you're not certain of. Prefer a search
  over a guess. Pass the required `query` alone by default; omit optional tuning fields
  unless the user explicitly needs them.
- **`nimble_extract`** — fetch one specific URL as clean content. Use when the user
  shares a link or a search result needs its full page.
- Other Nimble tools are intentionally unavailable in this group. Do not try to substitute
  a crawl, map, async job, or structured-data agent when search/extract is insufficient;
  state the exact gap instead.

For a bounded search request, finish the permitted searches in the current turn and then
answer with the evidence or state the exact gap. Do not send a progress-only reply and
continue researching after the caller has received it.

**Always cite sources.** When an answer uses web results, include the source URLs so the
user can verify. When Nimble tools are available, use them for web research and do not
use the built-in `WebSearch`; this keeps the configured provider choice effective and
auditable.
<!-- nimble-web-search:end -->
```

### 6. Restart and Test

```bash
ncl groups restart --id GROUP_ID
```

Confirm the registration landed:

```bash
ncl groups config get --id GROUP_ID
# Expect mcpServers.nimble.enabledTools with exactly nimble_search + nimble_extract,
# web_search_mode: "disabled", builtin_tool_mode: "mcp-only", and
# response_delivery_mode: "terminal".
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
  `type: "http"` and the `/mcp` URL, and `web_search_mode` must be `disabled`
  `builtin_tool_mode` must be `mcp-only`, and `response_delivery_mode` must be
  `terminal`; `enabledTools` must contain only
  `nimble_search` and `nimble_extract`
- A restart is required after config changes: `ncl groups restart --id GROUP_ID`
- The container logs `Additional MCP server: nimble (HTTP)` at boot; the repo's `/debug`
  skill shows how to surface container logs (`LOG_LEVEL=debug`)

**Auth errors (401/403) from Nimble:**

- The vault entry is missing, wrong, or was rotated: `onecli secrets list`, then redo
  step 1.
- The secret was never **granted** to this group's agent (selective mode): re-run step
  2's safe-merge and confirm the readback lists the Nimble secret id.
- Endpoint reachability can be checked without any key — an unauthenticated
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

**D. Guidance** — append step 5's `## Web search (Nimble)` section (without the marker
comments) to `container/CLAUDE.md`, the shared base these forks compose from (or to a
hand-edited `groups/main/CLAUDE.md` if the fork has no composed base).

**E. Build and restart** — only the host code compiles; the agent runner and
`container/CLAUDE.md` are mounted into the container at spawn, so no image rebuild:

```bash
pnpm run build
# macOS: launchctl kickstart -k "gui/$(id -u)/com.nanoclaw"   (or the install's label)
# Linux: systemctl --user restart nanoclaw
```

Then run step 6's functional test. To watch the registration line
(`Nimble web search MCP configured`), note container stderr reaches `logs/nanoclaw.log`
at **debug level only** — restart with `LOG_LEVEL=debug` per the repo's `/debug` skill.

**v2.1.x security note:** on this legacy path the key is forwarded into the container's
environment (the same model `/add-parallel` uses) — the agent process can read it, and
agents ingest untrusted web content by design. Use a dedicated key and rotate it from the
Nimble dashboard if in doubt. Current NanoClaw removes this tradeoff entirely.

## Uninstalling

Follow `REMOVE.md` in this skill's directory — it reverses every change this skill makes
on both current and older forks.

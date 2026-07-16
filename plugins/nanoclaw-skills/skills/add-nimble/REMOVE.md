# Remove Nimble Web Search

Idempotent — safe to run even if some steps were never applied.

## 1. Unregister the MCP server

In `container/agent-runner/src/index.ts`, remove the `if (process.env.NIMBLE_API_KEY) { … }`
block (including its comment) that registers `mcpServers['nimble']` after the
`config.mcpServers` merge loop.

**Older forks** (applied via the fallbacks): remove the same registration block, and
additionally remove `'mcp__nimble__*'` from the `allowedTools` array — on older forks
apply added both.

## 2. Revert the host-side edit in `src/container-runner.ts`

- Remove the `const nimbleApiKey = readEnvFile(['NIMBLE_API_KEY']).NIMBLE_API_KEY;` block
  (and its comment) that follows the `TZ` env line.
- Remove `import { readEnvFile } from './env.js';` **only if** nothing else in the file
  uses `readEnvFile` (grep first).

**Older forks:** remove `'NIMBLE_API_KEY'` from the `allowedVars` array instead.

## 3. Remove the agent guidance

Delete the `## Web search (Nimble)` section from `container/CLAUDE.md` (or from
`groups/main/CLAUDE.md` on older forks, if it was added there).

## 4. Remove the env var

```bash
sed -i.bak '/^NIMBLE_API_KEY=/d' .env && rm -f .env.bak
```

## 5. Build and restart

Only the host code needs compiling — the agent runner and `container/CLAUDE.md` are mounted
at spawn, so no image rebuild is needed.

```bash
pnpm run build

# macOS
source setup/lib/install-slug.sh 2>/dev/null && \
  launchctl kickstart -k "gui/$(id -u)/$(launchd_label)" || \
  launchctl kickstart -k "gui/$(id -u)/com.nanoclaw"

# Linux: systemctl --user restart nanoclaw
```

## Verification

After removal, asking the agent to "search the web with nimble" should report no such
tool, and the registration block is gone from the source:

```bash
grep -c "Nimble web search MCP configured" container/agent-runner/src/index.ts   # expect 0
```

(Don't verify via `logs/nanoclaw.log` — container stderr only reaches it at
`LOG_LEVEL=debug`, so an absent line proves nothing on a default install.)

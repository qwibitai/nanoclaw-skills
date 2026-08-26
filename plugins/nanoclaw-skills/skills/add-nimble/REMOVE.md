# Remove Nimble Web Search

Idempotent — safe to run even if some steps were never applied.

## Current NanoClaw (v2.2+, config-registered)

1. **Unregister the server.** List the groups, then for every group with a `nimble` MCP
   entry:

   ```bash
   ncl groups list
   ncl groups config get --id GROUP_ID
   ncl groups config remove-mcp-server --id GROUP_ID --name nimble
   ```

2. **Restore the provider's default web-search policy** for every group configured by
   this skill:

   ```bash
   ncl groups config update --id GROUP_ID --web-search-mode default
   ncl groups config update --id GROUP_ID --response-delivery-mode default
   ```

   This only clears the per-group override; it does not change any other group's policy.

3. **Remove the usage guidance** from every group whose `instructions.prepend.md`
   contains the `nimble-web-search` block:

   ```bash
   perl -0pi -e 's/\n?<!-- nimble-web-search:start -->.*?<!-- nimble-web-search:end -->\n?//s' groups/GROUP_FOLDER/instructions.prepend.md
   ```

   No-op when the block is absent.

4. **Restart the group** after the server, policy, and standing-instruction changes:

   ```bash
   ncl groups restart --id GROUP_ID
   ```

5. **Revoke the group grant — optional.** Removing the grant only affects this group;
   other groups' assignments are untouched. On NanoClaw's bundled OneCLI CLI (2.2.5,
   `versions.json`), read the current list, drop `SECRET_ID`, then replace it with the
   same safe pattern apply used, and read back:

   ```bash
   CURRENT=$(onecli agents secrets --id AGENT_ID | jq -r '[.data[]] | join(",")')
   REMAINING=$(printf '%s' "$CURRENT" | tr ',' '\n' | grep -vx SECRET_ID | paste -sd ',' -)
   onecli agents set-secrets --id AGENT_ID --secret-ids "$REMAINING"
   onecli agents secrets --id AGENT_ID   # read back: SECRET_ID must be gone
   ```

   On a different OneCLI version whose `agents secrets` / `set-secrets` are absent, have
   an operator apply the equivalent from that version's `onecli agents --help` — no
   guessed syntax.

6. **Vault secret — optional.** The secret may be shared by other groups or integrations
   on this host — leaving it in place is safe. Only if the **operator confirms** nothing
   else uses it:

   ```bash
   onecli secrets list                    # find the mcp.nimbleway.com entry's id
   onecli secrets delete --id SECRET_ID
   ```

## Older forks (v2.1.x, source-edited)

1. In `container/agent-runner/src/index.ts`, remove the
   `if (process.env.NIMBLE_API_KEY) { … }` block (including its comment) that registers
   `mcpServers['nimble']` after the `config.mcpServers` merge loop. If the fork's
   fallback also added `'mcp__nimble__*'` to a literal `allowedTools` array, remove that
   entry too — apply added both.

2. In `src/container-runner.ts`, remove the
   `const nimbleApiKey = readEnvFile(['NIMBLE_API_KEY']).NIMBLE_API_KEY;` block (and its
   comment) after the `TZ` env line, and remove
   `import { readEnvFile } from './env.js';` **only if** nothing else in the file uses
   `readEnvFile` (grep first). On `allowedVars` forks, remove `'NIMBLE_API_KEY'` from
   that array instead.

3. Delete the `## Web search (Nimble)` section from `container/CLAUDE.md` (or from
   `groups/main/CLAUDE.md` if it was added there).

4. Remove the env var:

   ```bash
   sed -i.bak '/^NIMBLE_API_KEY=/d' .env && rm -f .env.bak
   ```

5. Build and restart (host code only — no image rebuild):

   ```bash
   pnpm run build
   # macOS: launchctl kickstart -k "gui/$(id -u)/com.nanoclaw"   (or the install's label)
   # Linux: systemctl --user restart nanoclaw
   ```

## Verification

- Current NanoClaw: `ncl groups config get --id GROUP_ID` no longer lists `nimble`, shows
  the default/null web-search and response-delivery modes, `instructions.prepend.md` has no `nimble-web-search`
  block, and asking the agent to "search the web with nimble" reports no such tool.
- Older forks: additionally confirm the source edit is gone —

  ```bash
  grep -c "Nimble web search MCP configured" container/agent-runner/src/index.ts   # expect 0
  ```

  (Don't verify via `logs/nanoclaw.log` — container stderr only reaches it at
  `LOG_LEVEL=debug`, so an absent line proves nothing on a default install.)

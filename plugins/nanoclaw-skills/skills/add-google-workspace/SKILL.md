---
name: add-google-workspace
description: Add Google Workspace CLI integration to NanoClaw for Gmail, Calendar, Drive, Docs, and Sheets access. The agent calls gws commands via Bash — no MCP server needed.
---

# Add Google Workspace Integration

This skill adds Google Workspace CLI (`gws`) support to NanoClaw, enabling the assistant to access Gmail, Calendar, Drive, Docs, Sheets, and Slides by calling `gws` commands via Bash.

**Note:** gws v0.8.0 removed the `gws mcp` subcommand (context window bloat from 200-400 tools). The agent uses `gws` as a CLI tool via Bash instead, which returns structured JSON.

## Phase 1: Pre-flight

### Check if already applied

```bash
grep -q "googleworkspace" container/Dockerfile && echo "GWS_APPLIED=true" || echo "GWS_APPLIED=false"
```

If GWS_APPLIED=true, skip to Phase 3 (Host Setup).

### Check host prerequisites

```bash
command -v gws >/dev/null 2>&1 && echo "GWS_INSTALLED=true" || echo "GWS_INSTALLED=false"
```

```bash
ls ~/.config/gws/credentials.enc ~/.config/gws/.encryption_key 2>/dev/null && echo "GWS_AUTHENTICATED=true" || echo "GWS_AUTHENTICATED=false"
```

The `.encryption_key` fallback file is required for the container to decrypt credentials at runtime (since there's no interactive keyring inside the container).

## Phase 2: Apply Code Changes

### Ensure skill remote

```bash
git remote -v
```

If the skill branch is on `upstream`:

```bash
git fetch upstream skill/google-workspace
git merge upstream/skill/google-workspace --no-edit
```

Otherwise add it:

```bash
git remote add gws-skill https://github.com/qwibitai/nanoclaw.git
git fetch gws-skill skill/google-workspace
git merge gws-skill/skill/google-workspace --no-edit
```

Resolve any `package-lock.json` conflicts:

```bash
npm install
git add package-lock.json
git commit --no-edit
```

### Validate build

```bash
npm run build
```

Build must be clean before proceeding.

## Phase 3: Host Setup

### Install Google Workspace CLI (if not installed)

If GWS_INSTALLED=false:

```bash
npm install -g @googleworkspace/cli
```

### Authenticate (if not authenticated)

If GWS_AUTHENTICATED=false:

Tell the user:

> I'll guide you through Google Workspace authentication.
>
> First, run the setup wizard — this creates a Google Cloud project and configures OAuth:

```bash
gws auth setup
```

Tell the user:
> Follow the prompts:
> - Select "External" user type for personal accounts
> - Add your email as a test user
> - Choose "Desktop app" when creating OAuth client
>
> Then authenticate:

```bash
gws auth login
```

> Select "Core Consumer Scopes" for Gmail, Calendar, and Drive access.

### Verify authentication

```bash
gws auth status
```

Should show authenticated status with scopes.

## Phase 4: Rebuild and Restart

### Clear stale agent-runner copies

```bash
rm -r data/sessions/*/agent-runner-src 2>/dev/null || true
```

### Rebuild container image

```bash
cd container && ./build.sh
```

(Bash timeout: 300000ms — container builds can take a few minutes)

### Build and restart

```bash
npm run build
```

Restart the service:

```bash
# macOS (launchd)
launchctl kickstart -k gui/$(id -u)/com.nanoclaw

# Linux (systemd)
systemctl --user restart nanoclaw
```

## Phase 5: Verify

Tell the user:

> Google Workspace is connected! Try sending a message:
>
> - "Check my calendar for today"
> - "What are my recent emails?"
> - "Create a calendar event for Friday at 3pm"

### Add usage hints to group CLAUDE.md

Add to the main group's `CLAUDE.md`:

```markdown
## Google Workspace

You have access to the `gws` CLI for Google Workspace (Gmail, Calendar, Drive, Docs, Sheets, etc.). Credentials are mounted at `/home/node/.config/gws/`. Use Bash to run gws commands — they return structured JSON.

Common commands:
- `gws gmail messages list --params '{"maxResults": 10}'` — list recent emails
- `gws gmail messages get --params '{"userId": "me", "id": "MESSAGE_ID"}'` — read an email
- `gws calendar events list --params '{"calendarId": "primary", "maxResults": 10}'` — list calendar events
- `gws calendar events insert --params '{"calendarId": "primary"}' --body '{"summary": "...", "start": {...}, "end": {...}}'` — create event
- `gws schema <service>.<resource>.<method>` — inspect any method's parameters
```

## Troubleshooting

### "gws: command not found" in container

- Verify Dockerfile has `@googleworkspace/cli` in the npm install line
- Rebuild: `cd container && ./build.sh`

### "Authentication failed" or token errors

- Check host auth: `gws auth status`
- Re-authenticate: `gws auth login`
- Verify credentials exist: `ls ~/.config/gws/`

### Agent doesn't use gws commands

- Ensure gws is installed in container: `docker run --rm --entrypoint /bin/sh nanoclaw-agent -c "gws --version"`
- Ensure credentials are mounted: check container-runner.ts for the gws mount block
- Add usage instructions to your group's CLAUDE.md (see Phase 5)

### Token refresh fails

- Delete and re-authenticate: `rm ~/.config/gws/credentials.enc && gws auth login`
- Ensure `.encryption_key` fallback file exists: `ls ~/.config/gws/.encryption_key`

## Available Services

The agent can use any gws service via Bash. Example commands:

```bash
# List recent emails
gws gmail messages list --params '{"maxResults": 10}'

# List calendar events
gws calendar events list --params '{"calendarId": "primary", "maxResults": 10}'

# Inspect any method's parameters
gws schema gmail.users.messages.list
```

Available services: `gmail`, `calendar`, `drive`, `docs`, `sheets`, `slides`, `tasks`, `people`, `chat`, `keep`, `meet`

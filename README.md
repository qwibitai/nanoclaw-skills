# NanoClaw Skills

Official skill marketplace for [NanoClaw](https://github.com/qwibitai/nanoclaw) — a lightweight AI assistant that runs agents in isolated containers.

## Usage

This marketplace is auto-registered in NanoClaw's `.claude/settings.json` and loads automatically when you start Claude Code in a NanoClaw project. All skills are available immediately — no manual installation needed.

## Included Skills

### Channels
- `/add-whatsapp` — WhatsApp via QR code or pairing code
- `/add-telegram` — Telegram bot channel
- `/add-telegram-swarm` — Agent Swarm for Telegram (requires Telegram)
- `/add-slack` — Slack via Socket Mode
- `/add-discord` — Discord bot channel
- `/add-gmail` — Gmail integration (tool or full channel)
- `/add-emacs` — Emacs chat buffer and org-mode integration via local HTTP bridge

### Integrations
- `/add-voice-transcription` — OpenAI Whisper voice transcription (requires WhatsApp)
- `/use-local-whisper` — Local whisper.cpp transcription (requires voice-transcription)
- `/add-image-vision` — Image attachment processing (requires WhatsApp)
- `/add-pdf-reader` — PDF text extraction
- `/add-ollama-tool` — Ollama MCP server for local models
- `/add-parallel` — Parallel AI integration
- `/x-integration` — X (Twitter) posting and interaction
- `/add-reactions` — WhatsApp emoji reactions
- `/add-compact` — Manual context compaction command
- `/channel-formatting` — Convert Markdown to native channel syntax (WhatsApp, Telegram, Slack)

### Tools
- `/claw` — CLI tool to run NanoClaw agents from the terminal
- `/convert-to-apple-container` — Switch from Docker to Apple Container
- `/add-macos-statusbar` — macOS menu bar status indicator with start/stop controls
- `/init-onecli` — Install OneCLI Agent Vault and migrate .env credentials
- `/use-native-credential-proxy` — Use built-in .env credential proxy instead of OneCLI
- `/qodo-pr-resolver` — Review and resolve Qodo PR issues
- `/get-qodo-rules` — Load Qodo coding rules before code tasks

## License

MIT

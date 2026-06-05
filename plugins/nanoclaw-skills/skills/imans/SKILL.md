---
name: imans
description: Use the Imans CLI from NanoClaw or Claude Code containers to query Imans workspace, catalog, products, variants, and sales orders as JSON.
---

# Imans CLI

Use `imans` when a NanoClaw agent needs read-only Imans workspace, catalog, product variant, sales order, order item, or classification data.

## Container Setup

- The `imans` binary must be installed inside the container or host where the agent runs commands.
- Install: `curl -fsSL https://imans.ai/install | bash`
- Homebrew: `brew install imans-ai/tap/imans`
- Verify: `imans version`
- Login interactively: `imans login`
- Login for automation: `imans login --token-env IMANS_TOKEN` or `imans login --token-stdin < token.txt`
- Test auth: `imans auth test --quiet`

## Usage Rules

- Prefer `--json` for data parsing and summarization.
- Prefer narrow filters before using `--all --json`.
- Use `--profile <name>` when the user names a workspace.
- Keep chat/mobile responses compact; summarize large results.
- Confirm before exposing customer/order datasets or broad exports.
- Never print or request raw API tokens.

## Commands

- Workspace: `imans workspace get --json`
- Profiles: `imans profile list`
- Products: `imans products list --all --json`
- Product search: `imans products list --search "<query>" --json`
- Product details: `imans products get <id> --json`
- Variants: `imans product-variants list --product-id <product-id> --all --json`
- Sales orders: `imans sales-orders list --order-date-from <yyyy-mm-dd> --order-date-to <yyyy-mm-dd> --all --json`
- Sales order details: `imans sales-orders get <id> --json`
- Sales order items: `imans sales-order-items list --order-id <order-id> --all --json`
- Classifications: `imans sales-order-classifications list --all --json`

## Safety

- Treat Imans business data as sensitive.
- Prefer `--token-env` or `--token-stdin` over `--token`.
- Exit code `3` means auth failed, `4` means insufficient scope, `5` means not found, `6` means network failure, and `7` means API server error.

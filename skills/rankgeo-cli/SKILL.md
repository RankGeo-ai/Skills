---
name: rankgeo-cli
description: Use the RankGeo CLI to run GEO audits, manage suggestions, trigger ingestion, and check scores from the terminal. Use when the user asks about GEO Score, audits, suggestions, rankgeo CLI, or wants to analyze how GenAI platforms see their brand.
---

# RankGeo CLI

## Installation

```bash
npm install -g @rankgeo/cli
# or install a specific version:
npm install -g @rankgeo/cli@latest
```

Verify: `rankgeo --help`

## Authentication

```bash
rankgeo login                    # Opens browser for OAuth (default)
rankgeo login --token rgk_xxx    # Headless/CI — paste API key
rankgeo whoami                   # Show account email + active workspace
rankgeo logout                   # Clear stored credentials
```

Config is stored at `~/.config/rankgeo/config.json` (mode 600).

## Core concepts

- **Workspace** — one product URL. All data is scoped to a workspace.
- **GEO Score** — 0-100 metric: `mention_rate × 0.6 + citation_rate × 0.4`
- **Audit** — a GEO Score check run against Gemini with Google Search grounding.
- **Suggestion** — AI-generated recommendation to improve score. Statuses: `pending`, `applied`, `dismissed`.
- **Prompt** — a search query monitored during audits to check brand visibility.
- **Competitor** — a rival brand tracked in audit prompts to measure share of voice.

## Workspace commands

```bash
rankgeo workspace list                                      # List all (* = active)
rankgeo workspace get <id-or-name>                          # Show one workspace
rankgeo workspace use <id-or-name>                          # Set sticky active workspace
rankgeo workspace create <product-url> [--name <n>]         # Create new
rankgeo workspace delete <id-or-name>                       # Remove
```

Every command that needs a workspace resolves it in this order: `--workspace` flag > `current_workspace` in config > positional arg. Name resolution performs a one-shot lookup.

## Prompt commands

Prompts are the search queries used during audits to check brand visibility in GenAI responses. They replace what was previously called "keywords."

```bash
rankgeo prompt list [ws]                             # List all prompts
rankgeo prompt add [ws] "your search query"          # Add a prompt
rankgeo prompt remove [ws] <prompt-id>               # Remove a prompt
```

Options:
- `rankgeo prompt list` supports `--json` and `--plain`
- `rankgeo prompt add` supports `--json`
- `rankgeo prompt remove` supports `--json`

Example workflow:
```bash
# See current prompts
rankgeo prompt list <ws>

# Add a new prompt
rankgeo prompt add <ws> "AI Search Optimization"

# Remove an outdated prompt
rankgeo prompt remove <ws> <prompt-id>

# Re-run audit to test new prompts
rankgeo audit trigger <ws>
```

## Competitor commands

Competitors are brands tracked during audits. They appear in prompt results and feed into share-of-voice calculations.

```bash
rankgeo competitor list [ws]                              # List all competitors
rankgeo competitor add [ws] "Competitor Name"             # Add a competitor
rankgeo competitor add [ws] "Name" --url "https://..."   # Add with URL
rankgeo competitor remove [ws] <competitor-id>            # Remove a competitor
```

Options:
- `rankgeo competitor list` supports `--json` and `--plain`
- `rankgeo competitor add` supports `--url <url>` and `--json`
- `rankgeo competitor remove` supports `--json`

Example workflow:
```bash
# See who you're tracked against
rankgeo competitor list <ws>

# Add a new competitor
rankgeo competitor add <ws> "Otterly AI" --url "https://otterly.ai"

# Remove a competitor you no longer care about
rankgeo competitor remove <ws> <competitor-id>

# Re-audit to see updated share of voice
rankgeo audit trigger <ws>
```

## Audit commands

```bash
rankgeo audit list [ws]                                     # List audit history
rankgeo audit latest [ws]                                   # Latest completed score
rankgeo audit show [ws] <aid>                               # Full per-prompt detail
rankgeo audit trigger [ws]                                  # Run with live spinner
rankgeo audit trigger [ws] --no-wait                        # Fire-and-forget, returns audit ID
rankgeo audit trigger [ws] --poll-interval 5000             # Custom poll interval (default: 3000ms)
rankgeo audit export <ws> <aid> --format json               # JSON to stdout
rankgeo audit export <ws> <aid> --format md                 # Markdown to stdout
```

`trigger` polls every 3s (configurable via `--poll-interval`). Shows spinner with progress updates. Default timeout: 10 min.

## Ingestion

```bash
rankgeo ingest trigger [ws]                         # Re-scrape product URL (wait for completion)
rankgeo ingest trigger [ws] --no-wait               # Fire-and-forget, returns workspace ID
rankgeo ingest trigger [ws] --poll-interval 5000    # Custom poll interval (default: 3000ms)
```

Polls workspace status until `ingestionStatus` is `done` or `error`, showing page count in spinner.

## Suggestion commands

```bash
rankgeo suggestion list [ws]                                # All suggestions
rankgeo suggestion list [ws] --status pending               # Filter by status
rankgeo suggestion list [ws] --category content             # Filter by category (content|technical|credibility|competitive)
rankgeo suggestion show [ws] <sid>                          # Full detail
rankgeo suggestion apply [ws] <sid>                         # Mark applied (auto-schedules re-audit in 24h)
rankgeo suggestion dismiss [ws] <sid>                       # Mark dismissed
```

## Report commands

```bash
rankgeo report list [ws]    # List recommendations reports
rankgeo report show [ws] <rid>  # Show full structured report
```

## Output modes

These flags are available:

| Flag | Effect |
|------|--------|
| `--json` | Raw JSON to stdout (pipe to `jq`) — inherited by all subcommands |
| `--plain` | TSV output for `awk` — available on `list`-type subcommands only |
| `--no-color` | Disable ANSI |
| `--workspace <id-or-name>` | Override active workspace on per-command basis |

Logs/progress go to stderr. Results go to stdout. `rankgeo audit show <ws> <aid> > audit.json` works cleanly. Use `[ws]` notation where workspace is optional — resolved from config if omitted.

## Configuration & env vars

```bash
rankgeo config list                 # Show all stored config values
rankgeo config get api-url          # Show a specific config key
rankgeo config set api-url <url>    # Write a config key (api-url, api-key, consent-url, current-workspace)
```

`config list` redacts `api-key` as `<prefix>.****`. Use `--json` for structured output.

Environment variable overrides (per-invocation, never written to disk):

| Variable | Overrides |
|----------|-----------|
| `RANKGEO_API_URL` | Backend URL |
| `RANKGEO_API_KEY` | API key |
| `RANKGEO_CONFIG` | Config file path |
| `NO_COLOR` | Disable color output |

## API Key commands

```bash
rankgeo api-key list                              # List all API keys (label, prefix, last used)
rankgeo api-key create "my-key"                   # Generate a new API key (shown once)
rankgeo api-key revoke <key-id>                   # Revoke/delete an API key
```

Options:
- `rankgeo api-key list` supports `--json` and `--plain`

API keys use the format `rgk_<prefix>.<random>`. The full key is printed once on creation — it cannot be retrieved later.

## Exit codes for scripting

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error |
| 4 | Not found (bad workspace ID, etc.) |
| 5 | Auth error (run `rankgeo login`) |
| 6 | Conflict (audit already running) |
| 7 | Rate limited |
| 8 | Unknown tool |

## Common workflows

### Check current GEO Score
```bash
rankgeo audit latest <ws>
```

### Run a full audit and see the score
```bash
rankgeo audit trigger <ws>
```

### Apply the highest-priority suggestion
```bash
rankgeo suggestion list <ws> --status pending --json | jq -r '.[0].id'
rankgeo suggestion apply <ws> <id>
```

### Create and use an API key for CI
```bash
rankgeo api-key create "ci-key"
# Save the printed key as RANKGEO_API_KEY in CI secrets
rankgeo login --token <printed-key>
```

### Bulk create workspaces from a list
```bash
cat urls.txt | xargs -I{} rankgeo workspace create {} --name {}
```

### CI pipeline: audit → fail if score drops
```bash
rankgeo audit trigger <ws> --json | jq -r '.score'
# Compare with threshold, exit 1 if below
```

### Re-audit after applying a change
```bash
rankgeo suggestion apply <ws> <sid>
rankgeo audit trigger <ws>
```

### Full optimization loop
```bash
# 1. Check current state
rankgeo audit latest <ws>
rankgeo prompt list <ws>
rankgeo competitor list <ws>

# 2. Adjust prompts and competitors
rankgeo prompt add <ws> "new search query to rank for"
rankgeo competitor add <ws> "New Competitor" --url "https://..."

# 3. Run audit
rankgeo audit trigger <ws>

# 4. Review and act on suggestions
rankgeo suggestion list <ws> --status pending
rankgeo suggestion apply <ws> <suggestion-id>

# 5. Re-audit after changes take effect
rankgeo audit trigger <ws>
```

## Tips

- Use `rankgeo workspace use` to set a default workspace so you don't pass `<ws>` on every command.
- `RANKGEO_API_KEY` env var overrides the config file — useful for ephemeral containers and CI secrets.
- The CLI respects `NO_COLOR=1` and auto-disables color when stdout is not a TTY.
- API keys created via `rankgeo api-key create` are printed once and not stored in config.
- Prompts and competitors are auto-detected during ingestion, but you can manually add/remove to fine-tune what's tracked.
- After adding prompts or competitors, re-run the audit to see updated results.

<div align="center">

# Things AI

中文版本：[README.zh-CN.md](README.zh-CN.md)

**AI-first Things 3 CLI and MCP server for macOS**

[![CI](https://img.shields.io/github/actions/workflow/status/guangzan/things-ai/ci.yml?style=flat-square&label=CI)](https://github.com/guangzan/things-ai/actions)
[![Rust](https://img.shields.io/badge/Rust-1.89+-dea584?style=flat-square&logo=rust&logoColor=white)](https://www.rust-lang.org)
[![Platform](https://img.shields.io/badge/Platform-macOS-black?style=flat-square&logo=apple&logoColor=white)](https://culturedcode.com/things/)
[![License](https://img.shields.io/badge/License-MIT-blue?style=flat-square)](LICENSE)

[Features](#features) · [Install](#install) · [Quick start](#quick-start) · [MCP](#mcp) · [Configuration](#configuration) · [Docs](#documentation)

</div>

`ta` is a production-oriented command line interface and MCP server for
[Things 3](https://culturedcode.com/things/). It is written in Rust and built
around a strict safety boundary:

- **Reads** query the live Things SQLite database opened read-only
- **Writes** go through the official Things URL scheme or AppleScript, then are
  verified against the database
- **No direct database writes** are performed
- Repeating-rule edits are opened in Things rather than written to undocumented
  internal tables

> [!NOTE]
> This project is unofficial and is not affiliated with Cultured Code.

## Features

- Sidebar views: Inbox, Today, Upcoming, Anytime, Someday, Logbook, Trash,
  Deadlines, Repeating, and Projects
- Boolean filters for status, type, project, area, heading, tags, date ranges,
  overdue state, checklist, reminders, repeating items, and trash state
- Case-insensitive text search plus Rust-regex title/notes filters
- Stable UUID resolution by full id, 4+ character prefix, exact title, or unique
  title substring; ambiguous references return candidates instead of guessing
- Create, partial edit, complete, cancel, reopen, move between
  projects/areas/headings, and move to Trash
- Time-based reminders with strict `HH:MM` input
- Tag and area create/rename/delete with usage counts
- Bulk Things JSON import from a file or stdin, with per-item identity read-back
- Write verification based on pre-write snapshots and observed database state
- JSON and NDJSON output with stable error codes
- MCP over stdio with optional read-only mode and structured tool results
- AI-sized list results: notes are previewed to 200 characters, with full text
  available through `show` / `get`

## Install

Requires **macOS** and **Things 3**.

### Homebrew

```sh
brew install guangzan/tap/things-ai
```

This installs the prebuilt universal package for Apple Silicon and Intel Macs.
Upgrade it with `brew upgrade things-ai`.

### Shell installer

```sh
curl --proto '=https' --tlsv1.2 -LsSf \
  https://github.com/guangzan/things-ai/releases/latest/download/things-ai-installer.sh | sh
```

### From source

Source builds require **Rust 1.89+**:

```sh
cargo install --path crates/things-ai
```

Or build a release binary:

```sh
cargo build --release -p things-ai
./target/release/ta --help
```

### Migrating from the old `things` CLI

The command, Homebrew formula, config path, and environment variables changed
without compatibility shims:

```sh
brew uninstall things
brew install guangzan/tap/things-ai
mv ~/.config/things ~/.config/things-ai   # if it exists
```

Replace `things` with `ta` in scripts and rename environment variables:

| Old | New |
|---|---|
| `THINGSDB` | `THINGS_AI_DB` |
| `THINGS_AUTH_TOKEN` | `THINGS_AI_AUTH_TOKEN` |
| `THINGS_READ_ONLY` | `THINGS_AI_READ_ONLY` |
| `THINGS_VERIFY_TIMEOUT_MS` | `THINGS_AI_VERIFY_TIMEOUT_MS` |
| `THINGS_VERIFY_POLL_MS` | `THINGS_AI_VERIFY_POLL_MS` |

## macOS permissions

1. Grant **Full Disk Access** to the terminal or app that runs `ta`.
   The Things database lives under `~/Library/Group Containers`.
2. Open **Things → Settings → General** and enable **Things URLs**.
3. If you use `set_status`, `delete`, or tag mutations, allow automation access
   to Things when macOS prompts.

Run the built-in diagnostic:

```sh
ta doctor
ta doctor --json
```

## Quick start

```sh
# Read
ta today --json
ta search "quarterly" --json
ta show <id> --json
ta list --status open --tag focus --overdue --json

# Inspect a write without sending it
ta add "Review proposal" \
  --when tomorrow --reminder 09:30 --deadline 2026-10-24 --tags work \
  --dry-run --json

# Write and verify
ta add "Review proposal" --when tomorrow --reminder 09:30 --deadline 2026-10-24
ta edit <id> --append-notes "Ask legal for feedback"
ta edit <id> --reminder 08:45
ta done <id>
ta reopen <id>

# Tags, areas, and import
ta tag add "deep-work"
ta area add "Work"
ta import ./payload.json
cat payload.json | ta import -

# Destructive operations always require --yes
ta delete <id> --yes
ta empty-trash --yes
```

Use `--format ndjson` for streaming-friendly list output.
Use `--read-only` to disable every mutation for a process.

## MCP

Run the server over stdio:

```sh
ta mcp
```

Example Codex configuration:

```toml
[mcp_servers.things-ai]
command = "/absolute/path/to/ta"
args = ["mcp"]
```

Read-only agent:

```sh
ta mcp --read-only
```

See [docs/MCP.md](docs/MCP.md) for the full tool reference.

## Import

```sh
ta import payload.json
cat payload.json | ta import -
ta import payload.json --dry-run --json
```

The payload must be a JSON array in the format accepted by `things:///json`.

With verification enabled (the default), `ta import --json` returns two
per-item arrays in payload order:

- `created` for every top-level or nested to-do/project
- `updated` for every `operation: update`

`confirmed` proves identity for creates, and covers status-only updates for
updates. Mixed-field updates stay explicitly unconfirmed. When an item is
unconfirmed, search for its title before retrying so a retry does not duplicate
it.

> [!TIP]
> Prefer `--dry-run --json` before a bulk import. Full verification semantics
> are documented in [docs/COMMANDS.md](docs/COMMANDS.md#import).

## Configuration

Config file: `~/.config/things-ai/config.toml`

```toml
read_only = false

[output]
format = "text"   # text, json, or ndjson

[write]
verify_timeout_ms = 8000
verify_poll_ms = 250
create_missing_tags = false

[mcp]
read_only = false
default_page_size = 20
max_page_size = 100
```

Environment variables override the file:

| Variable | Purpose |
|---|---|
| `THINGS_AI_DB` | Explicit `main.sqlite` path |
| `THINGS_AI_AUTH_TOKEN` | URL-scheme auth token |
| `THINGS_AI_READ_ONLY` | `1` / `true` disables writes |
| `THINGS_AI_VERIFY_TIMEOUT_MS` | Verification timeout |
| `THINGS_AI_VERIFY_POLL_MS` | Verification poll interval |

If more than one `ThingsData-*` directory exists, database discovery refuses to
guess. Set `--db` or `THINGS_AI_DB`.

## Documentation

- [Command reference](docs/COMMANDS.md)
- [MCP setup and tool reference](docs/MCP.md)
- [Architecture and safety model](docs/ARCHITECTURE.md)
- [Compatibility and limitations](docs/COMPATIBILITY.md)
- [AI skill](skills/things-ai/SKILL.md)

## Development

```sh
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

Fixture tests do not require Things or personal data. Live read-only tests are
skipped when the Things database is unavailable.

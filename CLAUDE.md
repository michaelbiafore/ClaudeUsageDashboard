# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Tests use only `unittest` from the stdlib (no pytest, no deps to install).

```
# Run the full test suite
python -m unittest discover -s tests -v

# Run a single test module / class / method
python -m unittest tests.test_scanner -v
python -m unittest tests.test_scanner.TestProjectNameFromCwd -v
python -m unittest tests.test_scanner.TestProjectNameFromCwd.test_windows_path

# Scan logs, then start the dashboard at localhost:8080
python cli.py dashboard

# Scan only (incremental — keyed on file path + mtime)
python cli.py scan
python cli.py scan --projects-dir /path/to/transcripts

# Terminal reports
python cli.py today
python cli.py week
python cli.py stats
```

CI (`.github/workflows/tests.yml`) runs `python -m unittest discover -s tests -v` on Python 3.9 / 3.11 / 3.12 on every push and PR to `main`. Stdlib-only is a hard constraint — do not add third-party dependencies.

## Architecture

Three top-level modules form a pipeline: **scanner.py → SQLite → dashboard.py / cli.py**. All state lives in `~/.claude/usage.db`; the project itself ships no data.

### Data flow

1. Claude Code writes one JSONL file per session under `~/.claude/projects/` (and `~/Library/Developer/Xcode/CodingAssistant/ClaudeAgentConfig/projects/` on macOS).
2. `scanner.scan()` walks those directories, parses each JSONL, and writes to two tables: `sessions` (one row per session) and `turns` (one row per assistant message with token usage). A third table, `processed_files`, stores `(path, mtime, lines)` so re-scans only read newly appended lines.
3. `dashboard.py` queries the same DB and serves a single-page HTML/JS app at `localhost:8080`. The server sends *all* daily/hourly/session rows in one JSON payload; range and model filtering happen client-side in the browser.
4. `cli.py` is a thin shell over the same DB for terminal output.

### `turns` is the source of truth, not `sessions`

`sessions` row totals are aggregated rollups recomputed at the end of every scan by re-summing `turns` (see `scanner.scan()` final `UPDATE sessions ...` block). When adding new statistics, query `turns` and group/join up to `sessions` for project/branch context — do not rely on `sessions.total_*` columns staying in sync mid-scan. All commands in `cli.py` and dashboard queries already follow this pattern.

### Streaming dedup via `message.id`

Claude Code logs multiple JSONL records per API response, all sharing the same `message.id`; only the last one has the final usage tallies. `parse_jsonl_file` keeps the last record per `message_id` in a dict and drops earlier duplicates. A conditional unique index `idx_turns_message_id` (non-null only) plus `INSERT OR IGNORE` in `insert_turns` prevents the same message being counted twice across re-scans. If you change how turns are inserted, preserve this invariant — duplicate counting will roughly 2x reported usage.

### Incremental scan correctness

For an existing file, the scanner reads from line `old_lines + 1` onward and additively updates `sessions.total_*`. Because `INSERT OR IGNORE` may then drop newly-parsed turns that duplicate already-stored `message_id`s, the additive totals would drift. The final `UPDATE sessions SET total_* = (SELECT SUM ...) FROM turns` reconciles this. Any new write path that touches `turns` must run before this reconciliation block or trigger a similar resync.

### Model identification & pricing

`PRICING` in `cli.py` and an equivalent table in `dashboard.py` are keyed by exact model name. `get_pricing` falls back to a substring match on `opus` / `sonnet` / `haiku` — anything else (e.g. `gemma`, `glm`) returns `None` and is excluded from cost totals (rendered as `n/a`). When new Anthropic models ship, add them to both `PRICING` tables; do not widen the substring match to cover non-Anthropic models.

For a session's "primary" model, `MODEL_PRIORITY` (opus > sonnet > haiku) wins over the most-recent-update model — this matters because subagent calls in a session may use haiku while the main loop uses opus, and we want the session tagged with opus.

### Dashboard UI conventions

`dashboard.py` embeds a large HTML/JS template as a raw string (`HTML_TEMPLATE`). Chart.js is loaded from a CDN; no build step. Range buttons (`week`, `month`, `prev-month`, `7d`, `30d`, `90d`, `all`) and the model filter both live in the URL query string (`?range=...&models=...`, read via `URLSearchParams`) so views are bookmarkable. The server payload includes *all* historical rows — filtering is intentionally client-side to keep the server stateless and make the rescan button cheap.

## Conventions

- **No third-party packages.** README and CI assume stdlib only (`sqlite3`, `http.server`, `json`, `pathlib`). Adding a dependency is a breaking change for users.
- **Costs are computed per-turn, not per-session**, in both CLI and dashboard. Sessions that span multiple models would produce wrong totals if costed at the session level.
- **Timestamps in the DB are ISO8601 UTC strings** (e.g. `2026-04-08T09:30:00Z`). Day/hour filtering uses `substr(timestamp, 1, 10)` and `substr(timestamp, 12, 2)` — keep this format when inserting.

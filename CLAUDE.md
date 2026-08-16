# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

TrendRadar (v4.x) is a hot-news aggregation and analysis tool written in Python (>= 3.10). It crawls hot lists from ~11 Chinese platforms (知乎, 微博, 百度热搜, etc. via the `newsnow` API), filters them against user-defined keywords, generates HTML/TXT reports, and pushes notifications to channels like 飞书/企业微信/钉钉/Telegram/Slack/Email/ntfy/Bark. A companion FastMCP server exposes the collected data as AI-queryable tools (trend analysis, search, sentiment, etc.).

Two independently runnable components share the same codebase:
- `trendradar/` — the crawler/report/notification pipeline (CLI entry point `trendradar`, `python -m trendradar`).
- `mcp_server/` — a FastMCP server exposing TrendRadar's stored data as MCP tools (entry point `trendradar-mcp`, `python -m mcp_server.server`). It only *reads* data the crawler already collected; it does not crawl itself.

## Commands

Dependency management uses `uv` (see `setup-mac.sh` / `setup-windows.bat`); `requirements.txt` is the pip-compatible fallback used by CI and Docker.

```bash
uv sync                                   # install deps (local dev)
# or: pip install -r requirements.txt

python -m trendradar                      # run one crawl/report/notify cycle
python -m mcp_server.server                          # MCP server, stdio transport (for Claude Desktop/Cherry Studio/Cursor)
python -m mcp_server.server --transport http --port 3333   # MCP server, HTTP transport
```

There is no test suite and no configured linter/formatter in this repo (no `tests/`, no ruff/black/pytest config) — don't invent commands for them.

Docker (production deployment, see `docker/`):
```bash
docker compose -f docker/docker-compose.yml up -d
docker exec -it trend-radar python manage.py start_webserver   # serve output/ reports on :8080
```

## Architecture

### Crawler pipeline (`trendradar/__main__.py`)

`NewsAnalyzer` is the orchestrator. Startup order: `load_config()` → build an `AppContext` → init proxy/storage → `_crawl_data()` → `_execute_mode_strategy()`, wrapped so `ctx.cleanup()` always runs (flushes retention cleanup + closes DB connections).

**`AppContext` (`trendradar/context.py`) is the central seam** — nearly every module is wired through it rather than called directly, so it's the fastest way to see how the pieces connect (time/timezone, storage manager, frequency-word matching, stats, report/HTML generation, notification rendering/dispatch). New cross-cutting behavior should generally go through `AppContext`, not by importing `core`/`report`/`notification` submodules directly into `__main__.py`.

**Three report modes**, defined declaratively in `NewsAnalyzer.MODE_STRATEGIES` and driven by `REPORT_MODE` in config:
- `daily` — cumulative summary of all matched news for the day, sent once.
- `current` — current live rankings; re-evaluates against the full day's history each run so stats stay complete.
- `incremental` — only pushes when genuinely new titles appear (zero repeats); `current`-mode notifications diff against `read_today_titles`/`detect_new_titles` from the storage layer.

### Storage layer (`trendradar/storage/`)

`StorageManager` auto-selects a backend unless `STORAGE.BACKEND` is forced:
- On GitHub Actions with S3-compatible credentials present (bucket/key/secret/endpoint, from config or `S3_*` env vars) → `RemoteStorageBackend` (S3-compatible, e.g. Cloudflare R2/OSS/COS).
- Otherwise (Docker/local, or GH Actions without remote config) → `LocalStorageBackend` (SQLite in `output/`, which the MCP server also reads from).

This detection (`is_github_actions()`, `is_docker()`) determines behavior in several places (proxy usage, browser auto-open, storage backend) — check `StorageManager`/`NewsAnalyzer` env detection before adding new environment-dependent logic rather than re-implementing it.

### Config loading (`trendradar/core/loader.py`)

Config is merged from `config/config.yaml` with environment-variable overrides layered on top (`_get_env_str`/`_get_env_bool`/`_get_env_int`), so the same YAML works unchanged across local/Docker/GitHub Actions with secrets injected via env vars. Notification channels support multiple accounts per channel via `;`-separated values, parsed by `trendradar/core/config.py` (`parse_multi_account_config`); paired configs (e.g. Telegram token + chat_id) are validated for matching counts before dispatch.

### Notification pipeline (`trendradar/notification/`)

`renderer.py`/`formatters.py` build per-channel content (Markdown/mrkdwn/plain text), `splitter.py` batches long content to stay under each channel's size limits (feishu/dingtalk have distinct batch sizes), and `dispatcher.py` (`NotificationDispatcher`) fans a rendered report out to every configured channel. `push_manager.py` tracks once-per-day push state when `PUSH_WINDOW.ONCE_PER_DAY` is enabled.

### MCP server (`mcp_server/`)

`server.py` registers `@mcp.tool` functions grouped by concern in `tools/` (`data_query`, `analytics`, `search_tools`, `config_mgmt`, `system`, `storage_sync`), backed by `services/` (`data_service` wraps the SQLite storage backend, `cache_service`, `parser_service`). Tool instances are lazily created singletons via `_get_tools()`. `resolve_date_range` is meant to be called first for any natural-language date expression ("本周", "最近7天") so all tools compute dates the same way server-side instead of relying on model-side date arithmetic.

## Key config files

- `config/config.yaml` — all runtime settings (platforms, report mode, weight config, storage, push window, notification channels).
- `config/frequency_words.txt` — keyword watch list (supports `+must`, `!filter`, `@N` limit, `[GLOBAL_FILTER]` syntax); blank groups separated by empty lines are tracked independently.
- `output/` — generated reports and local SQLite data; never hand-edit.

## Working Guidelines

- **No double verification.** Edits are verified automatically by the tooling — do not add extra re-read/re-check steps after making a change, and do not include manual double-check instructions in code or docs.
- **Be concise.** Keep responses, summaries, and commit messages short and to the point.
- **Stay in scope.** Do exactly what was asked — no drive-by refactors, extra features, or unrelated cleanups.
- **Plan before executing.** For any non-trivial change, present a clear plan first, then implement.
- **Use high effort for complex code.** Apply high (or xhigh) reasoning effort when working on complex logic such as the crawler, report generation, or MCP server internals.

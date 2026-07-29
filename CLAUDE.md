# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Project Overview

TrendRadar (v4.x) is a hot-news aggregation and analysis tool written in Python (>= 3.10).

- `trendradar/` — main package: `core/`, `crawler/`, `notification/`, `report/`, `storage/`, `utils/`. CLI entry point: `trendradar` (`trendradar/__main__.py`).
- `mcp_server/` — FastMCP server exposing TrendRadar as MCP tools: `services/`, `tools/`, `utils/`. Entry point: `trendradar-mcp` (`mcp_server/server.py`).
- `config/config.yaml` — runtime configuration; `config/frequency_words.txt` — keyword watch list.
- `output/` — generated reports; do not hand-edit.
- Dependencies are declared in `pyproject.toml` (requests, pytz, PyYAML, fastmcp, websockets).

## Working Guidelines

- **No double verification.** Edits are verified automatically by the tooling — do not add extra re-read/re-check steps after making a change, and do not include manual double-check instructions in code or docs.
- **Be concise.** Keep responses, summaries, and commit messages short and to the point.
- **Stay in scope.** Do exactly what was asked — no drive-by refactors, extra features, or unrelated cleanups.
- **Plan before executing.** For any non-trivial change, present a clear plan first, then implement.
- **Use high effort for complex code.** Apply high (or xhigh) reasoning effort when working on complex logic such as the crawler, report generation, or MCP server internals.

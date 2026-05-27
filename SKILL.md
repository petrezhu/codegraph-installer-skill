---
name: codegraph-bootstrap
description: Ensure CodeGraph is installed, indexed for the current project, registered as an MCP server, and wired into startup hooks. Use when setting up or repairing CodeGraph for Codex, Claude Code, OpenCode, Hermes, MCP registration, or session-start hook automation.
---

# CodeGraph Bootstrap

Use this skill when the user wants CodeGraph to be available automatically across coding agents.

## Quick Start

Run the bundled wrapper:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "$HOME\.codex\skills\codegraph-bootstrap\scripts\Invoke-CodeGraphBootstrap.ps1"
```

The wrapper calls the bundled `scripts\Install-CodeGraphBootstrap.ps1` first and falls back to `$HOME\.codex\tools\codegraph\Install-CodeGraphBootstrap.ps1` only for older local installations.

## What It Ensures

- Installs `@colbymchenry/codegraph` with npm when the `codegraph` CLI is missing.
- Initializes and indexes the current project if `.codegraph\codegraph.db` is absent.
- Runs `codegraph install --target all --location global --yes` to register MCP for every agent supported by the installed CodeGraph version, including Codex CLI, Claude Code, OpenCode, and Hermes.
- Installs startup hook wrappers for Codex, Claude Code, OpenCode, and Hermes using the shared `Ensure-CodeGraph.ps1`.
- Logs status to `$HOME\.codegraph-agent-bootstrap.log`.

## SQL Fallback

When MCP tools are not visible in the current session or a custom graph query is needed, use:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "$HOME\.codex\skills\codegraph-bootstrap\scripts\Invoke-CodeGraphSql.ps1" -ListRecipes
```

Load `references/query-recipes.md` only when the task needs schema details, SQL examples, dead-code checks, hotspots, custom dependency queries, or fallback analysis against `.codegraph\codegraph.db`.

## Operational Notes

- Restart already-running agents after the first setup. New MCP server definitions and hooks usually load only at process startup.
- Use `-SkipIndex` when only repairing MCP/hook registration and avoiding a project indexing pass.
- Use `-Targets all` by default, or pass another CodeGraph-supported target list when needed.
- The hook script exits successfully even when bootstrap work fails, so agent startup is not blocked. Check the log file for details.
- Full user-facing documentation is in `README.md`.

## Verification

After running the wrapper, verify with:

```powershell
codegraph status --json
codex mcp list
```

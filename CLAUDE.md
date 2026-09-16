# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This folder holds small standalone browser projects: a tic-tac-toe game in `tictactoe/index.html` and a security posture dashboard in `FableTechDay/`. There is no package manager, build step, test runner, or linter. The folder is a git repository (default branch `main`).

## Running

Open the HTML file directly in a browser. From PowerShell:

```powershell
Start-Process .\tictactoe\index.html
Start-Process .\FableTechDay\index.html
```

After editing, refresh the browser tab to see changes. No server is required. Node is not installed on this machine; for a headless render check, Edge works: `msedge --headless=new --screenshot=out.png file:///<path>/index.html#<tab>`.

## Architecture: `tictactoe/index.html`

Everything (markup, CSS, JS) lives in this single file with no external dependencies. Keep it that way unless asked otherwise.

- **Theme colors** are CSS custom properties in the `:root` block at the top of the `<style>` section. Change colors there rather than in individual rules. The current theme is light; translucent `rgba` highlights (active mode button, winning cells, panel shadow) are tuned for a light background and should be adjusted if the theme changes.
- **State** is a 9-element `board` array (`'X'`, `'O'`, or `null`), plus `current`, `gameOver`, `vsComputer`, `startingPlayer`, and a `scores` object. The 9 cell buttons are created in JS at load and stored in `cells`, indexed to match `board`.
- **Game flow**: `handleMove` -> `place` -> (`winningLine` / draw check) -> `endGame` or switch turn. `place` is the only function that mutates `board` and the DOM for a move; both human and computer moves go through it.
- **Computer opponent** always plays `O` and uses a full minimax search (`computerMove` / `minimax`), so it is unbeatable. Scores are depth-adjusted so it prefers faster wins and slower losses. The computer's move is dispatched via `setTimeout` from `handleMove` and `newRound`, so guard any new turn logic against the human clicking while the computer's move is pending (`handleMove` already ignores clicks when it is `O`'s turn in computer mode).
- **Rounds**: the starting player alternates after each finished game. `resetScores` also resets the starting player to `X`; switching modes calls `resetScores`.

## Architecture: `FableTechDay/`

Security posture dashboard with four domains (Entra ID, Azure resources, Defender, AI & Agents) mapped to MCSB, CIS Azure v3.0 and CIS M365 v4; the AI domain is additionally mapped to the OWASP Top 10 for LLM Applications and OWASP Top 10 for Agentic Applications (2025), with MITRE ATLAS technique IDs shown in the drawer only. **Sample data only**; nothing calls a network. `README.md` in the folder has the user-facing tour, workbook import steps and permissions. `docs/ARCHITECTURE.md` (with Mermaid diagrams), `docs/architecture.svg` and `docs/DATA-MODEL.md` (full finding-to-control tables, generated from the constants) must be updated when the data or structure changes.

- `index.html` is a single file with no external dependencies (must keep working from `file://`). Theme tokens live in `:root`: the page is a fixed mesh gradient (`--bg-mesh`) with frosted-glass surfaces (`.topbar`, `.filters`, `.card`, `.drawer` share translucent `--panel` + `backdrop-filter`), gradient accents (`--accent-gradient`) on the hero number, active tab and meters. Keep hover fills translucent (`--hover`, `--accent-soft`) rather than solid `--bg`; domain colors (`--entra`, `--azure`, `--defender`, `--ai`) are categorical in the fixed order blue, orange, aqua, violet and must never be reassigned when filtering (violet is used instead of the palette's yellow slot because yellow collides with `--warn`); severity/status colors are always paired with a text label.
- **All data is in constants at the top of the `<script>`**: `FINDINGS` (one object per finding with `domain`, `severity`, `status` and the mapping arrays `mcsb[]`, `cisAzure[]`, `cisM365[]`, `owaspLlm[]`, `owaspAgentic[]`, `atlas[]`; missing arrays are normalized to `[]` right after the array), `CVES`, `SECURE_SCORE`, `DFC_CIS`, `STATS` (incl. `STATS.ai`), `AI_ALERTS`, `AGENT_PLATFORMS` (totals must equal `STATS.ai.agentIdentities`), `AGENTS`, `CONTROL_NAMES`, `FRAMEWORKS`. To add a finding, append to `FINDINGS` and add any new control ID to `CONTROL_NAMES`; everything else derives.
- **Frameworks**: `FRAMEWORKS` entries with `taxonomy: true` (the OWASP lists) list their 10 category IDs. `BENCHMARKS` feed the rollup percentages; `TAXONOMIES` render as 10-cell coverage strips (`coverage()` / `renderCoverageStrip`) with a "not assessed" state and are never shown as a percentage. Keep that distinction when adding a framework.
- **Flow**: `state` (tab, filters, sort) -> `applyFilters` -> `render()`, which rebuilds only the active tab panel. `rollup(frameworkKey, findings)` computes per-framework control pass/fail (a control fails if any mapped finding fails). `postureScore` is the mean of the findings score and the two secure-score percentages. `renderFindingsTable(findings, { showDomain, domain })` picks the column set: the AI tab drops CIS Azure and adds a combined "AI frameworks" chip column; the compliance cross-reference shows every mapping column.
- Charts are hand-rolled SVG (`renderBarChart`, `renderGroupedBars`, `renderStackedBars`, `renderArc`, `renderMeterRow`); every chart has a "Show table" twin. Tabs are deep-linkable via `#overview|#entra|#azure|#defender|#ai|#compliance`, which is also how headless screenshots reach each tab.
- The `el()` helper flattens nested child arrays; pass `null`/`false` to skip a child. Insert data with `text:`/`textContent`, never `innerHTML`.
- `workbook.json` mirrors the dashboard tiles for Azure Monitor Workbooks. Each tile exists twice: `<name>-sample` (`queryType: 8`, embeds a JSON-encoded copy of the same rows) and `<name>-live` (ARG or Log Analytics query), toggled by the `DataMode` parameter. **When `FINDINGS` changes, the sample tiles must be regenerated by hand**; `queries.kql` must match the `-live` items. Graph-only data (MFA registration, CA policies, PIM, M365 Secure Score, Copilot Studio and Entra Agent ID inventory, Purview DSPM for AI reports, the OWASP mapping) has no live tile by design and shows a text note in Live mode.

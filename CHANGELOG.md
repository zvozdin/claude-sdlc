# Changelog

All notable changes to the SDLC marketplace are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/), versioning is [SemVer](https://semver.org/) per plugin.

## [Unreleased]

### Added
- Initial repository scaffold (`marketplace.json`, LICENSE, README).
- `sdlc@0.0.1` skeleton with vanilla stack profile.
- `sdlc` Phase 1 contents: `pipeline-orchestrator` skill, `/sdlc:start` command, 5 cost-tiered default agents (business-analyst, developer, qa-engineer, security-analyst, document-writer).
- `laravel-plugin@0.0.1` first stack provider: `stack.md` profile, `laravel-architect` and `artisan-specialist` agents, `laravel-conventions` and `eloquent-patterns` skills, `.mcp.json` for laravel-boost, Pint Stop-hook.

### Added (post-Phase 2 patches)
- `<project>/.claude/sdlc.local.yaml` first-class override mechanism for `post_pipeline_checks`, `phase_command_overrides`, `extra_phase_prompts`, `skip_phases`, `convention_skills_extra` (was originally scoped to Phase 3, pulled forward). Implemented as Step 1b in `pipeline-orchestrator/SKILL.md`.
- `PROJECT_INTEGRATION.md` knowledge base: how plugins interact with project-local config (CLAUDE.md, `.claude/skills/`, `.mcp.json`, `sdlc.local.yaml`). Documents auto-respected channels, current limitations, recommended scenarios (Herd vs Docker, monorepo, PHPUnit vs Pest, external SAST).
- `/sdlc:list-stacks` slash command for verifying stack profile detection (Glob installed plugins, parse frontmatter, evaluate detect rules against current project).
- MUST-print announcement protocol in orchestrator (verbatim Step 0b stack detection, Step 3b phase boundaries, Step 5 final summary). Replaces softer "Announce" instructions that were collapsing silently.

### Changed (post-Phase 2 patches)
- Renamed plugin `core-sdlc-plugin` → `sdlc`. Slash command went from `/core-sdlc-plugin:sdlc-start` to `/sdlc:start`. Cleaner UX in plugin namespace.
- `plugin.json` `dependencies` switched from object form to native array (`["sdlc"]`) per Claude Code schema; runtime plugin checks moved to `runtime-dependencies.json`.
- License switched from MIT to GPL-3.0.

### Notes
- v0.0.1 series is pre-release scaffolding. v1.0.0 will be tagged after Phase 4 (Polish) per `IMPLEMENTATION_PLAN.md`.
- External plugin dependencies (e.g. `superpowers`) are declared in `runtime-dependencies.json` but the orchestrator preflight (Step 0a) is stubbed in v0.0.1; full implementation in Phase 3.

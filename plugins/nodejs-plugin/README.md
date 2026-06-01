# nodejs-plugin

Node.js backend stack provider for the [SDLC marketplace](../../README.md).

Adds a Node.js-specific implementation agent (`node-architect`) and conventions skills to the SDLC pipeline. Activates automatically on projects with `package.json` plus a backend marker (Express, Fastify, Koa, Hapi, `@types/node`, or `engines.node`).

## What this plugin adds

| Component | Purpose |
|---|---|
| `stack.md` | Stack profile, priority=100, aspects=[backend]. |
| `agents/node-architect.md` | Sonnet-tier developer for Node.js backend. Replaces vanilla `developer` for the development phase. |
| `skills/node-conventions/` | Project layout, config, logging, routing, error-handling patterns. |
| `skills/npm-patterns/` | Package manager detection, semver, scripts, lockfile hygiene. |
| `skills/typescript-patterns/` | Strict TS for backend: no-any, discriminated unions, branded IDs, validation-at-boundary, tsconfig hygiene. Activates when project has `tsconfig.json` + `typescript`. |
| `hooks/hooks.json` | Auto-format JS/TS files via Prettier + ESLint after Edit/Write (silent if not installed). |

## What it does NOT cover

- **Frontend** (React/Vue/RN) — separate plugins, frontend aspect.
- **NestJS** — `nest-plugin` (higher priority, opinionated).
- **Next.js** — `nextjs-plugin` (full-stack, multi-aspect).
- **Database design** — handled in core's BA / dev / security agents per the BA spec.

## Installation

```
/plugin marketplace add ROLIQUE/claude-plugins
/plugin install nodejs-plugin@claude-plugins
```

The core `sdlc` plugin is installed automatically as a dependency.

## Usage

On any Node.js backend project, the plugin activates automatically:

```
/sdlc:start "Add /healthz endpoint with uptime + version"
```

The orchestrator detects the Node.js stack profile and dispatches `node-architect` for the development phase. All other phases (BA, QA, Security, Docs) use the core agents.

## Verification

```
/sdlc:list-stacks
```

Should show `nodejs priority=100` with active match when run inside a Node.js project.

## Composition with frontend plugins

Once `react-plugin` / `vue-plugin` are installed, full-stack projects activate both backend (this plugin) and frontend (the React/Vue plugin) profiles. The orchestrator runs the development phase twice — once per aspect.

Example: a Node + React monorepo gets:

- `Phase 2/N: development — backend → node-architect`
- `Phase 2/N: development — frontend → react-architect`

## Package manager detection

`post_pipeline_checks` auto-detect the package manager from the lockfile:

| Lockfile present | Runner used |
|---|---|
| `pnpm-lock.yaml` | `pnpm test`, `pnpm run lint` |
| `yarn.lock` | `yarn test`, `yarn run lint` |
| (default) | `npm test`, `npm run lint --if-present` |

The `node-architect` agent applies the same detection for any install/script invocations during the development phase.

## Local override

To customize per-project (e.g., custom monorepo test runner, Yarn Berry workspaces, skip security phase if external SAST handles it), create `.claude/sdlc.local.yaml` in the project root and set `post_pipeline_checks` explicitly. Recognized top-level keys: `post_pipeline_checks` (replaces plugin defaults), `phase_command_overrides` (passed as context to agents), `extra_phase_prompts` (appends per-phase guidance), `skip_phases`, `convention_skills_extra`.

# SDLC Marketplace for Claude Code

Multi-stack AI-assisted SDLC pipelines built on the **Stack Provider Pattern**: a single core orchestrator runs the pipeline, framework plugins register themselves via declarative `stack.md` profiles. No core overrides, no slot registries, no copy-paste between stacks.

**v0.1.0** — 10 плагінів: 1 core + 1 shared lib + 7 JS/TS стеків + Laravel. Cost-optimized: model tiering + `effort` per-subagent.

---

## Quickstart

```bash
# 1. Додати маркетплейс
/plugin marketplace add AratKruglik/claude-sdlc

# 2. Встановити потрібний стек-плагін (sdlc — автоматично як dependency)
/plugin install laravel-plugin@sdlc-marketplace
# або для JS/TS проєктів:
/plugin install nodejs-plugin@sdlc-marketplace   # Express/Fastify/Koa
/plugin install nestjs-plugin@sdlc-marketplace   # NestJS
/plugin install nextjs-plugin@sdlc-marketplace   # Next.js (full-stack)
/plugin install react-plugin@sdlc-marketplace    # React SPA
/plugin install vue-plugin@sdlc-marketplace      # Vue 3 SPA
/plugin install angular-plugin@sdlc-marketplace  # Angular 18-21
/plugin install react-native-plugin@sdlc-marketplace  # React Native / Expo

# 3. Перевірка
/sdlc:doctor
/sdlc:list-stacks

# 4. Запуск
/sdlc:start "Add subscription billing with Stripe"
```

---

## Принцип роботи: Stack Provider Pattern

```
┌─────────────────────────────────────────────────────────────┐
│                    sdlc (core)                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  pipeline-orchestrator (skill) — НЕ ЗМІНЮЄТЬСЯ        │  │
│  │                                                       │  │
│  │  Phase 1: BA          → core's business-analyst       │  │
│  │  Phase 2: Dev         → ⚡ DISPATCH до stack provider │  │
│  │  Phase X: extra       → ⚡ стек-специфічні фази       │  │
│  │  Phase N-2: QA        → core's qa-engineer            │  │
│  │  Phase N-1: Security  → core's security-analyst       │  │
│  │  Phase N: Docs/PR     → core's document-writer        │  │
│  └──────────────────────────────────────────────────────┘  │
│                            ▲                                │
│                            │ читає stack.md profiles        │
└────────────────────────────┼────────────────────────────────┘
                             │
    ┌────────────────────────┼───────────────────────────┐
    │            │           │             │             │
┌───▼───┐  ┌────▼────┐ ┌────▼────┐  ┌────▼────┐  ┌────▼────┐
│laravel│  │ nodejs  │ │  nestjs │  │ nextjs  │  │  react  │
│plugin │  │ plugin  │ │  plugin │  │ plugin  │  │  plugin │
│stack.md│ │ stack.md│ │stack.md │  │stack.md │  │stack.md │
└───────┘  └─────────┘ └─────────┘  └─────────┘  └─────────┘
```

**Ключові принципи:**

1. **Core ніколи не змінюється.** Логіка пайплайну живе лише в `pipeline-orchestrator/SKILL.md`.
2. **Плагіни реєструють себе** через `stack.md` frontmatter — оголошують правила auto-detection, priority, агентів per-phase і convention skills.
3. **Per-aspect розподіл.** Проєкт може мати кілька аспектів (backend + frontend + database). Кожен аспект отримує свого спеціаліста.
4. **Priority wins.** Коли кілька профілів матчаться, перемагає найвищий priority.

### Таблиця пріоритетів стеків

| Priority | Плагін | Аспекти | Detect |
|---|---|---|---|
| 0 | `vanilla` (sdlc) | — | `*` (завжди) |
| 100 | `nodejs-plugin` | backend | `package.json` + express/fastify/koa/... |
| 100 | `laravel-plugin` | backend, database | `composer.json` + `laravel/framework` |
| 150 | `react-plugin` | frontend | `package.json` + `react` (без `next`, `react-native`) |
| 150 | `vue-plugin` | frontend | `package.json` + `vue` |
| 200 | `nestjs-plugin` | backend, database | `package.json` + `@nestjs/core` |
| 200 | `angular-plugin` | frontend | `package.json` + `@angular/core` |
| 250 | `nextjs-plugin` | backend, frontend | `package.json` + `next` |
| 300 | `react-native-plugin` | frontend | `package.json` + `react-native` |

---

## Фази пайплайну

### Стандартний 5-фазовий пайплайн

```
Phase 1: BA → business-analyst (opus/high)
          ↓ output: docs/plans/{slug}/01-business-analysis.md
Phase 2: Dev → [стек-агент] (sonnet/medium)
          ↓ output: docs/plans/{slug}/02-development.md
Phase 3: QA → qa-engineer (sonnet/medium, max 3 attempts)
          ↓ output: docs/plans/{slug}/03-qa.md
Phase 4: Security → security-analyst (opus/high)
          ↓ output: docs/plans/{slug}/04-security.md
Phase 5: Docs → document-writer (haiku/low)
          ↓ output: PR on GitHub
```

### Приклад: Laravel (6 фаз)

```
Phase 1: BA → business-analyst
Phase 2: Dev/backend → laravel-architect    (extra: aspect=backend)
Phase 3: Dev/database → artisan-specialist  (extra phase after backend)
Phase 4: QA → qa-engineer
Phase 5: Security → security-analyst
Phase 6: Docs → document-writer
```

### Per-aspect dispatch (multi-framework проєкти)

Якщо проєкт — це, наприклад, Node.js backend + React frontend:
- Phase 2/backend → `node-architect`
- Phase 2/frontend → `react-architect` (окремий прогін)

Аспекти в canonical order: `database → backend → frontend → testing`.

---

## Команди

| Команда | Призначення |
|---|---|
| `/sdlc:start "feature"` | Запуск повного 5-фазового пайплайну |
| `/sdlc:batch "task1" "task2"` | Паралельний запуск для кількох задач (ізольовані worktree) |
| `/sdlc:list-stacks` | Перегляд виявлених стек-профілів і priority |
| `/sdlc:doctor` | Preflight check: dep check, stack detection, cost baseline |
| `/sdlc:security-init` | Матеріалізувати security-patterns.yaml для security-guidance plugin |

---

## Cost-оптимізація: model + effort

### Чому `model` + `effort`, а не `temperature`

Claude Code підтримує у frontmatter субагента:
- `model` — `opus` / `sonnet` / `haiku` / повний ID / `inherit`
- `effort` — `low` / `medium` / `high` / `xhigh` / `max` — **перекриває session-рівень reasoning-бюджету**

`temperature` **не налаштовується per-subagent** у Claude Code. Керуємо виключно `model` + `effort`.

### Таблиця model+effort по всіх агентах

| Агент | Плагін | model | effort | Обґрунтування |
|---|---|---|---|---|
| `business-analyst` | sdlc | `opus` | `high` | Помилка вимог каскадує через 5 фаз; малий об'єм, максимальний важіль |
| `security-analyst` | sdlc | `opus` | `high` | Неочевидні вразливості (TOCTOU, JWT confusion, SSRF) потребують deep reasoning |
| `developer` | sdlc | `sonnet` | `medium` | Vanilla fallback — виконання за чітким спеком |
| `qa-engineer` | sdlc | `sonnet` | `medium` | Тести за чіткими критеріями; hard 3-attempt cap тримає cost |
| `document-writer` | sdlc | `haiku` | `low` | Структурований вивід із відомих фактів; ~10× економія vs Opus |
| `laravel-architect` | laravel | `sonnet` | `medium` | Laravel idioms + Inertia/Vue |
| `artisan-specialist` | laravel | `sonnet` | `low` | Механічна DB-робота: типи/індекси/factories |
| `node-architect` | nodejs | `sonnet` | `medium` | Express/Fastify — implementation за Node.js ідіомами |
| `nest-architect` | nestjs | `sonnet` | `medium` | Convention skills несуть per-domain глибину |
| `nextjs-architect` | nextjs | `sonnet` | `medium` | RSC/Client patterns добре визначені spec і skills |
| `react-architect` | react | `sonnet` | `medium` | React conventions та state/routing skills |
| `vue-architect` | vue | `sonnet` | `medium` | Vue 3/2 detection + convention skills |
| `angular-architect` | angular | `sonnet` | `medium` | Angular standalone/NgModule, signals, NgRx |
| `rn-architect` | react-native | `sonnet` | `medium` | Expo/bare + iOS/Android axes |

> `effort: high` на Opus — найдорожчий кут. Тому лише 2 агенти-важелі (BA і Security), де reasoning безпосередньо впливає на якість всіх наступних фаз.

### Орієнтовний кошторис medium-фічі

| Фаза | Агент | Cost |
|---|---|---|
| BA | opus/high | ~$0.25 |
| Dev | sonnet/medium | ~$0.60 |
| QA | sonnet/medium (≤3 спроби) | ~$0.30 |
| Security | opus/high | ~$0.25 |
| Docs | haiku/low | ~$0.07 |
| **Total** | | **~$1.47** |

### Додаткові cost-важелі

- **Skip-rules:** typo-fix, whitespace-only, config-only, lightweight-no-db — пропускають зайві фази.
- **QA hard cap:** max 3 спроби виправити тести, потім STOP.
- **Compact handoffs:** кожен агент повертає ≤2–3K-token summary.
- **Prompt caching:** стабільні system prompts (no timestamps, slugs, dynamic content) → ~60% cache hit rate на Sonnet.

---

## Доступні плагіни

| Плагін | Тип | Стек/технологія |
|---|---|---|
| `sdlc` | Core | Пайплайн-оркестратор, 5 агентів |
| `js-foundation` | Shared lib | TypeScript + npm patterns (без стек-профіля) |
| `nodejs-plugin` | Stack provider | Express / Fastify / Koa / plain Node.js |
| `nestjs-plugin` | Stack provider | NestJS + TypeORM/Prisma/Mongoose |
| `nextjs-plugin` | Stack provider | Next.js App Router (full-stack) |
| `react-plugin` | Stack provider | React SPA (Vite/Webpack) |
| `vue-plugin` | Stack provider | Vue 3 SPA |
| `angular-plugin` | Stack provider | Angular 18-21 |
| `react-native-plugin` | Stack provider | React Native / Expo |
| `laravel-plugin` | Stack provider | Laravel + Inertia + Vue |

### Зовнішні залежності (опціональні)

| Плагін | Source | Роль |
|---|---|---|
| `superpowers` | `obra/superpowers` | Додає brainstorming до BA, TDD до QA, verification-before-completion до архітекторів. Pipeline деградує gracefully без нього. |
| `security-guidance` | `anthropics/claude-plugins-official` | Hooks-based security review: per-edit pattern match, end-of-turn diff review. OWASP фаза й без нього повна. |

---

## Композиція стеків (multi-framework приклади)

| Проєкт | Профіль | Development dispatch |
|---|---|---|
| Laravel + Vue SPA (Inertia) | laravel (100) | laravel-architect (backend) + artisan-specialist (db) |
| Express + React | nodejs (100) + react (150) | node-architect (backend) + react-architect (frontend) |
| NestJS + Angular | nestjs (200) + angular (200) | nest-architect (backend) + angular-architect (frontend) |
| Next.js (full-stack) | nextjs (250) | nextjs-architect (owns backend + frontend) |
| Expo mobile | react-native (300) | rn-architect (frontend) |
| Vanilla Node.js | nodejs (100) | node-architect |
| Невідомий стек | vanilla (0) | developer (fallback) |

---

## Локальні оверайди

Файл `.claude/sdlc.local.yaml` у корені проєкту (не в плагіні) дозволяє адаптувати pipeline без зміни плагінів:

```yaml
post_pipeline_checks:
  - "composer test"
  - "php artisan route:list --json"

phase_command_overrides:
  qa: "php artisan test --coverage --min=80"

convention_skills_extra:
  - "local:custom-coding-standards"

skip_phases:
  - security  # для внутрішніх hotfix-branches

extra_phase_prompts:
  development: "Дотримуватись нашого internal-styleguide.md"
```

---

## Як додати новий стек-плагін

Контракт для нового framework provider:

```
plugins/your-framework-plugin/
├── .claude-plugin/
│   └── plugin.json          # { "name": "...", "dependencies": ["sdlc"] }
├── stack.md                 # YAML frontmatter: stack, priority, aspects, detect
├── agents/
│   └── your-architect.md   # frontmatter: name, model, effort, color, tools
├── skills/
│   └── your-conventions/
│       └── SKILL.md
└── README.md
```

### `stack.md` приклад

```yaml
---
stack: django
priority: 150
aspects: [backend, database]
detect:
  all:
    - file_exists: manage.py
    - file_contains:
        path: requirements.txt
        pattern: "django"
---

## Agents per phase
- business_analysis: business-analyst
- development:
    backend: django-architect
- qa: qa-engineer
- security: security-analyst
- documentation: document-writer

## Convention skills
- django-plugin:django-conventions
- django-plugin:orm-patterns
```

### Схеми для валідації

```bash
# Перевірка plugin.json
npx check-jsonschema --schemafile schemas/plugin.schema.json .claude-plugin/plugin.json

# Перевірка stack.md frontmatter
npx check-jsonschema --schemafile schemas/stack.schema.json <(yq '.frontmatter' stack.md)
```

---

## Встановлення (детально)

### 1. Додати маркетплейс

```bash
/plugin marketplace add AratKruglik/claude-sdlc
# або для локального dev:
/plugin marketplace add /path/to/claude-sdlc
```

### 2. Встановити core + потрібні плагіни

```bash
# Core встановлюється автоматично як dependency
/plugin install nodejs-plugin@sdlc-marketplace     # Node.js backend
/plugin install js-foundation@sdlc-marketplace     # потрібно для JS/TS плагінів
```

### 3. Опціональні зовнішні залежності

```bash
/plugin marketplace add obra/superpowers
/plugin install superpowers@superpowers-marketplace

/plugin marketplace add anthropics/claude-plugins-official
/plugin install security-guidance@claude-plugins-official
```

### 4. Перевірка

```bash
/sdlc:doctor
# → Stack profiles detected: vanilla(0), nodejs(100), react(150), ...
# → superpowers: ✅ installed
# → security-guidance: ⚠️ not found (pipeline will run in degraded mode)

/sdlc:list-stacks
# → Shows all matched stack profiles for current project
```

### 5. Запуск

```bash
/sdlc:start "Add user authentication with JWT"
# → Auto-detects stack, runs 5 phases, creates PR
```

---

## Вимоги

- Claude Code (latest)
- API Tier 2+ або Claude Max — medium-фіча займає ~445K input tokens; Pro plan rate limits throttle pipeline.
- Git репозиторій для `document-writer` (PR creation).

## Ліцензія

GPL-3.0 — see [`LICENSE`](./LICENSE).

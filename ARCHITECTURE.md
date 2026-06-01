# Claude SDLC Marketplace — Архітектура

> Marketplace плагінів Claude Code для повного AI-assisted SDLC, який працює на будь-якому технологічному стеку (Laravel, Django, NestJS, .NET, …).
>
> **Принцип:** core володіє пайплайном і не змінюється. Фреймворк-плагіни **реєструють себе** як stack providers через декларативний профіль і надають спеціалізовані агенти. Core читає профілі і композує виконання.

---

## 1. Ключова ідея: Stack Provider Pattern

```
┌─────────────────────────────────────────────────────────────┐
│                    sdlc                         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  pipeline-orchestrator (skill) — НЕ ЗМІНЮЄТЬСЯ        │  │
│  │                                                       │  │
│  │  Phase 1: BA          → core's business-analyst       │  │
│  │  Phase 2: Dev         → ⚡ DISPATCH до stack provider │  │
│  │  Phase 3: QA          → core's qa-engineer            │  │
│  │  Phase 4: Security    → core's security-analyst       │  │
│  │  Phase 5: Docs/PR     → core's document-writer        │  │
│  └──────────────────────────────────────────────────────┘  │
│                            ▲                                │
│                            │ читає stack profiles            │
└────────────────────────────┼────────────────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
       ┌──────▼─────┐ ┌──────▼─────┐ ┌─────▼──────┐
       │  laravel-  │ │  django-   │ │  nestjs-   │
       │   plugin   │ │   plugin   │ │   plugin   │
       │            │ │            │ │            │
       │ stack.md   │ │ stack.md   │ │ stack.md   │
       │ + agents   │ │ + agents   │ │ + agents   │
       │ + skills   │ │ + skills   │ │ + skills   │
       │ + .mcp     │ │ + .mcp     │ │ + .mcp     │
       └────────────┘ └────────────┘ └────────────┘
```

**Контракт між core і фреймворк-плагіном:**

1. Фреймворк-плагін розміщує файл `stack.md` за відомим шляхом (корінь плагіна).
2. Фреймворк-плагін надає агентів зі стек-специфічною спеціалізацією (наприклад, `laravel-architect`, `artisan-specialist`).
3. Core orchestrator читає `stack.md`, обирає найвищий-priority profile, чий `detect` спрацював, і диспатчить до правильних агентів.

**Що саме НЕ робимо** (свідомі рішення, не оптимізація):

- Жодного «Slot Registry», публічного контракту слотів, чотиришарової моделі (core/stack/capability/domain). Все живе в `stack.md` і конвенціях іменування агентів.
- Жодного override механізму у фреймворк-плагінах щодо core. Фреймворк **додає себе**, не редагує core.
- Жодних capability- чи domain-плагінів у v1.0. Якщо щось наскрізне (postgres, github-actions) потрібне — воно поки живе в межах фреймворк-плагіна, який його використовує.

---

## 2. Файлова структура

```
sdlc-marketplace/
├── .claude-plugin/
│   └── marketplace.json                 ← v0.1.0: 12 записів (2 зовнішніх + 10 локальних)
├── schemas/
│   ├── plugin.schema.json               ← JSON-схема для plugin.json
│   └── stack.schema.json                ← JSON-схема для stack.md frontmatter
│
├── plugins/
│   ├── sdlc/
│   │   ├── .claude-plugin/plugin.json
│   │   ├── commands/
│   │   │   ├── start.md                 ← /sdlc:start "feature description"
│   │   │   ├── batch.md                 ← /sdlc:batch (паралельний запуск)
│   │   │   ├── list-stacks.md           ← /sdlc:list-stacks
│   │   │   ├── doctor.md                ← /sdlc:doctor (preflight + dep check)
│   │   │   └── security-init.md         ← /sdlc:security-init (security patterns)
│   │   ├── skills/
│   │   │   └── pipeline-orchestrator/
│   │   │       └── SKILL.md             ← ЄДИНИЙ orchestrator (955 рядків)
│   │   ├── agents/
│   │   │   ├── business-analyst.md      ← opus + effort:high (critical reasoning)
│   │   │   ├── developer.md             ← sonnet + effort:medium (vanilla fallback)
│   │   │   ├── qa-engineer.md           ← sonnet + effort:medium (з iteration cap)
│   │   │   ├── security-analyst.md      ← opus + effort:high (critical reasoning)
│   │   │   └── document-writer.md       ← haiku + effort:low (structured output)
│   │   └── stack.md                     ← vanilla profile (priority: 0)
│   │
│   ├── js-foundation/                   ← shared TS/npm skills, no stack profile
│   │   └── skills/                      ← typescript-patterns, npm-patterns
│   ├── nodejs-plugin/                   ← Node.js/Express/Fastify (priority:100)
│   │   └── agents/node-architect.md     ← sonnet + effort:medium
│   ├── nestjs-plugin/                   ← NestJS backend (priority:200)
│   │   └── agents/nest-architect.md     ← sonnet + effort:medium
│   ├── nextjs-plugin/                   ← Next.js full-stack (priority:250)
│   │   └── agents/nextjs-architect.md   ← sonnet + effort:medium
│   ├── react-plugin/                    ← React SPA frontend (priority:150)
│   │   └── agents/react-architect.md    ← sonnet + effort:medium
│   ├── vue-plugin/                      ← Vue 3 SPA frontend (priority:150)
│   │   └── agents/vue-architect.md      ← sonnet + effort:medium
│   ├── angular-plugin/                  ← Angular 18-21 frontend (priority:200)
│   │   └── agents/angular-architect.md  ← sonnet + effort:medium
│   ├── react-native-plugin/             ← React Native mobile (priority:300)
│   │   └── agents/rn-architect.md       ← sonnet + effort:medium
│   │
│   └── laravel-plugin/
│       ├── .claude-plugin/plugin.json   ← dependencies: sdlc
│       ├── stack.md                     ← Laravel stack profile (priority: 100)
│       ├── agents/
│       │   ├── laravel-architect.md     ← Sonnet, замінює developer для Laravel
│       │   └── artisan-specialist.md    ← Sonnet, для extra phase "database"
│       ├── skills/
│       │   ├── laravel-conventions/SKILL.md
│       │   └── eloquent-patterns/SKILL.md
│       ├── .mcp.json                    ← laravel-boost тощо
│       └── hooks/hooks.json             ← pint автоформат на Stop
```

> **Ключова деталь:** у `laravel-plugin` немає `pipeline-orchestrator/`. Файли core залишаються недоторканими. Лараведь-плагін лише додає `stack.md` + спеціалізованих агентів + конвенційні скіли.

---

## 3. Stack Profile — контракт між core і фреймворком

`stack.md` — звичайний markdown з YAML frontmatter. Core orchestrator знає, як його читати.

### 3.1. Vanilla profile (`sdlc/stack.md`)

```markdown
---
stack: vanilla
priority: 0
detect:
  any: ["*"]
---

# Vanilla Stack Profile

## Agents per phase
- business_analysis: business-analyst
- development: developer
- qa: qa-engineer
- security: security-analyst
- documentation: document-writer

## Convention skills to apply
- (none)

## Pre-phase commands
- (none)

## Post-phase commands
- (none)
```

`priority: 0` + `detect.any: ["*"]` означає: завжди матчиться, але втрачає будь-якому профілю з вищим priority.

### 3.2. Laravel profile (`laravel-plugin/stack.md`)

```markdown
---
stack: laravel
priority: 100
detect:
  all:
    - file_exists: composer.json
    - file_contains:
        path: composer.json
        pattern: '"laravel/framework"'
---

# Laravel Stack Profile

## Agents per phase
- business_analysis: business-analyst        # core agent
- development: laravel-architect              # ⚡ Laravel-specific
- database: artisan-specialist                # ⚡ extra phase
- qa: qa-engineer                             # core agent
- security: security-analyst                  # core agent
- documentation: document-writer              # core agent

## Convention skills to apply
- laravel:laravel-conventions
- laravel:eloquent-patterns

## Extra phases
- name: database
  after: development
  agent: artisan-specialist
  description: "Run migrations, factories, seeders"

## Phase prompts injection

For development phase, inject:
  "Use Artisan commands for code generation: php artisan make:model -mfsc, etc.
   Follow PSR-12 and Laravel conventions.
   Apply skills: laravel:laravel-conventions, laravel:eloquent-patterns"

For qa phase, inject:
  "Use Pest/PHPUnit with Laravel testing helpers (RefreshDatabase, actingAs).
   Run: php artisan test --coverage"

For security phase, inject:
  "Check Laravel-specific issues: mass assignment, Gates/Policies coverage,
   raw query usage, .env exposure, debug mode in production."

## Post-pipeline checks
- ./vendor/bin/pint --test
- php artisan test
- php artisan route:list
```

### 3.3. Frontmatter spec

| Поле | Тип | Обовʼязкове | Опис |
|---|---|---|---|
| `stack` | string | ✅ | Унікальне імʼя стеку (`laravel`, `django`, `nestjs`, `vanilla`). |
| `priority` | int | ✅ | 0 — завжди-матч fallback; 100+ — конкретний фреймворк. Вище = переможець. |
| `detect.any` / `detect.all` | array | ✅ | Правила автодетекту. `["*"]` для vanilla. |
| `detect.*.file_exists` | string | — | Файл, що має існувати в корені проєкту. |
| `detect.*.file_contains` | object | — | `{path, pattern}` — regex-перевірка вмісту файлу. |

---

## 4. Pipeline Orchestrator (єдиний skill core)

`sdlc/skills/pipeline-orchestrator/SKILL.md` — серце системи.

### 4.1. Алгоритм (8 кроків)

```
Step 0a · Load declared external plugin dependencies (DEPENDENCIES.md)
Step 0b · Detect stack profile via Glob ~/.claude/plugins/cache/**/stack.md
Step 1  · Parse selected profile
Step 2  · Determine phase order (baseline + extras)
Step 3  · Execute each phase:
            - Look up agent in profile
            - Build prompt: base + injection + previous-phase summary
            - Spawn agent with token-budget aware prompt
            - Save COMPACT summary to CONTEXT.{phase}_output
Step 4  · Run post_pipeline_checks via Bash
Step 5  · Final summary (stack used, phases executed, costs, PR link)
```

### 4.2. Базові prompt-и фаз

```
business_analysis:
  > Analyze: $ARGUMENTS
  > Produce: user stories (Gherkin), acceptance criteria, data model, API contract, edge cases.
  > {INJECTED_PROMPT}
  > Return COMPACT summary (≤2K tokens): scope bullets + 5 user stories + open questions.

development:
  > Implement based on: $CONTEXT.business_analysis_output
  > Follow project conventions in CLAUDE.md.
  > Apply convention skills: {convention_skills}
  > {INJECTED_PROMPT}
  > Return COMPACT summary: files changed (list) + key decisions (3–5 bullets) + blockers.

qa:
  > Write and run tests for changes referenced in $CONTEXT.development_output.
  > Aim ≥80% coverage. Max 3 attempts to fix failing tests — then STOP and report.
  > {INJECTED_PROMPT}
  > Return COMPACT summary: tests added/passed/failed + coverage % + open issues.

security:
  > Review changes referenced in $CONTEXT.development_output for OWASP Top 10.
  > Fix Critical and High severity issues. Skip Low/Info unless trivially safe.
  > {INJECTED_PROMPT}
  > Return COMPACT summary: issues found (severity, file:line) + fixes applied.

documentation:
  > Create a Pull Request.
  > Inputs: $CONTEXT.business_analysis_output, $CONTEXT.qa_output, $CONTEXT.security_output.
  > {INJECTED_PROMPT}
  > Return: PR URL + 1-paragraph release-notes blurb.
```

«COMPACT summary» — критична оптимізація вартості (див. §6). Наступна фаза читає файли проєкту напряму, не отримує дамп виводу попередньої фази.

### 4.3. Skip-rules для тривіальних змін

Перед запуском пайплайну orchestrator аналізує scope (з $ARGUMENTS):

| Сигнал | Дія |
|---|---|
| Diff < 50 LOC, без нових файлів, без DB-migrations | Skip Security — використати lightweight check у dev. |
| Опис «typo», «rename var», «format» | Skip BA — використати $ARGUMENTS напряму як спек. |
| Опис «config tweak», «env var» | Skip QA — лишити post_pipeline_checks. |

Skip-rules економлять 30–60% вартості на дрібних задачах. Без них pipeline на typo-фікс коштує стільки ж, скільки на повну фічу.

---

## 5. Default Core Agents (5 агентів з вшитим tiering)

Усі 5 агентів живуть у `sdlc/agents/`. Модель і `effort` обрані за принципом «cost of mistakes» — Opus+high там, де помилки компаундяться через увесь пайплайн.

### 5.0. Повна таблиця model+effort (14 агентів включно зі стек-провайдерами)

| Агент | Плагін | model | effort | Обґрунтування |
|---|---|---|---|---|
| `business-analyst` | sdlc | `opus` | `high` | Помилка вимог каскадує крізь 5 фаз; малий об'єм токенів, максимальний важіль |
| `security-analyst` | sdlc | `opus` | `high` | Неочевидні вразливості (TOCTOU, JWT confusion) потребують deep reasoning |
| `developer` | sdlc | `sonnet` | `medium` | Vanilla fallback — виконання за чітким спеком |
| `qa-engineer` | sdlc | `sonnet` | `medium` | Тести за чіткими критеріями; hard 3-attempt cap тримає cost |
| `document-writer` | sdlc | `haiku` | `low` | Структурований вивід із відомих фактів; Haiku дає ~10× економію vs Opus |
| `laravel-architect` | laravel | `sonnet` | `medium` | Workhorse: Laravel idioms + Inertia frontend |
| `artisan-specialist` | laravel | `sonnet` | `low` | Механічна DB-робота: типи/індекси/factories |
| `node-architect` | nodejs | `sonnet` | `medium` | Express/Fastify — implementation за чіткими Node.js ідіомами |
| `nest-architect` | nestjs | `sonnet` | `medium` | Convention skills (nest-data-layer, nest-advanced) несуть per-domain глибину |
| `nextjs-architect` | nextjs | `sonnet` | `medium` | RSC/Client patterns добре визначені spec і convention skills |
| `react-architect` | react | `sonnet` | `medium` | React conventions та state/routing skills покривають варіативність |
| `vue-architect` | vue | `sonnet` | `medium` | Vue 3/2 detection та convention skills покривають вибір бібліотек |
| `angular-architect` | angular | `sonnet` | `medium` | Angular idioms (standalone, signals, NgRx) в convention skills |
| `rn-architect` | react-native | `sonnet` | `medium` | Expo/bare, iOS/Android axes — convention skills (rn-platform-specific) |

> **Про `temperature` і `effort`:**  
> Claude Code не підтримує `temperature` per-subagent у frontmatter. Контроль reasoning-бюджету — виключно через поле `effort` (`low`/`medium`/`high`/`xhigh`/`max`), яке перекриває session-рівень. `effort: high` на Opus — найдорожчий кут; тому лише 2 агенти-важелі.

| Агент | model | effort | Tools (least-privilege) |
|---|---|---|---|
| `business-analyst` | **opus** | `high` | Read, Glob, Grep, WebSearch, WebFetch |
| `developer` (vanilla fallback) | **sonnet** | `medium` | Read, Glob, Grep, Edit, Write, Bash |
| `qa-engineer` | **sonnet** | `medium` | Read, Glob, Grep, Edit, Write, Bash |
| `security-analyst` | **opus** | `high` | Read, Glob, Grep, WebSearch |
| `document-writer` | **haiku** | `low` | Read, Glob, Grep, Bash, mcp__github__* |

### 5.1. Iteration cap у QA (запобіжник runaway costs)

`qa-engineer.md`:

```markdown
## Hard limit: iteration cap

You have a maximum of **3 attempts** to fix failing tests.
After attempt #3:
  STOP. Do not iterate further.
  Return COMPACT summary including:
    - Tests still failing (file:test name)
    - Last error messages (1-2 lines each)
    - Hypothesis why they fail (1 paragraph)
  Mark phase as 'incomplete-blocked'.

This is non-negotiable. Runaway iterations have caused $50+ per pipeline.
```

Без цього cap-у єдиний крихкий test може спалити токенів на $5 за один запуск.

### 5.2. Tool restrictions = безпека + вартість

`business-analyst` має **тільки read-only** tools — не може випадково запустити migration або зачепити код. Це і безпека, і економія: менше тулів = менше випадкових heavy calls.

---

## 6. Cost-Conscious Design (вшито в архітектуру)

### 6.1. Бюджет на запуск

Цільовий бюджет для medium-фічі (eg. «Stripe billing module»):

| Сценарій | Cost/run |
|---|---|
| All-Opus (Rolique mandate — скасовано) | $4.05 |
| Model tiering (opus/sonnet/haiku) | $2.66 |
| + `effort: high` лише для BA/Security | ~$2.80 (трохи дорожче, але виправданий reasoning) |
| + prompt caching (60% hit) | $1.90 |
| + compact handoffs | **~$1.50** ← наша мета |

### 6.2. Чотири hotspots, які наш дизайн адресує by design

| Hotspot | Архітектурна відповідь |
|---|---|
| **Дублювання context між subagent-ами** (~30–50K дубль на pipeline) | Subagent-и читають проєктні файли самі через FS, не отримують їх у prompt'і. Compact summary між фазами. |
| **Tool call results роздуваються в input наступного агента** (test logs 5–20K, file reads 3–8K) | Кожен агент повертає COMPACT summary (≤2-3K). Деталі лишаються у власному context window агента і не пересилаються далі. |
| **Iteration loops в QA без обмеження** (3–5 циклів × 10K) | Iteration cap = 3 спроби в `qa-engineer.md`. Понад — STOP + report. |
| **Накопичення context від фази до фази** (60–100K додаткових input) | Orchestrator передає лише останню summary, не всю історію. Старі summary доступні через файли (`docs/plans/{slug}/0X-*.md`), а не через prompt. |

### 6.3. Prompt caching як принцип дизайну

Claude Code робить prompt caching автоматично, але **тільки для стабільних system prompts**. Це означає:

- Frontmatter агентів — стабільний (не генерується динамічно).
- Inject у prompt-фази йде наприкінці prompt-у (caching префіксу).
- Skill-вміст НЕ генеруємо динамічно — статичні markdown.

Очікуваний cache hit rate при стабільних prompts: 60% на Sonnet, 40% на Opus → ~30% знижки на input.

### 6.4. Skip-rules і phase parallelism

- **Skip-rules** (§4.3) знижують вартість тривіальних задач у 2–3 рази.
- **Параллелізм Security ∥ QA** (V2) — не залежать одна від одної. Дає половину wall-clock time. Реалізація — Phase 6+.

### 6.5. Telemetry для cost discipline

Орестратор пише в `docs/plans/{slug}/_telemetry.json` після кожного pipeline:

```json
{
  "stack": "laravel",
  "phases": [
    { "phase": "ba", "model": "opus", "input_tokens": 35000, "output_tokens": 3000, "cost_usd": 0.25 },
    { "phase": "dev", "model": "sonnet", "input_tokens": 42000, "output_tokens": 8500, "cost_usd": 0.25 }
  ],
  "total_cost_usd": 1.42,
  "total_wall_clock_s": 187,
  "skip_rules_applied": ["security:diff<50loc"]
}
```

Це дає тренд cost-per-feature і відповідає на «де горять токени». Якщо cost виходить за бюджет — telemetry показує конкретну фазу.

---

## 7. Зовнішні залежності плагінів (superpowers)

Плагіни нашого marketplace можуть депендити на зовнішні Claude Code плагіни (типу `obra/superpowers`), щоб перевикористовувати їхні скіли — без копіювання чужого коду в наш репо.

### 7.1. Реалістична межа

Claude Code не має нативного dependency resolution. Силент auto-install **технічно неможливий**: slash-команди (`/plugin install`) запускаються тільки користувачем. Тому стратегія — **guided install**:

1. Декларуємо залежність у `plugin.json`.
2. На старті pipeline orchestrator робить **одну** preflight-перевірку.
3. Якщо відсутня — обираємо за політикою (`block` / `warn` / `graceful-degrade`) і за наявності `mcp__plugins__suggest_plugin_install` пропонуємо однокліковий install.

Це **єдиний** check-point у новій архітектурі. Жодних 4 layer-ів, project-level config-ів, runtime-перевірок усередині кожного агента.

### 7.2. Маніфест залежностей у `plugin.json`

```jsonc
{
  "name": "sdlc",
  "version": "1.0.0",
  "dependencies": {
    "plugins": [
      {
        "name": "superpowers",
        "marketplace": "obra/superpowers",
        "version": ">=1.0.0",
        "policy": "warn",
        "skills_used": [
          "thinking-deeply",
          "test-driven-development",
          "verification-before-completion"
        ],
        "fallback_note": "Pipeline runs without these but with reduced rigor in BA and QA phases."
      }
    ]
  }
}
```

### 7.3. Три політики на missing-dep

| Policy | Поведінка | Коли використовувати |
|---|---|---|
| `block` | Pipeline halts. Suggest install. Юзер після install запускає `/sdlc:start` повторно. | Без скіла результат буде неправильний (рідкісний випадок). |
| `warn` | Pipeline продовжується з повідомленням «running in degraded mode for X». | **Дефолт.** Більшість зовнішніх скілів — поліпшення, не блокери. |
| `graceful-degrade` | Тиха підстановка fallback. Лише запис у telemetry. | Скіл — приємний бонус, користувач не помітить різниці. |

### 7.4. Preflight check — один-єдиний

В `pipeline-orchestrator/SKILL.md` Step 0a:

```
Step 0a — Verify external plugin dependencies

For each entry in plugin.json's dependencies.plugins:
  1. Call mcp__skills__list_skills.
  2. Check if every skill in skills_used is present (as `{name}:{skill}`).
  3. If all present: continue silently.
  4. If missing AND policy=block:
       Print install command and abort with exit=1.
       Suggest mcp__plugins__suggest_plugin_install if available.
  5. If missing AND policy=warn:
       Print warning. Set context flag {dep}_unavailable=true.
       Continue. Each agent that uses these skills falls back gracefully.
  6. If missing AND policy=graceful-degrade:
       Silently set context flag. Continue.

In headless mode (env SDLC_NONINTERACTIVE=true):
  - block → exit 1 with machine-readable JSON
  - warn → stderr message, continue
  - graceful-degrade → silent
```

Деталі імплементації + JSON Schema — в `DEPENDENCIES.md`. Коротка версія — у v1.0.

### 7.5. Що НЕ робимо щодо залежностей

- **Не** перевіряємо в кожному агенті — тільки в orchestrator-і на старті.
- **Не** реалізуємо project-level `.claude-sdlc.json` overrides у v1.0 — manifest достатньо.
- **Не** памʼятаємо decline між сесіями — користувач міг встановити плагін у проміжку.
- **Не** робимо poll-loop після `suggest_plugin_install` — асинхронний install неперевіряємий, чесніше abort + повторити команду.

---

## 8. Запуск на практиці

```bash
# Інсталяція (один раз)
/plugin marketplace add your-org/sdlc-marketplace
/plugin install laravel-plugin@sdlc-marketplace
# sdlc підтягнеться як залежність

# Перевірка стану
/sdlc:doctor
# → ✅ sdlc@1.0.0
# → ✅ laravel-plugin@1.0.0
# → ⚠️  superpowers: missing (policy=warn) — pipeline runs in degraded mode

# Перегляд знайдених профілів
/sdlc:list-stacks
# 🎯 vanilla   priority=0   (always matches)
# 🎯 laravel   priority=100 (matches: composer.json + laravel/framework)

# Запуск пайплайну (одна команда для всіх стеків)
/sdlc:start "Add subscription billing with Stripe"
# → Detected stack: laravel (from laravel-plugin/stack.md)
# → Phase 1/6: business_analysis (Opus)
# → Phase 2/6: development → laravel-architect (Sonnet)
# → Phase 3/6: database → artisan-specialist (Sonnet) [extra phase]
# → Phase 4/6: qa (Sonnet)
# → Phase 5/6: security (Opus)
# → Phase 6/6: documentation (Haiku)
# → Post-pipeline: pint --test, php artisan test, route:list
# → ✅ Completed in 187s, $1.42 spent, PR #142

# Явний override стеку
/sdlc:start --stack=vanilla "Add a /healthz endpoint"
# → Ігнорує Laravel, використовує дефолтні core агенти
```

---

## 9. Як додати новий фреймворк

Без жодних змін у core. Приклад для Django:

```
django-plugin/
├── .claude-plugin/plugin.json     ← dependencies: sdlc
├── stack.md                        ← detect: manage.py + django у requirements.txt
├── agents/
│   ├── django-architect.md        ← Sonnet
│   └── drf-specialist.md          ← Sonnet (extra phase: api-layer)
├── skills/
│   ├── django-conventions/SKILL.md
│   └── orm-patterns/SKILL.md
├── .mcp.json                       ← postgres MCP, django docs
└── hooks/hooks.json                ← black/ruff на Stop
```

`django-plugin/stack.md`:

```markdown
---
stack: django
priority: 100
detect:
  all:
    - file_exists: manage.py
    - file_contains: { path: requirements.txt, pattern: "django" }
---

## Agents per phase
- development: django-architect
- (інші використовують core)

## Convention skills
- django:django-conventions
- django:orm-patterns

## Phase prompts injection
For development phase, inject:
  "Use Django management commands: manage.py startapp, makemigrations.
   Follow PEP 8, Django coding style. Apply skills: django-conventions, orm-patterns."
```

Все. При наступному запуску `/sdlc:start` core orchestrator знаходить новий профіль через Glob, оцінює detect-правила і використовує `django-architect` замість vanilla `developer`.

---

## 10. Що ми отримуємо

| Властивість | Як працює |
|---|---|
| **Core не змінюється** | `pipeline-orchestrator` живе тільки в core. Жоден framework-плагін його не торкається. |
| **DRY** | Логіка пайплайну написана один раз. Bug fix у core автоматично доступний усім фреймворкам. |
| **Розширюваність** | Новий фреймворк = новий плагін з власним `stack.md` + спеціалізованими агентами. Без переписування orchestrator-а. |
| **Auto-detection** | Core читає файли проєкту і сам визначає стек. Override через `--stack=name`. |
| **Композиція без override** | Laravel використовує core's BA/QA/Security/Docs; підставляє свого тільки для development. |
| **Extra phases** | Laravel додає фазу `database`. Vanilla pipeline просто її не виконує. |
| **Конвенції через скіли** | Laravel skills (`laravel-conventions`, `eloquent-patterns`) застосовуються через профіль автоматично. |
| **Cost-conscious by design** | Smart model tiering, compact handoffs, iteration cap, skip-rules вшиті в дефолти. |

---

## 10.5. Profile composition for multi-aspect projects (Phase 4-5 evolution)

> **Поточне обмеження:** orchestrator на Step 0b обирає **один** профіль (найвищий priority серед матчучих). Це працює для single-stack проєктів, але **ламається на типовому Laravel**, де є і backend (`composer.json`), і frontend (`package.json` з Vue/React/Livewire). Зараз `laravel-plugin` ховає це шляхом монолітного `laravel-architect` («Full-stack Laravel + Inertia + Vue»). Тиха помилка для інших frontend-варіантів.

**Заплановане рішення (Phase 4-5):** **aspect-tagged profiles + phase fan-out.** Кожен профіль декларує `aspects:` (`backend`, `frontend`, `database`, `infra`, `testing`). Orchestrator обирає **переможця на КОЖЕН aspect окремо**, не на проєкт цілком. Aspect-aware фази (`development`, `qa`) виконують по агенту на кожен релевантний aspect послідовно.

Приклади (після Phase 5):

| Тип проєкту | Авто-активуються плагіни (через aspect resolution) |
|---|---|
| Laravel + Inertia + Vue | `laravel-plugin` (backend, database) + `inertia-vue-plugin` (frontend) |
| Laravel + Inertia + React | `laravel-plugin` + `inertia-react-plugin` |
| Laravel + Livewire | `laravel-plugin` + `laravel-livewire-plugin` |
| Laravel API-only | лише `laravel-plugin` (frontend slot пустий) |
| Pure Next.js (без PHP) | лише `nextjs-plugin` |

**`laravel-plugin` буде розщеплено** в Phase 5: backend+database залишаться, frontend (Inertia+Vue) переїде в окремий `inertia-vue-plugin`. Поточний laravel-architect стає backend-only; Inertia/Vue знання — у новому `inertia-vue-architect`.

**Робота сьогодні (v0.0.1) для не-Vue Laravel-проєктів:** workaround через `<project>/.claude/sdlc.local.yaml` `extra_phase_prompts` або CLAUDE.md — деталі в `PROJECT_INTEGRATION.md` §8.

**Повна архітектура, alternatives considered, migration path:** [`docs/decisions/ADR-014-aspect-tagged-profiles.md`](./docs/decisions/ADR-014-aspect-tagged-profiles.md).

---

## 11. Свідомі обмеження v1.0

| Не робимо | Чому |
|---|---|
| Slot Registry, 4 шари (core/stack/capability/domain) | Складність без виправдання — `stack.md` достатньо. |
| Capability-плагіни (postgres, github-actions окремо) | Якщо фреймворк цього потребує — кладемо у фреймворк-плагін. Винесемо у V2 за реальної потреби. |
| Domain-плагіни (fintech, saas) | Те саме. |
| Override механізми у фреймворк-плагінах | Композиція через профіль покриває use-cases. Override → каскад болю. |
| Project-level config для dep-policy | Manifest достатньо. |
| Власний CLI-installer | Native `/plugin install` дає leverage. |
| Параллелізм Security ∥ QA | V2 — складніша orchestration. У v1.0 sequential — надійніше. |
| 4-layer dep check (lint/doctor/preflight/runtime) | One-shot preflight у orchestrator-і. Решта — overengineering. |

---

## 12. Patterns, на які ми посилаємось

Цей підхід не вигаданий нами — це той самий патерн, що використовують:

| Система | Аналогія |
|---|---|
| **Webpack** | Loaders/plugins реєструються, бандлер незмінний. |
| **Symfony Bundles** | Бандли реєструють services, ядро не зачіпається. |
| **VS Code Extensions** | Extensions додають contributions, не переписуючи editor. |
| **Maven Lifecycle** | Плагіни bind-яться до lifecycle phases. |

Усі ці системи десятки років масштабуються, бо контракт «реєстрація через декларацію + конвенційні точки розширення» простіший за override.

---

## 13. Open questions (закриваємо при імплементації)

1. **Точний шлях кешу плагінів.** `~/.claude/plugins/cache/**/stack.md` — припущення. Перевірити в Phase 0 на live-системі і зафіксувати в orchestrator skill.
2. **Як саме core orchestrator вираховує `diff < 50 LOC`** для skip-rules — git diff від main, чи інакше? Зафіксуємо в Phase 3.
3. **Чи має `--stack=` override в `/sdlc:start` зберігатися між викликами в межах сесії**? Поки — ні, кожен запуск окремо.
4. **Версіонна сумісність між core@X і фреймворк-плагіном@Y.** Поки `dependencies` блок у `plugin.json` декларує semver. Стандартні правила.
5. **Telemetry-агрегація через декілька запусків** — чи будувати дашборд, чи лишити per-run JSON. Phase 6+ за реальною потребою.

---

## 14. Reference summary

> **Фреймворк-плагіни не override-ять core. Вони реєструють себе через декларативний профіль (`stack.md`) і надають спеціалізовані агенти/скіли. Core пайплайн читає профілі і композує виконання. Cost discipline — частина дизайну, не пост-факту оптимізація.**

Деталі реалізації покроково — `IMPLEMENTATION_PLAN.md`.
Зовнішні залежності (superpowers тощо) — `DEPENDENCIES.md` (потребує спрощення під цю архітектуру).

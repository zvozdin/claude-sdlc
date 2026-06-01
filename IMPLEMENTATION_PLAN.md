# Claude SDLC Marketplace — План Реалізації

> Покроковий план від нуля до v1.0 marketplace, що реалізує **Stack Provider Pattern** з `ARCHITECTURE.md`.
>
> **Ключові інпути:**
> - `ARCHITECTURE.md` — цільова архітектура (3 файли: `stack.md` + core agents + framework agents).
> - Cost analysis (вшита в дефолти): smart tiering, compact handoffs, iteration caps, skip-rules.
> - `DEPENDENCIES.md` — механіка superpowers/external deps (потребує спрощення під нову архітектуру).

---

## 0. Філософія

Три принципи, які тримають план:

1. **Композиція, не override.** Core має пайплайн і не змінюється. Фреймворк-плагіни **додають себе** через `stack.md`, не редагують core. Будь-яка спокуса дати фреймворку «трохи перевизначити» core означає, що ми проґавили розширення в самому профілі.
2. **Cost-conscious by default.** Кожен агент має `model:`, обраний за принципом «cost of mistakes». QA має iteration cap. Handoff-и компактні. Skip-rules для тривіального. Це не оптимізація — це дефолт з дня 1.
3. **Walking Skeleton рано.** До кінця Phase 2 хочемо робочий ланцюжок: `/plugin install laravel-plugin` → `/sdlc:start "..."` → реальний PR на тестовому Laravel-проєкті.

---

## 1. Roadmap

```
Phase 0 — Repo scaffolding                         (1-2 дні)
Phase 1 — Core: orchestrator skill + 5 agents      (5-7 днів)
Phase 2 — laravel-plugin: перший фреймворк-провайдер (5-7 днів)
Phase 3 — Cost optimizations + dep system          (3-5 днів)
Phase 4 — Polish: docs, /sdlc:doctor, /list-stacks (2-3 дні)
─── milestone v1.0 ───────────────────────────────────────────
Phase 5 — Другий фреймворк (Django або NestJS)     (5-7 днів)
─── milestone v1.1 ───────────────────────────────────────────
Phase 6+ — Beyond MVP (parallel QA∥Sec, дашборди, batch API)
```

Соло part-time → ~4–5 тижнів до v1.0, ще 1 тиждень до v1.1.

---

## 2. Phase 0 — Repository Scaffolding

**Goal:** валідний marketplace з 1 порожнім плагіном (`sdlc`-skeleton), що ставиться через `/plugin marketplace add`.

**Файли:**

```
sdlc-marketplace/
├── .claude-plugin/
│   └── marketplace.json
├── plugins/
│   └── sdlc/
│       ├── .claude-plugin/plugin.json   ← name, version 0.0.1, description
│       ├── stack.md                      ← vanilla profile (priority: 0)
│       └── README.md
├── README.md                             ← vision + Quickstart
├── LICENSE                               ← MIT
└── CHANGELOG.md
```

**Чек-лист:**
- [ ] Створити репо.
- [ ] `marketplace.json` з 1 плагіном-стабом.
- [ ] `sdlc/.claude-plugin/plugin.json` мінімальний.
- [ ] `sdlc/stack.md` — vanilla profile (priority: 0, detect.any: ["*"]).
- [ ] README з vision і двома командами install.
- [ ] Локальний smoke: `/plugin marketplace add file://$(pwd)` бачить плагін.

**Done criterion:** інстал `sdlc` локально проходить без помилок (плагін реєструється, навіть якщо нічого не вміє).

**Не робимо:** CLI installer, JSON Schema lint (тут ще нічого валідувати), GitHub Actions (V2). Інтегруємо те, що дає найбільше leverage першим — нативний marketplace.

---

## 3. Phase 1 — Core: Orchestrator Skill + 5 Default Agents

**Goal:** робочий core, який на `/sdlc:start "..."` виконує повний vanilla pipeline (BA → Dev → QA → Security → Docs) на будь-якому проєкті.

### 3.1. Файли цієї фази

```
sdlc/
├── .claude-plugin/plugin.json
├── stack.md                                    # vanilla з §3.1 ARCHITECTURE.md
├── commands/
│   └── start.md                           # /sdlc:start "<feature>"
├── skills/
│   └── pipeline-orchestrator/
│       └── SKILL.md                            # 8-крок алгоритм з §4 ARCHITECTURE
├── agents/
│   ├── business-analyst.md                     # model: opus
│   ├── developer.md                            # model: sonnet
│   ├── qa-engineer.md                          # model: sonnet, iteration cap=3
│   ├── security-analyst.md                     # model: opus
│   └── document-writer.md                      # model: haiku
└── README.md
```

### 3.2. Агенти: точний frontmatter (cost-conscious дефолти)

```yaml
# business-analyst.md
---
name: business-analyst
description: |
  Senior business analyst. Reads ambiguous task descriptions and produces user stories
  with acceptance criteria, edge cases, and data model.

  Trigger — EN: feature, story, requirements, analyze, scope.
  Trigger — UA: фіча, юзер сторі, вимоги, аналіз, скоуп.
model: claude-opus-4-7
tools: [Read, Glob, Grep, WebSearch, WebFetch]
---
```

```yaml
# developer.md  (vanilla fallback)
---
name: developer
description: Generic full-stack implementer; replaced by framework-specific agent when stack profile defines one.
model: claude-sonnet-4-6
tools: [Read, Glob, Grep, Edit, Write, Bash]
---
```

```yaml
# qa-engineer.md  (CRITICAL: iteration cap)
---
name: qa-engineer
description: Tests for the implemented changes; max 3 fix attempts then stops.
model: claude-sonnet-4-6
tools: [Read, Glob, Grep, Edit, Write, Bash]
---

# Hard limit: 3 attempts to fix failing tests, then STOP and report.
# This is enforced — runaway iterations are the #1 cost incident.
```

```yaml
# security-analyst.md
---
name: security-analyst
description: OWASP Top 10 review; fixes Critical/High; non-obvious vulnerabilities.
model: claude-opus-4-7
tools: [Read, Glob, Grep, WebSearch]
---
```

```yaml
# document-writer.md
---
name: document-writer
description: PR description, release-notes blurb, changelog entry — structured output from well-defined inputs.
model: claude-haiku-4-5
tools: [Read, Glob, Grep, Bash, mcp__github__pull_request_create, mcp__github__add_comment_to_pending_review]
---
```

### 3.3. Pipeline orchestrator skill — критично важливі деталі

`pipeline-orchestrator/SKILL.md` робить 8 кроків з §4.1 ARCHITECTURE.md. Ключові вшиті моменти:

- **Step 0a (deps preflight)** — заглушка в Phase 1 (просто continue). Реальна реалізація в Phase 3.
- **Step 0b (stack detection)** — `Glob ~/.claude/plugins/cache/**/stack.md`, парсинг frontmatter, оцінка `detect.all`/`detect.any` через `Read` + regex, вибір max-priority match.
- **Step 3 (phase execution)** — agent повертає COMPACT summary (≤2-3K tokens), повний детальний вивід пише у `docs/plans/{slug}/0X-<phase>.md` для майбутнього reference.
- **Step 5 (final summary)** — пише `docs/plans/{slug}/_telemetry.json` зі стеком, моделями, токенами, вартістю, skip-rules.

### 3.4. Skip-rules у v1 — мінімальні

В Phase 1 реалізуємо **тільки одне** skip-правило (просте і безпечне):

```
If $ARGUMENTS matches /typo|fix typo|rename .* to|format/i AND
   git diff against main shows < 30 LOC changed:
  Skip BA phase (use $ARGUMENTS as spec directly).
```

Решта skip-rules (skip Security для <50 LOC, skip QA для config-only) — Phase 3 за реальними даними telemetry.

### 3.5. Чек-лист Phase 1

- [ ] `pipeline-orchestrator/SKILL.md` написаний за §4 ARCHITECTURE.md.
- [ ] 5 агентів з точним frontmatter (моделі, tools allowlist).
- [ ] `qa-engineer.md` має жорстко прописаний iteration cap.
- [ ] Compact summary конвенція в кожному phase prompt.
- [ ] Skip-rule typo-fix реалізований і протестований.
- [ ] `/sdlc:start "test feature"` на пустому non-Laravel проєкті проходить vanilla pipeline до кінця.
- [ ] `_telemetry.json` пишеться і містить per-phase tokens + cost.

**Done criterion:**
1. Pipeline на vanilla-проєкті доходить до Docs у ≥80% спроб (10 запусків — ≥8 успішних).
2. Cost одного запуску для тривіальної фічі ≤ $0.80.
3. Cost для medium-фічі ≤ $2.00 (з prompt caching).
4. Iteration cap у QA реально спрацьовує: штучно зламана крихка тест зупиняється на 3-й спробі, не на 10-й.

---

## 4. Phase 2 — `laravel-plugin`: Перший Stack Provider

**Goal:** довести, що Stack Provider Pattern працює — фреймворк-плагін додає себе без жодних змін у core.

### 4.1. Файли

```
laravel-plugin/
├── .claude-plugin/plugin.json              # dependencies: sdlc
├── stack.md                                 # §3.2 з ARCHITECTURE.md
├── agents/
│   ├── laravel-architect.md                # model: sonnet
│   └── artisan-specialist.md               # model: sonnet (для extra phase "database")
├── skills/
│   ├── laravel-conventions/SKILL.md
│   └── eloquent-patterns/SKILL.md
├── .mcp.json                                # laravel-boost
├── hooks/hooks.json                         # pint --fix на Stop
└── README.md
```

### 4.2. Що в `stack.md` обовʼязково

Точно як §3.2 ARCHITECTURE.md:

- `priority: 100`, detect = `composer.json` + `"laravel/framework"` regex.
- `agents_per_phase`: development → laravel-architect, database → artisan-specialist; решта — core agents.
- `extra_phases`: database після development.
- `phase_prompts_injection` для development, qa, security з Laravel-конкретикою.
- `convention_skills`: laravel-conventions, eloquent-patterns.
- `post_pipeline_checks`: `vendor/bin/pint --test`, `php artisan test`, `php artisan route:list`.

### 4.3. Перенесення з `claude-laravel`

Беремо тільки те, що реально стек-специфічне:

| З `claude-laravel` | Куди в `laravel-plugin` |
|---|---|
| `agents/developer.md` (full-stack Laravel + Inertia) | `agents/laravel-architect.md` (model: sonnet) |
| `agents/dba.md` + queue/migration логіка | `agents/artisan-specialist.md` (для phase=database) |
| `skills/laravel-architecture`, `laravel-actions-patterns` | `skills/laravel-conventions/` (один консолідований) |
| `skills/php-pro` (релевантні частини) | `skills/eloquent-patterns/` (об'єднати) |
| `.mcp.json` `laravel-boost` фрагмент | `.mcp.json` цілком |
| Stop-hook Pint (з `settings.json`) | `hooks/hooks.json` |
| Rule `code-style.md`, `forms-authorization.md`, `inertia-vue.md` | Інлайн у `phase_prompts_injection` для development у `stack.md` |

**Що НЕ переносимо** (рішення):

- `agents/ba.md` — використовуємо core's `business-analyst`.
- `agents/reviewer.md`, `tester.md`, `qa.md`, `security-scanner.md`, `docs-writer.md`, `debugger.md`, `devil.md` — core's відповідні агенти або не потрібні в новій моделі.
- `agents/frontend.md`, `filament.md` — поки не переносимо (оцінимо в Phase 5, чи треба як другий extra phase, чи все робить laravel-architect).
- 23 окремі скіли — консолідуємо у 2 (laravel-conventions, eloquent-patterns). Решта (architecture-designer, security-reviewer тощо) живуть у core або заходить через superpowers.
- Постгрес-скіли × 3 → не в laravel-plugin (postgres — наскрізне; в Phase 5+ оцінимо чи виносити у власний плагін, чи лишити в Laravel поки що).
- Playwright-скіли × 2 → так само, V2.

### 4.4. Чек-лист Phase 2

- [ ] `laravel-plugin/.claude-plugin/plugin.json` з `dependencies.plugins[0].name = "sdlc"`.
- [ ] `stack.md` валідно парситься orchestrator-ом (тест: `/sdlc:list-stacks` показує laravel з priority=100).
- [ ] На реальному Laravel-проєкті (composer.json + laravel/framework) auto-detect обирає laravel profile, не vanilla.
- [ ] `--stack=vanilla` override працює.
- [ ] Extra phase `database` запускається після development.
- [ ] `phase_prompts_injection` доходить у промпт відповідного агента (логуємо в `_telemetry.json`).
- [ ] Post-pipeline `pint --test`, `php artisan test`, `route:list` проганяються.
- [ ] Pint Stop-hook автоформатує файли.
- [ ] **End-to-end smoke**: на тестовому Laravel-репо `/sdlc:start "Add subscription billing module"` доходить до PR.

**Done criterion:**
1. Pipeline проходить кінця в ≥80% спроб на 5 контрольних кейсах різного скоупу.
2. Cost одного medium-запуску (Stripe billing-style) ≤ $2.50 (з prompt caching).
3. **Жодних змін у `sdlc/`** під час цієї фази. Якщо щось змінилось — це сигнал, що архітектура має прогалину; оформлюємо як RFC і вирішуємо до завершення фази.

Цей пункт #3 — найважливіший тест архітектури. Якщо він пройде — Stack Provider Pattern довів свою життєздатність.

---

## 5. Phase 3 — Cost Optimizations + External Dep System

**Goal:** довести cost до бюджету $1.40-$1.80/medium-run і реалізувати spravзний superpowers preflight.

### 5.1. Cost-optimization checklist

- [x] **Compact handoff конвенція** — повторно перевірити кожен phase prompt: агент повертає тільки COMPACT summary (≤3K), деталі — у файли. _(Done: Step 3d-1 додає `compact_summary_chars` + `compact_handoff_violation` flag з stderr warning при перевищенні 3K chars.)_
- [x] **Skip-rules розширені** (Step 0c у `pipeline-orchestrator/SKILL.md`):
  - [x] `whitespace-only` → skip BA + QA.
  - [x] `config-only` (тільки `.env|.yaml|.json|.toml|.ini` AND <200 LOC) → skip QA.
  - [x] `lightweight-no-db` (<50 LOC + no migrations + no auth/secret paths) → skip Security з inline secret-leak check у Dev.
  - [x] Original `typo-fix` rule retained.
- [x] **Prompt caching audit** — Step 3b-1 reorganized into STABLE PREFIX + PER-CALL CONTEXT layout. New "Prompt-caching discipline" subsection in hard rules forbids `task_slug`/timestamps/UUIDs in the stable prefix. Verification (`cache_read_input_tokens > 0` on 2nd run) is part of the F-baseline done-criteria.
- [ ] **Telemetry cost — реальні дані:** після 20 запусків на тестовому Laravel-проєкті побудувати таблицю cost-per-stack-per-phase і зафіксувати baseline. _(Schema + aggregation methodology landed in `docs/cost-baseline.md`; numeric data fills in after 20 production runs.)_

### 5.2. Dependency system (один-єдиний preflight)

Спрощено від попередньої версії `DEPENDENCIES.md` — повного rewrite потребує (відкладено за рамки v1.0 plan, але механіка така):

- [x] **Розширити dep schema** — реалізовано через окремий `runtime-dependencies.json` (post-Phase 2 рішення); native `plugin.json → dependencies` лишається масивом імен. Step 0a у `pipeline-orchestrator/SKILL.md` читає `runtime-dependencies.json`.
- [x] **Step 0a в orchestrator skill** — реалізовано (0a-1..0a-6): FS fallback для `mcp__skills__list_skills`, агрегування `block` failures перед exit, MUST-print статус-блок. Stub видалено.
- [x] **Headless mode** — env `SDLC_NONINTERACTIVE=true`: `block` → exit 1 з JSON, `warn` → stderr і continue, `graceful-degrade` → silent. Документовано у `commands/start.md`.
- [x] **`/sdlc:doctor`** — окрема команда у `plugins/sdlc/commands/doctor.md`. Read-only варіант Step 0a + stack-profile звіт + cost-baseline summary. Підтримує `--json` flag.

### 5.3. Тригери реальних даних

Після Phase 3 зафіксовано в репо:

- [x] `docs/cost-baseline.md` schema + методологія агрегації (jq-pipeline, done-criteria); порожня таблиця до моменту збору 20 реальних запусків.
- [x] 4 skip-rule-и реалізовані; hit-rate замірюється у baseline-таблиці після 20 runs.
- [x] Superpowers preflight спрацьовує коректно у трьох сценаріях: installed, missing+warn, missing+block (логіка в Step 0a-4; baseline runs валідують end-to-end).

**Done criterion:** середній cost medium-запуску ≤ $1.80 (90% percentile ≤ $2.50). Iteration cap і skip-rules спрацьовують у телеметрії. _Перевіряється під час baseline-collection (operational follow-up: 15 Laravel + 5 vanilla запусків)._

---

## 6. Phase 4 — Polish + Docs

**Goal:** v1.0-готовий стан з документацією, /sdlc:doctor, list-stacks, authoring-guide для майбутніх контриб'юторів.

**Чек-лист:**

- [ ] **`/sdlc:list-stacks`** — команда показує всі знайдені `stack.md` з priority, detect-rules, чи матчиться поточний проєкт.
- [ ] **`/sdlc:doctor`** — preflight + dep-перевірка + cost-baseline read.
- [ ] **README marketplace** — vision, Quickstart (4 команди), приклад на Laravel.
- [ ] **README кожного плагіна** — що ставиться як залежність, як використовувати, що додає до vanilla.
- [ ] **`docs/authoring-stack-plugin.md`** — як написати свій фреймворк-плагін за 1 годину. Конкретно: скелет, `stack.md` template, мінімальний агент, як тестувати.
- [ ] **`docs/cost-discipline.md`** — публічна документація принципів tiering, compact handoffs, iteration cap. Щоб contributor-и розуміли, чому архітектура така.
- [ ] **`docs/superpowers-integration.md`** — як заявити залежність у своєму плагіні, які політики обирати.
- [ ] **GitHub Actions** — лінт `plugin.json` за схемою, валідація `stack.md` frontmatter.
- [ ] **3+ зовнішніх рев'юера** прогнали Quickstart без блокерів.

**Done criterion:** новий контриб'ютор може за 2 години локально пройти повний flow по docs (clone → install → /sdlc:start → PR).

---

## 7. Definition of Done для v1.0

```
✅ marketplace.json опублікований
✅ sdlc@1.0.0 і laravel-plugin@1.0.0 ставляться через native /plugin install
✅ /sdlc:start доходить кінця у ≥85% запусків на 10 контрольних кейсах
✅ Середній cost medium-запуску ≤ $1.80; 90-perc ≤ $2.50
✅ Iteration cap працює; runaway costs неможливі за дизайном
✅ Skip-rules працюють (логуються в telemetry)
✅ Dependency preflight (superpowers) працює у 3 сценаріях
✅ Stack Provider Pattern доведений: laravel-plugin не змінив жодного файлу core
✅ Documentation: README, Quickstart, authoring-guide, cost-discipline, superpowers-integration
✅ GitHub Actions з лінтом маніфестів
```

---

## 8. Phase 5 — Другий Фреймворк (Validation)

**Goal:** довести, що архітектура справді розширювана — додати другий фреймворк без змін у core.

**Кандидати** (обираємо за реальним інтересом):

- **`django-plugin`** — Python-стек, найбільший контраст з Laravel.
- **`nestjs-plugin`** — TypeScript, сучасний backend, інша екосистема.

Якщо вибір однаково цінний — обираємо **Django**, бо більший контраст показує гнучкість.

**Чек-лист (Django варіант):**

- [ ] Створити `plugins/django-plugin/`.
- [ ] `stack.md` з detect: `manage.py` + `django` у `requirements.txt`/`pyproject.toml`, priority=100.
- [ ] `agents/django-architect.md` (sonnet).
- [ ] (опційно) `agents/drf-specialist.md` для extra phase `api-layer`.
- [ ] Skills: `django-conventions`, `orm-patterns`.
- [ ] `.mcp.json` для postgres MCP, django-docs.
- [ ] `hooks/hooks.json` для black/ruff.
- [ ] Smoke test на тестовому Django-проєкті.

**Done criterion:**
1. Pipeline на Django-проєкті проходить кінця в ≥75% запусків.
2. **Жодних змін у core** під час цієї фази. Знову тест архітектури — якщо core ламається, виявляємо прогалину і фіксимо до завершення фази.
3. Час, витрачений на створення django-plugin, ≤ 5 робочих днів.

Цей пункт — реальна валідація. Якщо тут все встане гладко — архітектура жива, marketplace готовий приймати community-контрибуції.

---

## 9. Phase 6+ — Beyond MVP

Після v1.1 (core + 2 фреймворки) розглядаємо за реальною потребою:

- **Parallel Security ∥ QA** — складніша orchestration, дає 50% wall-clock economy.
- **`/sdlc:cost-dashboard`** — агрегована візуалізація telemetry.
- **Batch API для post-pipeline checks** — 50% знижка на heavy-аналіз (dep audit, full sec scan).
- **Background monitor для long-running tasks** — окремий клас задач (рев'ю PR щоранку, daily standup digest).
- **Більше фреймворків** (NestJS, Rails, FastAPI, .NET, Go).
- **Skip-rules за telemetry-даними** — auto-tune на основі реальних кейсів, де skip був виправданим.
- **Public marketplace launch** з featured-плагінами і community contributions.

Кожен з цих пунктів — окремий RFC, не починається без нього.

---

## 10. Cost Discipline (вшита в дефолти, не окрема фаза)

Підсумок принципів, які ми вшиваємо в кожний агент і кожен phase prompt **з Phase 1**, не пост-факто:

| Принцип | Реалізація |
|---|---|
| **Smart model tiering** | BA + Security → Opus; Dev + QA + extra phases → Sonnet; Docs → Haiku. Зашито у frontmatter агентів. |
| **Compact handoffs** | Кожен phase prompt просить агента повернути COMPACT summary (≤2-3K). Деталі живуть у файлах. |
| **Iteration cap** | QA жорстко обмежений 3 спробами. Прописано в prompt і в SKILL.md. |
| **Skip-rules** | Orchestrator аналізує scope перед запуском і пропускає фази для тривіальних задач. Phase 1: typo-fix; Phase 3: розширюємо. |
| **Tool restrictions** | BA — read-only. Security — read+web. Dev/QA — повний набір. Менше тулів = менше випадкових heavy calls. |
| **Prompt caching by design** | Стабільні system prompts, ніяких dynamic-генерованих частин. |
| **Telemetry per run** | `_telemetry.json` пише per-phase cost. Без цього неможливо оптимізувати. |

Цільові цифри:

- Trivial change: ≤ $0.50.
- Medium feature: ≤ $1.80 average, ≤ $2.50 p90.
- Large feature: ≤ $4.00.

Якщо реальні дані показують вищі цифри — переглядаємо в Phase 3 на основі telemetry.

---

## 11. Anti-patterns (свідомо не робимо)

| Не робимо | Чому |
|---|---|
| Дозволити фреймворк-плагіну редагувати/override-ити core agents | Каскад болю при оновленні core; ламає DRY. |
| Запускати `pipeline-orchestrator` як копію в кожному фреймворк-плагіні | Мета архітектури — один orchestrator. Дублювання = втрата DRY. |
| Покласти Haiku на BA, бо «щоб дешевше» | Помилки BA компаундяться через 5 наступних фаз. Економія на BA = 10x rework. |
| Дозволити QA крутитися без iteration cap | Найдорожчий incident у системі. Cap = $1-2 межа на runaway. |
| Передавати повний вивід попередньої фази у промпт наступної | 60-100K дублікату input. Compact summary + файли — норма. |
| Робити повний pipeline на typo-fix | Skip-rules економлять 50-70%. |
| Робити Slot Registry / 4-шарову архітектуру у v1.0 | Складність без виправдання. Stack Provider Pattern достатньо. |
| Покладатися на silent auto-install залежностей | Технічно неможливо в Claude Code. Guided install = чесно. |
| Перевіряти deps у кожному агенті | One-shot preflight у orchestrator. Решта = overhead. |
| Запускати orchestrator на `claude-pro` план | Швидко вб'є rate limits. Документуємо мінімальні вимоги в README. |

---

## 12. Decision Log (живий)

| Date | Decision | Rationale |
|---|---|---|
| 2026-04-26 | Прийняти Stack Provider Pattern як архітектуру | Спрощення проти 4-шарової моделі без втрати функціональності. |
| 2026-04-26 | Cost discipline вшити в дефолти з Phase 1 | Реальні цифри $1.40-$1.80/run потребують дисципліни by design. |
| 2026-04-26 | Iteration cap у QA = 3 спроби (hard) | Найдорожчий known incident. Hard cap > soft hint. |
| 2026-04-26 | Single preflight для deps в orchestrator | 4-layer check був overengineered для нової архітектури. |
| 2026-04-26 | Skip-rules мінімальні в Phase 1 (тільки typo-fix) | Інші skip-rules потребують реальних даних з telemetry. |
| 2026-04-26 | Не переносити з claude-laravel: ba, reviewer, tester, devil etc. | Stack Provider Pattern використовує core's BA/QA/Security/Docs. |
| 2026-04-26 | Phase 5 фреймворк = Django (preferred) | Найбільший контраст з Laravel — кращий тест архітектури. |
| 2026-04-26 | DEPENDENCIES.md потребує спрощення під нову архітектуру | Поточна 4-layer структура надмірна; виокремимо в окремий PR після Phase 3. |
| 2026-04-26 | `sdlc.local.yaml` override механіку перенесено з Phase 3 у post-Phase 2 patch | Реальна потреба для Herd/Docker dichotomy виявилась раніше за заплановане. Реалізована як Step 1b в orchestrator. Деталі — `PROJECT_INTEGRATION.md`. |
| 2026-04-26 | Перейменовано `core-sdlc-plugin` → `sdlc` для чистого `/sdlc:start` namespace | Claude Code префіксує slash-команди іменем плагіна. Старий формат був `/core-sdlc-plugin:sdlc-start` — потворний. |
| 2026-04-26 | Native `plugin.json → dependencies` як array of strings (не об'єкт) | Узгодження з Claude Code schema; кастомна логіка політик переїхала у `runtime-dependencies.json`. |
| 2026-04-26 | **ADR-014** — aspect-tagged profiles для multi-stack композиції (Laravel + Inertia/Vue/React/Livewire) | Поточне «один переможець за priority» не масштабується на реальні Laravel-проєкти, де є і backend (composer.json), і frontend (package.json). Переходимо на per-aspect winners + phase fan-out. Деталі — `docs/decisions/ADR-014-aspect-tagged-profiles.md`. План: Phase 4 (схема), Phase 5 (refactor `laravel-plugin`, новий `inertia-vue-plugin`). |

---

## 13. Стартовий 5-денний sprint (kickoff)

Конкретний план першого тижня, щоб не застрягнути в плануванні:

**Day 1 (4 год):** Створити `sdlc-marketplace/` repo, `marketplace.json`, `sdlc/.claude-plugin/plugin.json`, `sdlc/stack.md` (vanilla profile), README.md.

**Day 2 (6 год):** Написати `pipeline-orchestrator/SKILL.md` за §4 ARCHITECTURE.md (8 кроків). Заглушити Step 0a (deps preflight) — реалізуємо в Phase 3.

**Day 3 (6 год):** 5 default agents з точним frontmatter. Особливу увагу — `qa-engineer.md` (iteration cap) і `business-analyst.md` (read-only tools).

**Day 4 (4 год):** `commands/start.md` — slash-команда, що читає $ARGUMENTS, генерує task-slug, делегує в pipeline-orchestrator. Тест на пустому Laravel-репо: vanilla pipeline проходить кінця.

**Day 5 (4 год):** `_telemetry.json` запис; перший skip-rule (typo-fix). Smoke-test 5-ма різними кейсами. Зафіксувати baseline cost.

~24 години чистого часу. До кінця тижня — Phase 1 завершена, Walking Skeleton працює.

---

> **Один рядок-підсумок:** Stack Provider Pattern + cost discipline by default + один preflight для deps. Без слотів, шарів, override-ів. Архітектура працює бо проста; cost працює бо дисципліна вшита; розширюваність працює бо контракт декларативний.

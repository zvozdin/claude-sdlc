# Project Integration — як плагіни взаємодіють з конфігурацією проєкту

> База знань про те, які канали проєктної конфігурації плагіни SDLC marketplace **автоматично враховують**, які — **проігнорують**, і як адаптувати поведінку під специфіку конкретного проєкту (Herd vs Docker, monorepo, team styles тощо).
>
> Цей документ — джерело правди для авторів плагінів і користувачів. Будь-яке питання «а чи буде моя локальна X врахована?» має знаходити відповідь тут.

---

## 1. Чому це важливо

Плагіни SDLC marketplace задумані як **переносні** — один laravel-plugin працює і з Herd, і з Docker, і з monorepo, і з мікросервісом. Цього не можна досягти, якщо плагін жорстко зашиває команди типу `docker compose exec -T app php artisan ...`. Натомість плагін задає **скелет** (які агенти, які фази, які скіли активні), а специфіка проєкту приходить з:

- `<project>/CLAUDE.md`
- `<project>/.claude/skills/`
- `<project>/.claude/agents/`
- `<project>/.mcp.json`
- `<project>/.claude/sdlc.local.yaml` (Phase 3+, ще не реалізовано)

Цей документ описує точно, що з цих каналів **читається автоматично**, а що **поки що ігнорується**.

---

## 2. Матриця інтеграції

| Канал проєкту | Автоматично респектиться? | Хто читає | Phase у пайплайні |
|---|---|---|---|
| `<project>/CLAUDE.md` | ✅ Так | Усі агенти (orchestrator + всі subagent-и) | Усі фази |
| `<project>/.claude/skills/<name>/SKILL.md` | ✅ Так | Будь-який агент через Skill discovery | Будь-яка фаза, де triggers матчиться |
| `<project>/.claude/agents/<name>.md` | ✅ Так | Override плагінних агентів за іменем | Якщо ім'я співпадає з тим, що вказує `stack.md` |
| `<project>/.mcp.json` | ✅ Так (merge з plugin fragments) | MCP layer Claude Code | Будь-яка фаза з MCP tool |
| `stack.md` → `phase_prompts_injection` | ❌ Hardcoded у плагіні | Orchestrator інжектить у prompt subagent-а | Розписано per-phase |
| `stack.md` → `post_pipeline_checks` | ❌ Hardcoded у плагіні | Orchestrator виконує напряму через Bash | Після всіх фаз |
| `stack.md` → `commands.{build,test,lint}` | ❌ Hardcoded у плагіні | Subagent через injection або orchestrator | Залежить від фази |
| Команди типу `php artisan` всередині phase prompt | ⚠️ Адаптується через CLAUDE.md | Subagent читає CLAUDE.md і обирає форму | Development, Database, QA |
| `<project>/.claude/sdlc.local.yaml` | ✅ Так (з v0.0.1, post-Phase 2 patch) | Orchestrator у Step 1b | Усі фази (через merge з plugin profile) |

---

## 3. Що працює АВТОМАТИЧНО (без жодних дій з боку плагіна)

### 3.1. `CLAUDE.md` проєкту

Claude Code **автоматично** інжектить вміст `<project>/CLAUDE.md` у контекст:
- оркестратора (коли користувач запускає `/sdlc:start`),
- кожного subagent-а (коли orchestrator робить `Agent({subagent_type: ..., prompt: ...})`),
- кожної slash-команди.

Це означає: якщо ти напишеш у `CLAUDE.md` «Use Herd, NEVER docker compose» — laravel-architect, qa-engineer, artisan-specialist усі це побачать і адаптуються. Без потреби міняти плагін.

**У промпті кожного нашого агента** є директива: «Read project conventions in CLAUDE.md and follow them.» Це робить адаптацію реальною, а не теоретичною.

### 3.2. Локальні скіли в `<project>/.claude/skills/`

Будь-який скіл, що лежить за цим шляхом, автоматично доступний усім агентам через Skill discovery. Тобто можна в проєкті мати свій team-style скіл:

```
your-laravel-project/
└── .claude/
    └── skills/
        └── acme-team-style/
            └── SKILL.md
```

Якщо `SKILL.md` має description, що матчить тригер (наприклад, «applies to all Laravel code in this monorepo»), агенти підтягнуть його **поряд** з плагінними `laravel-conventions` і `eloquent-patterns`.

### 3.3. Локальні агенти в `<project>/.claude/agents/`

Якщо тебе категорично не влаштовує плагінний агент — можеш повністю його переписати локально. Файл з тим самим іменем (`laravel-architect.md`) у `<project>/.claude/agents/` **переважає** плагінного.

Це escape hatch для команд з нестандартним workflow. Ціна — втрата автоматичних оновлень з marketplace для цього агента.

### 3.4. `<project>/.mcp.json`

Project-level `.mcp.json` мерджиться з фрагментами, які постачає плагін. На конфліктах ключів **перемагає project**. Тобто якщо плагін постачає `laravel-boost` під Docker, а у проєкті ти переписав на Herd:

```jsonc
// <project>/.mcp.json
{
  "mcpServers": {
    "laravel-boost": {
      "command": "php",
      "args": ["artisan", "boost:mcp"]
    }
  }
}
```

— переможе твоя project-конфігурація. Plugin fragment ігнорується.

---

## 4. Що НЕ адаптується автоматично (поточні обмеження)

### 4.1. `post_pipeline_checks` у `stack.md`

Це shell-команди, які orchestrator виконує **сам**, через Bash, після завершення всіх фаз. Зараз вони hardcoded у плагіні:

```yaml
# laravel-plugin/stack.md
post_pipeline_checks:
  - ./vendor/bin/pint --test
  - php artisan test
  - php artisan route:list
```

Якщо твій проєкт вимагає інших команд (наприклад, `docker compose exec -T app php /var/www/artisan test`) — **orchestrator цього не знатиме і впаде**.

**Чому ми обрали non-Docker форму у Laravel-плагіні:** для більшості laravel + Herd-проєктів це працює. Docker-проєктам треба буде або переробити на свій плагін-форк, або дочекатись Phase 3 (sdlc.local.yaml).

### 4.2. `phase_prompts_injection` у `stack.md`

Це текст, що вшивається у prompt кожного агента під час фази. У `laravel-plugin/stack.md` він каже агентам, наприклад: «Use Pest 4», «Apply skill: laravel-conventions», «Check mass assignment».

Якщо твій проєкт використовує PHPUnit (не Pest), або має іншу версію Laravel із застарілими патернами — injection не адаптується. Workaround — **переоверайдити в `CLAUDE.md`**: «Use PHPUnit, not Pest. Ignore plugin's Pest recommendations.» Агент бачить **обидва** джерела і CLAUDE.md як specific override має превалювати.

### 4.3. `commands.{build,test,lint}` у `stack.md`

Поки що декларуються в плагіні. У Phase 3 переїдуть на per-project override.

---

## 5. Як адаптувати поведінку плагіна під специфіку проєкту

### 5.1. Робочий механізм СЬОГОДНІ — пишеш у `<project>/CLAUDE.md`

Це найпотужніший канал. Шаблон для Laravel + Herd проєкту:

```markdown
# Project: Acme Billing

## Execution environment
This project uses **Laravel Herd** for local development.
- PHP runner: `php` (NOT `docker compose exec -T app php`)
- Composer: `composer` (local)
- Artisan: `php artisan ...`
- Pint: `./vendor/bin/pint`
- Tests: `php artisan test` or `./vendor/bin/pest`

⚠️ Agents: do NOT prefix any command with `docker compose exec`.
The plugin's stack.md may suggest Docker — ignore those suggestions
and use Herd-native equivalents.

## Architecture conventions
- Use Action pattern (Spatie laravel-actions) for non-trivial logic
- Form Requests for ALL validation
- Policies for ALL authorization (registered in AuthServiceProvider)
- Eloquent over raw SQL; if raw, use bindings always
- Inertia v2 + Vue 3 Composition API
- DDD bounded contexts: app/Domain/{Billing,Users,Catalog}/

## Testing conventions
- Pest 4, NOT PHPUnit
- Use RefreshDatabase, never DatabaseTransactions
- Factories required for all models

## What's out of scope for AI changes
- Do NOT touch app/Legacy/* — pending replacement
- Do NOT modify migrations from before 2025_01_01 — already in production
```

Цей CLAUDE.md розв'язує 80% проблем інтеграції без жодних змін у плагіні.

### 5.2. Робочий механізм СЬОГОДНІ — повне переозначення агента

Якщо CLAUDE.md недостатньо (наприклад, тобі потрібен принципово інший pipeline-крок):

```bash
mkdir -p <project>/.claude/agents
cp ~/.claude/plugins/cache/laravel-plugin*/agents/laravel-architect.md \
   <project>/.claude/agents/laravel-architect.md
# редагуй під свої потреби
```

Тепер при Phase 2 Claude Code візьме твою локальну версію замість плагінної.

### 5.3. `<project>/.claude/sdlc.local.yaml` — first-class override механізм

**Це працює з v0.0.1.** Orchestrator на Step 1b читає `<project_root>/.claude/sdlc.local.yaml` (якщо існує) і мерджить його з `stack.md` плагіна.

```yaml
# <project>/.claude/sdlc.local.yaml
post_pipeline_checks:
  - ./vendor/bin/pint --test
  - ./vendor/bin/pest
  - php artisan route:list

phase_command_overrides:
  development:
    php_runner: php                    # NOT "docker compose exec -T app php"
    artisan_runner: php artisan
    composer_runner: composer
  database:
    migrate_command: php artisan migrate
    rollback_command: php artisan migrate:rollback --step=1

extra_phase_prompts:
  qa: |
    Use our snapshot helper at tests/Helpers/Snapshot.php for JSON comparisons.

skip_phases:
  - security                  # вже маємо external SAST у CI

convention_skills_extra:
  - acme:internal-api-style
```

**Merge semantics:**

| Ключ | Семантика |
|---|---|
| `post_pipeline_checks` | **REPLACES** plugin's value entirely. `[]` повністю вимикає default checks. |
| `phase_command_overrides` | Передається як context flag у промпт subagent-а. Subagent використовує overrides замість plugin defaults. |
| `extra_phase_prompts` | **APPENDS** до plugin's `phase_prompts_injection` (additive, не перезаписує). |
| `skip_phases` | Видаляє фази з канонічного порядку. |
| `convention_skills_extra` | APPENDS до `convention_skills`. |

**Що бачиш в output:**

Якщо overrides застосовано — orchestrator друкує:

```
🔧 Local overrides applied from .claude/sdlc.local.yaml:
   post_pipeline_checks: replaced (3 items)
   phase_command_overrides: development.{php_runner,artisan_runner}
   skip_phases: [security]
```

Якщо файла немає — silent, як ніби його не було.

Якщо файл є, але YAML битий — warning, продовжуємо з plugin defaults (не падаємо).

---

## 6. Як subagent читає CLAUDE.md (під капотом)

Технічний потік для laravel-architect у Phase 2 (Development):

```
1. Користувач: /sdlc:start "Add subscription billing"
2. Orchestrator (Skill: pipeline-orchestrator) починає роботу.
3. Orchestrator готує prompt для laravel-architect:
   - base prompt з SKILL.md
   - + injection з stack.md (laravel-specific)
   - + посилання на _brief.md і 01-business-analysis.md

4. Orchestrator: Agent({ subagent_type: "laravel-architect", prompt: <промпт> })

5. Claude Code запускає subagent з:
   - НОВИЙ context window (ізольований від orchestrator)
   - Системний prompt = frontmatter + body laravel-architect.md
   - АВТОМАТИЧНО: вміст <project>/CLAUDE.md
   - АВТОМАТИЧНО: список доступних skills, agents, MCP-серверів

6. Subagent починає виконання → читає CLAUDE.md → бачить «Use Herd, NEVER docker» →
   адаптує всі команди.

7. Subagent виконує роботу, повертає COMPACT summary до orchestrator.
```

Ключове: Claude Code **сам** інжектить CLAUDE.md у контекст subagent-а — це не наша логіка, це нативна поведінка платформи. Тому ти можеш бути впевнений, що навіть якщо я в `laravel-architect.md` забув написати «read CLAUDE.md» — Claude Code все одно його надасть.

---

## 7. Концептуальна ієрархія precedence

Коли є конфлікт між кількома джерелами вказівок, агент має дотримуватись цього порядку (від найвищого пріоритету):

```
1. <project>/CLAUDE.md         ← найвищий пріоритет, реальність проєкту
2. <project>/.claude/agents/   ← повне переозначення агента
3. <project>/.claude/skills/   ← локальні team-skills
4. <project>/.mcp.json         ← локальні MCP overrides
5. plugin's stack.md           ← stack-specific defaults
6. plugin's agent prompts      ← агент-специфічні defaults
7. plugin's skills             ← reusable conventions
8. base SKILL.md prompts       ← найзагальніші правила
```

Якщо ти бачиш, що агент дотримується pluginного дефолта замість твого CLAUDE.md — це **баг** (у нашому або Claude Code). Скинь приклад, виправлятимемо.

---

## 8. Поширені сценарії і рекомендації

### Сценарій A: Laravel + Herd

**Проблема:** plugin's `post_pipeline_checks` і деякі phase injections припускають Docker.

**Рекомендація:**
1. Створити `<project>/.claude/sdlc.local.yaml` з:
   ```yaml
   post_pipeline_checks:
     - ./vendor/bin/pint --test
     - ./vendor/bin/pest
     - php artisan route:list
   phase_command_overrides:
     development: { php_runner: php, artisan_runner: php artisan }
     database: { migrate_command: php artisan migrate }
   ```
2. Доповнити в `<project>/CLAUDE.md` «Use Herd locally. Plugin's Docker suggestions don't apply here.»
3. Готово — orchestrator адаптує і `post_pipeline_checks`, і phase prompts.

### Сценарій B: Laravel + Docker (default з plugin)

**Проблема:** немає, плагін під це і написаний.

**Рекомендація:** просто переконайся, що в проєкті docker-compose сервіс називається `app` (як у плагінній команді). Якщо інакше — переоверайдь у CLAUDE.md.

### Сценарій C: Monorepo з кількома Laravel-сервісами

**Проблема:** plugin `stack.md` припускає, що `composer.json` у корні. У monorepo це не так.

**Рекомендація для Phase 3+:** додамо `working_directory` параметр у профіль. Поки що — або запускати `/sdlc:start` з самого сервіс-кореня (cd `<service>`), або клонувати laravel-plugin локально і адаптувати detect.

### Сценарій D: PHPUnit замість Pest

**Проблема:** plugin's QA injection каже «Use Pest 4».

**Рекомендації (обери одну):**
- У `<project>/.claude/sdlc.local.yaml` додай:
  ```yaml
  extra_phase_prompts:
    qa: "Tests: PHPUnit only. Ignore any Pest-specific guidance from the plugin. Use ./vendor/bin/phpunit and standard PHPUnit assertions."
  ```
  (це APPEND до plugin's injection — qa-engineer прочитає і це override-ить.)
- Або у `CLAUDE.md` написати «Tests: PHPUnit only.» Менш точково, але працює.

### Сценарій E: Власний AI code style scanner у CI

**Проблема:** не хочу, щоб security-analyst робив повний OWASP-прохід — це робить наш external SAST.

**Рекомендація:** `<project>/.claude/sdlc.local.yaml`:
```yaml
skip_phases:
  - security
```
Готово — orchestrator пропускає фазу. У telemetry буде запис, що пропущено через local override.

---

## 9. Що вимагати від проєктів, що користуються плагіном

Базовий чек-лист для команди, що першого разу ставить marketplace:

- [ ] Створено `<project>/CLAUDE.md` з мінімум: execution environment, architecture conventions, testing framework, що-не-чіпати-секція.
- [ ] Перевірено, що `composer.json` (для Laravel) лежить у корені, або додано документацію про monorepo шлях.
- [ ] Якщо команда відрізняється від плагінних defaults — задокументовано в CLAUDE.md що саме і як адаптувати.
- [ ] Запущено `/sdlc:list-stacks` — переконано, що профіль обирається правильно.
- [ ] Запущено `/sdlc:start "trivial test"` — пайплайн доходить кінця без втручання.

---

## 10. Open questions

1. **Як саме Claude Code мерджить `.mcp.json`?** Я припускаю, що project takes priority — треба верифікувати на практиці у Phase 3.
2. **Чи `<project>/.claude/skills/` справді override-ить плагінні скіли** з тим самим іменем, чи вони існують паралельно? Перевірити, задокументувати тут.
3. **Чи Claude Code інжектить CLAUDE.md з parent-директорій** (типу `<repo-root>/CLAUDE.md` коли працюємо в `<repo-root>/services/billing/`)? Це важливо для monorepo.
4. **Який точний шлях** до cache плагінів — `~/.claude/plugins/cache/<name>@<version>/` припущення; на різних платформах може відрізнятись.

---

## 10.5. Multi-aspect projects (Laravel + Vue/React/Livewire тощо)

Сценарії D і E вище — це **workaround-и поки v0.0.1**. Архітектурне рішення для проєктів з кількома аспектами (backend + frontend + database) — **aspect-tagged profile composition**, заплановане на Phase 4-5.

Після Phase 5 в Laravel + Inertia + React проєкті авто-активуються **обидва** плагіни:
- `laravel-plugin` (aspects: backend, database)
- `inertia-react-plugin` (aspects: frontend)

Без потреби в `extra_phase_prompts` workaround-ах — кожен плагін відповідає за свій аспект, агенти запускаються по черзі у відповідних фазах.

Повна архітектурна аргументація, alternatives considered, migration path — [`docs/decisions/ADR-014-aspect-tagged-profiles.md`](./docs/decisions/ADR-014-aspect-tagged-profiles.md).

---

## 11. Roadmap

| Phase | Що додаємо |
|---|---|
| 0.0.1 (зараз) | CLAUDE.md, project skills, project agents, `.mcp.json` merge — все працює. **`sdlc.local.yaml` додано як post-Phase 2 patch.** |
| Phase 3 | Розширити `sdlc.local.yaml`: per-stage timeout overrides, custom telemetry sinks, agent-level overrides. |
| Phase 4 | `/sdlc:doctor` показує всі активні overrides і конфлікти. |
| V2 | Auto-detect Herd vs Docker і переключатись automatically (без потреби в `sdlc.local.yaml`). |

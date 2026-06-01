# External Plugin Dependencies — Design

> Документ описує, як плагіни нашого marketplace декларують і резолвлять залежність від **зовнішніх плагінів** (наприклад, `obra/superpowers`), щоб перевикористовувати їхні скіли в наших агентах без копіювання.

> **Споріднені документи:** `ARCHITECTURE.md` · `IMPLEMENTATION_PLAN.md` · ADR-013 у `docs/decisions/`.

---

## 1. Проблема

Наш core та стек-плагіни хочуть викликати готові скіли з популярних community-плагінів (стартовий приклад — **`superpowers`** від `obra`, який містить скіли на кшталт `thinking-deeply`, `rubber-ducking`, `pattern-recognition`). Без цього механізму ми б змушені були:

- Копіювати чужі скіли в свій репо (юридично/технічно сумнівно і завжди застаріле).
- Ігнорувати їх і писати власні (марне дублювання).
- Робити жорстку bundling-залежність (юзер не зможе обрати).

**Суть проблеми:** Claude Code plugins на сьогодні **не мають** вбудованого dependency resolution (зазначено навіть в офіційній документації marketplace). Тож систему перевірки/встановлення/деградації будуємо самі.

---

## 2. Базові факти про Claude Code, які формують дизайн

1. Плагіни встановлюються через `/plugin marketplace add ...` + `/plugin install ...` — **це slash-команди в сесії юзера**.
2. З коду агента **не можна** запустити іншу slash-команду фоном.
3. Встановлені плагіни лежать у `~/.claude/plugins/<plugin-name>/` (або в проектному `.claude/plugins/` для проектних плагінів) і кожен має `.claude-plugin/plugin.json`.
4. Skill-и плагіну доступні агентам у форматі `pluginname:skillname` (namespacing — на рівні Claude Code).
5. Тому «авто-інсталу» в сенсі silent-install **не існує** в межах сесії. Реалістичні варіанти: warn / guided-install з checkpoint-ом / hard-block.

Дизайн будуємо чесно навколо цих обмежень — без «магії», що не працює.

---

## 3. Чотири рішення, які приймаємо

| ID | Рішення | Альтернатива | Чому обрали |
|----|---------|--------------|-------------|
| D1 | Залежності декларуються у `plugin.json → requires.plugins` (machine-readable). | Документувати лише в README. | Інакше lint, doctor і preflight нічого не можуть перевірити автоматично. |
| D2 | Перевірка йде у **трьох точках**: lint (статика), `/sdlc:doctor` (за запитом), preflight pipeline (на старті фічі). | Лише в одній точці. | Кожна точка ловить інший клас помилок. |
| D3 | Три політики поведінки на missing-dep: **`block` / `warn` / `graceful-degrade`** — задаються per-залежність і per-скіл. | Єдина глобальна політика. | Різні залежності — різна критичність. Skill «thinking-deeply» — `warn`, а Skill «authoritative-source-validator» — `block`. |
| D4 | Auto-install — не silent, а **guided**: AskUserQuestion + копіпаст-блок з командою + checkpoint «coнтінью». | Silent install via shell. | Неможливо технічно (slash-команди не запускаються з агента) і небезпечно за дизайном. |

---

## 4. Декларативна модель

### 4.1. У `plugin.json`

Розширюємо вже існуючий блок `requires` з ARCHITECTURE.md §4.2:

```jsonc
{
  "name": "stack-laravel",
  "version": "1.0.0",
  "requires": {
    "core": "^1.0.0",

    "plugins": {
      "superpowers": {
        "marketplace": "obra/superpowers",          // де встановлювати
        "version":     "^2.0.0",                    // semver constraint
        "policy":      "warn",                      // дефолтна політика на цей плагін
        "installCommand": [
          "/plugin marketplace add obra/superpowers",
          "/plugin install superpowers@superpowers"
        ],
        "skills": {                                  // які саме скіли потрібні
          "thinking-deeply":      { "policy": "warn"             },
          "rubber-ducking":       { "policy": "warn"             },
          "pattern-recognition":  { "policy": "graceful-degrade" },
          "fact-check":           { "policy": "block"            }
        }
      }
    }
  }
}
```

**Семантика полів:**

- `marketplace` — звідки встановлювати. Може бути `org/repo`, або повний URL.
- `version` — semver. Перевірка проти `version` зі встановленого `plugin.json`.
- `policy` (на рівні плагіну) — дефолт, якщо для конкретного скіла не вказано окремо.
- `installCommand` — масив рядків, що буде показано юзеру з copy-paste.
- `skills` — мапа: ім'я скіла → налаштування. **Тільки скіли, перелічені тут, перевіряються** (інакше missing-skill було б шумом, бо ми не знаємо, що нам реально треба).

### 4.2. У frontmatter агента

Кожен агент **експліцитно декларує**, які зовнішні скіли йому потрібні. Це ПРИНЦИПОВО, бо інакше:
- Lint не може перевірити, що `requires.plugins` у `plugin.json` адекватний.
- При завантаженні агента Claude не знає, яку політику застосовувати.

```yaml
---
name: ba
slot: BA
model: opus
tools: [Read, Glob, Grep, WebSearch, WebFetch, SendMessage, Agent]

requires_skills:
  internal:
    - brainstorming
    - plan-writing

  external:
    - plugin: superpowers
      skill: thinking-deeply
      reason: "Deep mental models for ambiguous requirements"
      # policy успадковується з plugin.json → requires.plugins.superpowers.skills["thinking-deeply"]

    - plugin: superpowers
      skill: rubber-ducking
      reason: "Self-challenge before finalizing user stories"

    - plugin: superpowers
      skill: pattern-recognition
      reason: "Spot recurring requirement anti-patterns"
      optional: true                    # = policy: graceful-degrade
---
```

**Чому `requires_skills.external` — окремий список від `internal`:** агент має різну fallback-стратегію для своїх (core) скілів і чужих (superpowers). Свої завжди є; чужі можуть бути відсутні.

### 4.3. На рівні pipeline (в `pipelines/feature.yaml`)

Pipeline-фаза може теж декларувати залежність — наприклад, фаза `requirements` потребує `superpowers:thinking-deeply` через свого BA-агента, а фаза `documentation` — нічого зовнішнього. Це автоматично виводиться з `requires_skills.external` агентів, які беруть участь у фазі. **Окремої декларації на рівні pipeline не вводимо** — щоб не дублювати джерело правди.

---

## 5. Механіка перевірки

### 5.1. Layer A — Lint (CI на PR)

`tools/plugin-lint/` додає правила:

1. Валідує `requires.plugins` за схемою `schemas/plugin.v1.json`.
2. Для кожного агента в плагіні зіставляє `requires_skills.external` з `requires.plugins.<plugin>.skills` — якщо агент посилається на скіл, не задекларований у `plugin.json`, lint червоніє: «Agent X uses external skill `superpowers:foo`, але це не задекларовано в plugin.json».
3. Перевіряє, що `installCommand` непорожня для кожного `requires.plugins.<name>`.
4. Warn, якщо `policy: block` для optional-вигляду скіла (підозра на надмірно жорстку залежність).

**Виграш:** автор плагіну не закомітить агента, що мовчки посилається на superpowers.

### 5.2. Layer B — `/sdlc:doctor` (slash-команда у core)

```
> /sdlc:doctor

Checking SDLC marketplace dependencies for this project...

✓ claude-sdlc-core@1.0.0       installed
✓ stack-laravel@1.0.0          installed

External plugin dependencies (declared by stack-laravel):

✗ superpowers@^2.0.0           NOT FOUND
  Required for: ba, reviewer, ddd-architect (5 skills)
  Policy: warn (4 skills) / block (1 skill: fact-check used by reviewer)

  Run these commands in your Claude Code session to install:
    /plugin marketplace add obra/superpowers
    /plugin install superpowers@superpowers

  Then re-run /sdlc:doctor to verify.

Summary: 1 dependency missing (1 with policy=block — pipeline will halt
on Reviewer phase until installed).
```

**Реалізація:** маленький Node-скрипт у `core/lib/doctor.js`. Slash-команда `core/commands/doctor.md` запускає його через `Bash` і друкує результат. Detection — це список перевірок:

```js
// псевдокод detection
function checkPlugin({ name, version }) {
  const path = `~/.claude/plugins/${name}/.claude-plugin/plugin.json`;
  if (!fs.existsSync(path)) return { status: 'missing' };
  const installed = JSON.parse(fs.readFileSync(path));
  if (!semverSatisfies(installed.version, version)) {
    return { status: 'version-mismatch', installed: installed.version, want: version };
  }
  return { status: 'ok', installed: installed.version };
}

function checkSkill({ plugin, skill }) {
  const skillDir = `~/.claude/plugins/${plugin}/skills/${skill}/SKILL.md`;
  return fs.existsSync(skillDir) ? 'ok' : 'missing';
}
```

(Адаптувати під реальну структуру Claude Code — побутово я підтверджу шляхи в Phase 6 при імплементації, бо вони можуть еволюціонувати.)

### 5.3. Layer C — Pre-pipeline preflight (всередині `/feature`, `/bugfix`)

`core/commands/feature.md` перед першою фазою робить:

```
1. Determine which agents will run in this pipeline.
2. Collect all `requires_skills.external` from those agents.
3. Run dependency check (same as /sdlc:doctor) for that subset.
4. Apply per-skill policy:
   - any `block` missing  → halt with install instructions
   - all `warn` missing   → proceed; emit warnings; record in
                            docs/plans/{slug}/_dependencies.md
   - any `graceful-degrade` missing → proceed, but mark agent
                            to skip those skills
5. If user policy = "prompt-install" and any deps missing →
   AskUserQuestion: "Install missing plugins now? [yes/no]"
   If yes → emit install commands → ask user to paste & confirm
   continuation. If no → apply step 4.
```

**Виграш:** ми ловимо проблему **до** того, як втратили час на BA-фазу і впали на Reviewer.

### 5.4. Layer D — Per-agent runtime check (опційно)

Кожен агент при старті може виконати перевірку **для своїх** external skills (через окремий рядок у промпті: «Перед тим, як використати `superpowers:thinking-deeply`, перевір її доступність через `Bash: ls ~/.claude/plugins/superpowers/skills/thinking-deeply 2>/dev/null`. Якщо порожньо — продовжуй без неї і зазнач у звіті»).

Це більш-менш honor system, але як другий пояс. Не обов'язково для v1.0; додамо лише якщо preflight (Layer C) виявиться недостатнім.

---

## 6. Три політики на missing dependency

| Політика | Семантика | Коли використовувати |
|----------|-----------|----------------------|
| **`block`** | Pipeline зупиняється з повідомленням «це залежність обов'язкова». Чек-лист команд для install. Юзер після install запускає `/feature ...` ще раз. | Скіл, без якого якість роботи агента впаде нижче прийнятного (наприклад, security-related скіл). |
| **`warn`** | Pipeline продовжується. Агент запускається без скіла. У звіт фази записується попередження «без `X:Y` результат може бути менш якісним». | Дефолт. Більшість зовнішніх скілів — поліпшення, не блокери. |
| **`graceful-degrade`** | Pipeline продовжується, але агент **пропускає** ту частину роботи, яка явно завʼязана на скіл. У звіт записується, що саме пропустили. | Скіли, що додають окрему здатність (наприклад, «pattern-recognition» — можна обійтись без формального аналізу патернів). |

**Чому НЕ робимо політику `auto-install`:** з §2 ми вже знаємо, що це фактично неможливо без переходу до guided-flow з `prompt-install` (нижче).

---

## 7. Конфігурація на рівні проєкту

Юзер задає глобальну стратегію в `.claude-sdlc.json` робочого репо:

```jsonc
{
  "dependencies": {
    "policy": "warn",                  // "warn" | "prompt-install" | "strict"
    "overrides": {
      "superpowers": "warn",           // override per-plugin
      "superpowers:fact-check": "block"
    }
  }
}
```

| Project policy | Поведінка |
|---|---|
| `warn` (дефолт) | Per-skill `policy` з `plugin.json` застосовується as-is. Жодних запитів. |
| `prompt-install` | На кожному preflight, якщо є missing-deps, AskUserQuestion: «Бракує X. Install? [yes / skip / abort]». При yes — гайд по install. |
| `strict` | Будь-яка missing-dep трактується як `block`, навіть якщо в `plugin.json` стоїть `warn`. |

`overrides` дозволяє командам, які не хочуть superpowers в принципі, явно сказати «ми погоджуємось на warn по всіх скілах superpowers».

---

## 8. Guided install flow (для policy=`prompt-install`)

Реалістичний UX, бо ми не можемо silent-install:

```
[orchestrator running /feature "add user dashboard"]

Preflight: superpowers (^2.0.0) is required by ba, reviewer.
  Skills missing: thinking-deeply, rubber-ducking.

? Install superpowers now? Use ↑↓ then Enter.
  ▶ Yes — show install commands and pause for me to run them
    No  — proceed with warn (some skills will be skipped)
    Abort

[user picks Yes]

  Please paste the following into your Claude Code prompt
  (in the same window), then return here and type "continue":

      /plugin marketplace add obra/superpowers
      /plugin install superpowers@superpowers

  Awaiting "continue"...

[user runs slash-commands, types "continue"]

  Re-checking dependencies... ✓ superpowers@2.1.0 detected.
  Resuming /feature: phase 1 of 4 — requirements (BA)...
```

Технічно це реалізується в `feature.md` через `AskUserQuestion` + текстовий блок з командами + `AskUserQuestion("type 'continue' when done")`. Ніякої магії.

---

## 8.5. MCP-інструменти для detection і install

У §5 і §8 описана механіка через filesystem-checks і копіпаст команд. Це працює, але Claude Code має MCP-інструменти, що дають кращу UX і робастніше виявлення. Використовуємо їх там, де доступні, з FS-fallback.

### 8.5.1. `mcp__skills__list_skills` — основний detection-сигнал

Замість того, щоб лише читати `~/.claude/plugins/superpowers/skills/...`, основний сигнал «встановлено / ні» — це список скілів, які Claude Code **реально завантажив** і знає про них:

```js
// core/lib/doctor.js (псевдокод)
const skills = await callMcp('mcp__skills__list_skills', {});
const found = skills.some(s => s.name === `${plugin}:${skill}`);
```

Чому це краще за чисто-FS-probe:
- Якщо плагін встановлено, але реєстрація скілів ще не пройшла — FS-probe скаже «ОК», а агент усе одно не зможе викликати скіл. MCP-сигнал чесніший.
- Не залежимо від конкретного шляху встановлення (юзер може мати custom prefix).
- Один універсальний механізм для глобальних і проєктних плагінів.

FS-probe лишаємо як **fallback**, якщо MCP-інструмент недоступний у поточній сесії (наприклад, headless без full Claude Code env).

### 8.5.2. `mcp__plugins__search_plugins` — валідація адреси

Перед тим як показати юзеру `installCommand`, варто перевірити, що плагін реально існує в реєстрі за вказаною адресою. Це захищає від невірно скопійованих `marketplace`-полів у дочірніх плагінах:

```js
const results = await callMcp('mcp__plugins__search_plugins', { query: 'superpowers' });
const match = results.find(r => r.marketplace?.includes('obra/superpowers'));
if (!match) warn(`Declared marketplace 'obra/superpowers' not found in registry.`);
```

Цей крок — soft warning, не блокер. Лінтер у CI це **не** перевіряє (бо не має доступу до реєстру з ізольованого ранера); doctor — перевіряє при `/sdlc:doctor`.

### 8.5.3. `mcp__plugins__suggest_plugin_install` — заміна копіпасту

Замість виводу 2-х slash-команд як тексту, що юзер сам копіює, у guided-flow (policy=`prompt-install`) використовуємо MCP-інструмент:

```
[orchestrator running /feature "add user dashboard"]
Preflight: superpowers (^2.0.0) missing — required by ba, reviewer.

[виклик mcp__plugins__suggest_plugin_install
   name=superpowers, marketplace=obra/superpowers]

→ Claude Code показує юзеру нативний install-діалог
→ Юзер натискає Approve / Decline
→ Pipeline aborts з повідомленням
   "Re-run /feature after install completes."
```

Переваги над копіпастом:
- Один клік, не дві команди у вікні.
- Не треба інструктувати юзера «вставте у те саме вікно Claude Code».
- Менше шансів, що юзер встановить старішу версію вручну.

**Чому навіть тут не «справжній silent auto-install»:** після `suggest_plugin_install` Claude Code обробляє install **асинхронно**. Поточна сесія агента не знає, коли install завершиться. Тож чесно завершуємо pipeline (`abort`) і просимо повторити команду — детерміновано і надійно. Альтернатива (sleep-loop з polling `mcp__skills__list_skills`) технічно можлива, але блокує сесію і робить поведінку непередбачуваною — відмовляємось.

### 8.5.4. Decision matrix: який інструмент коли

| Точка | Primary tool | Fallback |
|---|---|---|
| `/sdlc:doctor` detection | `mcp__skills__list_skills` | FS read `~/.claude/plugins/<name>/skills/<skill>/SKILL.md` |
| Pre-pipeline preflight detection | те саме | те саме |
| Validate `marketplace` поля | `mcp__plugins__search_plugins` | Skip with note |
| Запропонувати install (policy=`prompt-install`) | `mcp__plugins__suggest_plugin_install` | Текстовий копіпаст з §8 |
| Виявити версію встановленого плагіну | (немає прямого — читаємо `plugin.json` з FS) | — |

Якщо MCP-інструменти недоступні взагалі — doctor друкує: «running in fallback mode (MCP plugin tools unavailable)».

---

## 8.6. CI / headless режим

Pipeline може запускатись не з інтерактивної сесії (наприклад, batch-обробка PR-ів через GitHub Actions). У такому режимі `AskUserQuestion` не має сенсу — нема кому відповідати.

**Активація через env:** `SDLC_NONINTERACTIVE=true`

| Ситуація | Поведінка в headless |
|---|---|
| `block`-залежність відсутня | `exit 1` з machine-readable повідомленням і списком missing skills |
| `warn`-залежність відсутня | Warning у stderr, pipeline продовжує з fallback |
| `graceful-degrade` відсутня | Тиха підстановка fallback (як в інтерактивному режимі) |
| `policy: prompt-install` спрацював би | Автоматично вибирається `skip`, поведінка як у `warn` |
| Версія мисматч | Warning, не блокує |

`/sdlc:doctor` у headless-режимі друкує JSON замість таблиці:

```bash
$ SDLC_NONINTERACTIVE=true claude /sdlc:doctor --json
{
  "checks": [
    {
      "name": "superpowers",
      "status": "missing",
      "policy": "warn",
      "missing_skills": ["thinking-deeply", "rubber-ducking"],
      "install_command": ["..."]
    }
  ],
  "summary": { "total": 1, "missing": 1, "blocked": 0 },
  "exit_code": 0
}
```

Документуємо в `docs/ci-usage.md` зі зразком GitHub Actions:

```yaml
- name: Run SDLC pipeline
  env:
    SDLC_NONINTERACTIVE: "true"
  run: claude /feature "${{ github.event.issue.title }}"
```

---

## 9. Lifecycle перевірок (звідки→коли→куди)

```
┌──────────────────────────────────────────────────────────────────┐
│ AUTHORING TIME (CI на PR у marketplace)                          │
│  → plugin-lint валідує plugin.json і requires_skills.external    │
│  → блокує merge, якщо контракт неконсистентний                   │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│ INSTALL TIME (юзер робить /plugin install our-plugin)            │
│  → /onboarding (Phase 6) автоматично запускає /sdlc:doctor       │
│  → юзер бачить, що бракує, і чи готовий встановити               │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│ ON-DEMAND (юзер хоче перевірити стан)                            │
│  → /sdlc:doctor                                                  │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│ PIPELINE START (юзер запускає /feature чи /bugfix)               │
│  → preflight перед фазою 1                                       │
│  → block | warn | graceful-degrade | prompt-install              │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│ AGENT RUNTIME (опційно, V2)                                      │
│  → агент перед використанням скіла перевіряє, чи доступний       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 10. Application до конкретного `superpowers`

**Орієнтовний мапінг наших агентів на скіли superpowers** (фінал затвердиться в Phase 5/6 за реальним експериментом):

| Наш агент | Зовнішній скіл (superpowers) | Policy | Чому |
|---|---|---|---|
| `core/agents/ba` | `thinking-deeply` | `warn` | Глибокий аналіз вимог; без нього BA працює, але слабше. |
| `core/agents/ba` | `rubber-ducking` | `graceful-degrade` | Self-challenge step — якщо немає, BA не робить цей крок. |
| `core/agents/reviewer` | `pattern-recognition` | `warn` | Помічає повторювані anti-patterns. |
| `core/agents/security-scanner` | (TBD) | `block` | Якщо superpowers має authoritative-source-validator — критично. |
| `core/agents/debugger` | `systematic-debugging` (якщо існує) | `warn` | Доповнює `core/skills/debugging-wizard`. |
| `stack-laravel/agents/ddd-architect` | `pattern-recognition` | `warn` | Patterns-aware design. |

> Список орієнтовний. Перед фіналізацією треба **перевірити фактичний інвентар скілів superpowers** (читання їхньої документації або `~/.claude/plugins/superpowers/skills/`-каталогу) і узгодити фактичні імена скілів та їхній корисний внесок. Зробимо у Phase 5 (стек-laravel migration) на реальних кейсах.

---

## 11. Чого НЕ робимо у v1.0

| Не робимо | Чому |
|---|---|
| Silent auto-install з агента. | Технічно неможливо в Claude Code, не намагатимемось. |
| Повна інтроспекція скілів superpowers (динамічне виявлення доступних). | Достатньо `requires.plugins.<name>.skills` як explicit allowlist. Динамічне — V2. |
| Власний package-resolver з графом залежностей. | Глибина 1 (наш плагін → external) вистачає. Транзитивні залежності — V3+. |
| Кешування dependency-checks між сесіями. | Файлові stat-перевірки дешеві; не варто ускладнювати кешем. |
| GUI / TUI для install. | Текстовий чек-аут UX достатньо для v1.0. |

---

## 12. Інтеграція в IMPLEMENTATION_PLAN.md (зведено)

| Phase | Що додається | Деталі |
|---|---|---|
| Phase 0 | Записати ADR-013 «External plugin dependencies». | У `docs/decisions/`. |
| Phase 1 | Розширити `core/CLAUDE.md` згадкою про залежності, додати конвенції. | Без коду — лише місце для подальшого. |
| Phase 2 | Кожен core-агент отримує поле `requires_skills.external` (поки порожнє). | Контракт з v1.0. |
| Phase 4 | Розширення `schemas/plugin.v1.json` блоком `requires.plugins`. Lint-правила R-D1..D4. | JSON Schema + правила в `tools/plugin-lint/`. |
| Phase 5 | На реальному стек-laravel заповнити `requires.plugins.superpowers` і `requires_skills.external` в агентах. Eval — як змінюється якість з/без superpowers. | Реальна перевірка. |
| Phase 6 | `/sdlc:doctor` команда; preflight у `/feature` і `/bugfix`; `/onboarding` запускає doctor; `.claude-sdlc.json` з policy. | Імплементація механіки. |
| Phase 7 | На stack-react перевірити, що механіка працює без змін у core. | Регресія. |

---

## 13. Open questions (закриваємо при імплементації)

1. **Точна схема скілового namespacing у Claude Code.** Скіл superpowers — це `superpowers:thinking-deeply` чи інший формат? Підтвердити в Phase 5 на реальному встановленні.
2. **Шлях до встановлених плагінів.** `~/.claude/plugins/<name>/` припускаємо за дефолт; може бути проектний `.claude/plugins/`. Зробити detection-функцію, що пробує обидва.
3. **Чи має `installCommand` бути одним рядком чи масивом slash-команд?** Поки масив — обмеження одного рядка надто.
4. **Обробка підверсії плагіну** (e.g. superpowers `2.0.0` → `2.1.0`). Наразі semver, але у Claude Code marketplace може бути інша конвенція. Перевірити.
5. **Як саме показати install-команди в чаті, щоб юзеру було легко скопіювати** — code block з тонами правильного синтаксису або плейн-текст?
6. **Cross-session state.** Якщо юзер у сесії А обрав «proceed without superpowers», чи памʼятати це в сесії Б? Поточне рішення — **не памʼятати** (юзер міг встановити superpowers між сесіями; зайвий silence — гірше, ніж зайве запитання). Документуємо явно: у `auto memory` decline-decisions не зберігаємо.
7. **Trust boundary для зовнішнього `marketplace`.** Якщо стек-плагін від community декларує залежність на маловідоме джерело — чи лінтер червоніє автоматично, чи лише warn? Поточно — warn з вимогою README-обґрунтування. Можливо, у V2 знадобиться allowlist довірених marketplace-ів (`obra/*`, `anthropic/*`, etc.), щоб новачок не встановив сумнівне.
8. **Поведінка, коли `mcp__plugins__suggest_plugin_install` був викликаний, але юзер задеклайнив у нативному UI.** Pipeline aborts (як зараз заплановано) — чи маємо ми дізнатися про факт декланенію, щоб перейти у warn-flow без явного повторного запиту? Залежить від того, що повертає MCP-інструмент. Перевірити в Phase 6 на реальному API.

---

## 14. Чек-лист готовності механіки до v1.0

- [ ] ADR-013 написаний.
- [ ] `schemas/plugin.v1.json` має блок `requires.plugins` з валідацією.
- [ ] Lint-правило: `requires_skills.external` агента має бути задекларованим у `plugin.json`.
- [ ] `core/lib/doctor.js` детектує установлений плагін і конкретні скіли.
- [ ] `core/commands/doctor.md` (`/sdlc:doctor`) друкує адекватний звіт.
- [ ] `core/commands/feature.md` і `bugfix.md` мають preflight-блок.
- [ ] Підтримка `.claude-sdlc.json → dependencies.policy`.
- [ ] `/onboarding` викликає doctor.
- [ ] Документація для авторів плагінів: «як декларувати external deps» — секція в `docs/authoring-guide.md`.
- [ ] Eval-кейс: BA з/без `superpowers:thinking-deeply`, виміряти якість виводу.

---

> Цей документ — джерело правди для механіки залежностей. Будь-яка зміна політик, схеми чи lifecycle — оновлюється тут і тригерить bump версії `core` (бо це частина публічного контракту marketplace).

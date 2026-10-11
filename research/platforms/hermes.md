# Hermes Agent: хост JediKit

Досье описывает, как Hermes Agent подключает и исполняет пять скиллов JediKit: `jedikit-tasks`, `jedikit-habits`, `jedikit-calendar`, `jedikit-planning` и навигатор `jedikit`. Принятые решения — [ADR 0005](../../docs/adr/0005-hermes-host-connections.md), [ADR 0017](../../docs/adr/0017-declarative-preferences.md), [ADR 0011](../../docs/adr/0011-navigator-always-loaded.md), [ADR 0003](../../docs/adr/0003-no-unattended-writes.md), [ADR 0006](../../docs/adr/0006-memory-holds-object-marks.md), [ADR 0013](../../docs/adr/0013-clarification.md) и [ADR 0014](../../docs/adr/0014-confirmations.md); здесь — их основания. Рекомендации значений предпочтений и приглашений — [досье setup](../setup.md).

## Версия и типы доказательств

Последний stable release на 2026-10-11 — **v0.21.6**, commit **`818c13be1dc4fd28987e1e881a9408224afd4535`** (annotated tag object `f27cdbce5ab63eafe8633b293a75151409da5769`). Настройки скиллов, `skills.auto_load`, cron и валидация прочитаны на этом commit 2026-10-11. [PL-H-RELEASE-0.21.6](../sources.md), [PL-H-1](../sources.md)

Остальные source facts — индекс скиллов, MCP, Tool Search, `clarify`, `memory`, прочие инструменты — прочитаны на **0.21.5 / `v2026.9.24`**, commit **`f97608f178d1ffeca59860195ab7da295f7c8e5f`**, 2026-10-05 и 2026-10-08; для v0.21.6 они не перепроверялись. Каждый тезис ссылается на ID с точным commit. [PL-H-2](../sources.md)

- **Чтение source:** parser, discovery, loader, resolver настроек, cron, validators на pin. Это чтение реализации, не runtime test; прочитанный upstream test обозначает намерение, а не выполненную проверку.
- **Vendor docs:** официальные страницы о skills, plugins, MCP, memory, cron и Nix; они могут описывать более новый `main` или расходиться с source. [PL-H-3](../sources.md), [PL-H-4](../sources.md), [PL-H-5](../sources.md), [PL-H-6](../sources.md), [PL-H-7](../sources.md), [PL-H-18](../sources.md), [PL-H-SKILL-DOCS-0.21.6](../sources.md)
- **Конфигурация владельца:** датированное чтение clanwright HEAD `6641f66`; декларация, не подтверждение deployment. [PL-H-66](../sources.md)
- **Live-наблюдение:** одно сообщение clanwright о setup `jedikit-calendar` ниже. [CLANWRIGHT-SETUP-2026-10-08](../sources.md)

Hermes на машине исследования не установлен. Установка, validators, OAuth, MCP calls, записи `memory`, задания cron и поведение модели здесь не выполнялись.

## Подключение: `skills.external_dirs`

JediKit подключается каталогом `skills/` закреплённой ревизии в `skills.external_dirs`; ревизию задаёт источник каталога (checkout, Nix input). Приоритет каталогов — `project → local → skills.create_dir → external_dirs`; изменения подхватываются в новой сессии. [PL-H-4](../sources.md)

| Поверхность | Поведение |
| --- | --- |
| Имена скиллов | Голые: `jedikit-tasks` |
| Индекс скиллов в system prompt | Имя и первые 57 символов `description` + `...` |
| Поиск и загрузка | Индекс, `skills_list`, `skill_view`, preload `hermes chat --skills '<имя>'` |
| Slash `/jedikit-tasks` | Да, scanner обходит физические каталоги |
| Защита от записи | Не даёт Hermes: обеспечивается файловой системой (у владельца — Nix store) |

Основания: индекс и усечение description — [PL-H-50](../sources.md), [PL-H-51](../sources.md); `skills_list`/`skill_view` — [PL-H-49](../sources.md); slash scanner — [PL-H-29](../sources.md); preload — [PL-H-30](../sources.md).

Reader `external_dirs` нормализует пути, проверяет существующий каталог и читает `SKILL.md`; запись в каталог для discovery и чтения деклараций настроек не нужна. Отсюда вывод: каталог из `/nix/store` пригоден, если файлы доступны процессу; реальный mount и права не проверялись. Тот же reader читает сырой профильный `config.yaml` (см. ниже). [PL-H-SKILL-CONFIG-0.21.6](../sources.md)

`external_dirs` — граница владения, а не запрет записи: `skill_manage` способен изменить внешний скилл, и upstream test прямо ожидает правку внешнего файла. Неизменяемость даёт файловая система Потребителя. Скилл с тем же именем в каталоге более высокого уровня затеняет внешний. [PL-H-SKILL-WRITE-GUARD-0.21.6](../sources.md), [PL-H-SKILLS-CLI-0.21.6](../sources.md), [PL-H-SKILL-CONFIG-0.21.6](../sources.md)

### Исследованные и отвергнутые способы установки

**Portable plugin (`hermes plugins install`).** Требует `plugin.json` в корне и полный commit SHA в `--ref`; даёт qualified names вида `agent-plugin-jedikit-805a716c:jedikit-tasks`, plugin skills не попадают в индекс, slash не гарантирован. Portable parser допускает `metadata` только как отображение строка→строка: скилл с вложенным `metadata.hermes` даёт diagnostic и пропускается discovery. Поэтому объявление `metadata.hermes.config` у навигатора несовместимо с этим способом, и JediKit его не использует ([ADR 0005](../../docs/adr/0005-hermes-host-connections.md), [ADR 0017](../../docs/adr/0017-declarative-preferences.md)). Чтение 0.21.5: [PL-H-10](../sources.md), [PL-H-8](../sources.md), [PL-H-9](../sources.md), [PL-H-12](../sources.md), [PL-H-16](../sources.md), [PL-H-17](../sources.md), [PL-H-48](../sources.md); наблюдение имени на 0.20.6 — [PL-H-35](../sources.md).

**Skills Hub.** `hermes skills install ibelyasov/jedikit/skills/<скилл>` ставит один скилл за команду без выбора ревизии, берёт default branch и создаёт изменяемую копию в `$HERMES_HOME/skills`; пять установок могут получить разные ревизии. Для закреплённой ревизии не подходит. [PL-H-63](../sources.md), [PL-H-64](../sources.md), [PL-H-11](../sources.md)

## Ресурсы скилла и соседние скиллы

`skill_view(name="<скилл>")` возвращает `SKILL.md`; файлы скилла читаются тем же инструментом: `skill_view(name="<скилл>", file_path="references/write-policy.md")`, `file_path="tools.json"`. Markdown-ссылка сама файл не загружает. `..` и выход за корень скилла запрещены, поэтому общий файл вне скилла недоступен: соседний скилл загружается по своему голому имени, например `skill_view(name="jedikit-calendar")`. У владельца файловые инструменты и terminal выключены, поэтому `skill_view` — единственный путь к ресурсам. [PL-H-49](../sources.md), [PL-H-SKILL-VIEW-0.21.6](../sources.md), [PL-H-66](../sources.md)

`skill_view` только загружает текст; выбор и исполнение сценария остаются за моделью. Markdown skill не даёт инструментов: доступны toolsets из конфигурации Hermes. [PL-H-49](../sources.md), [PL-H-4](../sources.md)

## Frontmatter

У скиллов JediKit `name` и `description`; у навигатора `jedikit` ещё `metadata.hermes.config` с объявлением ключей предпочтений. Native reader понимает вложенный `metadata.hermes`. [PL-H-SKILL-CONFIG-0.21.6](../sources.md)

Native reader терпим к ошибкам: невалидный YAML разбирается построчно как `key: value`, отсутствие fences даёт пустой frontmatter, catalog берёт имя каталога и описание из тела. Поэтому успешное появление скилла в списке не доказывает валидный YAML. [PL-H-SKILL-CONFIG-0.21.6](../sources.md), [PL-H-SKILL-VIEW-0.21.6](../sources.md)

`metadata.hermes.config` сам не включает фильтры зависимостей. `requires_toolsets`/`requires_tools` JediKit не использует: они сравниваются с eager-набором инструментов, а при включённом Tool Search отложенные MCP в него не входят, и исправный скилл может исчезнуть из индекса (вывод из source, не воспроизведён). Имена MCP toolsets — `mcp-<server>`. [PL-H-51](../sources.md), [PL-H-50](../sources.md), [PL-H-52](../sources.md), [PL-H-53](../sources.md), [PL-H-54](../sources.md)

Индекс показывает `- имя: описание`, где описание усечено до первых 57 символов и `...`; полный `SKILL.md` не загружается. Поэтому начало `description` должно различать пять скиллов. `skills_list` читает metadata из первых 4000 символов файла. Качество выбора скилла моделью не измерялось. [PL-H-50](../sources.md), [PL-H-51](../sources.md), [PL-H-49](../sources.md)

## Настройки скиллов: `skills.config`

### Объявление и путь значения

`metadata.hermes.config` — список объектов (одиночный объект превращается в список). Обязательны непустые `key` и `description`; повторный ключ пропускается, сохраняется первый; `default` переносится, если не `null`; `prompt` по умолчанию равен `description`. Типы значений не валидируются: это описание, а не JSON Schema. [PL-H-SKILL-CONFIG-0.21.6](../sources.md)

Ключ `jedikit.timezone` ищется по вложенному пути `skills.config.jedikit.timezone`; литеральный YAML-ключ с точкой этому не равен, индексация списков (`key.0`) не реализована. Список или словарь допустим как значение конечного пути. `$VAR` и `~` раскрываются у строкового значения и default, но не внутри списков. Необъявленные ключи в блок не попадают. [PL-H-SKILL-CONFIG-0.21.6](../sources.md)

```yaml
skills:
  external_dirs:
    - /nix/store/EXAMPLE-jedikit/skills
  auto_load:
    - jedikit
  config:
    jedikit:
      timezone: Europe/Moscow
      tasks:
        workdays: [mon, tue, wed, thu, fri]
```

Пример формы, не данные владельца.

### Блок значений

Подстановка — блок после текста скилла, а не замена переменных в Markdown:

```text
[Skill config (from ~/.hermes/config.yaml):
  jedikit.timezone = Europe/Moscow
  jedikit.tasks.workdays = ['mon', 'tue', 'wed', 'thu', 'fri']
  jedikit.calendar.gap_minutes = (not set)
]
```

Путь в заголовке вычисляется для профиля и может отличаться. Отступ — два пробела. Значение печатается выражением `str(value) if value else '(not set)'`: это Python `str()`, не JSON и не YAML. Отсутствие ключа, `null`, пустая строка дают `default`, иначе `''`; итоговое пустое значение и любые falsey значения — `0`, `false`, `[]`, `{}`, `''` — печатаются `(not set)`, хотя resolver их сохраняет. Поэтому «не задано» и falsey должны совпадать по смыслу. Ошибка подстановки перехватывается целиком, и сообщение остаётся без блока. [PL-H-SKILL-CONFIG-0.21.6](../sources.md), [PL-H-SKILL-INJECT-0.21.6](../sources.md)

### Откуда resolver читает

Resolver читает **сырой профильный `config.yaml`** (`get_config_path()`) через YAML reader; defaults и managed overlay к нему не подмешиваются. Managed overlay — `/etc/hermes/config.yaml` или `$HERMES_MANAGED_DIR/config.yaml` поверх пользовательского config — для `skills.config` и `skills.external_dirs` поэтому недостаточен: Потребитель должен доставить эти секции в профильный файл. `hermes config migrate` для подстановки не нужен. [PL-H-SKILL-CONFIG-0.21.6](../sources.md), [PL-H-MANAGED-SCOPE-0.21.6](../sources.md), [PL-H-CONFIG-WRITERS-0.21.6](../sources.md)

### Когда агент получает значения

| Путь загрузки | Блок значений | Основание |
| --- | --- | --- |
| Индекс system prompt, `skills_list` | Нет: имя и описание | [PL-H-AUTOLOAD-0.21.6](../sources.md), [PL-H-SKILL-VIEW-0.21.6](../sources.md) |
| `skill_view(name)` и `skill_view(name, file_path=…)` | **Нет.** В raw frontmatter видны объявления и defaults, но не значения | [PL-H-SKILL-VIEW-0.21.6](../sources.md) |
| Slash `/jedikit`, preload `hermes chat --skills` | Да, в activation message | [PL-H-SKILL-INJECT-0.21.6](../sources.md) |
| `skills.auto_load` | Да: полный скилл и блок в system context, включая Telegram | [PL-H-SKILL-INJECT-0.21.6](../sources.md), [PL-H-AUTOLOAD-0.21.6](../sources.md) |
| Cron `skills: [...]` | Да: scheduler после `skill_view` сам добавляет блок | [PL-H-CRON-SKILLS-0.21.6](../sources.md) |

Поэтому носитель предпочтений — навигатор `jedikit` в `skills.auto_load`: значения всегда в контексте, а провайдерные скиллы, загруженные через `skill_view`, читают их из этого блока ([ADR 0017](../../docs/adr/0017-declarative-preferences.md)). Auto-load не применяется без skills tools, при `skip_context_files` или `HERMES_IGNORE_RULES`, поэтому он не заменяет явный `jedikit` в `skills` задания cron. Отдельный `auto_skill` канала или темы вставляет активацию в user message новой conversation — это другой механизм. [PL-H-AUTOLOAD-0.21.6](../sources.md), [PL-H-AGENT-CACHE-0.21.6](../sources.md)

Документация того же commit говорит о подстановке «when a skill loads» и предлагает `hermes skills config <name>` для значений; source шире не подтверждает: обычный `skill_view` блока не добавляет, а `hermes skills config` — TUI включения и отключения скиллов. Значения задаются в `config.yaml` или `hermes config set skills.config.<key>`. [PL-H-SKILL-DOCS-0.21.6](../sources.md), [PL-H-CONFIG-WRITERS-0.21.6](../sources.md)

### Кэш агента и перезапуск

Блок `skills.auto_load` сохраняется на время жизни экземпляра AIAgent, в том числе при сжатии контекста. Gateway Telegram может повторно использовать кэшированный агент, а ключи сброса кэша не включают `skills.config` и `skills.auto_load`. **Вывод:** изменённые значения гарантированно появляются в новом экземпляре агента, например после перезапуска gateway; реальный lifecycle у Потребителя не проверен. [PL-H-AUTOLOAD-0.21.6](../sources.md), [PL-H-AGENT-CACHE-0.21.6](../sources.md)

### Native writers `config.yaml`

Профильный `config.yaml` — не только декларация: его переписывают `hermes config set`/`unset`, `config migrate` (интерактивно спрашивает недостающие значения скиллов) и неинтерактивный шаг миграции при bootstrap CLI и gateway для устаревшей schema (`_config_version: 50` на теге) с backup; `/sethome` пишет `platforms.<platform>.home_channel`; переключатели `/memory`, `/skills`, `/tool-progress`, `/footer`, смена модели и CLI enable/disable скиллов тоже пишут config. Полный inventory writers не устанавливался. Atomic writer разрешает symlink и пишет в реальную цель; неперезаписываемая цель даёт ошибку записи. `HERMES_MANAGED` — грубая защита: `save_config` и `config set/unset` отказываются писать, но это не доказательство отсутствия всех побочных записей. [PL-H-CONFIG-WRITERS-0.21.6](../sources.md)

**Вывод для Потребителя:** рендерить профильный YAML декларативно и сознательно отказаться от изменений из чата и CLI; read-only файл совместим с чтением настроек, но не со штатной работой команд сохранения и миграций. Проверка Nix и service lifecycle — у Потребителя.

### Другие каналы

`SOUL.md`, файлы контекста проекта (`.hermes.md`, `AGENTS.md`), seed `memory` и env технически способны доставить текст, но смешивают настройки с характером Хоста, зависят от рабочего каталога (в cron без `workdir` контекст проекта выключен) или делают изменяемое состояние источником декларации; произвольный env сам в контекст не попадает. Эти варианты отвергнуты в [ADR 0017](../../docs/adr/0017-declarative-preferences.md). [PL-H-CONTEXT-CHANNELS-0.21.6](../sources.md)

## MCP-инструменты и Tool Search

Чтение 0.21.5. Callable name — `mcp__<server>__<tool>`, где любой символ вне `[A-Za-z0-9_]`, включая `-`, заменён на `_`; длинное имя ограничивается 64 символами с hash suffix. Например, `get-event` на `google_calendar` — `mcp__google_calendar__get_event`. Базовые имена в `tools.json` менять не нужно. [PL-H-46](../sources.md)

У сервера с `trust=untrusted` Hermes сам запрашивает approval на вызов инструмента без `readOnlyHint=true`; это может затронуть и чтения без annotation. Host approval не заменяет согласие по правилам JediKit, и наоборот. [PL-H-47](../sources.md)

При включённом Tool Search schemas MCP отложены: отсутствие инструмента в видимом списке не означает его отсутствия. Путь — `tool_search(queries=[...])` → `tool_describe(names=[...])` → `tool_call(calls=[{name, arguments}])`; schemas можно получить группой, но для MCP в `calls` допускается один элемент. Текущая web-страница описывает несколько local calls иначе; pin однозначен. Tool Search сохраняет approvals исходного инструмента. [PL-H-54](../sources.md)

Пакет JediKit MCP не объявляет; подключения, trust и OAuth — native конфигурация Hermes у Потребителя. Native конфигурация поддерживает `auth: oauth`; effective config и рабочая OAuth-сессия — разные утверждения. [PL-H-14](../sources.md), [PL-H-18](../sources.md)

Фильтр `tools.include`/`tools.exclude` (срез 0.20.6, 2026-09-13): include — allowlist имён и globs с приоритетом над exclude; пустой `include: []` не регистрирует инструменты. Denylist или широкий glob могут пропустить будущий write-tool; фильтр хоста не отзывает OAuth scope сервера. [PL-H-36](../sources.md)

## `clarify`

Чтение 0.21.5. Схема: `questions:[{question, choices?, multi_select?}]`, 1–5 вопросов, до 4 вариантов; первый вариант помечается рекомендованным, UI добавляет «Other». Без `choices` ответ свободный. Ответ — `responses[]`; timeout, skip и недоставка не являются ответом. Timeout по умолчанию 3600 с. В Telegram пакет вопросов приходит отдельными последовательными карточками, полноценного multi-select нет. [PL-H-55](../sources.md)

Отсюда канал JediKit: все блокирующие пробелы — один вопрос без `choices` со всем списком, чтобы сохранить «одним сообщением» ADR 0013; одиночное предложение, общий Preview и подтверждение безвозвратной потери Habitify — одна карточка «Да»/«Нет» после полного текста. Порядок вариантов: «Да», «Нет»; для подтверждения потерь Habitify — «Нет», «Да», чтобы рекомендованным не стало удаление. Это продуктовая интерпретация, не security approval. В cron `clarify` недоступен. [PL-H-55](../sources.md), [PL-H-66](../sources.md)

## `memory`

Чтение 0.21.5. Схема: `target: memory | user`, `action: add | replace | remove`, `content`, `old_text` для replace/remove; есть атомарный `operations` одного target. `replace` меняет всю запись, `old_text` лишь находит её. Отдельного чтения нет: записи видны в контексте памяти. Лимиты — 2200 символов на `memory` и 1375 на `user`, с разделителями. [PL-H-31](../sources.md), [PL-H-MEMORY-0.21.5](../sources.md)

При `memory.write_approval=true` foreground пытается получить approval сразу, без callback и в фоне запись ставится в очередь: `staged:true`, `pending_id`, `/memory pending`, `/memory approve <id>`. Применённая запись возвращает `done:true` с `usage` и `entry_count`, без содержимого. `done:true` — ответ инструмента о применении, не независимое чтение. [PL-H-56](../sources.md), [PL-H-31](../sources.md)

Описание `memory` в 0.21.5 советует хранить предпочтения «для вида работы» в скилле через `skill_manage`. Датированное наблюдение 2026-10-08: на хосте владельца setup `jedikit-calendar` по этой подсказке создал ожидающие изменения файлов скилла вместо записи в память, хотя скиллы подключены из `/nix/store`. В JediKit `memory` хранит только пометки объектов провайдеров, предпочтения задаёт Потребитель в `skills.config`, и скиллы не пишут настройки ни через `memory`, ни через `skill_manage` ([ADR 0006](../../docs/adr/0006-memory-holds-object-marks.md), [ADR 0017](../../docs/adr/0017-declarative-preferences.md)). В Telegram `/skills` управляет ожидающими записями скиллов, а не каталогом. [PL-H-MEMORY-0.21.5](../sources.md), [CLANWRIGHT-SETUP-2026-10-08](../sources.md), [PL-H-65](../sources.md)

## Cron

Приглашения JediKit — задания cron, которые создаёт и сопровождает Потребитель; скиллы задания не создают и не меняют ([ADR 0017](../../docs/adr/0017-declarative-preferences.md), [ADR 0003](../../docs/adr/0003-no-unattended-writes.md)). Синтаксис расписания для приглашений (`#`, «или» дня месяца и дня недели, `[SILENT]`) разобран в [досье setup, §6](../setup.md#6-cron-hermes-v0216).

### Управление и хранилище

В `config.yaml` есть параметры scheduler, но не список заданий. CLI `hermes cron` умеет `list`, `create`/`add`, `edit`, `pause`, `resume`, `run`, `remove` и диагностику; `import`, `sync`, `upsert` или файла-манифеста нет. CLI create вызывает тот же API, что модельный инструмент `cronjob_manage`. `create_job` генерирует случайный ID (`uuid4().hex[:12]`) и не принимает свой; `name` не уникален; `edit` ищет точный ID, затем точное имя без учёта регистра, а несколько одноимённых заданий дают ошибку. **Вывод:** первоначальный create и edit по уникальному имени возможны штатно, повторный create не идемпотентен, а автоматическая сверка требует логики Потребителя. [PL-H-CRON-CLI-0.21.6](../sources.md), [PL-H-CRON-STORE-0.21.6](../sources.md)

Blueprint skills (`metadata.hermes.blueprint`), каталог Automation Blueprint и `/blueprint` создают обычные задания по предложению, а не синхронизируют расписание; автоматическое обнаружение blueprint из `external_dirs` не доказано. [PL-H-CRON-BLUEPRINT-0.21.6](../sources.md)

`<profile-home>/cron/jobs.json` — **изменяемое хранилище scheduler**: `{"jobs": [...], "updated_at": ...}`, `schedule` внутри задания — нормализованный объект (`{"kind":"cron","expr":…,"display":…}`), рядом runtime-поля: claims, `next_run_at`, `last_run_at`, счётчики, pause state. Save переписывает файл целиком через temp и atomic replace под lock; слияние при сохранении не объединяет поля записей с одинаковым ID. Внешняя перезапись может потерять состояние и вызвать повторный или пропущенный запуск; read-only файл останавливает dispatch. Отдельный SQLite-журнал `cron/executions.db` хранит попытки, не определения. Публичной versioned schema у `jobs.json` нет. [PL-H-CRON-STORE-0.21.6](../sources.md)

### Скиллы задания

`skills` задания — голые имена, например `[jedikit, jedikit-tasks]`. Scheduler загружает их через `skill_view` и сам добавляет блок значений `skills.config`; отсутствующий или неоднозначный скилл пропускается с notice, поэтому успешное задание не доказывает загрузку всех скиллов. Вызов соседнего `skill_view` внутри задания блока не добавляет, поэтому задание, которому нужны предпочтения, прикрепляет `jedikit` явно. [PL-H-CRON-SKILLS-0.21.6](../sources.md), [PL-H-SKILL-VIEW-0.21.6](../sources.md)

### Доставка

| Механизм | Смысл | Основание |
| --- | --- | --- |
| `deliver: telegram:<chat_id>:<thread_id>` | Явная цель: signed numeric chat и numeric thread через двоеточие, без `topic=` и URL. Chat без thread не наследует home thread | [PL-H-CRON-DELIVERY-0.21.6](../sources.md) |
| `origin`, `deliver: origin` | Чат и тема сессии создания; без origin — первый подходящий home, иначе доставка не гарантирована. CLI create вне чата origin обычно не имеет | [PL-H-CRON-DELIVERY-0.21.6](../sources.md) |
| `TELEGRAM_CRON_THREAD_ID` | Тема только для выбора **home**-цели; явную цель и origin не меняет | [PL-H-CRON-DELIVERY-0.21.6](../sources.md) |
| `failure_deliver` | Та же грамматика; для уведомлений о сбоях **заменяет** `deliver`; `local` подавляет внешние уведомления | [PL-H-CRON-DELIVERY-0.21.6](../sources.md) |
| `attach_to_session` | Зеркалирование в продолжаемую сессию; к адресации темы не относится | [PL-H-CRON-DELIVERY-0.21.6](../sources.md) |

Для декларативного подключения подходит явный `deliver` с chat и thread; тема для сбоев, если нужна другая, задаётся `failure_deliver`. Права бота и фактическая доставка в супергруппу не проверялись.

### Timezone

Своего timezone у задания нет: recurring cron считает wall clock по `HERMES_TIMEZONE`, затем по `timezone` effective config профиля, затем по поясу сервера; при multiplex env-пояс подавляется ради изоляции профилей, неверное имя IANA даёт warning и fallback. Пояс кэшируется по identity config без mtime: после смены нужен перезапуск scheduler или gateway; пересчёт сохранённого `next_run_at` не проверен. Timezone читается из effective config, а `skills.config` — из сырого профильного файла, поэтому относительно managed overlay они ведут себя по-разному. [PL-H-CRON-TIMEZONE-0.21.6](../sources.md), [PL-H-SKILL-CONFIG-0.21.6](../sources.md)

### Toolsets и исполнение

Чтение 0.21.5: per-job `enabled_toolsets` имеет приоритет над `platform_toolsets.cron`, а без явного выбора к заданию могут добавиться включённые MCP. Поэтому toolsets cron по умолчанию не доказывают read-only любого задания; фоновое чтение без записи обеспечивает текст скиллов по ADR 0003, а выбор toolsets — Потребитель. [PL-H-58](../sources.md), [PL-H-57](../sources.md)

Срез 0.20.6 (2026-09-13): ISO без offset трактуется в настроенной зоне, отдельный DST-переход мог пропустить срабатывание; preflight проверял auth, настройку skill и доставку, но не гарантировал готовность; попытка проходила claimed → running → completed/failed/unknown в `executions.db` без автоматического retry. Задание, попытка, ответ модели и доставка — разные наблюдения. Общий денежный или токенный лимит не найден; `approvals.cron_mode: deny` ограничивает опасные команды, не расходы. [PL-H-38](../sources.md), [PL-H-39](../sources.md), [PL-H-40](../sources.md), [PL-H-41](../sources.md)

## Другие инструменты

Чтение 0.21.5.

- `todo_list`: список `{id, content, status}` до 256 элементов, вызов без аргументов читает. Подходит для шагов агента в длинном обзоре, не для задач пользователя. [PL-H-59](../sources.md)
- `session_search`: поиск и чтение прошлых сессий с ограниченным окном. Найденное — контекст разговора, а не актуальное состояние провайдера. [PL-H-62](../sources.md)
- `send_message` модели не зарегистрирован; фоновые уведомления — доставка cron. [PL-H-60](../sources.md)
- `delegate_task` у владельца выключен, а дочерним агентам недоступны `clarify`, `memory` и `cronjob`; соседний скилл JediKit загружается в той же сессии. [PL-H-61](../sources.md), [PL-H-66](../sources.md)

Local terminal (срез 0.20.6) работает от пользователя ОС и не является sandbox; фильтрация окружения terminal, `execute_code` и MCP различается. JediKit не требует terminal. [PL-H-37](../sources.md), [PL-H-39](../sources.md)

## Валидация

`hermes plugins validate` ищет `plugin.yaml`, `plugin.yml` или `plugin.json` и без manifest завершается ошибкой **до** security scan; `hermes plugins doctor --ci` тоже требует manifest и отдаёт exit `1`; официальный Action `plugin-validate` вызывает тот же validator. Для каталога скиллов в `external_dirs` эти проверки неприменимы. [PL-H-PLUGIN-VALIDATE-0.21.6](../sources.md)

Штатной CLI-замены для каталога скиллов в v0.21.6 нет:

| Команда | Что проверяет | Почему не заменяет validator |
| --- | --- | --- |
| `hermes skills list --source local` | Native catalog, включая external tier | Inventory, не frontmatter и не security; нет `--json`, `--ci`, аргумента каталога; создаёт каталоги Hub — только в изолированном home |
| `hermes skills audit [name] [--deep]` | Security scan записей Hub lock | Не обходит `external_dirs`; findings не меняют exit |
| `hermes skills check`, `inspect`, `snapshot export` | Обновления Hub, preview из registry, экспорт Hub lock | Не проверяют локальный каталог |

`tools.skills_guard.scan_skill` и `tools.skill_linter` — внутренние Python API без CLI entrypoint; строгая проверка frontmatter принадлежит пути записи `skill_manage`. Вызов этих функций собственным wrapper противоречит [ADR 0004](../../docs/adr/0004-no-custom-code.md). Native reader терпим к невалидному YAML, поэтому даже успешный `skills list` не доказывает валидный frontmatter. [PL-H-SKILLS-CLI-0.21.6](../sources.md), [PL-H-SKILL-CONFIG-0.21.6](../sources.md)

Следствие ([ADR 0017](../../docs/adr/0017-declarative-preferences.md)): CI JediKit проверяет `git diff --check` (пробелы во всём дереве HEAD), остальное — независимое ревью; валидность frontmatter, security scan и discovery до появления штатной проверки Hermes автоматически не проверяются.

## Конфигурация владельца

Датированное наблюдение: clanwright HEAD `6641f66d74a1877f4c28f98bf9589af648a85d4c`, прочитан 2026-10-08; JediKit там закреплён на `v0.4.0` (`647b0d1`). Это декларация до перехода на `skills.config`, не проверка deployment. [PL-H-66](../sources.md)

- JediKit подключён через `skills.external_dirs` из Nix store; plugin registry не используется. Hermes закреплён на `f97608f`. Nix-модули Hermes официально имеют уровень поддержки Tier 2 best effort. [PL-H-7](../sources.md)
- MCP: `singularity` — официальный hosted MCP с OAuth и `trust=untrusted`; `habitify_read`, `habitify`, `google_calendar` — локальные адаптеры с `trust=full`. Sampling, elicitation, resources и prompts выключены. Включены 35 инструментов Singularity, 7 чтений и 16 записей Habitify; allowlist Calendar берётся из `tools.json` JediKit.
- Toolsets Telegram: `clarify`, `todo`, `memory`, `session_search`, `web`, `search`, `skills`, `mcp-singularity`, `mcp-habitify`, `mcp-habitify_read`, `mcp-google_calendar`, с cron — `cronjob`. Tool Search включён.
- Фоновые toolsets: `web`, `skills`, `mcp-habitify_read`, `memory` — без Singularity и Calendar, поэтому фоновая утренняя сводка задач и календаря по ним невозможна. Задания и их toolsets заводит Потребитель; скиллы JediKit их не создают.
- Cron включён, timezone `Europe/Moscow`; `memory.write_approval=true`, `skills.write_approval=true`, `approvals.mode=manual`. Terminal, file, code execution, delegation, browser выключены.
- `grill-me` поставляется из официальных optional skills закреплённого Hermes; фактический discovery не проверен.

## Ограничения доступа gateway

Срез 0.20.6: доступ к боту требует pairing, allowlist, явного allow-all или trusted adapter; pairing grants и allowlists объединяются. Admin/user slash policy ограничивает slash-команды, не обычный чат. Доступ к боту, разрешение инструмента и согласие изменить данные — разные границы. [PL-H-42](../sources.md), [PL-H-44](../sources.md), [PL-H-45](../sources.md)

## Непроверенные слои

Не проверены: установленная версия Hermes у Потребителя, подключение закреплённой ревизии через `external_dirs`, фактические имена скиллов, индекс и slash, появление блока `skills.config` в Telegram и cron, свежесть значений после перезапуска, поведение native writers с декларативным `config.yaml`, MCP discovery и OAuth, вызовы Tool Search, карточки `clarify` в Telegram, записи и approval `memory`, задания cron, timezone, запуск и доставка в тему, поведение модели и Read-back. Source и статические проверки эти результаты не подтверждают. Аккаунты и данные провайдеров этой работой не затрагивались.

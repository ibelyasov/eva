# Один каталог скиллов для Hermes Agent, подключения настраиваются в Hermes

JediKit работает на одном хосте — Hermes Agent. `skills/` в корне — единственный источник пяти скиллов. Генерации, копий и manifests нет.

Hermes подключает `skills/` закреплённой ревизии каталогом в `skills.external_dirs`. Скиллы обычные: голые имена (`jedikit-tasks`), имя и начало `description` в индексе скиллов system prompt, slash `/jedikit-tasks`. Соседний скилл JediKit загружается голым именем: `skill_view(name="jedikit-calendar")`. Навигатор `jedikit` Потребитель включает в `skills.auto_load` ([ADR 0011](0011-navigator-always-loaded.md)).

Frontmatter скилла содержит только `name` и `description`. У навигатора `jedikit` есть ещё `metadata.hermes.config` — объявление ключей предпочтений, значения которых Потребитель задаёт в `skills.config` ([ADR 0017](0017-declarative-preferences.md)). Других расширений `metadata.hermes` нет.

Провайдерные скиллы объявляют ожидания в [`skills/jedikit-tasks/tools.json`](../../skills/jedikit-tasks/tools.json), [`skills/jedikit-habits/tools.json`](../../skills/jedikit-habits/tools.json) и [`skills/jedikit-calendar/tools.json`](../../skills/jedikit-calendar/tools.json): сервер у каждой операции, базовое имя без префикса, `read`/`write`; у Habitify и Google Calendar ещё `required`. Чтения Habitify объявлены на сервере `habitify_read`, записи — на `habitify`: это отдельные экземпляры с разным уровнем доверия. Google Calendar объявлен на сервере `google_calendar`. Агент вызывает их как `mcp__<server>__<base_name>`. Это декларативный внешний контракт, задаваемый JediKit; изменение имён выполняется вместе со скиллом и отмечается в release notes. У `jedikit-planning` и навигатора `jedikit` своих провайдеров и `tools.json` нет.

MCP-серверы настраиваются в Hermes. SingularityApp — официальный hosted MCP на сервере `singularity`, OAuth проводит Hermes. Habitify предоставляет доверенный адаптер по REST/OpenAPI v2 с ключом только у адаптера ([ADR 0001](0001-habitify-via-rest.md)). Google Calendar — сервер `google_calendar` ([ADR 0009](0009-separate-calendar-skill.md)). У владельца серверы, toolsets, `skills.config` и cron настраивает Потребитель. Для глубокого уточнения намерения нужен внешний скилл `grill-me` ([ADR 0013](0013-clarification.md)); JediKit не реализует собственное интервью.

## Considered Options

- Portable `plugin.json` и `hermes plugins install`: portable loader отбрасывает скилл с вложенным `metadata`, поэтому несовместим с `metadata.hermes.config` навигатора; скиллы получают имена `agent-plugin-…:<скилл>` и не попадают в индекс system prompt.
- Native `plugin.yaml`: требует Python-модуля с `register()`, то есть кода ([ADR 0004](0004-no-custom-code.md)), и не подключает MCP или toolsets.
- Только Skills Hub (`hermes skills install`): ставит скиллы по одному с default branch без выбора ревизии, копия изменяема.
- Другие поля `metadata.hermes` (`tags`, `related_skills`, `requires_*`): `requires_*` при включённом Tool Search может скрыть исправный скилл из индекса, остальные не нужны сценариям.

## Consequences

Подключение JediKit не подключает провайдеров. Если нужных инструментов нет, скилл называет недостающие базовые имена из своего `tools.json` и рекомендует подключить их в Hermes; сам ничего не устанавливает и не настраивает. После каждой записи — Read-back. Штатной проверки frontmatter, security scan и discovery для каталога `skills.external_dirs` у Hermes нет: frontmatter проверяет ревью ([ADR 0004](0004-no-custom-code.md)). Владелец пользуется скиллами и сообщает о проблемах, обязательного ручного runtime-гейта нет. Публикация версии (тег и GitHub release) остаётся отдельным решением владельца. Основания — [досье Hermes](../../research/platforms/hermes.md).

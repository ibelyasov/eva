# Hermes Agent: хост Eva

Досье описывает, как Hermes Agent подключает и исполняет шесть скиллов Eva: `eva-tasks`, `eva-habits`, `eva-calendar`, `eva-planning`, `eva-notes` и навигатор `eva`. Принятые решения — [ADR 0005](../../docs/adr/0005-hermes-host-connections.md), [ADR 0018](../../docs/adr/0018-notes-over-llm-wiki.md), [ADR 0017](../../docs/adr/0017-declarative-preferences.md), [ADR 0011](../../docs/adr/0011-navigator-always-loaded.md), [ADR 0003](../../docs/adr/0003-no-unattended-writes.md), [ADR 0006](../../docs/adr/0006-state-in-eva-journal.md), [ADR 0013](../../docs/adr/0013-clarification.md), [ADR 0014](../../docs/adr/0014-confirmations.md), [ADR 0022](../../docs/adr/0022-eva-journal-and-initiative.md) и [ADR 0024](../../docs/adr/0024-portable-notes-skill.md); здесь — их основания. Как `eva-notes` работает в других клиентах Agent Skills — [досье клиентов Agent Skills](agent-skills.md). Рекомендации значений предпочтений — [досье setup](../setup.md); механизмы Инициативы, Ритма и Возвратов в сравнении с наукой и практикой — [досье проактивности](../agents/proactivity.md).

## Версия и типы доказательств

Последний stable release на 2026-10-11 — **v0.21.6**, commit **`818c13be1dc4fd28987e1e881a9408224afd4535`** (annotated tag object `f27cdbce5ab63eafe8633b293a75151409da5769`). Настройки скиллов, `skills.auto_load`, cron, валидация, встроенные `llm-wiki` и `obsidian`, toolsets встроенных инструментов, `web_extract`, монтирование кэша в песочницу и кэш вложений gateway прочитаны на этом commit 2026-10-11; проектные скиллы, context files и ответ `skill_view` для файлов скилла — на нём же 2026-10-13. [PL-H-RELEASE-0.21.6](../sources.md), [PL-H-1](../sources.md)

Остальные source facts — индекс скиллов, MCP, Tool Search, `clarify`, `memory`, прочие инструменты — прочитаны на **0.21.5 / `v2026.9.24`**, commit **`f97608f178d1ffeca59860195ab7da295f7c8e5f`**, 2026-10-05 и 2026-10-08; для v0.21.6 они не перепроверялись. Каждый тезис ссылается на ID с точным commit. [PL-H-2](../sources.md)

- **Чтение source:** parser, discovery, loader, resolver настроек, cron, validators на pin. Это чтение реализации, не runtime test; прочитанный upstream test обозначает намерение, а не выполненную проверку.
- **Vendor docs:** официальные страницы о skills, plugins, MCP, memory, cron и Nix; они могут описывать более новый `main` или расходиться с source. [PL-H-3](../sources.md), [PL-H-4](../sources.md), [PL-H-5](../sources.md), [PL-H-6](../sources.md), [PL-H-7](../sources.md), [PL-H-18](../sources.md), [PL-H-SKILL-DOCS-0.21.6](../sources.md)
- **Конфигурация владельца:** датированное чтение clanwright HEAD `6641f66` и handoff `eva-notes` от 2026-10-11; декларации, не подтверждение deployment. [PL-H-66](../sources.md), [CLANWRIGHT-NOTES-HANDOFF-2026-10-11](../sources.md)
- **Live-наблюдение:** одно сообщение clanwright о setup `eva-calendar` ниже. [CLANWRIGHT-SETUP-2026-10-08](../sources.md)

Hermes на машине исследования не установлен. Установка, validators, OAuth, MCP calls, записи `memory`, задания cron и поведение модели здесь не выполнялись.

## Подключение: `skills.external_dirs`

Eva подключается каталогом `skills/` закреплённой ревизии в `skills.external_dirs`; ревизию задаёт источник каталога (checkout, Nix input). Приоритет каталогов — `project → local → skills.create_dir → external_dirs`; изменения подхватываются в новой сессии. [PL-H-4](../sources.md)

| Поверхность | Поведение |
| --- | --- |
| Имена скиллов | Голые: `eva-tasks` |
| Индекс скиллов в system prompt | Имя и первые 57 символов `description` + `...` |
| Поиск и загрузка | Индекс, `skills_list`, `skill_view`, preload `hermes chat --skills '<имя>'` |
| Slash `/eva-tasks` | Да, scanner обходит физические каталоги |
| Защита от записи | Не даёт Hermes: обеспечивается файловой системой (у владельца — Nix store) |

Основания: индекс и усечение description — [PL-H-50](../sources.md), [PL-H-51](../sources.md); `skills_list`/`skill_view` — [PL-H-49](../sources.md); slash scanner — [PL-H-29](../sources.md); preload — [PL-H-30](../sources.md).

Reader `external_dirs` нормализует пути, проверяет существующий каталог и читает `SKILL.md`; запись в каталог для discovery и чтения деклараций настроек не нужна. Отсюда вывод: каталог из `/nix/store` пригоден, если файлы доступны процессу; реальный mount и права не проверялись. Тот же reader читает сырой профильный `config.yaml` (см. ниже). [PL-H-SKILL-CONFIG-0.21.6](../sources.md)

`external_dirs` — граница владения, а не запрет записи: `skill_manage` способен изменить внешний скилл, и upstream test прямо ожидает правку внешнего файла. Неизменяемость даёт файловая система Потребителя. Скилл с тем же именем в каталоге более высокого уровня затеняет внешний. [PL-H-SKILL-WRITE-GUARD-0.21.6](../sources.md), [PL-H-SKILLS-CLI-0.21.6](../sources.md), [PL-H-SKILL-CONFIG-0.21.6](../sources.md)

### Проектные скиллы и Хранилище

**Чтение source, 2026-10-13.** Уровни скиллов по приоритету: доверенный проект > локальный `~/.hermes/skills` (туда копируются встроенные скиллы) > `skills.create_dir` > `skills.external_dirs`; порядок общий для списка, индекса, `skill_view`, preload и cron ([skill_utils.py L381–403](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/agent/skill_utils.py#L381-L403)). Проектные каталоги — только `<root>/.hermes/skills` и `<root>/.agents/skills`; `.claude/skills` не ищется. `<root>` — ближайший предок рабочего каталога с `.git`, домашний каталог проектом не считается ([L497–532](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/agent/skill_utils.py#L497-L532)). Проектные скиллы загружаются, только если корень есть в `skills.trusted_project_dirs` (CLI `hermes skills trust`); по умолчанию список пуст, и каждый проектный `SKILL.md` проходит `skills_guard`: вердикт `dangerous` или сбой сканера — карантин ([L535–644](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/agent/skill_utils.py#L535-L644), [config_defaults.py L1449–1459](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/hermes_cli/config_defaults.py#L1449-L1459)). В cron проектный корень берётся из `workdir` задания. `external_dirs` обходится с `followlinks=True`, скрытые каталоги, кроме `.git`, `.github`, `.hub`, `.archive`, `node_modules` и подобных, тоже обходятся ([L23–27, L842–853](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/agent/skill_utils.py#L842-L853)). [PL-H-PROJECT-SKILLS-0.21.6](../sources.md)

Одноимённые скиллы: на разных уровнях побеждает более высокий, нижний скрыт как `shadowed` с warning в логе; на одном уровне идентичные копии одного корня сводятся к менее глубокому пути, а разные корни или разное содержимое дают отказ `skill_view` «Ambiguous skill name». `.hermes/skills` и `.agents/skills` одного проекта — один уровень. `skills.disabled` выключает все копии по имени ([skill_utils.py L431–489](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/agent/skill_utils.py#L431-L489), [skills_tool.py L463–505](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/tools/skills_tool.py#L463-L505)). [PL-H-PROJECT-SKILLS-0.21.6](../sources.md)

Вывод: если Хранилище с `.git` сделать доверенным проектом, копия `eva-notes` из `<Хранилище>/.agents/skills` перекроет закреплённую ревизию из `external_dirs`. Поэтому Хранилище в Hermes доверенным проектом не делается, а Отдельная установка для другого клиента в том же Хранилище Hermes без доверия не загружает. Копии основ Отдельной установки лежат в `<Хранилище>/.agents/eva-notes-bases/` — вне `.hermes/skills` и `.agents/skills`, поэтому Hermes не находит их как скиллы и при доверии, а встроенные `llm-wiki` и `obsidian` они не перекрывают ([ADR 0024](../../docs/adr/0024-portable-notes-skill.md)). Карантин `skills_guard` для этих текстов не проверялся.

### Исследованные и отвергнутые способы установки

**Portable plugin (`hermes plugins install`).** Требует `plugin.json` в корне и полный commit SHA в `--ref`; даёт qualified names вида `agent-plugin-eva-805a716c:eva-tasks`, plugin skills не попадают в индекс, slash не гарантирован. Portable parser допускает `metadata` только как отображение строка→строка: скилл с вложенным `metadata.hermes` даёт diagnostic и пропускается discovery. Поэтому объявление `metadata.hermes.config` у навигатора несовместимо с этим способом, и Eva его не использует ([ADR 0005](../../docs/adr/0005-hermes-host-connections.md), [ADR 0017](../../docs/adr/0017-declarative-preferences.md)). Чтение 0.21.5: [PL-H-10](../sources.md), [PL-H-8](../sources.md), [PL-H-9](../sources.md), [PL-H-12](../sources.md), [PL-H-16](../sources.md), [PL-H-17](../sources.md), [PL-H-48](../sources.md); наблюдение имени на 0.20.6 — [PL-H-35](../sources.md).

**Skills Hub.** `hermes skills install ibelyasov/eva/skills/<скилл>` ставит один скилл за команду без выбора ревизии, берёт default branch и создаёт изменяемую копию в `$HERMES_HOME/skills`; пять установок могут получить разные ревизии. Для закреплённой ревизии не подходит. [PL-H-63](../sources.md), [PL-H-64](../sources.md), [PL-H-11](../sources.md)

## Ресурсы скилла и соседние скиллы

`skill_view(name="<скилл>")` возвращает `SKILL.md`; файлы скилла читаются тем же инструментом: `skill_view(name="<скилл>", file_path="references/write-policy.md")`, `file_path="tools.json"`. Markdown-ссылка сама файл не загружает. `..` и выход за корень скилла запрещены, поэтому общий файл вне скилла недоступен: соседний скилл загружается по своему голому имени, например `skill_view(name="eva-calendar")`. Пять скиллов Eva, кроме `eva-notes`, читают свои ресурсы только через `skill_view`. [PL-H-49](../sources.md), [PL-H-SKILL-VIEW-0.21.6](../sources.md)

**Чтение source, 2026-10-13: host-нейтральная ссылка на файл.** Ответ `skill_view(name)` содержит `skill_dir` — абсолютный путь каталога скилла на Хосте, `linked_files` (`references/*.md` без рекурсии, `templates/`, `assets/`, `scripts/`) и `usage_hint` с подсказкой повторного вызова с `file_path`; `tools.json` в `linked_files` не входит, но `file_path="tools.json"` его отдаёт ([skills_tool.py L386–404, L635–650](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/tools/skills_tool.py#L635-L650), [skills_tool_plugin.py L68–102](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/tools/skills_tool_plugin.py#L68-L102)). При preload и `skills.auto_load` сообщение содержит `[Skill directory: <abs>]` и просьбу разрешать относительные пути от него ([skill_commands.py L299–303, L345–365](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/agent/skill_commands.py#L345-L365)). В Docker каталоги скиллов смонтированы только на чтение под `/root/.hermes/…`, но модели сообщается хостовый путь; `read_file` переводит хостовый путь в путь контейнера только для каталога, смонтированного как рабочий, а относительный путь разрешает от рабочего каталога задачи ([credential_files.py L165–206](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/tools/credential_files.py#L165-L206), [file_operations.py L503–512](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/tools/file_operations.py#L503-L512), [file_tools_paths.py L267–274](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/tools/file_tools_paths.py#L267-L274)). [PL-H-SKILL-DIR-0.21.6](../sources.md)

Вывод: ожидается, что фраза `eva-notes` «файлы скилла — из каталога этого скилла, тем же инструментом, которым загружен скилл, с путём файла» в Hermes ведёт к `skill_view(name, file_path)`, который читает файл на Хосте при любом backend; запасной путь фразы — чтение по абсолютному пути каталога — в Docker по хостовому пути не работает. Голый относительный путь `read_file` разрешил бы от корня Хранилища, поэтому текст называет каталог скилла явно ([ADR 0024](../../docs/adr/0024-portable-notes-skill.md)). Какой путь выберет модель, не проверено.

`skill_view` только загружает текст; выбор и исполнение сценария остаются за моделью. Markdown skill не даёт инструментов: доступны toolsets из конфигурации Hermes. [PL-H-49](../sources.md), [PL-H-4](../sources.md)

## Frontmatter

У скиллов Eva `name` и `description`; у навигатора `eva` ещё `metadata.hermes.config` с объявлением ключей предпочтений. Native reader понимает вложенный `metadata.hermes`. [PL-H-SKILL-CONFIG-0.21.6](../sources.md)

Native reader терпим к ошибкам: невалидный YAML разбирается построчно как `key: value`, отсутствие fences даёт пустой frontmatter, catalog берёт имя каталога и описание из тела. Поэтому успешное появление скилла в списке не доказывает валидный YAML. [PL-H-SKILL-CONFIG-0.21.6](../sources.md), [PL-H-SKILL-VIEW-0.21.6](../sources.md)

`metadata.hermes.config` сам не включает фильтры зависимостей. `requires_toolsets`/`requires_tools` Eva не использует: они сравниваются с eager-набором инструментов, а при включённом Tool Search отложенные MCP в него не входят, и исправный скилл может исчезнуть из индекса (вывод из source, не воспроизведён). Имена MCP toolsets — `mcp-<server>`. [PL-H-51](../sources.md), [PL-H-50](../sources.md), [PL-H-52](../sources.md), [PL-H-53](../sources.md), [PL-H-54](../sources.md)

Индекс показывает `- имя: описание`, где описание усечено до первых 57 символов и `...`; полный `SKILL.md` не загружается. Поэтому начало `description` должно различать шесть скиллов. `skills_list` читает metadata из первых 4000 символов файла. Качество выбора скилла моделью не измерялось. [PL-H-50](../sources.md), [PL-H-51](../sources.md), [PL-H-49](../sources.md)

## Настройки скиллов: `skills.config`

### Объявление и путь значения

`metadata.hermes.config` — список объектов (одиночный объект превращается в список). Обязательны непустые `key` и `description`; повторный ключ пропускается, сохраняется первый; `default` переносится, если не `null`; `prompt` по умолчанию равен `description`. Типы значений не валидируются: это описание, а не JSON Schema. [PL-H-SKILL-CONFIG-0.21.6](../sources.md)

Ключ `eva.timezone` ищется по вложенному пути `skills.config.eva.timezone`; литеральный YAML-ключ с точкой этому не равен, индексация списков (`key.0`) не реализована. Список или словарь допустим как значение конечного пути. `$VAR` и `~` раскрываются у строкового значения и default, но не внутри списков. Необъявленные ключи в блок не попадают. [PL-H-SKILL-CONFIG-0.21.6](../sources.md)

```yaml
skills:
  external_dirs:
    - /nix/store/EXAMPLE-eva/skills
  auto_load:
    - eva
  config:
    eva:
      timezone: Europe/Moscow
      tasks:
        workdays: [mon, tue, wed, thu, fri]
```

Пример формы, не данные владельца.

### Блок значений

Подстановка — блок после текста скилла, а не замена переменных в Markdown:

```text
[Skill config (from ~/.hermes/config.yaml):
  eva.timezone = Europe/Moscow
  eva.tasks.workdays = ['mon', 'tue', 'wed', 'thu', 'fri']
  eva.calendar.gap_minutes = (not set)
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
| Slash `/eva`, preload `hermes chat --skills` | Да, в activation message | [PL-H-SKILL-INJECT-0.21.6](../sources.md) |
| `skills.auto_load` | Да: полный скилл и блок в system context, включая Telegram | [PL-H-SKILL-INJECT-0.21.6](../sources.md), [PL-H-AUTOLOAD-0.21.6](../sources.md) |
| Cron `skills: [...]` | Да: scheduler после `skill_view` сам добавляет блок | [PL-H-CRON-SKILLS-0.21.6](../sources.md) |

Поэтому носитель предпочтений — навигатор `eva` в `skills.auto_load`: значения всегда в контексте, а провайдерные скиллы, загруженные через `skill_view`, читают их из этого блока ([ADR 0017](../../docs/adr/0017-declarative-preferences.md)). Auto-load не применяется без skills tools, при `skip_context_files` или `HERMES_IGNORE_RULES`, поэтому он не заменяет явный `eva` в `skills` задания cron. Отдельный `auto_skill` канала или темы вставляет активацию в user message новой conversation — это другой механизм. [PL-H-AUTOLOAD-0.21.6](../sources.md), [PL-H-AGENT-CACHE-0.21.6](../sources.md)

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

У сервера с `trust=untrusted` Hermes сам запрашивает approval на вызов инструмента без `readOnlyHint=true`; это может затронуть и чтения без annotation. Host approval не заменяет согласие по правилам Eva, и наоборот. [PL-H-47](../sources.md)

При включённом Tool Search schemas MCP отложены: отсутствие инструмента в видимом списке не означает его отсутствия. Путь — `tool_search(queries=[...])` → `tool_describe(names=[...])` → `tool_call(calls=[{name, arguments}])`; schemas можно получить группой, но для MCP в `calls` допускается один элемент. Текущая web-страница описывает несколько local calls иначе; pin однозначен. Tool Search сохраняет approvals исходного инструмента. [PL-H-54](../sources.md)

Пакет Eva MCP не объявляет; подключения, trust и OAuth — native конфигурация Hermes у Потребителя. Native конфигурация поддерживает `auth: oauth`; effective config и рабочая OAuth-сессия — разные утверждения. [PL-H-14](../sources.md), [PL-H-18](../sources.md)

Фильтр `tools.include`/`tools.exclude` (срез 0.20.6, 2026-09-13): include — allowlist имён и globs с приоритетом над exclude; пустой `include: []` не регистрирует инструменты. Denylist или широкий glob могут пропустить будущий write-tool; фильтр хоста не отзывает OAuth scope сервера. [PL-H-36](../sources.md)

## `clarify`

Чтение 0.21.5. Схема: `questions:[{question, choices?, multi_select?}]`, 1–5 вопросов, до 4 вариантов; первый вариант помечается рекомендованным, UI добавляет «Other». Без `choices` ответ свободный. Ответ — `responses[]`; timeout, skip и недоставка не являются ответом. Timeout по умолчанию 3600 с. В Telegram пакет вопросов приходит отдельными последовательными карточками, полноценного multi-select нет. [PL-H-55](../sources.md)

Отсюда канал Eva: все блокирующие пробелы — один вопрос без `choices` со всем списком, чтобы сохранить «одним сообщением» ADR 0013; одиночное предложение, общий Preview и подтверждение безвозвратной потери Habitify — одна карточка «Да»/«Нет» после полного текста. Порядок вариантов: «Да», «Нет»; для подтверждения потерь Habitify — «Нет», «Да», чтобы рекомендованным не стало удаление. Это продуктовая интерпретация, не security approval. В cron `clarify` недоступен. [PL-H-55](../sources.md), [PL-H-66](../sources.md)

## `memory`

Чтение 0.21.5. Схема: `target: memory | user`, `action: add | replace | remove`, `content`, `old_text` для replace/remove; есть атомарный `operations` одного target. `replace` меняет всю запись, `old_text` лишь находит её. Отдельного чтения нет: записи видны в контексте памяти. Лимиты — 2200 символов на `memory` и 1375 на `user`, с разделителями. [PL-H-31](../sources.md), [PL-H-MEMORY-0.21.5](../sources.md)

При `memory.write_approval=true` foreground пытается получить approval сразу, без callback и в фоне запись ставится в очередь: `staged:true`, `pending_id`, `/memory pending`, `/memory approve <id>`. Применённая запись возвращает `done:true` с `usage` и `entry_count`, без содержимого. `done:true` — ответ инструмента о применении, не независимое чтение. [PL-H-56](../sources.md), [PL-H-31](../sources.md)

Описание `memory` в 0.21.5 советует хранить предпочтения «для вида работы» в скилле через `skill_manage`. Датированное наблюдение 2026-10-08: на хосте владельца setup `eva-calendar` по этой подсказке создал ожидающие изменения файлов скилла вместо записи в память, хотя скиллы подключены из `/nix/store`. Eva `memory` не использует: Пометки объектов живут в Журнале Eva (`Eva/Пометки.md` в Хранилище), предпочтения задаёт Потребитель в `skills.config`, и скиллы не пишут настройки ни через `memory`, ни через `skill_manage` ([ADR 0006](../../docs/adr/0006-state-in-eva-journal.md), [ADR 0017](../../docs/adr/0017-declarative-preferences.md)). В Telegram `/skills` управляет ожидающими записями скиллов, а не каталогом. [PL-H-MEMORY-0.21.5](../sources.md), [CLANWRIGHT-SETUP-2026-10-08](../sources.md), [PL-H-65](../sources.md)

## Cron

Задания Ритма Eva — Рефлексию, утро, Дневную проверку и закрытие дня — создаёт и сопровождает Потребитель; Eva их не создаёт и не удаляет и по своей инициативе не меняет, а по явной просьбе владельца выставляет у названного задания только `attach_to_session`, потому что CLI его не задаёт (см. «Доставка»). Сама Eva ставит только Возвраты — разовые задания через инструмент `cronjob_manage`; из фонового запуска это возможно, только если Потребитель включил `cron.allow_agent_scheduling: true` (см. «Самоназначенные задания и обвязка ответа») ([ADR 0022](../../docs/adr/0022-eva-journal-and-initiative.md), [ADR 0016](../../docs/adr/0016-consumer-boundary.md), [ADR 0003](../../docs/adr/0003-no-unattended-writes.md)). Синтаксис расписания (`#`, «или» дня месяца и дня недели, `[SILENT]`) разобран в [досье setup, §6](../setup.md#6-cron-hermes-v0216); heartbeat, `/loop`, webhooks, `context_from` и стоимость фоновых запусков — в [досье проактивности](../agents/proactivity.md).

### Управление и хранилище

В `config.yaml` есть параметры scheduler, но не список заданий. CLI `hermes cron` умеет `list`, `create`/`add`, `edit`, `pause`, `resume`, `run`, `remove` и диагностику; `import`, `sync`, `upsert` или файла-манифеста нет. CLI create вызывает тот же API, что модельный инструмент `cronjob_manage`. `create_job` генерирует случайный ID (`uuid4().hex[:12]`) и не принимает свой; `name` не уникален; `edit` ищет точный ID, затем точное имя без учёта регистра, а несколько одноимённых заданий дают ошибку. **Вывод:** первоначальный create и edit по уникальному имени возможны штатно, повторный create не идемпотентен, а автоматическая сверка требует логики Потребителя. [PL-H-CRON-CLI-0.21.6](../sources.md), [PL-H-CRON-STORE-0.21.6](../sources.md)

Blueprint skills (`metadata.hermes.blueprint`), каталог Automation Blueprint и `/blueprint` создают обычные задания по предложению, а не синхронизируют расписание; автоматическое обнаружение blueprint из `external_dirs` не доказано. [PL-H-CRON-BLUEPRINT-0.21.6](../sources.md)

`<profile-home>/cron/jobs.json` — **изменяемое хранилище scheduler**: `{"jobs": [...], "updated_at": ...}`, `schedule` внутри задания — нормализованный объект (`{"kind":"cron","expr":…,"display":…}`), рядом runtime-поля: claims, `next_run_at`, `last_run_at`, счётчики, pause state. Save переписывает файл целиком через temp и atomic replace под lock; слияние при сохранении не объединяет поля записей с одинаковым ID. Внешняя перезапись может потерять состояние и вызвать повторный или пропущенный запуск; read-only файл останавливает dispatch. Отдельный SQLite-журнал `cron/executions.db` хранит попытки, не определения. Публичной versioned schema у `jobs.json` нет. [PL-H-CRON-STORE-0.21.6](../sources.md)

### Скиллы задания

`skills` задания — голые имена, например `[eva, eva-tasks]`. Scheduler загружает их через `skill_view` и сам добавляет блок значений `skills.config`; отсутствующий или неоднозначный скилл пропускается с notice, поэтому успешное задание не доказывает загрузку всех скиллов. Вызов соседнего `skill_view` внутри задания блока не добавляет, поэтому задание, которому нужны предпочтения, прикрепляет `eva` явно. [PL-H-CRON-SKILLS-0.21.6](../sources.md), [PL-H-SKILL-VIEW-0.21.6](../sources.md)

### Доставка

| Механизм | Смысл | Основание |
| --- | --- | --- |
| `deliver: telegram:<chat_id>:<thread_id>` | Явная цель: signed numeric chat и numeric thread через двоеточие, без `topic=` и URL. Chat без thread не наследует home thread | [PL-H-CRON-DELIVERY-0.21.6](../sources.md) |
| `origin`, `deliver: origin` | Чат и тема сессии создания; без origin — первый подходящий home, иначе доставка не гарантирована. CLI create вне чата origin обычно не имеет | [PL-H-CRON-DELIVERY-0.21.6](../sources.md) |
| `TELEGRAM_CRON_THREAD_ID` | Тема только для выбора **home**-цели; явную цель и origin не меняет | [PL-H-CRON-DELIVERY-0.21.6](../sources.md) |
| `failure_deliver` | Та же грамматика; для уведомлений о сбоях **заменяет** `deliver`; `local` подавляет внешние уведомления | [PL-H-CRON-DELIVERY-0.21.6](../sources.md) |
| `attach_to_session` | Зеркалирование в продолжаемую сессию; к адресации темы не относится | [PL-H-CRON-DELIVERY-0.21.6](../sources.md) |

Для декларативного подключения подходит явный `deliver` с chat и thread; тема для сбоев, если нужна другая, задаётся `failure_deliver`. Права бота и фактическая доставка в супергруппу не проверялись.

Для явной цели `platform:chat` продолжаемость даёт только флаг задания `attach_to_session: true`; общий `cron.mirror_delivery` по умолчанию `false`. Brief попадает в целевую беседу как помеченный user turn, best-effort, без transcript cron; ответ владельца — новый foreground turn, в котором `clarify` доступен. [PL-H-CRON-DELIVERY-0.21.6](../sources.md), [PL-H-TELEGRAM-REPLY-0.21.6](../sources.md)

### Самоназначенные задания и обвязка ответа

Чтение v0.21.6.

- **Закрытые toolsets cron.** Агенту, запущенному scheduler, всегда закрыты `messaging` и `clarify`, а `cronjob` — по умолчанию; поверх добавляется `agent.disabled_toolsets`, поэтому per-job `enabled_toolsets` этот запрет не расширяет. Запрет `cronjob` в source описан как защита от петель расписания, а не граница безопасности. [PL-H-CRON-AGENT-SCHEDULING-0.21.6](../sources.md)
- **`cron.allow_agent_scheduling`.** Default `false`; `true` снимает только запрет `cronjob`. Документация того же commit: задание из запуска cron попадает в общую плоскую таблицу как обычное задание пользователя; `deliver: origin` внутри запуска cron разрешается в момент создания в конкретную цель создающего задания; задание может удалить себя и всё же доставить последний ответ. [PL-H-CRON-AGENT-SCHEDULING-0.21.6](../sources.md)
- **`cronjob_manage` create.** Инструмент `cronjob_manage` в toolset `cronjob`; для create обязательны `schedule` и `prompt`. `schedule` — интервал (`every 2h`), разовое смещение (`in 2h`), естественная форма, cron-выражение или ISO-время. Агент передаёт `skills`, `deliver`, `attach_to_session`, `context_from`, `continuity`, `enabled_toolsets`, `name`, `repeat` и другие поля; `model` и `provider` в список переданных агентом аргументов намеренно не входят — модель задания задаёт владелец через CLI или dashboard, иначе — `cron.model`, затем основная модель. Поле `pinned` агент выставляет только по явной просьбе пользователя: `true` закрепляет за заданием текущую основную модель, другую модель агент выбрать не может. Без `deliver` задание адресуется чату создания. [PL-H-CRONJOB-TOOL-0.21.6](../sources.md)
- **`cron.wrap_response`.** Default `true`: доставка обёрнута строками `Cronjob Response: <имя>`, `(job_id: …)` и подсказкой «To stop or manage this job…». `false` в профильном `config.yaml` отключает обвязку для всего профиля; per-job флага нет. [PL-H-CRON-WRAP-0.21.6](../sources.md)
- **Пределы.** Тихих часов, дневного предела уведомлений и денежного лимита не найдено; ограничивают `agent.max_turns` и 600 секунд неактивности (`HERMES_CRON_TIMEOUT`). Модель задания: `model` задания → `cron.model` → основная. [PL-H-CRON-LIMITS-0.21.6](../sources.md)

**Вывод для Eva:** Возврат из фонового запуска требует настройки Потребителя `cron.allow_agent_scheduling: true`; без неё `cronjob` недоступен, и Наблюдение остаётся со сроком в Журнале ([ADR 0022](../../docs/adr/0022-eva-journal-and-initiative.md)). Ограничение числа самоназначенных заданий после включения держится на правилах Eva, а не на Хосте. Создание Возвратов в runtime не проверялось.

### Timezone

Своего timezone у задания нет: recurring cron считает wall clock по `HERMES_TIMEZONE`, затем по `timezone` effective config профиля, затем по поясу сервера; при multiplex env-пояс подавляется ради изоляции профилей, неверное имя IANA даёт warning и fallback. Пояс кэшируется по identity config без mtime: после смены нужен перезапуск scheduler или gateway; пересчёт сохранённого `next_run_at` не проверен. Timezone читается из effective config, а `skills.config` — из сырого профильного файла, поэтому относительно managed overlay они ведут себя по-разному. [PL-H-CRON-TIMEZONE-0.21.6](../sources.md), [PL-H-SKILL-CONFIG-0.21.6](../sources.md)

### Toolsets и исполнение

Чтение 0.21.5: per-job `enabled_toolsets` имеет приоритет над `platform_toolsets.cron`, а без явного выбора к заданию могут добавиться включённые MCP. Поэтому toolsets cron по умолчанию не доказывают read-only любого задания; граница фоновых записей — Журнал Eva и Возвраты — держится на тексте скиллов по ADR 0003, а выбор toolsets — у Потребителя. [PL-H-58](../sources.md), [PL-H-57](../sources.md) На v0.21.6 запрет `messaging`, `clarify` и по умолчанию `cronjob` наложен поверх per-job выбора (см. выше). [PL-H-CRON-AGENT-SCHEDULING-0.21.6](../sources.md)

Чтение v0.21.6:

- **Имена MCP в `platform_toolsets`.** Сервер распознаётся только по имени из `mcp_servers` (`singularity`); перечисленные серверы образуют allowlist. `mcp-singularity` — имя toolset в registry: запись проходит как toolset, но allowlist серверов не образует, и в набор попадают все включённые серверы, в cron — и записывающий `habitify`. Конфигурация владельца ниже использует форму `mcp-…`. [PL-H-CRON-TOOLSETS-0.21.6](../sources.md)
- **`workdir`.** Абсолютный путь; при create и update должен существовать каталогом на Хосте scheduler, на запуске несуществующий отбрасывается с warning, и задание идёт без него. `workdir` становится cwd файловых инструментов, в контейнере — без проверки на Хосте, поэтому Хранилище должно быть смонтировано в песочницу по тому же абсолютному пути. С `workdir` в контекст запуска загружаются context files проекта, в том числе `AGENTS.md` Хранилища. [PL-H-CRON-WORKDIR-0.21.6](../sources.md)

Срез 0.20.6 (2026-09-13): ISO без offset трактуется в настроенной зоне, отдельный DST-переход мог пропустить срабатывание; preflight проверял auth, настройку skill и доставку, но не гарантировал готовность; попытка проходила claimed → running → completed/failed/unknown в `executions.db` без автоматического retry. Задание, попытка, ответ модели и доставка — разные наблюдения. Общий денежный или токенный лимит не найден; `approvals.cron_mode: deny` ограничивает опасные команды, не расходы. [PL-H-38](../sources.md), [PL-H-39](../sources.md), [PL-H-40](../sources.md), [PL-H-41](../sources.md)

## Другие инструменты

Чтение 0.21.5.

- `todo_list`: список `{id, content, status}` до 256 элементов, вызов без аргументов читает. Подходит для шагов агента в длинном обзоре, не для задач пользователя. [PL-H-59](../sources.md)
- `session_search`: поиск и чтение прошлых сессий с ограниченным окном. Найденное — контекст разговора, а не актуальное состояние провайдера. [PL-H-62](../sources.md) На v0.21.6 toolset в cron не закрыт; поиск возвращает по умолчанию 3, максимум 10 сессий. [PL-H-SESSION-SEARCH-0.21.6](../sources.md), [PL-H-CRON-AGENT-SCHEDULING-0.21.6](../sources.md)
- `send_message` модели не зарегистрирован; фоновые уведомления — доставка cron. [PL-H-60](../sources.md)
- `delegate_task` у владельца выключен, а дочерним агентам недоступны `clarify`, `memory` и `cronjob`; соседний скилл Eva загружается в той же сессии. [PL-H-61](../sources.md), [PL-H-66](../sources.md)

Local terminal (срез 0.20.6) работает от пользователя ОС и не является sandbox; фильтрация окружения terminal, `execute_code` и MCP различается. [PL-H-37](../sources.md), [PL-H-39](../sources.md) Из скиллов Eva `terminal` нужен только `eva-notes` — для `sha256` и перемещения файлов в Хранилище, в песочнице Потребителя (см. «Файлы, terminal и песочница»).

## Встроенные скиллы `llm-wiki` и `obsidian`

`eva-notes` надстраивается над двумя скиллами, которые поставляются с Hermes ([ADR 0018](../../docs/adr/0018-notes-over-llm-wiki.md)). Оба — только текст инструкций, без кода, `references/` и `scripts/`; оба под MIT. [PL-H-LLM-WIKI-0.21.6](../sources.md), [PL-H-OBSIDIAN-SKILL-0.21.6](../sources.md)

| Скилл | Что даёт | Основание |
| --- | --- | --- |
| `llm-wiki` 2.1.0 | Структура `raw/`, `entities/`, `concepts/`, `comparisons/`, `queries/`, `_archive/` и `SCHEMA.md`, `index.md`, `log.md`; путь из `WIKI_PATH`; ingest, query, lint; wiki frontmatter из шести полей и raw-схема `source_url`/`ingested`/`sha256`; пороги — сущность при двух источниках или центральности, минимум две ссылки у страницы, подтверждение ingest на 10+ существующих страниц; ценные ответы сохраняются в `queries/` или `comparisons/` | [PL-H-LLM-WIKI-0.21.6](../sources.md) |
| `obsidian` 1.0.0 | Работа с хранилищем файловыми инструментами: путь из `OBSIDIAN_VAULT_PATH` (fallback `~/Documents/Obsidian Vault`), абсолютные пути без переменных, `terminal` — только чтобы определить путь; чтение, поиск, запись, wikilinks | [PL-H-OBSIDIAN-SKILL-0.21.6](../sources.md) |

Русские имена папок, несколько корней сырья и правила Eva основа не описывает; `eva-notes` переопределяет её текстом, а в Hermes берёт основы встроенными скиллами по имени: `skill_view(name="llm-wiki")` и `skill_view(name="obsidian")`; каталог копий Отдельной установки в Hermes он не читает. Места основ, специфичные для Hermes, и их пути по умолчанию вне Hermes — [досье клиентов Agent Skills](agent-skills.md#основы-llm-wiki-и-obsidian-вне-hermes). Кодом соблюдение переопределений никто не проверяет. `related_skills` возвращается как metadata ответа `skill_view` и зависимостей не загружает; выключенный скилл `skill_view` не загружает, поэтому Потребитель включает `llm-wiki` и `obsidian` вместе с `eva-notes`. [PL-H-SKILL-DEPS-0.21.6](../sources.md)

**Чтение source, 2026-10-13: context files.** В system prompt попадает один тип контекста проекта, первый найденный: `.hermes.md`/`HERMES.md` (вверх до git root) → цепочка `AGENTS.override.md`/`AGENTS.md`/`agents.md` от git root до рабочего каталога → `CLAUDE.md` (только рабочий каталог) → `.cursorrules` и `.cursor/rules/*.mdc`; файл с найденным паттерном prompt injection блокируется ([prompt_builder.py L83–96, L1720–1760, L1852–1876](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/agent/prompt_builder.py#L1852-L1876)). Рабочий каталог — session cwd или `TERMINAL_CWD` на Хосте, существующий каталог; в gateway `TERMINAL_CWD` берётся из `terminal.cwd` ([system_prompt.py L713–726](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/agent/system_prompt.py#L713-L726), [runtime_cwd.py L84–120](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/agent/runtime_cwd.py#L84-L120), [gateway/run.py L5759–5772](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/gateway/run.py#L5759-L5772)). В разговоре gateway context files загружаются, если не выставлен `skip_context_files` платформы; в cron — только при `workdir` ([cron/scheduler.py L2491–2492](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/cron/scheduler.py#L2491-L2492)). При вызовах инструментов с путём в подкаталоге Hermes добавляет `AGENTS.md`/`CLAUDE.md` этого подкаталога ([subdirectory_hints.py](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/agent/subdirectory_hints.py)). `SCHEMA.md` не загружается никогда. [PL-H-CONTEXT-FILES-0.21.6](../sources.md)

Вывод: `AGENTS.md` Хранилища Hermes читает сам, когда его рабочий каталог на Хосте — Хранилище или каталог внутри него, и в cron с `workdir` — корнем Хранилища; при рабочем каталоге вне Хранилища, как в схеме clanwright, не читает ([CLANWRIGHT-NOTES-HANDOFF-2026-10-11](../sources.md)). `.hermes.md` в Хранилище отменил бы загрузку `AGENTS.md`. Поэтому `eva-notes` всегда явно читает `SCHEMA.md` и `AGENTS.md` Хранилища; при рабочем каталоге в Хранилище это повтор, безвредный и нужный другим клиентам ([ADR 0024](../../docs/adr/0024-portable-notes-skill.md)). Фактический `terminal.cwd` у Потребителя не проверен.

## Файлы, terminal и песочница

Инструменты `eva-notes` и их toolsets на v0.21.6: `read_file`, `write_file`, `patch`, `search_files` — `file`; `terminal` — `terminal`; `web_search`, `web_extract` — `web`; `skills_list`, `skill_view` — `skills`; `clarify` — `clarify`. В `tools.json` встроенные инструменты указываются ключом `toolset` вместо `server`. [PL-H-TOOLSETS-0.21.6](../sources.md)

**Чтение source.** Для backend docker каталоги кэша Hermes (`cache/documents`, `cache/images`, `cache/web` и другие) монтируются в контейнер под `/root/.hermes/…` флагом `:ro`, только на чтение; сетевой режим контейнера задаётся параметром `network` и `docker_extra_args`. [PL-H-SANDBOX-MOUNTS-0.21.6](../sources.md) Переменные окружения хоста в контейнер сами не попадают: их передают `terminal.docker_forward_env` (имена) или `terminal.docker_env` (пары имя–значение), поэтому `printenv WIKI_PATH` в песочнице видит значение, только если Потребитель его передал и задал путём внутри контейнера ([config_defaults.py L352–356](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/hermes_cli/config_defaults.py#L352-L356), [docker.py L53–83](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/tools/environments/docker.py#L53-L83), прочитано 2026-10-11).

**Чтение source: защищённые инструкционные файлы.** `write_file` и `patch` перед записью вызывают `_check_protected_instruction_write`: basename `AGENTS.md`, `CLAUDE.md`, `SOUL.md` или `.cursorrules` в любом каталоге, без учёта регистра и после realpath, требует approval человека; `--yolo` и allowlist его не снимают, без канала к человеку запись отклоняется, timeout — отказ, а текст отказа велит не повторять запись через `terminal`. Включено по умолчанию: `security.protected_instruction_files: true` ([file_tools_write_guards.py L175–189, L259–278, L345–355](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/tools/file_tools_write_guards.py#L175-L189), [file_tools.py L788, L875](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/tools/file_tools.py#L788), [config_defaults.py L1787](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/hermes_cli/config_defaults.py#L1787), прочитано 2026-10-11). Поэтому каждая запись `eva-notes` в `AGENTS.md` Хранилища, включая его первичное создание, проходит approval Hermes. Описание `write_file` сообщает `verified:true` и просит не перечитывать файл ([file_tools.py L1176](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/tools/file_tools.py#L1176)); `eva-notes` всё равно проверяет запись `read_file`, сверяя путь, frontmatter и фрагмент с разрешённым. Живой approval в Telegram не проверялся. Защищены только эти четыре имени: `SKILL.md` скиллов, лежащих в Хранилище на запись, approval не требует, а индекс скиллов сам предлагает модели чинить скиллы через `skill_manage(action='patch')` ([file_tools_write_guards.py L187–188](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/tools/file_tools_write_guards.py#L187-L188), [prompt_builder.py L1487](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/agent/prompt_builder.py#L1487)). [PL-H-VAULT-SKILL-WRITE-0.21.6](../sources.md)

**Декларация Потребителя** (handoff clanwright, не проверенный deployment): `terminal`, `file` и `execute_code` исполняются в одноразовом Docker без сети; Хранилище смонтировано на запись, `.obsidian/` и `.livesync/` — только на чтение; `WIKI_PATH` и `OBSIDIAN_VAULT_PATH` указывают на корень Хранилища; в образе bash, coreutils, `rg`, `jq`, diff/patch, Python с pypdf и pdfplumber, Poppler, Tesseract RU/EN, OCRmyPDF, tzdata; Node, curl, git нет, установка во время работы невозможна. Монтирование Хранилища, синхронизация LiveSync и Git окружения — у Потребителя ([ADR 0016](../../docs/adr/0016-consumer-boundary.md)). [CLANWRIGHT-NOTES-HANDOFF-2026-10-11](../sources.md)

## `web_extract` и вложения Telegram

**Чтение source.** `web_extract` исполняет сам Hermes, вне песочницы. Модель получает до `web.extract_char_limit` — по умолчанию 15 000 символов: около 75% начала и 25% конца с пометкой `[TRUNCATED]` и путём к полному тексту. `web.extract_char_limit` задаёт Потребитель в пределах 2 000–500 000. Полный текст до 2 000 000 символов пишется в `cache/web` под именем из хоста и hash URL и перезаписывается при повторном извлечении того же URL, поэтому копировать его нужно сразу. Более длинная копия обрезается, и пометка `[... stored copy truncated at …]` дописывается в конец самого файла кэша, а не в ответ модели ([web_tools_truncate.py L59–81](https://github.com/NousResearch/hermes-agent/blob/818c13be1dc4fd28987e1e881a9408224afd4535/tools/web_tools_truncate.py#L59-L81), прочитано 2026-10-11): `eva-notes` проверяет хвост файла перед копированием. В песочнице docker путь — `/root/.hermes/cache/web/`, только на чтение. [PL-H-WEB-EXTRACT-0.21.6](../sources.md), [PL-H-SANDBOX-MOUNTS-0.21.6](../sources.md)

Фото и документы gateway кладёт в `cache/images` и `cache/documents`; в песочнице они видны только на чтение. `agent.image_input_mode: native` передаёт картинки модели напрямую. [PL-H-GATEWAY-MEDIA-0.21.6](../sources.md), [PL-H-SANDBOX-MOUNTS-0.21.6](../sources.md)

**По handoff clanwright, не перечитано здесь:** провайдер `web_extract` — Firecrawl, закрепляет Потребитель; Markdown возвращается с картинками-удалёнными ссылками, байты картинок не сохраняются; исполнение JavaScript зависит от провайдера; у пересланных сообщений Telegram теряется автор, поэтому пересланный текст не выдаётся за слова владельца. [CLANWRIGHT-NOTES-HANDOFF-2026-10-11](../sources.md)

## Валидация

`hermes plugins validate` ищет `plugin.yaml`, `plugin.yml` или `plugin.json` и без manifest завершается ошибкой **до** security scan; `hermes plugins doctor --ci` тоже требует manifest и отдаёт exit `1`; официальный Action `plugin-validate` вызывает тот же validator. Для каталога скиллов в `external_dirs` эти проверки неприменимы. [PL-H-PLUGIN-VALIDATE-0.21.6](../sources.md)

Штатной CLI-замены для каталога скиллов в v0.21.6 нет:

| Команда | Что проверяет | Почему не заменяет validator |
| --- | --- | --- |
| `hermes skills list --source local` | Native catalog, включая external tier | Inventory, не frontmatter и не security; нет `--json`, `--ci`, аргумента каталога; создаёт каталоги Hub — только в изолированном home |
| `hermes skills audit [name] [--deep]` | Security scan записей Hub lock | Не обходит `external_dirs`; findings не меняют exit |
| `hermes skills check`, `inspect`, `snapshot export` | Обновления Hub, preview из registry, экспорт Hub lock | Не проверяют локальный каталог |

`tools.skills_guard.scan_skill` и `tools.skill_linter` — внутренние Python API без CLI entrypoint; строгая проверка frontmatter принадлежит пути записи `skill_manage`. Вызов этих функций собственным wrapper противоречит [ADR 0004](../../docs/adr/0004-no-custom-code.md). Native reader терпим к невалидному YAML, поэтому даже успешный `skills list` не доказывает валидный frontmatter. [PL-H-SKILLS-CLI-0.21.6](../sources.md), [PL-H-SKILL-CONFIG-0.21.6](../sources.md)

Следствие ([ADR 0017](../../docs/adr/0017-declarative-preferences.md)): CI Eva проверяет `git diff --check` (пробелы во всём дереве HEAD), остальное — независимое ревью; валидность frontmatter, security scan и discovery до появления штатной проверки Hermes автоматически не проверяются.

## Конфигурация владельца

Датированное наблюдение: clanwright HEAD `6641f66d74a1877f4c28f98bf9589af648a85d4c`, прочитан 2026-10-08; Eva там закреплён на `v0.4.0` (`647b0d1`). Это декларация до перехода на `skills.config` и до `eva-notes`, не проверка deployment. Окружение `eva-notes` по handoff 2026-10-11 описано в разделе «Файлы, terminal и песочница». [PL-H-66](../sources.md)

- Eva подключён через `skills.external_dirs` из Nix store; plugin registry не используется. Hermes закреплён на `f97608f`. Nix-модули Hermes официально имеют уровень поддержки Tier 2 best effort. [PL-H-7](../sources.md)
- MCP: `singularity` — официальный hosted MCP с OAuth и `trust=untrusted`; `habitify_read`, `habitify`, `google_calendar` — локальные адаптеры с `trust=full`. Sampling, elicitation, resources и prompts выключены. Включены 35 инструментов Singularity, 7 чтений и 16 записей Habitify; allowlist Calendar берётся из `tools.json` Eva.
- Toolsets Telegram: `clarify`, `todo`, `memory`, `session_search`, `web`, `search`, `skills`, `mcp-singularity`, `mcp-habitify`, `mcp-habitify_read`, `mcp-google_calendar`, с cron — `cronjob`. Tool Search включён.
- Фоновые toolsets: `web`, `skills`, `mcp-habitify_read`, `memory` — без Singularity и Calendar, поэтому фоновая утренняя сводка задач и календаря по ним невозможна. Задания Ритма и их toolsets заводит Потребитель; Eva сама ставит только Возвраты ([ADR 0022](../../docs/adr/0022-eva-journal-and-initiative.md)).
- Cron включён, timezone `Europe/Moscow`; `memory.write_approval=true`, `skills.write_approval=true`, `approvals.mode=manual`. Terminal, file, code execution, delegation, browser на 2026-10-08 выключены; для `eva-notes` Потребитель объявил `file`, `terminal` и `execute_code` в песочнице Docker.
- `grill-me` поставляется из официальных optional skills закреплённого Hermes; фактический discovery не проверен.

## Ограничения доступа gateway

Срез 0.20.6: доступ к боту требует pairing, allowlist, явного allow-all или trusted adapter; pairing grants и allowlists объединяются. Admin/user slash policy ограничивает slash-команды, не обычный чат. Доступ к боту, разрешение инструмента и согласие изменить данные — разные границы. [PL-H-42](../sources.md), [PL-H-44](../sources.md), [PL-H-45](../sources.md)

## Непроверенные слои

Не проверены: установленная версия Hermes у Потребителя, подключение закреплённой ревизии через `external_dirs`, фактические имена скиллов, индекс и slash, появление блока `skills.config` в Telegram и cron, свежесть значений после перезапуска, поведение native writers с декларативным `config.yaml`, MCP discovery и OAuth, вызовы Tool Search, карточки `clarify` в Telegram, записи и approval `memory`, задания cron, timezone, запуск и доставка в тему, `cron.allow_agent_scheduling` и создание Возвратов из фонового запуска, `cron.wrap_response`, `attach_to_session` в Telegram, поведение модели и Read-back. Для `eva-notes` не проверены: включение `llm-wiki` и `obsidian`, соблюдение переопределений моделью, песочница Docker без сети и монтирование Хранилища, доступ к `/root/.hermes/cache/web/` и вложениям, `web_extract` и его провайдер, запись в Хранилище и проверка `read_file`, синхронизация LiveSync, загрузка `AGENTS.md` Хранилища при фактическом `terminal.cwd`, выбор моделью `skill_view` или `read_file` по фразе «из каталога этого скилла», вердикт `skills_guard` для скиллов в Хранилище. Source и статические проверки эти результаты не подтверждают. Аккаунты и данные провайдеров этой работой не затрагивались.

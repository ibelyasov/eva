# Клиенты Agent Skills: переносимый `eva-notes`

Досье — основания [ADR 0024](../../docs/adr/0024-portable-notes-skill.md): почему `eva-notes` пишется одним host-нейтральным текстом, ставится в `<Хранилище>/.agents/skills/` с симлинком для Claude Code, почему копии основ лежат вне каталогов скиллов и чего статика не подтверждает. Механика Hermes — [досье Hermes](hermes.md): там проектные скиллы и доверие, чтение файлов скилла, context files и защита инструкционных файлов.

## Версии и типы доказательств

Все источники прочитаны 2026-10-13, только чтение; клиенты не устанавливались и не запускались, модельных прогонов нет.

| Метка | Что значит | Где |
| --- | --- | --- |
| SRC | Прочитано в source на commit | спецификация `agentskills/agentskills` @ `69ef37e9424c0a7ea9dd2293b559e43ec8176379`; `openai/codex` @ `cf0ed4c600161fcaa18eeac02b5fc715e1076955`; Hermes `NousResearch/hermes-agent` @ `818c13be1dc4fd28987e1e881a9408224afd4535`; `obsidianmd/obsidian-help` @ `08ef563f07b78dd589f19b1583d089c1e2d2377c` |
| DOC | Официальная документация vendor без version pin | страницы Claude Code, Codex, Gemini CLI, GitHub Copilot, VS Code, Cursor, OpenCode, Goose, Hermes, Obsidian, Self-hosted LiveSync |
| VENDOR | Release notes или блог vendor | VS Code 1.109 и 1.110 |
| FORUM | Форум или сторонний справочник | форум Obsidian, справочник Goose на mintlify.com |
| НЕ ПРОВЕРЕНО | Подтверждения нет | отмечено в тексте |

Сайт agentskills.io из среды исследования не отвечал (TLS reset). Спецификацию и руководство для клиентов прочитали по исходникам сайта в репозитории `agentskills/agentskills` на commit `69ef37e` от 2026-08-09. Выборочное сравнение со снимком [SC-S-1](../sources.md) от 2026-10-07 показало различия только в разметке. [PL-AS-SPEC-69EF37E](../sources.md), [PL-AS-CLIENT-GUIDE-69EF37E](../sources.md)

## Формат Agent Skills

**SRC, спецификация.** Frontmatter: обязательные `name` (1–64 символа, `a-z`, `0-9`, `-`, без `-` на краях и `--`, совпадает с именем каталога) и `description` (1–1024 символа: что делает скилл и когда его применять); необязательные `license`, `compatibility` (1–500 символов, «most skills do not need»), `metadata` (отображение строка → строка) и экспериментальный `allowed-tools`. Другие файлы тело называет относительными путями от корня скилла, на один уровень от `SKILL.md`. Рекомендуемый объём `SKILL.md` — меньше 500 строк и около 5000 токенов; `references/` загружаются по требованию. [PL-AS-SPEC-69EF37E](../sources.md)

**SRC, руководство для клиентов.** Клиент передаёт модели каталог скилла (`location` в каталоге скиллов или «Skill directory» при активации); модель читает файлы своим инструментом чтения, относительные пути разрешаются от каталога `SKILL.md`. Проверка мягкая: несовпадение `name` с каталогом или длина больше 64 — предупреждение; без `description` или с неразбираемым YAML скилл пропускается. Одноимённые скиллы клиент должен детерминированно свести к одной копии и предупредить. [PL-AS-CLIENT-GUIDE-69EF37E](../sources.md)

**DOC, отличия клиентов.** Claude Code молча игнорирует незнакомые поля; в листинге обрезает `description` вместе с `when_to_use` до 1536 символов; имеет подстановку `${CLAUDE_SKILL_DIR}`, только свою. Codex отдаёт списку скиллов не больше 2 % контекста и при нехватке места сначала укорачивает `description`, поэтому главный сценарий ставится в начало. OpenCode повторяет ограничения спецификации на `name` и `description` и хранит свои флаги в `metadata`. Hermes подставляет `${HERMES_SKILL_DIR}`, только свою. [PL-CC-DOCS-2026-10-13](../sources.md), [PL-CODEX-DOCS-2026-10-13](../sources.md), [PL-OPENCODE-DOCS-2026-10-13](../sources.md), [PL-H-SKILL-DIR-0.21.6](../sources.md)

**Локальная проверка, 2026-10-13.** У текста `eva-notes` по этому решению `description` — 365 символов, `SKILL.md` — 6974 байта и 56 строк, frontmatter — только `name` и `description`, `name` совпадает с каталогом: это укладывается во все перечисленные лимиты и в лимит Eva 7000 байт.

## Как модель читает файлы скилла

| Клиент | Как читаются `references/` | Основание |
| --- | --- | --- |
| Claude Code | Файлы, на которые ссылается `SKILL.md` | DOC [PL-CC-DOCS-2026-10-13](../sources.md) |
| Codex | В начальный список попадает путь скилла; после выбора модель читает `SKILL.md`, затем нужные файлы | DOC [PL-CODEX-DOCS-2026-10-13](../sources.md) |
| Gemini CLI | `activate_skill` с подтверждением пользователя; тело и структура папки попадают в разговор, каталог скилла — в разрешённые пути | DOC [PL-GEMINI-DOCS-2026-10-13](../sources.md) |
| VS Code / Copilot | Пути от `SKILL.md`; файл без ссылки не загружается | DOC [PL-VSCODE-DOCS-2026-10-13](../sources.md) |
| Cursor | Пути от корня скилла, `references/` по требованию | DOC [PL-CURSOR-DOCS-2026-10-13](../sources.md) |
| Goose | Supporting files читаются файловыми инструментами расширения Developer | DOC [PL-GOOSE-DOCS-2026-10-13](../sources.md) |
| Hermes | `skill_view(name, file_path)`; ответ на загрузку скилла содержит каталог и список файлов | SRC [PL-H-SKILL-DIR-0.21.6](../sources.md), DOC [PL-H-DOCS-2026-10-13](../sources.md) |

Вывод: общий знаменатель — относительный путь от каталога скилла. Голый `references/x.md` в Хранилище опасен: файловый инструмент разрешит его от рабочего каталога, то есть от корня Хранилища. Поэтому текст говорит «из каталога этого скилла» и добавляет, как читать: тем же инструментом, которым загружен скилл, с путём файла, иначе чтением по абсолютному пути каталога скилла; `skill_view` он не содержит ([досье Hermes](hermes.md#ресурсы-скилла-и-соседние-скиллы)). Ссылки между файлами `references/` в Markdown разрешаются от папки `references/`, а спецификация и Cursor задают пути от корня скилла; текст оговаривает, что такие ссылки ведут в ту же папку `references/`. Что модель каждого клиента делает по этой фразе — НЕ ПРОВЕРЕНО.

## Клиенты: каталоги, контекст, вопрос, веб

| Клиент | Project-level каталоги скиллов | `AGENTS.md` / `CLAUDE.md` рабочего каталога | Вопрос с вариантами | Извлечение страницы |
| --- | --- | --- | --- | --- |
| Claude Code | `.claude/skills/` (от рабочего каталога вверх до корня репозитория); `.agents/skills` в документации нет | `CLAUDE.md`; `AGENTS.md` — только если нет `CLAUDE.md` | `AskUserQuestion` | `WebFetch` |
| Codex CLI | `.agents/skills` в рабочем каталоге, родителях и корне репозитория | `AGENTS.override.md`, затем `AGENTS.md` от корня Git до рабочего каталога; `CLAUDE.md` — только через `project_doc_fallback_filenames` | `request_user_input`: 1–3 вопроса по 2–3 варианта и «Other»; по умолчанию только в Plan mode, флаг вне Plan mode в разработке и выключен (SRC) | Только `web_search`, отдельного fetch страницы в документации нет |
| Gemini CLI | `.gemini/skills/`, `.agents/skills/` (на одном уровне побеждает `.agents`) | `GEMINI.md`; `AGENTS.md` — только через `context.fileName` | `ask_user`: 1–4 вопроса, `choice`, `text`, `yesno` | `web_fetch` с подтверждением |
| GitHub Copilot (CLI, cloud agent, VS Code) | `.github/skills`, `.claude/skills`, `.agents/skills` | CLI: `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`; VS Code: `AGENTS.md`, `CLAUDE.md` | CLI: `ask_user`, формат вариантов не описан; VS Code: `askQuestions` с вариантами и свободным текстом (VENDOR) | CLI: `web_fetch`; VS Code: `#web/fetch` |
| Cursor | `.agents/skills/`, `.cursor/skills/`, для совместимости `.claude/skills/` и `.codex/skills/` | `AGENTS.md`; `CLAUDE.md` в документации не указан | «Ask questions», формат вариантов не описан | Web fetch |
| OpenCode | `.opencode/skills/`, `.claude/skills/`, `.agents/skills/` (вверх до git worktree) | `AGENTS.md`, `CLAUDE.md` — запасной | `question`: варианты и свой ответ | `webfetch` |
| Goose | `.agents/skills/` (рекомендован), для совместимости `.goose/skills/` и `.claude/skills/` | `AGENTS.md` и `.goosehints`; `CLAUDE.md` — через `CONTEXT_FILE_NAMES` | Встроенного не найдено (НЕ ПРОВЕРЕНО); MCP Elicitation для расширений | Расширение Computer Controller, имя `web_scrape` (FORUM); fetch — внешнее расширение |
| Hermes Agent | `.hermes/skills/`, `.agents/skills/` — только в git checkout, после `hermes skills trust`, со сканом `skills_guard` | Первый найденный: `.hermes.md` → `AGENTS.md` → `CLAUDE.md` → `.cursorrules` (SRC) | `clarify`: до 4 вариантов, «Other», открытый вопрос | `web_extract`, нужен ключ backend |

Основания: [PL-CC-DOCS-2026-10-13](../sources.md), [PL-CODEX-DOCS-2026-10-13](../sources.md), [PL-CODEX-SRC-CF0ED4C](../sources.md), [PL-GEMINI-DOCS-2026-10-13](../sources.md), [PL-COPILOT-DOCS-2026-10-13](../sources.md), [PL-VSCODE-DOCS-2026-10-13](../sources.md), [PL-VSCODE-RELEASES-1.109-1.110](../sources.md), [PL-CURSOR-DOCS-2026-10-13](../sources.md), [PL-OPENCODE-DOCS-2026-10-13](../sources.md), [PL-GOOSE-DOCS-2026-10-13](../sources.md), [PL-GOOSE-MINTLIFY-2026-10-13](../sources.md), [PL-H-DOCS-2026-10-13](../sources.md), [PL-H-PROJECT-SKILLS-0.21.6](../sources.md), [PL-H-CONTEXT-FILES-0.21.6](../sources.md).

Выводы для текста `eva-notes` (продуктовая интерпретация, [ADR 0024](../../docs/adr/0024-portable-notes-skill.md)):

- **Вопрос владельцу.** Карточка с вариантами есть почти везде, но называется по-разному, у Codex вне Plan mode недоступна, у Goose не подтверждена. Поэтому вопрос задаётся карточкой, если Хост её даёт, иначе обычным сообщением, а timeout и пропуск согласием не считаются.
- **Извлечение страницы.** У Codex его нет, у Goose оно зависит от расширения, у Hermes — от ключа backend. Без инструмента извлечения ссылка захватывается с `capture: partial`.
- **`AGENTS.md` Хранилища.** Почти все клиенты подхватывают его сами, когда рабочий каталог — Хранилище. Исключения — Gemini CLI без `context.fileName`, Claude Code при соседнем `CLAUDE.md` и Hermes при рабочем каталоге вне Хранилища. `SCHEMA.md` не грузит никто. Поэтому скилл читает оба файла явно.

## Каталог в Хранилище и симлинки

**DOC.** `.agents/skills/` читают 7 из 8 целевых клиентов: Codex, Gemini CLI, Copilot, Cursor, OpenCode, Goose и Hermes (последний — только в доверенном git checkout). Не читает только Claude Code. `.claude/skills/` читают 5 из 8: Claude Code, Copilot, Cursor, OpenCode, Goose; не читают Codex, Gemini CLI и Hermes. Остальные каталоги покрывают по 1–2 клиента.

**DOC.** Claude Code поддерживает симлинк на каталог скилла и загружает скилл один раз, даже если на одну цель указывают несколько мест; Codex следует за симлинками. Copilot, Cursor, OpenCode и Goose сканируют и `.agents/skills`, и `.claude/skills`, поэтому после симлинков могут увидеть скилл дважды. OpenCode требует уникальных имён во всех местах, VS Code при совпадении имён показывает обе копии. Фактическое поведение дублей — НЕ ПРОВЕРЕНО; содержимое у копий одинаковое. [PL-CC-DOCS-2026-10-13](../sources.md), [PL-CODEX-DOCS-2026-10-13](../sources.md), [PL-OPENCODE-DOCS-2026-10-13](../sources.md), [PL-VSCODE-DOCS-2026-10-13](../sources.md), [PL-AS-CLIENT-GUIDE-69EF37E](../sources.md)

Вывод: основной каталог `eva-notes` — `<Хранилище>/.agents/skills/`, для Claude Code — отдельный шаг с относительным симлинком `.claude/skills/eva-notes` → `../../.agents/skills/eva-notes`. Основам симлинки не нужны: они лежат вне каталогов скиллов. Hermes `.claude/skills` не сканирует, а `.agents/skills` без доверия не загружает ([досье Hermes](hermes.md#проектные-скиллы-и-хранилище)).

## Основы `llm-wiki` и `obsidian` вне Hermes

**SRC.** На commit `818c13be` в `skills/research/llm-wiki/` и `skills/note-taking/obsidian/` лежит только `SKILL.md` (507 и 68 строк). Последний commit, затронувший оба файла, — `1c9433897c7b8a3b2754a84875e8f86d0a42991a`. Лицензия — `license: MIT` во frontmatter и корневой `LICENSE` Hermes: MIT, «Copyright (c) 2025 Nous Research». [PL-H-LLM-WIKI-0.21.6](../sources.md), [PL-H-OBSIDIAN-SKILL-0.21.6](../sources.md)

| Место | Что специфично для Hermes |
| --- | --- |
| `llm-wiki` L4–12, `obsidian` L4–11 | Поля `version`, `author`, `platforms` верхнего уровня и `metadata.hermes`; для строгих валидаторов Agent Skills они нестандартны — отвергает ли их какой-то клиент, НЕ ПРОВЕРЕНО |
| `llm-wiki` L38–44, L374–383 | `WIKI_PATH` из `${HERMES_HOME:-~/.hermes}/.env`, без него — `~/wiki` |
| `llm-wiki` L262–263, L323–331 | `web_extract` для URL и PDF; `execute_code` для lint |
| `llm-wiki` L271–272 | Пропуск обсуждения в cron |
| `llm-wiki` L418–475 | `OBSIDIAN_VAULT_PATH`, Obsidian Headless: `npm install -g`, `ob login`, systemd |
| `obsidian` L22–26 | `OBSIDIAN_VAULT_PATH`, без него — `~/Documents/Obsidian Vault`; путь определять через `terminal` |
| Оба | Имена инструментов Hermes: `read_file`, `search_files`, `write_file`, `patch`, `terminal` |

Вывод: по методу основы переносимы — это инструкции без кода, а имена инструментов модели других клиентов обычно переводят на свои (поведение модели, НЕ ПРОВЕРЕНО). По пути — нет: без переменных основы уходят в `~/wiki` и `~/Documents/Obsidian Vault`. Поэтому `eva-notes` передаёт им Корень Хранилища явно, а `execute_code`, headless-настройку и cron-оговорку переопределяет нейтрально ([ADR 0018](../../docs/adr/0018-notes-over-llm-wiki.md)). Каталог копируется без категории (`research/`, `note-taking/`): в Hermes она выводится только из глубины пути. MIT требует сохранить уведомление, поэтому `LICENSE` кладётся рядом с каждой копией ([право и атрибуция](../legal-and-attribution.md#mit-и-публикация-материалов)).

**SRC.** `description` основы `obsidian` на `818c13be` — «Read, search, create, and edit notes in the Obsidian vault.»: в каталоге скиллов клиента такой скилл подходит под любой запрос о заметках. [PL-H-OBSIDIAN-SKILL-0.21.6](../sources.md)

Вывод (решение владельца, [ADR 0024](../../docs/adr/0024-portable-notes-skill.md)): копии основ лежат в `<Хранилище>/.agents/eva-notes-bases/{llm-wiki,obsidian}/`, а не в `.agents/skills/`. Клиенты сканируют только свои каталоги скиллов (таблица выше), поэтому копии не попадают в список скиллов и клиент не выбирает их сам в обход `eva-notes` — без Плана сущностей, правил согласия и с путями по умолчанию вне Хранилища. Загрузки скилла по имени у Codex и Goose нет, так что `eva-notes` в любом случае читает `SKILL.md` основы по пути от корня Хранилища. Hermes этот каталог тоже не сканирует ([досье Hermes](hermes.md#проектные-скиллы-и-хранилище)). Применит ли модель основу, прочитанную как обычный файл, так же, как загруженный скилл, — НЕ ПРОВЕРЕНО.

## Obsidian и синхронизация скрытых папок

- **SRC/DOC.** Obsidian Sync считает файлы и папки на «.» скрытыми и не синхронизирует их; исключение — `.obsidian`. Прямого утверждения, что Obsidian не индексирует dot-папки, справка не содержит. [PL-OBS-SYNC-08EF563](../sources.md)
- **FORUM.** Сотрудники Obsidian на форуме говорят, что файлы и папки на «.» скрыты по замыслу: их нет в проводнике, ссылки на них не работают. Значит, `.agents/` не попадёт в граф, поиск и вики. [PL-OBS-FORUM-1622](../sources.md), [PL-OBS-FORUM-106685](../sources.md)
- **DOC.** Obsidian «strongly advise[s] against» симлинки и игнорирует симлинк с одной папки хранилища на другую того же хранилища; Git симлинкам не следует и хранит их как ссылки. Затронет ли симлинк между двумя скрытыми папками индекс Obsidian — НЕ ПРОВЕРЕНО. [PL-OBS-SYMLINKS-2026-10-13](../sources.md)
- **DOC.** Self-hosted LiveSync переносит скрытые файлы только опциональной Hidden File Sync, выключенной по умолчанию; её советуют включать после того, как обычная синхронизация заметок работает. [PL-LIVESYNC-HIDDEN-FILES-2026-10-13](../sources.md)

Вывод: на другие устройства скиллы попадают Git Хранилища или повторной установкой, не Obsidian Sync и не LiveSync без Hidden File Sync. Симлинки на Windows требуют поддержки в Git и правах — НЕ ПРОВЕРЕНО.

## Непроверенные слои

Загрузка `eva-notes` в каждом клиенте; чтение копий основ по пути и их применение моделью; чтение `references/` по фразе «из каталога этого скилла»; команды установки и скилла в Windows без POSIX-оболочки (Git Bash, WSL); поведение дублей при симлинках; обход скрытых симлинков Obsidian; вопрос с вариантами в Goose и имя его веб-инструмента; работа Codex с `.agents/skills` вне git-репозитория (документирован `$CWD/.agents/skills`); вердикт `skills_guard` Hermes; поведение моделей по переопределениям основ. Source, документация и статические проверки эти результаты не подтверждают.

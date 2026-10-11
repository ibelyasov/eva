---
name: jedikit-tasks
description: "Задачи и проекты в SingularityApp по «Джедайским техникам» Максима Дорофеева: записать дело во Inbox, разобрать Inbox item, сформулировать задачу, дерево Областей и Папок, следующий шаг проекта, утренний, вечерний, недельный и месячный обзоры — capture, triage, project, daily open, daily close, weekly. Привычки ведёт jedikit-habits, календарь — jedikit-calendar, План дня с календарём — jedikit-planning."
---

# jedikit-tasks

Помогай записывать обязательства, формулировать стартуемые Задачи и вести Проекты к результату. Состояние задач и проектов — в SingularityApp; решает пользователь. Отвечай по-русски, identifiers не переводи.

## Вход в сценарий

1. Выбери строку таблицы по команде или фразе и прочитай её reference; другие — по ссылке из текущего.
2. До первого обращения к SingularityApp или `memory` прочитай [write-policy.md](references/write-policy.md) и [tools.md](references/tools.md).
3. Файлы скилла загружай `skill_view(name="jedikit-tasks", file_path="references/<файл>.md")`, контракт — `file_path="tools.json"`; Markdown-ссылка файл не загружает. Соседний скилл — по имени: `skill_view(name="jedikit-calendar")`.
4. Предпочтения читай из блока `[Skill config …]` навигатора `jedikit` по [settings.md](references/settings.md#предпочтения).

## Маршрутизация

| Запрос | Результат | Reference |
| --- | --- | --- |
| «Настрой дерево», Области и Папки | Дерево, Режимы поддерева, новые Области | [tree.md](references/tree.md) |
| Рабочие дни, timezone, режим Области или Папки; `status` | Предпочтения, режимы, их состояние | [settings.md](references/settings.md) |
| `capture`, «запиши», «сохрани, чтобы не забыть» | Обязательство — Inbox item; идея, мысль, ссылка — `jedikit-notes` | [inbox.md](references/inbox.md#capture) |
| `triage`, «разберём Inbox», «уточни задачу» | Разобранный Inbox item | [inbox.md](references/inbox.md#triage) |
| «Накопился Inbox» | Сессия Inbox debt | [inbox.md](references/inbox.md#inbox-debt) |
| Формулировка, дата, приоритет, длительность; «сделал», «отмени задачу», архив | Изменённая Задача | [tasks.md](references/tasks.md) |
| `project`, «проверь проект», «следующий шаг», «закрой проект»; обратная ссылка из Хранилища | Результат, Следующий шаг, судьба открытых Задач | [projects.md](references/projects.md) |
| Разделы, Теги, Чек-лист | Структура внутри и поперёк Проектов | [structure.md](references/structure.md) |
| `daily open`, «начнём день» | Хвосты, Ресурс дня, Фокус-лист | [daily.md](references/daily.md#daily-open) |
| `daily close`, «закроем день» | Остатки, затронутые Проекты, Гвоздодёр | [daily.md](references/daily.md#daily-close) |
| `weekly`, «еженедельный обзор» | Inbox, все Проекты, Следующие шаги, структура | [weekly.md](references/weekly.md) |
| «Пересмотрим когда-нибудь» | Месячный обзор поддеревьев Когда-нибудь | [someday.md](references/someday.md) |
| Запуск `[расписание jedikit]`; «напоминай о weekly» | Чтение и приглашение; задания — у Потребителя | [background.md](references/background.md) |
| `help` | Команды и предложение сценария | этот файл |

## Уточнение и согласие

- Используй известное; необязательное и названное не спрашивай, относительную дату покажи в результате. Все блокирующие пробелы — один `clarify` без `choices`; полную явную команду не переспрашивай.
- Согласие на предложение агента или Preview — одна карточка `clarify` «Да»/«Нет» с полным текстом; без явного «Да» (timeout, пропуск, «Other») запись не выполняй.
- Размытое намерение уточняй редко через `grill-me` (`skills_list` → `skill_view`); если его нет, назови его и предложи установить в Hermes, своё интервью не веди.
- Запрос другого скилла или провайдера, кроме «запиши …», — одной строкой предложи ход и выполни после «Да». Перед записью в чужого провайдера или Хранилище загрузи скилл-владелец и его `references/write-policy.md`; без фактической загрузки передачу не объявляй.

## Границы

- SingularityApp — только MCP-сервер `singularity` по [контракту](references/tools.md).
- У каждого действия один ответственный скилл: привычки и Ритуалы — `jedikit-habits`, События — `jedikit-calendar`, Хранилище Obsidian — `jedikit-notes`. Встреча из Inbox — `jedikit-calendar`, Идея и Справка — `jedikit-notes` по Triage.
- Daily open собирает Фокус-лист без календаря и проверки вместимости.
- Kanban, удаление объектов и служебные Теги состояния агента вне скилла.
- Фоновый запуск только читает и приглашает; задания cron не создавай.
- Предпочтения только читай; в `memory` — только Режимы поддерева по [settings.md](references/settings.md#где-хранить). Файлы скиллов JediKit не меняй, `skill_manage` не вызывай.

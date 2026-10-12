---
name: eva-habits
description: "Привычки и Ритуалы в Habitify: поведенческие эксперименты, Отметки, Тяга. Используй, чтобы завести привычку, отметить выполнение, срыв, пропуск или количество, пересмотреть план, поставить паузу, архивировать или удалить, вести Области и Ритуалы («ритуал выполнен»); для коучинга по еде и весу, порно и мастурбации, отказу от веществ; команды design, log, urge, review, adjust, rituals. Задачи ведёт eva-tasks, календарь — eva-calendar."
---

# Eva Habits

Помогай проверять поведение экспериментом: одна гипотеза, План эксперимента в заметке привычки, Отметки, Обзор эксперимента. Состояние живёт в Habitify, знания о привычке и «зачем мне» — Заметка о привычке у `eva-notes` ([experiments.md](references/experiments.md)); решает пользователь. По-русски, identifiers не переводи.

## Вход в сценарий

1. Определи сценарий по таблице; читай только его references.
2. Еда, вес, потеря контроля, вещества, алкоголь, никотин, лекарства, сексуальное поведение или любой сигнал риска — сначала [safety.md](references/safety.md), затем тема: [food-behavior.md](references/food-behavior.md), [sexual-behavior.md](references/sexual-behavior.md), [cessation.md](references/cessation.md). Safety-стоп важнее любой команды.
3. Перед записью в Habitify прочитай [write-policy.md](references/write-policy.md) и [tools.md](references/tools.md); Журнал Eva — по `skill_view(name="eva", file_path="references/notebook.md")`.
4. Файлы скилла загружай `skill_view(name="eva-habits", file_path="references/<файл>.md")`, контракт — `file_path="tools.json"`, соседний скилл — `skill_view(name="eva-calendar")`; Markdown-ссылка файл не загружает.

| Запрос | Результат | Reference |
| --- | --- | --- |
| «давай настроим привычки» | Привычки и один ход | [settings.md](references/settings.md#открытый-запрос) |
| `status`, «какие привычки сегодня?» | Настройки, эксперимент или день | [settings.md](references/settings.md#status) |
| `design`, «хочу проверить…»; `adjust` | Гипотеза, привычка, План эксперимента; правка одной вещи | [experiments.md](references/experiments.md), [coaching.md](references/coaching.md) |
| `log`, «отметь…», отмена Отметки | Отметка за точную дату; отмена с подтверждением потерь | [tracking.md](references/tracking.md) |
| `urge`, «тянет…» | 1–3 действия из плана, без записи | [tracking.md](references/tracking.md#тяга) |
| `review` | Оставить, изменить одну вещь, пауза или завершить | [review.md](references/review.md), [coaching.md](references/coaching.md) |
| `pause`, Off Mode, `archive`, удаление | Инструкция для приложения, команды `off` нет; архив с историей; удаление | [review.md](references/review.md#pause); Область — [areas.md](references/areas.md#удалить-область) |
| `areas`, группировка | Области и назначения | [areas.md](references/areas.md) |
| `rituals`, «ритуал выполнен», статус, обзор, старт | Состав, старт, массовая Отметка | [rituals.md](references/rituals.md) |
| Доказательства, сроки формирования, серия | Границы уверенности | [evidence.md](references/evidence.md) |
| Инициатива Eva по привычкам | Сигнал, практика, «зачем» | [signals.md](references/signals.md) |
| `help` | Команды без обращения к аккаунту | — |

## Уточнение и согласие

- Известное и необязательное не спрашивай, относительную дату покажи в результате; блокирующие пробелы — один `clarify` без `choices`; полную явную команду не переспрашивай.
- Согласие на предложение агента или Preview — карточка `clarify` «Да»/«Нет» с полным текстом; без явного «Да» (timeout, пропуск, «Other») не пиши.
- Размытое намерение уточняй редко через `grill-me` (`skills_list` → `skill_view`); нет его — назови и предложи установить в Hermes, своё интервью не веди.
- Запрос другого скилла — одной строкой предложи ход, выполни после «Да»; перед записью в чужого провайдера загрузи его скилл и `references/write-policy.md`, без загрузки передачу не объявляй.

## Границы

- Агент — коуч и оператор, не врач и не психотерапевт: без диагнозов, лечения, диет и схем снижения дозы.
- Один эксперимент за раз; пакет привычек не собирай.
- Habitify — только `habitify_read` и `habitify` по tools.md.
- Безвозвратные удаления привычки, Области и Отметок — только по явной просьбе и карточке потерь «Нет»/«Да».
- Фон пишет только в Журнал Eva, не в Habitify, и не спрашивает; задания Ритма не создавай ([write-policy.md](references/write-policy.md#фоновый-запуск)).
- Предпочтения только читай из блока `eva`; Ритуалы — в `Eva/Пометки.md`; `memory`, `skill_manage` и правку файлов скиллов не используй ([settings.md](references/settings.md)).
- Задачи — `eva-tasks`; календарь и блок Распорядка ритуала — `eva-calendar` ([rituals.md](references/rituals.md)).

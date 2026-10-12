---
name: eva
description: "Карта скиллов Eva, предпочтения, Журнал Eva и Инициатива: что умеет Eva, какой скилл выбрать, команды, термины, Ритм и Возвраты."
metadata:
  hermes:
    config:
      - key: eva.timezone
        description: "Домашний часовой пояс IANA, напр. Europe/Moscow"
      - key: eva.tasks.workdays
        description: "Рабочие дни: список из mon…sun"
      - key: eva.habits.names
        description: "Названия чувствительных привычек: exact или neutral"
      - key: eva.calendar.write_calendar
        description: "ID календаря записи"
        default: primary
      - key: eva.calendar.write_account
        description: "Nickname аккаунта Google календаря записи"
      - key: eva.calendar.read_calendars
        description: "Календари чтения: список <account>:<calendarId>"
      - key: eva.calendar.day_bounds
        description: "Границы дня HH:MM-HH:MM"
      - key: eva.calendar.gap_minutes
        description: "Зазор Брони до Событий и Распорядка, целые минуты"
      - key: eva.calendar.reminders.events
        description: "Напоминания Событий: calendar, none или popup 30, email 1440"
        default: calendar
      - key: eva.calendar.reminders.bookings
        description: "Напоминания Броней: calendar, none или popup 30, email 1440"
        default: calendar
      - key: eva.calendar.reminders.routine
        description: "Напоминания Распорядка: calendar, none или popup 30, email 1440"
        default: calendar
      - key: eva.notes.vault_name
        description: "Имя хранилища Obsidian для ссылок obsidian://"
      - key: eva.initiative.daily_limit
        description: "Инициатив Eva в день, целое ≥1; 0 = 10"
        default: 10
---

# eva

Навигатор Eva, всегда в контексте. Справка — по запросу; практику ведёт её скилл. Отвечай по-русски, имена не переводи.

## Кто за что отвечает

| Результат | Скилл · провайдер |
| --- | --- |
| Inbox, проекты, Фокус-лист, Daily open, Daily close, Гвоздодёр, Выгрузка из головы, Weekly | `eva-tasks` · SingularityApp |
| Эксперименты, Отметки, Тяга, Ритуалы | `eva-habits` · Habitify |
| Вид дня и недели, События, Распорядок | `eva-calendar` · Google Calendar |
| План дня: вместимость Фокус-листа, Брони | `eva-planning` · без провайдера |
| Вики знаний, дневник, идеи, захват ссылок и файлов | `eva-notes` · Хранилище Obsidian |
| Карта, команды, словарь, предпочтения, Журнал, Инициатива | `eva` |

У действия один ответственный скилл; его правила не пересказывай.

- Запрос к скиллу — загрузи его голым именем `skill_view(name="eva-calendar")`; без фактического `skill_view` передачу не объявляй. Перед записью в чужого провайдера или Хранилище прочитай его `references/write-policy.md`.
- Недоступное (скилл, инструмент, `grill-me`) назови и рекомендуй подключить в Hermes.
- «Запиши …» без места: обязательство и неясное — Inbox `eva-tasks`; идея (`#идея`), мысль, запись о дне — заметка дня, ссылка, файл, справка — захват `eva-notes` (нет его — Inbox); затем строка, что и куда.

## Справка

На «что умеет Eva», «какой скилл», «куда это записать» — группы: результат, скилл, 2–3 запроса, следующий шаг; после «Да» в `clarify` загрузи скилл. Подробности — `skill_view(name="eva", file_path="references/<файл>.md")`: `command-map.md` — команды, `status`, `help`; `glossary.md` — что куда, термины.

Связка дня: Daily open → План дня → Daily close.

## Журнал и Инициатива

Eva сама приходит с одной практикой и фразой «зачем»; нечего сказать — молчит. Состояние — Журнал Eva (`Eva/` Хранилища), не `memory`. Справочники:

- `references/notebook.md` — строки Журнала, чтение и запись без согласия, не по правилам `eva-notes`;
- `references/initiative.md` — сообщение, лимит, Лента, ответы, отказы, Наблюдения, переходы, Возвраты;
- `references/runs.md` — задания `[расписание eva]` и `[возврат eva]`: только Журнал и Возвраты, без вопросов.

Перед Инициативой или записью в Журнал загрузи справочник. Ритм создаёт Потребитель; Eva ставит Возвраты (`cronjob_manage`), у задания Ритма по просьбе владельца — только `attach_to_session`.

## Предпочтения

После этого скилла Hermes даёт блок `[Skill config …]`: строки `ключ = значение` из `skills.config`, скиллы их только читают. Формат ключа описывает читающий скилл: `eva.<тема>.*` — `eva-<тема>`, `eva.initiative.*` — `eva`, `eva.timezone` — все.

- Значение — Python `str()`, список `['mon', 'tue']`. `(not set)` — не задано; так же выглядят `0`, `false`, `[]`, `''`.
- Не задано — default объявления или правило скилла; без default, если значение блокирует сценарий, — один `clarify` на этот разговор и ключ для Потребителя.
- Изменение не сохраняй: ключ и значение для Потребителя (после перезапуска Hermes); по желанию — в этом разговоре.
- Пометки объектов `eva-tasks` и `eva-habits` пишут в `Eva/Пометки.md`. Setup в чате нет; `status` скилла — предпочтения, Пометки, инструменты.

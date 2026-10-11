---
name: jedikit
description: "Карта скиллов JediKit и предпочтения владельца: что умеет JediKit, какой скилл выбрать, команды и термины («куда это записать», «что такое Бронь»)."
metadata:
  hermes:
    config:
      - key: jedikit.timezone
        description: "Домашний часовой пояс IANA, напр. Europe/Moscow"
      - key: jedikit.tasks.workdays
        description: "Рабочие дни: список из mon…sun"
      - key: jedikit.habits.names
        description: "Названия чувствительных привычек: exact или neutral"
      - key: jedikit.calendar.write_calendar
        description: "ID календаря записи"
        default: primary
      - key: jedikit.calendar.write_account
        description: "Nickname аккаунта Google календаря записи"
      - key: jedikit.calendar.read_calendars
        description: "Календари чтения: список <account>:<calendarId>"
      - key: jedikit.calendar.day_bounds
        description: "Границы дня HH:MM-HH:MM"
      - key: jedikit.calendar.gap_minutes
        description: "Зазор Брони до Событий и Распорядка, целые минуты"
      - key: jedikit.calendar.reminders.events
        description: "Напоминания Событий: calendar, none или popup 30, email 1440"
        default: calendar
      - key: jedikit.calendar.reminders.bookings
        description: "Напоминания Броней: calendar, none или popup 30, email 1440"
        default: calendar
      - key: jedikit.calendar.reminders.routine
        description: "Напоминания Распорядка: calendar, none или popup 30, email 1440"
        default: calendar
---

# jedikit

Карта JediKit и предпочтения владельца; скилл всегда в контексте. Справку даёт по запросу, но не является обязательным шагом перед запросом и чужие сценарии не выполняет. Отвечай по-русски, имена скиллов и команд сохраняй.

## Кто за что отвечает

| Результат | Скилл · провайдер |
| --- | --- |
| Inbox, проекты, Фокус-лист, Daily open, Daily close, Weekly | `jedikit-tasks` · SingularityApp |
| Эксперименты с поведением, Отметки, Тяга, Ритуалы | `jedikit-habits` · Habitify |
| Вид дня и недели, События, Распорядок | `jedikit-calendar` · Google Calendar |
| План дня: вместимость Фокус-листа, Брони | `jedikit-planning` · без провайдера |
| Карта, команды, словарь, предпочтения | `jedikit` |

У действия один ответственный скилл; его правила не пересказывай.

- Запрос к скиллу — сразу загрузи его голым именем: `skill_view(name="jedikit-calendar")`; сценарий ведёт он. Без фактического `skill_view` передачу не объявляй.
- Перед записью в чужого провайдера прочитай `skill_view(name="<владелец>", file_path="references/write-policy.md")`.
- Недоступные скилл, MCP-сервер или `grill-me` назови и рекомендуй подключить в Hermes; сам ничего не устанавливай.

## Справка

На «что умеет JediKit», «какой скилл», «куда это записать» отвечай нужными группами: результат, скилл, 2–3 запроса обычными словами, следующий шаг. Переход предложи одной строкой, после «Да» в `clarify` загрузи скилл. Подробности — `skill_view(name="jedikit", file_path="references/<файл>.md")`: `command-map.md` — команды, `status`, `help`; `glossary.md` — что куда относится и термины.

## Связка дня

1. Утром `jedikit-tasks`, Daily open «Начнём день»: хвосты, Ресурс дня, Фокус-лист.
2. Затем `jedikit-planning`, План дня «Спланируй день»: окна, вместимость, Брони.
3. Вечером `jedikit-tasks`, Daily close «Закроем день».

Приглашения к обзорам по расписанию заводит Потребитель; скиллы задания не создают.

## Предпочтения

Hermes добавляет после этого скилла блок `[Skill config (from …/config.yaml): …]` со строками `ключ = значение`: предпочтения владельца, заданные Потребителем в `skills.config`. Формат ключа описывает читающий его скилл: `jedikit.tasks.*` — `jedikit-tasks`, `jedikit.habits.*` — `jedikit-habits`, `jedikit.calendar.*` — `jedikit-calendar`, `jedikit.timezone` — все.

- Значение напечатано Python `str()`, список — `['mon', 'tue']`. `(not set)` — не задано; так же выглядят `0`, `false`, `[]`, пустая строка.
- Скиллы предпочтения только читают: не пишут в `memory`, файлы или куда-либо ещё.
- Не задано — действует default объявления или правило читающего скилла. Если default нет и значение блокирует сценарий, спроси одним `clarify` для этого разговора и назови ключ для Потребителя. Ненужное не спрашивай.
- Просьба изменить предпочтение («рабочие дни вт–сб»): не сохраняй, назови ключ и новое значение для Потребителя; по желанию применяй только в этом разговоре. Новое значение действует после перезапуска Hermes.
- Пометки объектов — Режимы поддерева и Ритуалы — сохраняют в `memory` `jedikit-tasks` и `jedikit-habits` по ходу работы.
- Отдельного setup в чате нет; `status` скилла показывает предпочтения, пометки и доступность инструментов.

---
name: eva
description: "Карта скиллов Eva и предпочтения владельца: что умеет Eva, какой скилл выбрать, команды и термины («куда это записать», «что такое Бронь»)."
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
---

# eva

Карта Eva и предпочтения владельца; всегда в контексте. Справка — по запросу, не обязательный шаг; чужие сценарии не выполняет. Отвечай по-русски, имена скиллов и команд не переводи.

## Кто за что отвечает

| Результат | Скилл · провайдер |
| --- | --- |
| Inbox, проекты, Фокус-лист, Daily open, Daily close, Weekly | `eva-tasks` · SingularityApp |
| Эксперименты, Отметки, Тяга, Ритуалы | `eva-habits` · Habitify |
| Вид дня и недели, События, Распорядок | `eva-calendar` · Google Calendar |
| План дня: вместимость Фокус-листа, Брони | `eva-planning` · без провайдера |
| Вики знаний, дневник, идеи, захват ссылок и файлов | `eva-notes` · Хранилище Obsidian |
| Карта, команды, словарь, предпочтения | `eva` |

У действия один ответственный скилл; его правила не пересказывай.

- Запрос к скиллу — сразу загрузи его голым именем `skill_view(name="eva-calendar")`; сценарий ведёт он. Без фактического `skill_view` передачу не объявляй.
- Перед записью в чужого провайдера или Хранилище прочитай `skill_view(name="<владелец>", file_path="references/write-policy.md")`.
- Недоступные скилл, MCP-сервер или `grill-me` назови и рекомендуй подключить в Hermes; сам не устанавливай.
- «Запиши …» без префиксов: обязательство, действие или неясное — Inbox `eva-tasks`; у `eva-notes`: идея — строка с `#идея` в заметке дня; мысль, наблюдение, запись о дне — заметка дня; ссылка, файл, справка — захват. Затем — строка, что и куда; владелец поправит. Нет `eva-notes` — назови и предложи Inbox.

## Справка

На «что умеет Eva», «какой скилл», «куда это записать» отвечай группами: результат, скилл, 2–3 запроса обычными словами, следующий шаг; переход — одной строкой, после «Да» в `clarify` загрузи скилл. Подробности — `skill_view(name="eva", file_path="references/<файл>.md")`: `command-map.md` — команды, `status`, `help`; `glossary.md` — что куда, термины.

Связка дня: Daily open `eva-tasks` → План дня `eva-planning` → Daily close `eva-tasks`. Приглашения по расписанию заводит Потребитель; скиллы задания не создают.

## Предпочтения

После этого скилла Hermes добавляет блок `[Skill config (from …/config.yaml): …]` со строками `ключ = значение` из `skills.config` Потребителя; скиллы их только читают и никуда не записывают. Формат ключа описывает читающий его скилл: `eva.<тема>.*` — `eva-<тема>` (`tasks`, `habits`, `calendar`, `notes`), `eva.timezone` — все.

- Значение напечатано Python `str()`, список — `['mon', 'tue']`. `(not set)` — не задано; так же выглядят `0`, `false`, `[]`, пустая строка.
- Не задано — default объявления или правило читающего скилла; без default, если значение блокирует сценарий, спроси одним `clarify` на этот разговор и назови ключ для Потребителя. Ненужное не спрашивай.
- Просьбу изменить предпочтение не сохраняй: назови ключ и новое значение для Потребителя, действует после перезапуска Hermes; по желанию применяй только в этом разговоре.
- Пометки объектов (Режимы поддерева, Ритуалы) сохраняют в `memory` `eva-tasks` и `eva-habits` по ходу работы.
- Setup в чате нет; `status` скилла показывает предпочтения, пометки и доступность инструментов.

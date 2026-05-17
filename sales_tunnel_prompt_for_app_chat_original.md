# Prompt for another chat / developer: implement Structured Sales Tunnel

У нас есть приложение для звонков/CRM по недвижимости Дубая. Я загрузил структурированную базу sales tunnel из файла `structured_sales_tunnel_database.json` или Excel `structured_sales_tunnel_database.xlsx`.

## Что это такое
Это не просто скрипт звонка. Это **structured sales tunnel**: сценарии и ответы продавца зависят от стадии лида и ответа клиента.

Основная логика:
1. Новый лид попадает в `S01 New lead / 1st touch`.
2. Агент использует opening script.
3. После ответа клиента приложение переводит лида в нужный node:
   - `YES` → qualification/discovery
   - `NO` → short response + follow-up/close
   - `NOT INTERESTED` → clarify if not now or never
   - `WORKING WITH SOMEONE` → competitive objection
   - `ALREADY BOUGHT` → future plans/referral/nurture
   - `DID NOT LEAVE` → apologize/clarify/close
   - any objection → objection router
4. В каждом node приложение показывает правильный script, next action, status update и follow-up rule.
5. Тексты нельзя хардкодить в UI: они должны читаться из базы по ID.

## Главные сущности
Создай/адаптируй такие таблицы или коллекции:

### stages
`stage_id`, `stage_name`, `purpose`, `entry_condition`, `success_exit`, `next_stage`

### decision_nodes
`node_id`, `stage_id`, `node_type`, `trigger`, `linked_record_ids`, `routing_rule`, `next_route`, `developer_notes`

### scripts
`script_id`, `stage_id`, `category`, `trigger_label`, `channel`, `touch`, `step_order`, `script_text`, `suggested_next_action`, `source_cell`

### objections
`objection_id`, `category`, `trigger_label`, `lead_phrase`, `response_text`, `next_action`, `stage_id`, `source_cell`

### follow_up_rules
`follow_up_id`, `condition`, `channel`, `cadence_or_touch`, `message_or_rule`, `next_action`, `source_cell`

### small_talk_topics
`topic_id`, `topic`, `opener_text`, `usage`, `source_range`

### lead_sessions
`lead_id`, `current_stage_id`, `current_node_id`, `lead_status`, `detected_objection_id`, `next_action`, `follow_up_due_at`, `notes`

### lead_events
`event_id`, `lead_id`, `timestamp`, `agent_input`, `selected_script_id`, `client_reply_type`, `old_stage_id`, `new_stage_id`, `next_action`

## UI, который мне нужен
Сделай экран sales tunnel:

- Левая панель: карточка лида, статус, источник, последний контакт, next follow-up.
- Центральная панель: текущий script text.
- Правая панель: кнопки выбора ответа клиента:
  `YES`, `NO`, `NOT INTERESTED`, `WORKING WITH SOMEONE`, `ALREADY BOUGHT`, `DID NOT LEAVE`, `OBJECTION`, `NO ANSWER`, `CLOSING`, `FOLLOW UP`.
- Если выбран `OBJECTION`, показывай список objection categories из таблицы `objections`.
- После выбора app обновляет `current_stage_id`, `current_node_id`, `lead_status`, создаёт запись в `lead_events` и показывает следующий script.

## Важные правила маршрутизации
- `S01` = первый контакт.
- `S02` = qualification questions.
- `S03` = education/value framing: off-plan, EOI, forced savings, Zoom proposal.
- `S04` = objection handling.
- `S05` = follow-up cadence: call/WhatsApp/email.
- `S06` = warm client nurturing + small talk.
- `S07` = closing.
- `S08` = disqualification / not now.

## Что сделать
1. Импортируй JSON/Excel в seed data.
2. Создай data model и migration/seed file.
3. Построй routing function:

```ts
function getNextSalesStep(session, clientReply) {
  // 1. classify clientReply
  // 2. find matching decision_node
  // 3. return scripts/objections/follow_up for next UI step
  // 4. return status/stage updates
}
```

4. Сделай UI так, чтобы агент не видел огромный spreadsheet, а видел только следующий лучший шаг.
5. Добавь возможность редактировать scripts/admin panel позже, но сейчас достаточно seed-data.

## Acceptance criteria
- Агент может начать с нового лида и пройти минимум flow: first touch → yes → qualification → objection → response → close/follow-up.
- Все script texts берутся из базы.
- Каждый клик агента сохраняется в `lead_events`.
- Повторяющиеся возражения из исходника не дублируются; используется единая таблица `objections`.
- Follow-up protocol работает отдельно от scripts и может создавать next follow-up task.

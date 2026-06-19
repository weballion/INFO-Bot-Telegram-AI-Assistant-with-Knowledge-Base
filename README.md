# HairBook — Фаза 1 (MVP)
## Telegram-бот для управления записями

---

## 1. Что делаем в Фазе 1

Минимальный рабочий продукт: Telegram-бот, через который мастер **текстом или голосом** управляет расписанием. Никакого веб-интерфейса. Тестовые данные — без DSGVO.

### Что входит

- Диалог на естественном языке (DE/RU): «запиши Анну на завтра в 14:00 на стрижку»
- Голосовые сообщения (транскрибация через Groq Whisper, бесплатно)
- Reply Keyboard — меню с кнопками, которое раскрывается/сворачивается
- Inline кнопки внутри диалога (выбор услуги, даты, слота)
- Проверка рабочих часов, перерыва и свободных слотов
- Создание / отмена / перенос записей
- Просмотр расписания (день, неделя, свободные окна)


### Что НЕ входит

- Веб-интерфейс
- DSGVO-функции (согласия, аудит-лог, экспорт)
- WhatsApp-уведомления
- Напоминания клиентам
- Фото клиентов
- Поддержка нескольких мастеров

---

## 2. Стек и сервисы

| Компонент | Сервис | Стоимость |
|-----------|--------|-----------|
| Оркестрация | **n8n** self-hosted на Railway | €0 (Railway free credit $5/мес) |
| База данных | **Supabase** PostgreSQL (Frankfurt) | €0 (free tier) |
| Telegram-бот | n8n Telegram Trigger + BotFather | €0 |
| Голос → текст | **Groq Whisper** (whisper-large-v3-turbo) | €0 (2000 запр/день) |
| Понимание текста | **Groq LLM** (llama-3.3-70b-versatile) | €0 (14 400 запр/день) |
| **Итого** | | **€0/мес** |

### Почему Groq, а не OpenAI

- Полностью бесплатно, без привязки карты
- Whisper: 2000 аудиозапросов/день — для 1 мастера огромный запас
- LLM: Llama 3.3 70B — качество на уровне GPT-4o, 14 400 запр/день
- Скорость: 300–800 токенов/сек (быстрее всех провайдеров)

---

## 3. Архитектура

```
 МАСТЕР (Telegram)
     │
     │ текст / голос / кнопка меню
     ▼
┌─────────────┐
│  n8n        │
│  (Railway)  │
│             │
│  ┌──────────────────────────────────────┐
│  │ Workflow: Telegram Handler           │
│  │                                      │
│  │  [Telegram Trigger]                  │
│  │       │                              │
│  │  [IF: authorized chat_id?]           │
│  │       │                              │
│  │  [Switch: text / voice]              │
│  │       │           │                  │
│  │    text        voice                 │
│  │       │           │                  │
│  │       │     [Download .ogg]          │
│  │       │     [Groq Whisper]           │
│  │       └─────┬─────┘                  │
│  │             │                        │
│  │      unified text                    │
│  │             │                        │
│  │   [Groq LLM: parse intent → JSON]   │
│  │             │                        │
│  │   [Switch: action]                   │
│  │    │   │   │   │   │   │   │         │
│  │   new view free cancel move client  │
│  │    │   │   │   │   │   │   │         │
│  │   [Supabase operations]              │
│  │             │                        │
│  │   [Format + Reply Keyboard]          │
│  │             │                        │
│  │   [Telegram: send reply]             │
│  └──────────────────────────────────────┘
│
└──────────── Supabase PostgreSQL (Frankfurt) ─────────
```

---

## 4. Telegram UI: Reply Keyboard (меню кнопок)

### Как работает

**Reply Keyboard** — постоянная клавиатура внизу чата. Мастер нажимает кнопку
«☰ Меню» рядом с полем ввода — раскрываются кнопки. Нажимает снова — прячутся.
Кнопка отправляет текст боту как обычное сообщение, LLM его парсит так же, как
свободный текст — никакой специальной логики не нужно.

```
┌────────────────────────────────────────┐
│  📅 Сегодня       📅 Завтра           │
│  📆 Неделя        🟢 Свободные окна   │
│  ➕ Новая запись  ❌ Отменить запись   │
│  📋 Услуги                            │
└────────────────────────────────────────┘
```

7 кнопок — удобно одной рукой на телефоне.

### JSON для n8n (reply_markup)

```json
{
  "keyboard": [
    ["📅 Сегодня",      "📅 Завтра"],
    ["📆 Неделя",       "🟢 Свободные окна"],
    ["➕ Новая запись", "❌ Отменить запись"],
    ["📋 Услуги"]
  ],
  "resize_keyboard": true,
  "one_time_keyboard": false
}
```

`resize_keyboard: true` — кнопки компактные, не на весь экран.
`one_time_keyboard: false` — меню остаётся после нажатия.

### Как кнопки попадают в LLM

Кнопка «📅 Сегодня» → бот получает текст `"📅 Сегодня"` → LLM парсит как
`{action: "view_schedule", period: "today"}`. Никакого специального кода.
LLM понимает эмодзи и любой текст.

---

## 5. Inline кнопки — уточняющий диалог (новая запись)

Когда мастер нажимает **➕ Новая запись** или говорит «запиши Анну», бот
задаёт уточняющие вопросы через inline-кнопки под сообщением.

```
Бот: 👤 Клиент?

[Max Mustermann]  [Erika Musterfrau]
[Anna Beispiel]   [➕ Новый клиент]

─────────────────────────────────────

Бот: ✂️ Услуга?

[Стрижка ж. 60м]     [Стрижка м. 30м]
[Окрашивание 120м]   [Мелирование 90м]
[Балаяж 150м]        [Укладка 30м]
[Стрижка+окраска]    [Другое...]

─────────────────────────────────────

Бот: 📅 Дата?

[Сегодня Чт 19.06]  [Завтра Пт 20.06]
[Сб 21.06]          [Пн 23.06]
[Другая дата...]

─────────────────────────────────────

Бот: 🕐 Свободные слоты на Пт 20.06:

[09:00]  [10:00]  [11:30]
[14:00]  [15:30]  [16:45]

─────────────────────────────────────

Бот: ✅ Готово!

👤 Max Mustermann
✂️ Стрижка (60 мин)
📅 Пт, 20.06.2026
🕐 14:00 – 15:00
```

Последние 3–4 клиента в кнопках подтягиваются из Supabase
(последние по `created_at` или `last_visit`).

---

## 6. База данных

### Таблица `stylists`

```sql
CREATE TABLE stylists (
  id               UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
  name             TEXT    NOT NULL,
  telegram_chat_id BIGINT  UNIQUE NOT NULL,
  language         TEXT    DEFAULT 'de',  -- 'de' | 'ru'
  buffer_minutes   INT     DEFAULT 15,    -- буфер между записями
  working_hours    JSONB   NOT NULL,
  created_at       TIMESTAMPTZ DEFAULT now()
);
```

### Рабочие часы и дни — тестовые данные

Типичный немецкий салон: среда — выходной, четверг — длинный день.

```sql
INSERT INTO stylists (name, telegram_chat_id, language, buffer_minutes, working_hours)
VALUES (
  'Maria Testova',
  YOUR_CHAT_ID,
  'ru',
  15,
  '{
    "mon": {"start":"09:00","end":"18:00","break_start":"13:00","break_end":"14:00"},
    "tue": {"start":"09:00","end":"18:00","break_start":"13:00","break_end":"14:00"},
    "wed": null,
    "thu": {"start":"09:00","end":"20:00","break_start":"13:00","break_end":"14:00"},
    "fri": {"start":"09:00","end":"18:00","break_start":"13:00","break_end":"14:00"},
    "sat": {"start":"09:00","end":"14:00","break_start":null,"break_end":null},
    "sun": null
  }'::jsonb
);
```

Ключи `null` = выходной день (среда, воскресенье).
`break_start: null` = без обеда (суббота).

### Таблица `services`

```sql
CREATE TABLE services (
  id                  UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
  stylist_id          UUID    REFERENCES stylists(id) ON DELETE CASCADE,
  name                TEXT    NOT NULL,
  default_duration_min INT    NOT NULL DEFAULT 60,
  is_active           BOOLEAN DEFAULT true
);
```

### Услуги — тестовые данные

```sql
INSERT INTO services (stylist_id, name, default_duration_min) VALUES
  -- Стрижки
  (STYLIST_ID, 'Damenhaarschnitt / Стрижка женская',      60),
  (STYLIST_ID, 'Herrenhaarschnitt / Стрижка мужская',     30),
  (STYLIST_ID, 'Kinderhaarschnitt / Стрижка детская',     30),

  -- Окрашивание
  (STYLIST_ID, 'Färben komplett / Окрашивание полное',   120),
  (STYLIST_ID, 'Ansatz färben / Окрашивание корни',       60),
  (STYLIST_ID, 'Strähnchen / Мелирование',                90),
  (STYLIST_ID, 'Balayage / Балаяж',                      150),

  -- Уход
  (STYLIST_ID, 'Haarkur / Маска для волос',               30),
  (STYLIST_ID, 'Keratin-Behandlung / Кератин',           180),

  -- Стайлинг
  (STYLIST_ID, 'Föhnen / Укладка феном',                  30),
  (STYLIST_ID, 'Hochsteckfrisur / Причёска торжество',    60),

  -- Комбо
  (STYLIST_ID, 'Schnitt + Färben / Стрижка + окраска',   150),
  (STYLIST_ID, 'Schnitt + Strähnchen / Стрижка + мелир', 120);
```

13 услуг. В Фазе 2 переносятся в раздел настроек веб-интерфейса.

### Таблица `clients`

```sql
CREATE TABLE clients (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name       TEXT NOT NULL,
  phone      TEXT,
  notes      TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

### Тестовые клиенты

```sql
INSERT INTO clients (name, phone, notes) VALUES
  ('Max Mustermann',   '+49 170 0000001', 'Stammkunde, kurzer Haarschnitt'),
  ('Erika Musterfrau', '+49 170 0000002', 'Blondierung alle 8 Wochen'),
  ('Anna Beispiel',    '+49 170 0000003', 'Empfindliche Kopfhaut'),
  ('Иван Тестов',      '+49 170 0000004', 'Предпочитает говорить по-русски'),
  ('Лиза Пробная',     '+49 170 0000005', 'Новый клиент');
```

### Таблица `appointments`

```sql
CREATE TABLE appointments (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  stylist_id  UUID REFERENCES stylists(id),
  client_id   UUID REFERENCES clients(id),
  service_id  UUID REFERENCES services(id),
  start_time  TIMESTAMPTZ NOT NULL,
  end_time    TIMESTAMPTZ NOT NULL,
  duration_min INT NOT NULL,
  status      TEXT DEFAULT 'confirmed'
              CHECK (status IN ('confirmed','cancelled','completed','no_show')),
  notes       TEXT,
  created_at  TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_appointments_time
  ON appointments (stylist_id, start_time)
  WHERE status = 'confirmed';
```

### Таблица `conversation_state`

```sql
CREATE TABLE conversation_state (
  chat_id      BIGINT PRIMARY KEY,
  current_step TEXT,
  context      JSONB DEFAULT '{}',
  updated_at   TIMESTAMPTZ DEFAULT now()
);
```

---

## 7. Логика проверки рабочих часов и слотов

Три проверки при любой операции со временем:

```
1. Рабочий день?
   working_hours["wed"] = null → "В среду не работаю 🙅"

2. Время в рамках дня?
   start < "09:00" или end > "18:00"
   → "Последний слот в 17:00, работаю до 18:00"

3. Попадает на перерыв?
   пересечение с break_start/break_end
   → "В это время обед (13:00–14:00), могу в 12:00 или 14:00"
```

### Function Node n8n — расчёт свободных слотов

```javascript
const DAY_MAP = {
  0:'sun', 1:'mon', 2:'tue', 3:'wed',
  4:'thu', 5:'fri', 6:'sat'
};

const date    = new Date(input.date);
const dayKey  = DAY_MAP[date.getDay()];
const hours   = input.working_hours[dayKey];

if (!hours) return { free_slots: [], reason: 'day_off' };

const toMin  = t => { const [h,m] = t.split(':').map(Number); return h*60+m; };
const fromMin = m => `${String(Math.floor(m/60)).padStart(2,'0')}:${String(m%60).padStart(2,'0')}`;

// Занятые блоки: перерыв + подтверждённые записи + буфер
const busy = [];

if (hours.break_start) {
  busy.push({ start: toMin(hours.break_start), end: toMin(hours.break_end) });
}

for (const appt of input.appointments) {
  busy.push({
    start: toMin(appt.start_time.slice(11,16)),
    end:   toMin(appt.end_time.slice(11,16)) + input.buffer_minutes
  });
}

busy.sort((a,b) => a.start - b.start);

// Найти слоты нужной длины
const workStart = toMin(hours.start);
const workEnd   = toMin(hours.end);
const duration  = input.requested_duration;
const free_slots = [];
let cursor = workStart;

for (const block of busy) {
  if (block.start - cursor >= duration) {
    free_slots.push({ start: fromMin(cursor), end: fromMin(cursor + duration) });
  }
  cursor = Math.max(cursor, block.end);
}

if (workEnd - cursor >= duration) {
  free_slots.push({ start: fromMin(cursor), end: fromMin(cursor + duration) });
}

return { free_slots, day_off: false };
```

---

## 8. Промпт для Groq LLM (парсинг намерений)

```
SYSTEM PROMPT:

Du bist ein Assistent für einen Friseursalon-Terminkalender.
Analysiere die Nachricht und extrahiere die Aktion als JSON.

Mögliche Aktionen:
- "new_appointment"    — Termin anlegen
- "cancel_appointment" — Termin absagen
- "move_appointment"   — Termin verschieben
- "view_schedule"      — Termine anzeigen
- "check_free_slots"   — Freie Slots anzeigen

- "list_services"      — Dienstleistungen anzeigen
- "unknown"            — Aktion unklar

Antworte NUR mit JSON:

{
  "action": "new_appointment",
  "client_name": "Anna",
  "service": "Haarschnitt",
  "date": "2026-06-20",
  "time": "14:00",
  "duration_min": null,
  "notes": null,
  "period": null,
  "missing_fields": [],
  "language": "de"
}

Regeln:
- Heute ist {CURRENT_DATE}, Wochentag: {WEEKDAY}
- "morgen"/"завтра" = tomorrow, "heute"/"сегодня" = today
- "übermorgen"/"послезавтра" = in 2 Tagen
- При нажатии кнопок меню (📅 Сегодня и т.д.) — парсить как обычный текст
- missing_fields = поля, которых не хватает для выполнения действия
- language = "de" или "ru"
- period: "today" | "tomorrow" | "week" для view_schedule
```

### Примеры

| Ввод | action | детали |
|------|--------|--------|
| «запиши Анну завтра в 14:00 стрижка» | new_appointment | client: Анна, date: завтра, time: 14:00 |
| «📅 Сегодня» (кнопка) | view_schedule | period: today |
| «was hab ich morgen?» | view_schedule | period: tomorrow |
| «отмени Лизу в пятницу» | cancel_appointment | client: Лиза, date: пятница |
| «🟢 Свободные окна» (кнопка) | check_free_slots | date: today |
| «📋 Услуги» (кнопка) | list_services | — |
| голос: «перенеси Макса с трёх на пять» | move_appointment | client: Макс, time_from: 15:00, time_to: 17:00 |

---

## 9. Формат ответов бота

### Расписание на день
```
📅 Donnerstag, 19.06.2026

09:00 – 10:00  Max Mustermann — Стрижка женская
10:15 – 12:15  Erika Musterfrau — Окрашивание
      ☕ 13:00 – 14:00  Mittagspause
14:00 – 15:30  Anna Beispiel — Мелирование
15:45 – 17:00  ——— frei (75 мин) ———
17:00 – 18:00  Лиза Пробная — Укладка

✂️ 4 записи  |  🟢 75 мин свободно
```

### Свободные слоты
```
🟢 Freie Slots — Пт 20.06 (для услуги 60 мин):

• 09:00 – 10:00
• 11:15 – 12:15
• 14:00 – 15:00
• 16:00 – 17:00

Записать кого-то?
```

### Подтверждение записи
```
✅ Запись создана:

👤 Anna Beispiel
✂️ Мелирование (90 мин)
📅 Пт, 20.06.2026
🕐 14:00 – 15:30
```

### Слот занят — предложение альтернативы
```
⚠️ 14:00 занято (Anna Beispiel, до 15:30 + 15 мин буфер).

Ближайшие свободные слоты:
  • 15:45 – 17:15
  • 17:15 – 18:00  (только 45 мин — короче услуги)

Выбрать 15:45?
```

### Выходной день
```
🙅 Среда — выходной.

Ближайшие рабочие дни: Чт 19.06, Пт 20.06, Сб 21.06
```

---

## 10. n8n Workflows

### Workflow 1: Main Telegram Handler

```
[Telegram Trigger]
  → [IF: authorized?] — нет → [Reply: "Нет доступа"]
  → [Switch: text / voice]
      voice → [GET file from Telegram]
            → [HTTP: Groq Whisper transcribe]
      text  → (pass through)
  → [Merge: unified text]
  → [Set: inject current date/weekday into system prompt]
  → [HTTP: Groq LLM → JSON intent]
  → [JSON Parse]
  → [Switch: action]
      new_appointment    → [Sub-WF: New Appointment]
      view_schedule      → [Supabase: get day/week] → [Format] → [Reply]
      check_free_slots   → [Supabase: get appts] → [Function: calc slots] → [Reply + Inline KB]
      cancel_appointment → [Supabase: find appt] → [Confirm inline] → [Update status]
      move_appointment   → [Sub-WF: Move]
      list_services      → [Supabase: get services] → [Reply: список]
      unknown            → [Reply: "Не понял, попробуй иначе 🤔"]
```

### Workflow 2: New Appointment Sub-workflow

```
[Input: intent JSON]
  → [IF: missing_fields empty?]
      нет → [Save conversation_state]
          → [Telegram: inline KB для первого missing field]
          → СТОП (ждём следующего сообщения)
      да  → [Supabase: find/create client]
          → [Supabase: match service by name (ILIKE)]
          → [Check: working day?] — нет → [Reply: выходной]
          → [Supabase: get appointments for date]
          → [Function: check slot availability]
          → свободно → [Supabase: INSERT appointment]
                     → [Clear conversation_state]
                     → [Reply: ✅ подтверждение]
          → занято   → [Function: find nearest free slot]
                     → [Reply: ⚠️ + альтернативы + inline KB]
```

### Workflow 3: Conversation State (многошаговый диалог)

```
При каждом входящем сообщении:
  [Supabase: GET conversation_state WHERE chat_id = ?]
    → есть активный шаг?
        да  → [Merge context + новый ввод]
            → [Continue New Appointment from saved step]
        нет → [Main Workflow (Workflow 1)]
```

---

## 11. Пошаговый план запуска

### Шаг 1 — Supabase (30 мин)
1. supabase.com → New project → регион **EU Central (Frankfurt)**
2. SQL Editor → выполнить все CREATE TABLE из раздела 6
3. Вставить тестовые данные (мастер, услуги, клиенты)
4. Сохранить: `SUPABASE_URL`, `SUPABASE_ANON_KEY`

### Шаг 2 — Telegram Bot (10 мин)
1. @BotFather → `/newbot` → `hairbook_test_bot`
2. Сохранить: `TELEGRAM_BOT_TOKEN`
3. Написать боту `/start`, сохранить свой `chat_id`

### Шаг 3 — Groq API (5 мин)
1. console.groq.com → Sign up (Google/email, без карты)
2. API Keys → Create → сохранить: `GROQ_API_KEY`

### Шаг 4 — n8n на Railway (20 мин)
1. railway.com → New Project → шаблон n8n
2. Дождаться деплоя, открыть n8n UI
3. Credentials → добавить:
   - Telegram API (bot token)
   - HTTP Header Auth (Groq: `Bearer GROQ_API_KEY`)
   - Supabase или Postgres (URL + key)

### Шаг 5 — Echo Bot тест (15 мин)
```
[Telegram Trigger] → [Telegram: Send Message "Получил: {{text}}"]
```
Убедиться что бот отвечает.

### Шаг 6 — Reply Keyboard (15 мин)
Добавить в ответный узел `reply_markup` из раздела 4.
Убедиться что меню появляется и кнопки отправляют текст.

### Шаг 7 — Whisper (20 мин)
```
[Switch: voice?] → [GET file] → [HTTP: Groq Whisper] → [Reply: транскрипт]
```
Проверить голосовое на DE и RU.

### Шаг 8 — LLM парсинг + view_schedule (30 мин)
```
[Groq LLM] → [JSON parse] → [Supabase: get today's appts] → [Format] → [Reply]
```
Проверить: «📅 Сегодня» и «was hab ich morgen?»

### Шаг 9 — Свободные слоты (30 мин)
Function Node из раздела 7 + inline кнопки со слотами.

### Шаг 10 — Новая запись (1–2 дня)
Sub-workflow + conversation_state + все проверки.

### Шаг 11 — Отмена, перенос (1 день)
Ветки `cancel_appointment` и `move_appointment`.

### Шаг 12 — Финальное тестирование (1 день)
- Все сценарии текстом и голосом на DE и RU
- Граничные случаи: выходной, перерыв, конфликт слотов, несуществующий клиент
- Reply Keyboard + inline кнопки

---

## 12. Что дальше (Фаза 2+)

- **Веб-интерфейс** — календарь, карточки клиентов
- **Настройки** — редактирование услуг, рабочих часов через UI
- **Напоминания** — cron n8n → Telegram мастеру, потом WhatsApp клиентам
- **Фото клиентов** — Supabase Storage + загрузка через Telegram
- **DSGVO** — при переходе на реальных клиентов
- **Несколько мастеров** — stylist_id во все запросы
- **Клиентская самозапись** — Telegram mini-app или веб-форма

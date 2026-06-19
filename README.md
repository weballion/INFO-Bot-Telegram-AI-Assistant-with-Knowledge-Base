# INFO Bot — Telegram AI Assistant with Knowledge Base

Two linked n8n workflows: a Telegram bot with RAG-powered search over a custom knowledge base, and an automated document loader from Google Drive.

**[RU verinfo-bot--telegram-ai-ассистент-с-базой-знаний)**

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    Google Drive                          │
│                 (documents folder)                       │
└────────────────────┬─────────────────────────────────────┘
                     │ new file (checked every hour)
                     ▼
          ┌─────────────────────┐
          │  INFO -> LOADING    │
          │  PDF → chunks →     │
          │  embeddings → DB    │
          └─────────┬───────────┘
                    │
                    ▼
          ┌─────────────────────┐
          │  Supabase           │
          │  • documents (RAG)  │
          │  • chat_logs        │
          └─────────▲───────────┘
                    │
                    │ semantic search
                    │
┌───────────┐     ┌─────────────────────┐     ┌───────────┐
│ Telegram  │────▶│  INFO               │────▶│ Telegram  │
│ (input)   │     │  Agent (GPT-4o)     │     │ (reply)   │
└───────────┘     └─────────────────────┘     └───────────┘
```

---

## INFO — Telegram Bot

The main bot for AI & automation consulting. Accepts text and voice messages, retrieves answers from a custom knowledge base via RAG.

### How it works

1. **Telegram Trigger** — receives an incoming message.
2. **Switch** — routes by type: voice or text.
3. **Voice branch** — downloads audio → transcribes via OpenAI Whisper.
4. **Set Final Text** — builds a unified object with text, session ID, user ID, username.
5. **Save User Message** — logs the message to Supabase (`chat_logs`).
6. **Info Agent (GPT-4o)** — processes the request:
   - **Memory** — PostgreSQL, last 10 messages per session.
   - **DATA_TOOL** — semantic search over the Supabase vector store (`documents`), up to 30 chunks.
7. **Response** — sends the reply back to Telegram.
8. **Save Bot Message** — logs the bot's reply to Supabase (`chat_logs`).

### Features

- Full conversation logging (user + assistant) in Supabase.
- RAG over a knowledge base with real AI implementation cases.
- Prompt flow: identify the problem → search the knowledge base → actionable plan.
- Responses formatted in Telegram Markdown.

---

## INFO -> LOADING — Knowledge Base Loader

Automatically populates the vector database from files in Google Drive. Works in tandem with INFO.

### How it works

1. **Google Drive Trigger** — checks for new files in a specified folder every hour.
2. **Edit Fields** — assigns the `knowledge` category and stores the file ID.
3. **Google Drive Download** — downloads the file, converting Google Docs/Sheets/Slides to PDF.
4. **Extract PDF** — extracts text from the PDF.
5. **Supabase Vector Store** — splits text into chunks (800 chars, 150 overlap), generates OpenAI embeddings, and saves to the `documents` table.

### Features

- Supports all Google Workspace formats (Docs, Sheets, Slides, Drawings → PDF).
- Chunking with overlap to preserve context across boundaries.

---

## Tech Stack

| Component | Technology |
|---|---|
| Orchestration | n8n |
| LLM | OpenAI GPT-4o (via OpenRouter) |
| Transcription | OpenAI Whisper |
| Vector DB | Supabase (pgvector) |
| Embeddings | OpenAI Embeddings |
| Conversation memory | PostgreSQL |
| Chat logging | Supabase (`chat_logs`) |
| Document storage | Google Drive |
| Messenger | Telegram Bot API |

---

## Setup

### Prerequisites

- A running n8n instance (self-hosted or cloud).
- API keys for all services (see below).
- Supabase database with `documents` and `chat_logs` tables and a `match_documents` function.

### Installation

1. Import the workflow JSON files into n8n.
2. Create the required credentials (see table below).
3. Activate **INFO -> LOADING** first, then **INFO**.

### Credentials

| Credential | Workflow | Purpose |
|---|---|---|
| Telegram Bot API | INFO | Receive & send messages |
| OpenAI API (OpenRouter) | INFO | LLM (GPT-4o), transcription |
| OpenAI API (direct) | INFO -> LOADING | Embeddings |
| Supabase API | both | Vector DB, chat logs |
| PostgreSQL | INFO | Conversation memory |
| Google Drive OAuth2 | INFO -> LOADING | File access |

### Supabase Tables

- **`documents`** — vector store (pgvector) with `match_documents` function.
- **`chat_logs`** — conversation log. Fields: `session_id`, `user_id`, `username`, `first_name`, `role`, `content`, `metadata`.

---

## Security

The JSON files **do not contain API keys** — n8n stores credentials separately; only internal IDs are present.

**One exception:** the `Typing 1` node (INFO) has a Telegram bot token hardcoded in its URL. Before publishing, replace it with a placeholder or remove the node (it is disabled and does not affect the workflow).

---

---

# INFO Bot — Telegram AI-ассистент с базой знаний

Два связанных n8n-сценария: Telegram-бот с RAG-поиском по собственной базе знаний и автоматическая загрузка документов из Google Drive.

---

## Архитектура

```
┌──────────────────────────────────────────────────────────┐
│                    Google Drive                          │
│               (папка с документами)                      │
└────────────────────┬─────────────────────────────────────┘
                     │ новый файл (проверка каждый час)
                     ▼
          ┌─────────────────────┐
          │  INFO -> LOADING    │
          │  PDF → чанки →      │
          │  эмбеддинги → БД    │
          └─────────┬───────────┘
                    │
                    ▼
          ┌─────────────────────┐
          │  Supabase           │
          │  • documents (RAG)  │
          │  • chat_logs        │
          └─────────▲───────────┘
                    │
                    │ семантический поиск
                    │
┌───────────┐     ┌─────────────────────┐     ┌───────────┐
│ Telegram  │────▶│  INFO               │────▶│ Telegram  │
│ (вход)    │     │  Инфо-агент (GPT-4o)│     │ (ответ)   │
└───────────┘     └─────────────────────┘     └───────────┘
```

---

## INFO — Telegram-бот

Основной бот для консультаций по AI и автоматизации. Принимает текст и голос, ищет ответы в собственной базе знаний через RAG.

### Как работает

1. **Telegram Trigger** — получает входящее сообщение.
2. **Switch** — определяет тип: голосовое или текстовое.
3. **Голосовая ветка** — скачивает аудио → транскрибирует через OpenAI Whisper.
4. **Set Final Text** — формирует объект с текстом, session ID, user ID, username.
5. **Save User Message** — сохраняет сообщение в Supabase (`chat_logs`).
6. **Инфо-агент (GPT-4o)** — обрабатывает запрос:
   - **Память** — PostgreSQL, 10 последних сообщений сессии.
   - **DATA_TOOL** — семантический поиск по векторной базе Supabase (`documents`), до 30 чанков.
7. **Response** — отправляет ответ в Telegram.
8. **Save Bot Message** — сохраняет ответ бота в Supabase (`chat_logs`).

### Особенности

- Полное логирование диалогов (user + assistant) в Supabase.
- RAG по базе знаний с реальными кейсами внедрения AI.
- Промпт: выяснение проблемы → поиск в базе → конкретный план действий.
- Ответы в формате Telegram Markdown.

---

## INFO -> LOADING — Загрузка документов в базу знаний

Автоматически наполняет векторную базу из файлов в Google Drive. Работает в связке с INFO.

### Как работает

1. **Google Drive Trigger** — каждый час проверяет новые файлы в заданной папке.
2. **Edit Fields** — присваивает категорию `knowledge` и сохраняет ID файла.
3. **Google Drive Download** — скачивает файл, конвертируя Google Docs/Sheets/Slides в PDF.
4. **Extract PDF** — извлекает текст из PDF.
5. **Supabase Vector Store** — разбивает текст на чанки (800 символов, overlap 150), генерирует эмбеддинги через OpenAI и сохраняет в таблицу `documents`.

### Особенности

- Поддержка форматов Google Workspace (Docs, Sheets, Slides, Drawings → PDF).
- Разбиение на чанки с перекрытием для сохранения контекста.

---

## Стек технологий

| Компонент | Технология |
|---|---|
| Оркестрация | n8n |
| LLM | OpenAI GPT-4o (через OpenRouter) |
| Транскрибация | OpenAI Whisper |
| Векторная БД | Supabase (pgvector) |
| Эмбеддинги | OpenAI Embeddings |
| Память диалогов | PostgreSQL |
| Логирование | Supabase (`chat_logs`) |
| Хранение документов | Google Drive |
| Мессенджер | Telegram Bot API |

---

## Настройка и запуск

### Предварительные требования

- Работающий инстанс n8n (self-hosted или cloud).
- API-ключи для всех сервисов (см. ниже).
- База Supabase с таблицами `documents`, `chat_logs` и функцией `match_documents`.

### Установка

1. Импортируйте JSON-файлы воркфлоу в n8n.
2. Создайте credentials (см. таблицу ниже).
3. Активируйте сначала **INFO -> LOADING**, затем **INFO**.

### Credentials

| Credential | Воркфлоу | Назначение |
|---|---|---|
| Telegram Bot API | INFO | Приём и отправка сообщений |
| OpenAI API (OpenRouter) | INFO | LLM (GPT-4o), транскрибация |
| OpenAI API (direct) | INFO -> LOADING | Эмбеддинги |
| Supabase API | оба | Векторная БД, логи чатов |
| PostgreSQL | INFO | Память диалогов |
| Google Drive OAuth2 | INFO -> LOADING | Доступ к файлам |

### Supabase — таблицы

- **`documents`** — векторное хранилище (pgvector) с функцией `match_documents`.
- **`chat_logs`** — лог диалогов. Поля: `session_id`, `user_id`, `username`, `first_name`, `role`, `content`, `metadata`.

---

## Безопасность

JSON-файлы **не содержат API-ключей** — n8n хранит credentials отдельно, в файлах только внутренние ID.

**Единственное исключение:** в ноде `Typing 1` (INFO) в URL вшит токен Telegram-бота. Перед публикацией замените его на плейсхолдер или удалите ноду (она отключена и не влияет на работу).

---

## Лицензия / License

MIT

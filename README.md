# News Autoposter — n8n Automation

Автоматический пайплайн: YouTube URL → транскрибация → AI-пост → Telegram

## Как работает

```
POST /webhook/news-autoposter
        ↓
Валидация (url обязателен) → 400 если нет
        ↓
Проверка дубликата в БД → 409 если уже обрабатывался
        ↓
yt-dlp скачивает аудио в mp3
        ↓
AssemblyAI транскрибирует аудио (язык: ru)
        ↓
AI Agent (OpenRouter GPT-4.1) генерирует новостной пост
        ↓
Сохранение в PostgreSQL (news_posts)
        ↓
Отправка в Telegram канал
        ↓
Удаление временного mp3 файла
        ↓
Ответ 200 клиенту
```

## Стек

- **n8n** — платформа автоматизации
- **PostgreSQL** — база данных
- **yt-dlp** — скачивание аудио с YouTube
- **AssemblyAI** — транскрибация аудио
- **OpenRouter** — генерация поста через GPT-4.1
- **Telegram Bot API** — публикация поста

## Структура проекта

```
.
├── docker-compose.yml    # Инфраструктура (n8n + PostgreSQL)
├── Dockerfile.n8n        # Кастомный образ n8n с yt-dlp
├── init.sql              # Создание таблиц БД
├── .env.example          # Пример переменных окружения
├── workflow.json         # Экспорт воркфлоу n8n
└── README.md
```

## Таблицы БД

**news_posts** — обработанные посты:
```sql
id, source_url, transcript, post_text, status, created_at
```

**error_logs** — лог ошибок:
```sql
id, node_name, error_message, input_data, created_at
```

## Как поднять

1. Клонировать репозиторий:
```bash
git clone https://github.com/hrwwtzz-rgb/test.git
cd test
```

2. Создать `.env` из примера:
```bash
cp .env.example .env
```

3. Заполнить `.env`:
```
POSTGRES_USER=n8n_user
POSTGRES_PASSWORD=your_password
POSTGRES_DB=n8n_db
N8N_ENCRYPTION_KEY=your_32_char_key
WEBHOOK_URL=http://your-server-ip:5678/
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=your_password
OPENAI_API_KEY=your_key
TELEGRAM_BOT_TOKEN=your_token
TELEGRAM_CHAT_ID=your_chat_id
```

4. Запустить стек:
```bash
docker compose up -d
```

5. Открыть n8n на `http://localhost:5678`

6. Импортировать `workflow.json` через Settings → Import from file

7. Настроить credentials:
   - PostgreSQL (host: postgres, port: 5432)
   - Telegram Bot
   - OpenRouter API
   - AssemblyAI API

8. Активировать воркфлоу

## Использование

Отправить POST запрос:
```bash
curl -X POST http://your-server:5678/webhook/news-autoposter \
  -H "Content-Type: application/json" \
  -d '{"url": "https://www.youtube.com/watch?v=VIDEO_ID", "id": "unique_id"}'
```

**Ответы:**
- `200` — пост успешно опубликован
- `400` — отсутствует поле url
- `409` — URL уже был обработан ранее
- `500` — внутренняя ошибка (детали в error_logs и Telegram)

## Обработка ошибок

- Ошибки записываются в таблицу `error_logs`
- Уведомление с деталями отправляется в Telegram
- Временные файлы удаляются даже при падении воркфлоу
- Retry: Telegram 2×

## Исправленные ошибки в docker-compose.yml

1. **Опечатка** `DB_POSTGRESDB_HSOT` → `DB_POSTGRESDB_HOST`
2. **Отсутствие тома** `n8n_storage` не был примонтирован → данные терялись при перезапуске
3. **Отсутствие Basic Auth** → n8n был открыт без аутентификации

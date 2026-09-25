# PEd02 — «Карманный ассистент»: Telegram AI-бот на n8n

Домашнее задание к уроку «Интерфейсы: как подключить Telegram, формы, виджеты» (Модуль 6).

**Задача:** бот в Telegram принимает любое сообщение, отправляет текст в AI-модель (Claude) и возвращает сгенерированный ответ обратно в чат.

## Схема workflow

```
Telegram Trigger ──► Есть текст? ──да──► Печатает… ──► AI Agent ──► Ответ в Telegram
                          │                              ▲   ▲
                          нет                            │   │
                          ▼                 Anthropic Chat   Память диалога
                      «Не текст»               Model         (10 сообщений, по chat.id)
```

| Нода | Что делает |
|---|---|
| **Telegram Trigger** (On Message) | Точка входа (Input): запускает цепочку на каждое сообщение боту |
| **Есть текст?** (IF) | Отсекает стикеры, фото, голосовые — у них нет `message.text` |
| **Печатает…** | Показывает в чате статус «печатает…», пока модель думает |
| **AI Agent** + **Anthropic Chat Model** | Обработка (Processing): системный промпт + Claude Sonnet 5 |
| **Память диалога** | Бот помнит последние 10 сообщений отдельно для каждого чата |
| **Ответ в Telegram** | Выход (Output): ответ реплаем на сообщение пользователя |
| **Не текст** | Вежливо просит писать текстом |

Файл для импорта: [workflow/pocket-assistant-telegram.json](workflow/pocket-assistant-telegram.json)

## Как запустить

### 1. Создать бота в BotFather

1. В Telegram открыть **@BotFather** → `/newbot`.
2. Имя (Name): например, `Карманный ассистент`.
3. Username — **нельзя поменять потом**, должен заканчиваться на `bot`: например, `my_pocket_assistant_bot`.
4. Скопировать токен вида `6123456789:AAH...`. Потерялся — `/token`.

> ⚠️ Токен не коммитить в репозиторий и не показывать на скриншотах. Один токен — одна активная цепочка (не держать его одновременно в Альбато и n8n).

### 2. Получить API-ключ Anthropic

[console.anthropic.com](https://console.anthropic.com) → **API Keys** → **Create Key**. Нужен положительный баланс на аккаунте.

### 3. Импортировать workflow в n8n

1. n8n → **Workflows** → **Create Workflow** → меню `⋯` справа вверху → **Import from File…** → выбрать `workflow/pocket-assistant-telegram.json`.
2. Открыть **Telegram Trigger** → *Credential to connect with* → **Create New Credential** → вставить токен BotFather в *Access Token* → **Save**.
3. Тот же credential выбрать в нодах **Печатает…**, **Ответ в Telegram**, **Не текст**.
4. Открыть **Anthropic Chat Model** → **Create New Credential** → вставить API-ключ → **Save**. Проверить, что в поле *Model* выбрана модель (если `claude-sonnet-5` нет в списке — выбрать любую актуальную Claude из выпадающего списка).
5. **Save** workflow.

> Хотите GPT вместо Claude? Удалите ноду *Anthropic Chat Model*, нажмите `+` под *Chat Model* у AI Agent и добавьте *OpenAI Chat Model* — остальное менять не нужно.

### 4. Тест и запуск

1. Нажать **Test workflow** (Execute workflow) — триггер начнёт слушать.
2. Написать боту `/start`, затем любой вопрос. В n8n все ноды должны загореться зелёным, в Telegram — прийти ответ.
3. Переключить workflow в **Active** (тумблер справа вверху). Теперь бот работает постоянно, без нажатия Test.

> Если n8n self-hosted на `localhost`, Telegram не сможет достучаться до вебхука — нужен публичный HTTPS-адрес (n8n Cloud, VPS с доменом или туннель: `n8n start --tunnel`, ngrok, cloudflared) и переменная `WEBHOOK_URL`.

## Сценарий тестирования для скриншотов

Проверяем все ветки workflow и память:

| # | Сообщение боту | Что показывает |
|---|---|---|
| 1 | `/start` | Приветствие от модели |
| 2 | `Придумай 3 идеи для ужина из курицы и риса` | Обычный запрос → ответ модели |
| 3 | `А какая из них самая быстрая?` | Память диалога: бот понимает, о чём речь |
| 4 | `Explain what a webhook is in two sentences` | Отвечает на языке вопроса |
| 5 | Стикер или фото | Ветка «Не текст» |

Скриншоты для сдачи:
- переписка с ботом в Telegram (шаги 1–5);
- workflow в n8n: открыть **Executions** → последний успешный запуск, где все ноды зелёные.

Положить их в папку [screenshots/](screenshots/).

## Частые проблемы

| Симптом | Причина и решение |
|---|---|
| `Conflict: terminated by other getUpdates request` / бот молчит | Токен используется ещё где-то (Альбато, другой workflow). Отключить там или создать нового бота |
| Ответ приходит только при нажатом *Test workflow* | Workflow не переведён в **Active** |
| Ошибка `Bad Request: can't parse entities` | Где-то включён `Parse Mode: Markdown`, а модель прислала несбалансированные `*`/`_`. Оставить Parse Mode пустым |
| `message is too long` | Ответ длиннее 4096 символов. Уменьшить `Maximum Number of Tokens` в Chat Model или попросить краткость в системном промпте |
| Ошибка авторизации Anthropic | Неверный ключ или нулевой баланс в Anthropic Console |

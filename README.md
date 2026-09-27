# PEd02 — «Карманный ассистент»: Telegram AI-бот на n8n

Домашнее задание к уроку «Интерфейсы: как подключить Telegram, формы, виджеты» (Модуль 6).

**Задача:** бот в Telegram принимает любое сообщение, отправляет текст в AI-модель (Claude) и возвращает сгенерированный ответ обратно в чат.

## Схема workflow

```
Telegram Trigger ──► Есть текст? ──да──► Печатает… ──► AI Agent ──► Ответ в Telegram ──► Собрать лог запроса ──► Лог в Google Sheets
                          │                              ▲   ▲
                          нет                            │   │
                          ▼                 Anthropic Chat   Память диалога
                      «Не текст»               Model         (10 сообщений, по chat.id)
```

Две последние ноды — логирование успешных запросов, добавлены в PEd03 (см. ниже).

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

---

# PEd03 — Логирование и обработка ошибок

Домашнее задание к уроку «Логирование и отладка: как искать ошибки» (Модуль 6).

**Задача:** каждый запрос к боту оставляет строку в Google Таблице. Успешный пишется со временем ответа, упавший — с типом и текстом ошибки, и об ошибке сразу приходит письмо на почту. По таблице считаются метрики: error rate, среднее и максимальное время ответа.

## Схема логирования

Обе ветки пишут в **один лист** `Logs` с одинаковыми колонками. Колонка `status` = `success` / `error`.

```
Бот:            … ──► Ответ в Telegram ──► Собрать лог запроса ──► Лог в Google Sheets   (status = success)

Error Handler:                     ┌──► Запись в Google Sheets   (status = error)
Error Trigger ──► Собрать лог ─────┤
                                   └──► Письмо в Gmail           (уведомление сразу)
```

**Успешные запросы** (в workflow бота):

| Нода | Что делает |
|---|---|
| **Собрать лог запроса** (Code) | Собирает строку: `status = success`, `duration_sec` (от отправки сообщения до ответа бота), `response_chars` (длина ответа) |
| **Лог в Google Sheets** | Добавляет строку в `Logs`. Стоит **после** ответа пользователю: если таблица недоступна, пользователь всё равно получит ответ, а сбой записи поймает Error Handler и пришлёт письмо |

**Ошибки** (отдельный workflow Error Handler):

| Нода | Что делает |
|---|---|
| **Error Trigger** | Срабатывает, когда падает любой workflow, у которого этот сценарий указан как *Error Workflow* |
| **Собрать лог** (Code) | Собирает данные ошибки в плоскую строку и определяет тип ошибки (см. ниже) |
| **Запись в Google Sheets** | Добавляет строку в лист логов. Настроено *On Error → Continue*: если таблица недоступна, письмо всё равно уйдёт |
| **Письмо в Gmail** | Тема: `🚨 [n8n] <workflow>: <тип> в ноде «<нода>»`, в теле — сообщение ошибки и ссылка на выполнение |

Файл для импорта: [workflow/error-handler.json](workflow/error-handler.json)

### Какие ошибки распознаются

Нода «Собрать лог» ищет в тексте ошибки характерные признаки и записывает тип в колонку `error_type`. Это типичные ошибки AI-сценариев из урока:

| `error_type` | Признаки в тексте ошибки | Пример в нашем боте |
|---|---|---|
| `timeout` | timeout, timed out, ETIMEDOUT | Claude слишком долго отвечает |
| `invalid_token` | 401, 403, unauthorized, invalid api key | Отозван ключ Anthropic или токен бота |
| `rate_limit` | 429, rate limit, overloaded, quota | Превышены лимиты API |
| `invalid_json` | json | Сервис вернул не тот формат |
| `empty_response` | empty, no output | Модель вернула пустой ответ |
| `telegram` | chat not found, bot was blocked, message is too long | Пользователь заблокировал бота, ответ длиннее 4096 символов |
| `other` | всё остальное | Смотреть `error_message` и `error_stack` |

> В лог **не пишутся** текст сообщения пользователя и его chat_id: в логе нет персональных данных. Подробности конкретного случая открываются по ссылке `execution_url`.

## Как настроить

### 1. Google Таблица для логов

1. Создать таблицу **Bot Logs** и переименовать первый лист в **Logs**.
2. В первую строку вставить заголовки колонок из [sheets/logs-template.csv](sheets/logs-template.csv) (**Файл → Импорт → Загрузить** или скопировать вручную):

   | A | B | C | D | E | F | G | H | I | J | K | L | M |
   |---|---|---|---|---|---|---|---|---|---|---|---|---|
   | timestamp | status | workflow_id | workflow_name | node_name | execution_id | execution_url | mode | error_type | error_message | error_stack | duration_sec | response_chars |

   > Названия колонок должны совпадать с полями точь-в-точь: нода Sheets сопоставляет их автоматически (*Map Automatically*). Порядок колонок важен для формул дашборда.

   В строках `success` колонки ошибок пустые, в строках `error` пустые `duration_sec` и `response_chars`.

3. Закрепить первую строку (**Вид → Закрепить → 1 строку**).

### 2. Доступ к Google (credentials)

**n8n Cloud:** в ноде Google Sheets → *Create New Credential* → **Sign in with Google**. Для Gmail то же самое. Больше ничего не нужно.

**Self-hosted n8n** — нужен свой OAuth-клиент в [Google Cloud Console](https://console.cloud.google.com):

1. Создать проект → **APIs & Services → Library** → включить **Google Sheets API**, **Google Drive API** и **Gmail API**.
2. **OAuth consent screen** → External → заполнить название → в **Test users** добавить свой Gmail.
3. **Credentials → Create Credentials → OAuth client ID** → тип *Web application* → в **Authorized redirect URIs** вставить *OAuth Redirect URL* из окна credential в n8n.
4. Скопировать **Client ID** и **Client Secret** в credential n8n → **Sign in with Google** → разрешить доступ.
   На экране «Приложение не проверено» нажать **Дополнительно → Перейти в …**.
5. Повторить для credential **Gmail OAuth2**. Можно использовать тот же Client ID и Client Secret.

> ⚠️ Client Secret часто вставляется с ошибкой. Если авторизация не проходит — очистить поле и вставить заново. На один OAuth-клиент можно создать не больше двух секретов.

### 3. Импорт error-workflow

1. n8n → **Create Workflow** → `⋯` → **Import from File…** → `workflow/error-handler.json`.
2. **Запись в Google Sheets** → выбрать credential → *Document*: `Bot Logs` → *Sheet*: `Logs`.
3. **Письмо в Gmail** → выбрать credential → в поле *To* заменить `your-email@gmail.com` на свой адрес.
4. **Save**. Активировать этот workflow не нужно: Error Trigger срабатывает и так.

### 4. Настроить бота

1. Если бот уже импортирован из PEd02, импортировать [workflow/pocket-assistant-telegram.json](workflow/pocket-assistant-telegram.json) заново (в нём появились ноды логирования) и снова выбрать credentials. Старый workflow выключить: один токен — один активный workflow.
2. **Лог в Google Sheets** → тот же credential Google → *Document*: `Bot Logs` → *Sheet*: `Logs`.
3. `⋯` → **Settings** → **Error Workflow** → выбрать `Error Handler — логи в Sheets + Gmail` → **Save**.
4. Включить **Active**.

## Как проверить (и наполнить таблицу для сдачи)

> ⚠️ Error Workflow срабатывает **только на автоматических запусках** активного workflow. Ошибки при ручном *Test workflow* в лог не попадут. Поэтому бот должен быть в режиме **Active**, а сообщения нужно слать ему в Telegram.

**Шаг 1 — успешные запросы.** Написать боту 5–10 обычных вопросов (разной длины, чтобы различалось время ответа). В `Logs` появятся строки `success` с `duration_sec` и `response_chars`.

**Шаг 2 — ошибки.** Три способа сломать бота, чтобы получить ошибки разных типов:

| # | Как сломать | Ожидаемый `error_type` | Как вернуть |
|---|---|---|---|
| 1 | Создать второй credential Anthropic с ключом `sk-ant-invalid`, выбрать его в *Anthropic Chat Model*, сохранить, написать боту | `invalid_token` | Вернуть рабочий credential |
| 2 | В *Anthropic Chat Model* → *Model* → *By ID* → `claude-does-not-exist` | `other` (404 от API) | Вернуть модель из списка |
| 3 | В *Ответ в Telegram* заменить Chat ID на `123` | `telegram` (chat not found) | Вернуть выражение `{{ $('Telegram Trigger').item.json.message.chat.id }}` |

После каждого сломанного запуска:
- в таблице **Logs** появляется новая строка;
- на почту приходит письмо;
- в **Executions** бота этот запуск отмечен красным.

Не забыть вернуть все настройки и убедиться, что бот снова отвечает.

## Дашборд с метриками

Создать в таблице второй лист **Dashboard** и вставить формулы (разделитель `;` — для русской локали Google Таблиц):

| Метрика | Формула |
|---|---|
| Всего запросов | `=COUNTA(Logs!B2:B)` |
| Успешных | `=COUNTIF(Logs!B2:B;"success")` |
| Ошибок | `=COUNTIF(Logs!B2:B;"error")` |
| **Error rate** | `=IFERROR(COUNTIF(Logs!B2:B;"error")/COUNTA(Logs!B2:B);0)` (формат ячейки: %) |
| Ошибок сегодня | `=COUNTIFS(Logs!B2:B;"error";Logs!A2:A;">="&TODAY();Logs!A2:A;"<"&TODAY()+1)` |
| **Avg Success Time**, сек | `=IFERROR(AVERAGEIF(Logs!B2:B;"success";Logs!L2:L);0)` |
| **Max Duration**, сек | `=IFERROR(MAXIFS(Logs!L2:L;Logs!B2:B;"success");0)` |
| Средняя длина ответа, симв. | `=IFERROR(AVERAGEIF(Logs!B2:B;"success";Logs!M2:M);0)` |
| Ошибки по типам | `=QUERY(Logs!A:M;"select I, count(A) where B = 'error' group by I order by count(A) desc";1)` |
| Ошибки по нодам | `=QUERY(Logs!A:M;"select E, count(A) where B = 'error' group by E order by count(A) desc";1)` |

Ориентиры из урока: error rate около 5% — сценарий стабилен, 30–40% и выше — явная проблема. Если Avg Success Time растёт день ото дня или Max Duration сильно выше среднего, где-то появилось узкое место.

> `duration_sec` считается от момента, когда пользователь отправил сообщение (`message.date` из Telegram), поэтому это время ожидания глазами пользователя с точностью до секунды. Если бот был выключен и Telegram доставил накопившиеся сообщения позже, у них будет большое время. Такие выбросы видны в Max Duration.

## Сдача

Таблица → **Настройки доступа** → «Все, у кого есть ссылка» → *Читатель* → скопировать ссылку и отправить её в ответ на задание.

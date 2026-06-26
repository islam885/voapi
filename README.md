# vo.api — OSINT Сервис поиска

## Что это вообще такое?

Это сервис для поиска информации о людях: по номеру телефона, IP-адресу, домену, Telegram ID.
Состоит из:
- **Telegram ботов** (@itmone_bot, @promptleek_bot и другие) — в них ты пишешь запрос и получаешь результат
- **Сайта** (unixclient.ru) — админка, документация, личный кабинет
- **API** (api.unixclient.ru) — можно стучаться программой, получать данные скриптами
- **Gateway бота** (@play_blackrussia_bot) — проверяет, подписан ли ты на каналы, и пускает в ботов

---

## Как всё устроено (схема)

```
          ИНТЕРНЕТ
              │
      ┌───────┴────────┐
      │                │
  unixclient.ru    api.unixclient.ru        Telegram
  (сайт :443)      (API :443)               │
      │                │              ┌─────┼─────┐
  nginx           nginx          @itmone_bot  @promptleek_bot  @Free_probive...
      │                │              │          │               │
      ▼                ▼              ▼──────────▼───────────────▼
  Express(:3000)   Python API(:1337)   все боты читают API(:1337)
  - админка         - поиск            чтобы проверитьgateway/подписку
  - профиль         - ключи API
  - документация    - gateway                  @play_blackrussia_bot
                    - blacklist                  (gateway_bot.py)
                    - отчёты HTML                проверяет подписку
                    - авторизация                на каналы
```

Всё хранится в одном файле базы данных: `/home/itmone/bot/api/bots.db`

---

## Как работает поиск от начала до конца (для чайников)

### Шаг 1: Человек пишет боту

```
Юзер: 79123456789
Бот:  🔐 Для доступа к поиску подпишись на каналы через @play_blackrussia_bot
```

Почему так? Бот вызвал функцию `check_gateway()`, которая:
1. Сделала HTTP-запрос к Python API: `GET /api/v1/gateway/check?q=123456789`
2. API проверила в таблице `gateway_users` — есть ли там этот user_id
3. Если нет → ответила `{"verified": false, "gateway_bot": "play_blackrussia_bot"}`
4. Бот увидел `verified: false` и отправил сообщение: "Подпишись через @play_blackrussia_bot"

### Шаг 2: Человек идёт в gateway бота

Юзер открывает @play_blackrussia_bot, жмёт `/start`.
Бот показывает кнопки:
- 📢 Канал (1) — ссылка-приглашение в первый канал
- 📢 Канал (2) — ссылка-приглашение во второй канал  
- ✅ Проверить подписку — проверяет, вступил ли юзер

### Шаг 3: Человек вступает в каналы и жмёт "Проверить"

Gateway бот вызывает `check_subscription()`:
1. Берёт ID каналов из таблицы `gateway_config` (channel_1_id, channel_2_id)
2. Вызывает `bot.get_chat_member(channel_id, user_id)` для каждого канала
3. Если юзер состоит в обоих → вызывает `set_gateway_user_verified(user_id)`
4. Функция пишет в таблицу `gateway_users`: `user_id = 123456789`
5. Бот отвечает: "✅ Подписка подтверждена! Переходи в бота: @itmone_bot"

### Шаг 4: Человек идёт в любого бота и ищет

Юзер пишет `79123456789` в @itmone_bot.
Опять вызывается `check_gateway()`:
1. Запрос к API: `GET /api/v1/gateway/check?q=123456789`
2. API проверяет `gateway_users` — юзер ТАМ ЕСТЬ
3. Ответ: `{"verified": true}`
4. Бот пропускает → запускается поиск

### Шаг 5: Поиск и результат

Бот вызывает `perform_general_search(message, query)`:
1. Собирает данные из нескольких источников:
   - depsearch.sbs (база утечек)
   - easyapi (ещё база)
   - ip-api.com (IP геолокация)
   - dns.google (DNS записи доменов)
   - funstat (Telegram ID статистика)
2. Формирует HTML-страницу с результатами
3. Отправляет файл .html в чат
4. Сохраняет HTML в таблицу `shared_reports`
5. Присылает ссылку: `https://unixclient.ru/api/v1/report/...` (живёт 24 часа)

---

## Из чего состоит проект (все файлы)

### Папка /home/itmone/bot/

```
bot.py                 — ГЛАВНЫЙ ФАЙЛ. Telegram бот.
                          Тут:
                          - Все команды (/start, /passport, /snils, /inn, /bal, /stat, /login, /admin)
                          - Функции поиска (perform_general_search, perform_ip_search, perform_domain_search, perform_telegram_search)
                          - Gateway проверка (check_gateway)
                          - Генерация HTML отчётов (generate_html_report, HTML_STYLES, HTML_JS)
                          - Запуск зеркал (run_mirror_bot, start_all_mirrors)
                          - Сохранение отчётов в БД (send_report_link)

config.py              — Настройки. Токены ботов, админы, тарифы, тексты сообщений, клавиатуры.

gateway_bot.py         — Gateway бот. Проверяет подписку на каналы.
                          - check_subscription() — проверяет get_chat_member
                          - set_gateway_user_verified() — пишет в БД
                          - send_random_bot() — отправляет юзера к случайному онлайн-боту

api.key                — Мастер-ключ API. Авто-генерируется при первом запуске.
                          Нужен для входа в админку и всех админских операций.

requirements.txt       — Список Python библиотек.

api/
  database.py          — Работа с SQLite. Все таблицы, все запросы к БД.
                          Таблицы:
                          - bots — зарегистрированные боты
                          - api_keys — ключи для доступа к API
                          - api_logs — логи запросов к API
                          - gateway_config — настройки gateway (токен, ID каналов)
                          - gateway_users — кто подписался на каналы
                          - ip_blacklist — заблокированные IP
                          - web_auth_tokens — токены для входа на сайт
                          - shared_reports — HTML отчёты с сроком 24 часа

  server.py            — HTTP сервер API (порт 1337).
                          Принимает запросы, проверяет ключи, вызывает поиск, gateway, blacklist.

  search_api.py        — Функции поиска (запросы к внешним API).

web/
  server.js            — Express сервер сайта (порт 3000).
                          Все страницы админки, профиль, документация.
                          Читает config.py для редактирования.
                          Вызывает systemctl для управления сервисами.

  package.json         — Node.js зависимости.
  package-lock.json

  views/               — HTML шаблоны (EJS).
    index.ejs          — Главная страница сайта. Показывает активного бота.
    admin.ejs          — Страница входа в админку.
    dashboard.ejs      — Список всех ботов с онлайном.
    bot.ejs            — Управление сервисами (start/stop/restart для 4 сервисов).
    config.ejs         — Редактор config.py (прямо из браузера).
    api-keys.ejs       — Создание/удаление/редактирование API ключей.
    api-key-logs.ejs   — Логи запросов по каждому ключу.
    gateway.ejs        — Настройка gateway бота (токен, каналы).
    status.ejs         — Статус сервера (CPU, RAM, диск, uptime, боты онлайн).
    blacklist.ejs      — Блокировка IP для API.
    profile.ejs        — Личный кабинет (вход по одноразовому токену из /login).
    api-docs.ejs       — Документация API.
```

### Папка /etc/nginx/sites-enabled/

```
probiv                 — Конфиг для unixclient.ru:
                          - Порт 80 → редирект на 443
                          - Порт 443 с SSL
                          - / → прокси на Express :3000
                          - /api/ → прокси на Python API :1337

api.unixclient.ru      — Конфиг для api.unixclient.ru:
                          - Порт 80 → редирект на 443
                          - Порт 443 с SSL
                          - Всё прокси на Python API :1337
```

### Папка /etc/systemd/system/

```
probiv-api.service     — Python API сервер. Порт 1337.
                          Команда: systemctl start/stop/restart probiv-api

probiv-bot.service     — Главный Telegram бот + зеркала.
                          Команда: systemctl start/stop/restart probiv-bot

probiv-gateway.service — Gateway бот (@play_blackrussia_bot).
                          Команда: systemctl start/stop/restart probiv-gateway

probiv-web.service     — Сайт (Express). Порт 3000.
                          Команда: systemctl start/stop/restart probiv-web
```

---

## API Endpoints (как стучаться программой)

### Python API (:1337)

Для поиска нужен API-ключ в заголовке: `Authorization: Bearer sk-api-xxxx...`

```
Поиск:
  GET /api/v1/search?q=79123456789          — общий поиск
  GET /api/v1/search/ip?q=8.8.8.8           — поиск по IP
  GET /api/v1/search/domain?q=google.com    — поиск по домену
  GET /api/v1/search/telegram?q=123456789   — поиск по Telegram ID
  GET /api/v1/check                         — проверка своего ключа (лимиты, использовано)

Управление ключами (требуют мастер-ключ):
  GET  /api/v1/keys                         — список всех ключей
  POST /api/v1/keys                         — создать новый ключ
  POST /api/v1/keys/ID/update               — изменить ключ (лимит, название, active/disabled)
  POST /api/v1/keys/ID/delete               — удалить ключ
  GET  /api/v1/keys/ID/logs                 — логи запросов по ключу

Gateway:
  GET  /api/v1/gateway/check?q=USER_ID      — проверка, подписан ли юзер
  GET  /api/v1/gateway/config               — получить настройки gateway
  POST /api/v1/gateway/config               — сохранить настройки gateway
  POST /api/v1/gateway/verify?q=USER_ID     — отметить юзера как подписанного

Blacklist:
  GET  /api/v1/blacklist                    — список заблокированных IP
  POST /api/v1/blacklist/add                — добавить IP в блокировку
  POST /api/v1/blacklist/remove             — убрать IP из блокировки

Авторизация для сайта:
  GET  /api/v1/auth/generate?q=USER_ID      — создать одноразовый токен (5 минут)
  GET  /api/v1/auth/validate?token=TOKEN    — проверить токен

Отчёты:
  POST /api/v1/report                       — сохранить HTML, получить UUID
  GET  /api/v1/report/UUID                  — открыть HTML отчёт в браузере

Регистрация ботов:
  GET  /api/status                          — получить мастер-ключ
  GET  /api/bots                            — список ботов
  POST /api/register                        — зарегистрировать бота
```

### Express Web (:3000) — страницы сайта

```
GET  /                        — Главная (показывает активного бота)
GET  /admin                   — Вход в админку
POST /admin                   — Отправить мастер-ключ для входа
GET  /admin/dashboard         — Боты
GET  /admin/bot               — Управление сервисами
POST /admin/bot/restart       — Рестарт выбранного сервиса
POST /admin/bot/stop          — Стоп
POST /admin/bot/start         — Старт
GET  /admin/config            — Редактор config.py
POST /admin/config            — Сохранить config.py
GET  /admin/api-keys          — API ключи
POST /admin/api-keys/create   — Создать ключ
POST /admin/api-keys/delete/ID— Удалить ключ
POST /admin/api-keys/update/ID— Обновить ключ
GET  /admin/api-keys/ID/logs  — Логи по ключу
GET  /admin/gateway           — Настройки gateway
POST /admin/gateway           — Сохранить gateway
GET  /admin/status            — Статус сервера и ботов
GET  /admin/blacklist         — IP блокировки
POST /admin/blacklist/add     — Добавить IP
POST /admin/blacklist/remove  — Убрать IP
GET  /profile                 — Личный кабинет (с токеном из /login)
GET  /docs/api/v1             — Документация
```

---

## Команды Telegram ботов

| Команда | Кто может | Что делает |
|---------|-----------|------------|
| `/start` | Все | Приветствие, главное меню |
| `/passport <номер>` | Все | Поиск по паспорту |
| `/snils <номер>` | Все | Поиск по СНИЛС |
| `/inn <номер>` | Все | Поиск по ИНН |
| `/login` | Все | Создать одноразовый токен для входа на /profile |
| `/bal <id> <сумма>` | Админ | Пополнить баланс пользователю |
| `/stat` | Админ | Статистика бота (юзеры, запросы, балансы) |
| `/ras` | Админ | Рассылка всем пользователям |
| `/admin` | Админ | Статус сервера (CPU, RAM, диск, боты) |

---

## Как gateway проверяет подписку (пошагово)

1. `check_subscription(bot, user_id)`:
   - Достаёт ID каналов из `gateway_config` (channel_1_id, channel_2_id)
   - Вызывает `bot.get_chat_member(channel_1_id, user_id)` — Telegram API проверяет, есть ли юзер в канале
   - Если юзер в статусе `member`, `administrator` или `creator` — ок
   - Если нет или ошибка — возвращает False

2. Если всё ок → `set_gateway_user_verified(user_id)`:
   - `INSERT OR IGNORE INTO gateway_users (user_id) VALUES (?)`
   - Юзер добавлен в таблицу

3. `send_random_bot(message)`:
   - Стучится к API: `GET /api/bots/online`
   - API возвращает случайного бота, который сейчас онлайн (is_online=1)
   - Отправляет юзеру: "Переходи в бота: @имя_бота"

4. В любом боте при поиске:
   - `check_gateway()` стучится к API: `GET /api/v1/gateway/check?q=user_id`
   - API смотрит в `gateway_users` — если есть, отвечает `verified: true`
   - Если нет — `verified: false` + gateway_bot: "play_blackrussia_bot"

---

## Как работают зеркала (mirror bots)

Зеркала — это копии основного бота с другими токенами.

1. В админке можно создать зеркало (нажать "Create Mirror", ввести токен от @BotFather)
2. Токен сохраняется в `mirrors.db`
3. При старте основного бота вызывается `start_all_mirrors()`:
   - Читает все токены из `mirrors.db`
   - Для каждого запускает `run_mirror_bot(token, user_id, ...)`
4. `run_mirror_bot` запускает subprocess:
   ```python
   subprocess.Popen([sys.executable, bot.py], env={MIRROR_BOT_TOKEN: token})
   ```
5. В новом процессе `bot.py` видит `MIRROR_BOT_TOKEN` и создаёт `bot = TeleBot(MIRROR_BOT_TOKEN)`
6. Все хендлеры (@bot.message_handler) регистрируются с этим ботом
7. Зеркало работает как самостоятельный бот, но использует тот же код и ту же БД

---

## Как работает личный кабинет (/profile)

1. Юзер пишет `/login` любому боту
2. Бот вызывает `GET /api/v1/auth/generate?q=user_id`
3. API создаёт запись в `web_auth_tokens`: случайный токен + user_id + срок 5 минут
4. Бот присылает токен: `<code>abc123...</code>`
5. Юзер открывает `unixclient.ru/profile`, вставляет токен, жмёт Login
6. Сайт вызывает `GET /api/v1/auth/validate?token=abc123...`
7. API проверяет: токен есть, не использован, не истёк → возвращает user_id
8. Сайт запрашивает Telegram API: `getChat?chat_id=user_id` (через токен бота)
9. Получает username, имя, фамилию, аватарку
10. Показывает на странице

---

## Что где хранится

| Что | Где | Как |
|-----|-----|-----|
| Токены ботов | `config.py` | Переменные BOT_TOKEN, DEPSEARCH_TOKEN, FUNSTAT_TOKEN, EASYAPI_TOKEN |
| Админы | `config.py` | Список ADMINS = [...] |
| Тарифы | `config.py` | TARIFFS, DAILY_LIMIT, PAYMENT_MIN/MAX |
| Тексты сообщений | `config.py` | START_TEXT, ABOUT_TEXT, SUB_REQUIRED_TEXT и т.д. |
| Клавиатуры | `config.py` | kb_start(), kb_subscription(), kb_profile() и т.д. |
| Мастер-ключ API | `/home/itmone/bot/api.key` | Генерируется при первом запуске |
| Пользователи | `bot_data.pkl` (pickle) | Для каждого юзера: баланс, запросы, рефка |
| Зеркала | `mirrors.db` (SQLite) | Токены зеркал |
| Всё остальное | `api/bots.db` (SQLite) | Боты, ключи, логи, gateway, blacklist, отчёты, токены |

---

## Как поднять сервер с нуля (для новичка)

### 1. Установить Python пакеты
```bash
cd /home/itmone/bot
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 2. Установить Node.js пакеты
```bash
cd /home/itmone/bot/web
npm install
```

### 3. Запустить сервисы
```bash
# Скопировать systemd файлы
cp /home/itmone/src/etc/systemd/system/*.service /etc/systemd/system/
systemctl daemon-reload

# Запустить всё
systemctl enable --now probiv-api
systemctl enable --now probiv-bot
systemctl enable --now probiv-gateway
systemctl enable --now probiv-web

# Проверить
systemctl status probiv-api probiv-bot probiv-web probiv-gateway
```

### 4. Настроить nginx
```bash
# Скопировать конфиги
cp /home/itmone/src/etc/nginx/sites-enabled/* /etc/nginx/sites-enabled/

# Получить SSL сертификаты
certbot --nginx -d unixclient.ru -d api.unixclient.ru

# Перезагрузить nginx
nginx -s reload
```

### 5. Проверить
```bash
curl http://127.0.0.1:1337/api/status
curl http://127.0.0.1:3000
```

---

## Полезные команды (частые действия)

```bash
# Посмотреть статус всех сервисов
for s in probiv-api probiv-bot probiv-web probiv-gateway; do
  echo "$s: $(systemctl is-active $s)"
done

# Посмотреть логи бота
journalctl -u probiv-bot --no-pager -n 50

# Перезапустить API
systemctl restart probiv-api

# Открыть порт в фаерволе
ufw allow 22/tcp

# Проверить, кто в БД верифицирован
python3 -c "
import sqlite3
db = sqlite3.connect('/home/itmone/bot/api/bots.db')
for r in db.execute('SELECT * FROM gateway_users').fetchall():
    print(r)
"

# Проверить, что API отвечает для конкретного юзера
curl "http://127.0.0.1:1337/api/v1/gateway/check?q=123456789"
```

---

## Типичные проблемы и их решения

**Бот не отвечает** → `systemctl status probiv-bot` → если inactive: `systemctl start probiv-bot` → смотри логи: `journalctl -u probiv-bot -n 20`

**API не отвечает** → `systemctl status probiv-api` → проверь порт: `curl http://127.0.0.1:1337/api/status` → смотри логи

**Gateway бот не стартует** → Зайди в админку → Gateway → Убедись что заполнен gateway_bot_token → сохрани → перезапусти: `systemctl restart probiv-gateway`

**Gateway проверка не работает** → `journalctl -u probiv-gateway -n 20` → скорее всего бот не состоит в каналах для get_chat_member → добавь бота в каналы админом

**Сайт не открывается** → `systemctl status probiv-web nginx` → проверь что сертификаты есть: `ls /etc/letsencrypt/live/unixclient.ru/`

**Юзер верифицировался, но бот всё равно блокирует** → Проверь что юзер в БД: `python3 -c "import sqlite3; print(sqlite3.connect('/home/itmone/bot/api/bots.db').execute('SELECT * FROM gateway_users').fetchall())"` → Проверь API: `curl "http://127.0.0.1:1337/api/v1/gateway/check?q=USER_ID"` → Если verified=true, то проблема в боте (перезапусти)

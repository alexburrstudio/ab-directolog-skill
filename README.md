# Yandex.Direct Agent — OpenClaw Skill

Практический гайд: как превратить OpenClaw-агента в эксперта по Яндекс.Директу.

## Содержание

1. [Быстрый старт](#быстрый-старт)
2. [Регистрация приложения](#1-регистрация-приложения-в-яндексе)
3. [Получение OAuth-токенов](#2-получение-oauth-токенов)
4. [Подключение к OpenClaw](#3-подключение-к-openclaw)
5. [Настройка агента](#4-настройка-агента)
6. [Примеры использования](#5-примеры-использования)
7. [Деплой на GitHub](#6-деплой-на-github)

---

## Быстрый старт

```bash
# 1. Клонируем репозиторий скилла
git clone https://github.com/AlexBurrOne/ab-directolog-skill.git
cd ab-directolog-skill

# 2. Устанавливаем скилл
openclaw skill install ./ab-directolog-skill

# 3. Настраиваем переменные
cp .env.example .env
# Заполни YANDEX_DIRECT_CLIENT_ID, CLIENT_SECRET, REFRESH_TOKEN

# 4. Активируем
openclaw agent update directolog --skill ab-directolog-skill
```

---

## 1. Регистрация приложения в Яндексе

### Шаг 1.1 — Создай проект

Перейди на https://developer.tech.yandex.ru/projects и нажми **Создать проект**.

### Шаг 1.2 — Заполни форму

```
Название:         OpenClaw Direct Agent
Описание:         AI-агент для управления Яндекс.Директом
Платформы:        Backend-приложение
```

### Шаг 1.3 — Добавь API

В разделе **API** нажми **Добавить API** и выбери:
- **Yandex.Direct API** (уровень доступа: `direct:api`)

### Шаг 1.4 — Сохрани credentials

После создания проекта ты получишь:
```
Client ID:     xxx
Client Secret: yyy
```

Эти данные нужно сохранить — они понадобятся для OAuth.

---

## 2. Получение OAuth-токенов

### Схема авторизации

```
┌──────────┐     ┌──────────────┐     ┌─────────────┐
│  Бот     │────▶│ Яндекс OAuth │────▶│ Direct API  │
└──────────┘     └──────────────┘     └─────────────┘
```

### Шаг 2.1 — Авторизация (первый раз)

Открой в браузере:

```
https://oauth.yandex.com/authorize?
  response_type=code&
  client_id=<CLIENT_ID>&
  redirect_uri=https://example.com&
  scope=direct:api
```

После авторизации тебя перенаправит на:
```
https://example.com?code=<AUTHORIZATION_CODE>
```

Скопируй код из URL.

### Шаг 2.2 — Обмен кода на токен

```bash
curl -X POST https://oauth.yandex.com/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code" \
  -d "code=<CODE>" \
  -d "client_id=<CLIENT_ID>" \
  -d "client_secret=<CLIENT_SECRET>" \
  -d "redirect_uri=https://example.com"
```

Ответ:
```json
{
  "access_token": "AQAAA...",
  "refresh_token": "1:xxx",
  "token_type": "bearer",
  "expires_in": 86400
}
```

**Access Token** живёт 24 часа. **Refresh Token** — 1 год.

### Шаг 2.3 — Обновление токена (автоматически)

```bash
curl -X POST https://oauth.yandex.com/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=refresh_token" \
  -d "refresh_token=<REFRESH_TOKEN>" \
  -d "client_id=<CLIENT_ID>" \
  -d "client_secret=<CLIENT_SECRET>"
```

---

## 3. Подключение к OpenClaw

### Вариант A — Через переменные окружения

В `.env` файле агента:

```env
YANDEX_DIRECT_CLIENT_ID=your_client_id_here
YANDEX_DIRECT_CLIENT_SECRET=your_client_secret_here
YANDEX_DIRECT_ACCESS_TOKEN=your_access_token_here
YANDEX_DIRECT_REFRESH_TOKEN=your_refresh_token_here
```

### Вариант B — Через OpenClaw Config

```bash
openclaw config set plugins.entries.yandex-direct.clientId "your_client_id"
openclaw config set plugins.entries.yandex-direct.clientSecret "your_client_secret"
openclaw config set plugins.entries.yandex-direct.accessToken "your_access_token"
openclaw config set plugins.entries.yandex-direct.refreshToken "your_refresh_token"
```

---

## 4. Настройка агента

### 4.1 — Установка скилла

Скопируй файлы скилла в рабочую директорию:

```bash
mkdir -p ~/.openclaw/workspace/skills/ab-directolog-skill
cp -r ab-directolog-skill/* ~/.openclaw/workspace/skills/ab-directolog-skill/
```

### 4.2 — Подключение к агенту

В конфигурации агента добавь:

```json
{
  "skills": ["ab-directolog-skill"]
}
```

### 4.3 — Тест подключения

```bash
curl -X POST "https://api.direct.yandex.com/json/v5/campaigns" \
  -H "Authorization: Bearer $YANDEX_DIRECT_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "method": "get",
    "params": {
      "SelectionCriteria": {},
      "FieldNames": ["Id", "Name", "Status"]
    }
  }'
```

Ожидаемый ответ — список кампаний в JSON.

---

## 5. Примеры использования

### Пример 1: Аудит кампании

```
Пользователь: Проверь кампанию "Летняя распродажа" — где сливается бюджет?
```

Агент:
1. Запрашивает данные по кампании
2. Анализирует CTR каждого объявления
3. Находит ключевые слова с низкой эффективностью
4. Формирует отчёт

### Пример 2: Оптимизация ставок

```
Пользователь: Понизь ставки на 20% для ключевых слов с CTR < 1%
```

Агент:
1. Получает список keywords с CTR
2. Фильтрует низкоэффективные
3. Рассчитывает новые ставки
4. Отправляет batch-update через API

### Пример 3: Отчёт за период

```
Пользователь: Сделай отчёт по всем кампаниям за май: клики, расход, конверсии
```

Агент:
1. Запрашивает данные через Reports API
2. Агрегирует метрики
3. Формирует таблицу с выводами

### Пример 4: Создание кампании

```
Пользователь: Создай новую кампанию "Осень 2026" с бюджетом 1000₽/день
```

Агент:
1. Формирует JSON для Create кампании
2. Отправляет POST request
3. Возвращает ID новой кампании

---

## 6. Деплой на GitHub

### 6.1 — Создай репозиторий

```bash
# На GitHub создай репозиторий: ab-directolog-skill
# Затем локально:

cd ab-directolog-skill
git init
git add .
git commit -m "Initial commit: Yandex.Direct Agent Skill"
git branch -M main
git remote add origin https://github.com/AlexBurrOne/ab-directolog-skill.git
git push -u origin main
```

### 6.2 — Структура репозитория

```
ab-directolog-skill/
├── README.md          ← Этот файл
├── SKILL.md          ← Инструкция для OpenClaw агента
├── config/
│   ├── .env.example   ← Пример переменных окружения
│   └── agent.json     ← Пример конфига агента
├── memory/
│   └── direct-memory.md  ← Шаблон памяти агента
└── examples/
    ├── get-campaigns.sh   ← Пример запроса кампаний
    ├── get-keywords.sh    ← Пример запроса ключевых слов
    ├── get-reports.sh     ← Пример отчёта
    ├── token-refresh.sh   ← Обновление токена
    └── create-campaign.sh ← Создание кампании
```

---

## Troubleshooting

### Ошибка 401 Unauthorized

Токен истёк. Выполни refresh через `/token` endpoint.

### Ошибка 1000 Daily Limit Reached

Достигнут дневной лимит API. Подожди до следующего дня или напиши в поддержку Яндекса.

### Ошибка 8001 Request Limit Exceeded

Превышен лимит запросов в секунду. Добавь задержку 1 секунду между запросами.

---

## Полезные ссылки

- [Yandex.Direct API v5](https://api.direct.yandex.com/json/v5/)
- [OAuth Яндекса](https://yandex.ru/dev/oauth/)
- [Документация](https://yandex.ru/dev/direct/)
- [Лимиты API](https://yandex.ru/dev/direct/doc/api-concepts/domains.html)

---

## 💰 Support / Поддержать

Если скилл оказался полезен:

🥝 **TON:** @AlexBurrOne
💳 **Т-БАНК:** https://www.tbank.ru/cf/3CfaY0mpVIt

---

*Создано AB Agents*
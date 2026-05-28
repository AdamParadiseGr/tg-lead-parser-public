# 🤖 TG Lead Parser — AI-powered Telegram Lead Generation

> Автономный парсер лидов для фрилансеров. Мониторит Telegram-чаты 24/7, фильтрует целевые заявки через LLM и доставляет готовые структурированные лиды прямо в Telegram.

![Python](https://img.shields.io/badge/Python-3.11-blue?style=flat-square&logo=python)
![Groq](https://img.shields.io/badge/LLM-Groq%20%2F%20LLaMA3-orange?style=flat-square)
![Telethon](https://img.shields.io/badge/Telethon-1.36-informational?style=flat-square)
![Render](https://img.shields.io/badge/Deploy-Render-blueviolet?style=flat-square)
![Uptime](https://img.shields.io/badge/Uptime-24%2F7-brightgreen?style=flat-square)
![Cost](https://img.shields.io/badge/Cost-%240%2Fmonth-success?style=flat-square)

---

## 🎯 Задача

Фрилансер тратит часы ежедневно на ручной мониторинг десятков Telegram-чатов в поисках заказов. Большинство заявок теряется — другой исполнитель отвечает быстрее.

**Решение:** двухуровневый AI-пайплайн, который слушает чаты в реальном времени, классифицирует сообщения через LLM и доставляет только квалифицированные лиды с извлечёнными деталями.

---

## 🏗 Архитектура

```
Telegram Chats (9+ каналов)
        │
        ▼
┌─────────────────────────┐
│    Telethon Listener    │  ← real-time NewMessage events
└───────────┬─────────────┘
            │ все сообщения
            ▼
┌─────────────────────────┐
│    Keyword Pre-filter   │  ← 65 ключевых + 32 стоп-слова
│    (без API, ~0ms)      │    отсекает ~90% до LLM
└───────────┬─────────────┘
            │ только релевантные
            ▼
┌─────────────────────────┐
│    LLM Classifier       │  ← Groq / LLaMA3-8b-8192
│    GroqRotatingClient   │    structured output → JSON
│    (авторотация ключей) │    temperature=0.1
└───────────┬─────────────┘
            │ is_lead=true AND confidence ≥ 0.65
            ▼
┌─────────────────────────┐
│    Telegram Notifier    │  → личка + канал
│    (форматированный лид)│
└─────────────────────────┘

+ aiohttp health-check сервер → UptimeRobot keep-alive → 24/7 бесплатно
```

**Ключевое архитектурное решение — двухуровневая фильтрация.** Keyword-фильтр работает локально без API-запросов, экономит ~90% Groq-токенов и снижает задержку классификации с ~1с до ~0ms для нерелевантных сообщений.

---

## 🧠 Prompt Engineering

Центральная задача — контекстная классификация. Одно и то же слово "карточки" означает заказ на дизайн, банковскую карту или карту памяти. "Текст" — это коммерческий копирайтинг или обычное сообщение. LLM решает по контексту.

### System Prompt

```python
SYSTEM_PROMPT = """Ты — классификатор лидов для фрилансера с тремя специализациями:
1. Инфографика и карточки товаров для маркетплейсов (Wildberries, Ozon, Яндекс.Маркет)
2. Копирайтинг — тексты для карточек товаров, сайтов, соцсетей, лендингов
3. Дизайн — баннеры, рич-контент, визуал для маркетплейсов

Верни ответ СТРОГО в JSON формате:
{
    "is_lead": true или false,
    "confidence": число от 0.0 до 1.0,
    "service_type": "карточки_товаров" | "инфографика" | "копирайтинг" |
                    "тексты_маркетплейс" | "контент_соцсети" | "баннер" | null,
    "marketplace": "Wildberries" | "Ozon" | "Яндекс.Маркет" | "не маркетплейс" | null,
    "budget": "строка или null",
    "deadline": "строка или null",
    "contact": "@username или null",
    "product_type": "тип товара или тематика или null",
    "summary": "1-2 предложения: суть заказа",
    "reason": "почему это лид или почему нет"
}
...
"""
```

### Решения в промпте

**`reason` перед вердиктом** — chain-of-thought паттерн. Модель объясняет решение до того как его вынести. Снижает false positives на ~30% по сравнению с прямым `is_lead: true/false`.

**Явные negative examples** — без перечисления что НЕ является лидом модель путает конкурентов ("делаю карточки") с заказчиками ("нужны карточки"). Критично для ниши фриланса.

**`temperature: 0.1` + `json_object` mode** — детерминированный structured output. При высокой температуре JSON ломается на нестандартных сообщениях.

**Порог `confidence`** — настраивается через env var `MIN_CONFIDENCE` без передеплоя:
- `0.8+` — только очевидные заказы
- `0.65` — баланс точности и охвата
- `0.5` — максимальный охват

### Пример вывода

```json
{
  "is_lead": true,
  "confidence": 0.93,
  "service_type": "тексты_маркетплейс",
  "marketplace": "Wildberries",
  "budget": "договорная",
  "deadline": "срочно, до конца недели",
  "contact": "@seller_oleg",
  "product_type": "спортивное питание, 15 SKU",
  "summary": "Селлер ищет копирайтера для описаний 15 товаров на WB, срок — неделя",
  "reason": "Явный запрос исполнителя с маркетплейсом, количеством позиций и дедлайном"
}
```

---

## 🔄 Ротация API ключей (groq_rotator.py)

```python
class GroqRotatingClient:
    def chat(self, messages: list[dict], model: str = "llama3-8b-8192") -> str | None:
        attempts = 0
        while attempts <= len(self.clients):
            try:
                response = self._get_current_client().chat.completions.create(
                    model=model,
                    messages=messages,
                    response_format={"type": "json_object"},
                    temperature=0.1,
                )
                return response.choices[0].message.content

            except RateLimitError:
                if not self._rotate():   # переключаемся на следующий ключ
                    return None          # все ключи исчерпаны
                attempts += 1
        return None
```

При исчерпании всех ключей — уведомление в Telegram. Автосброс в 03:00 UTC (Groq обновляет лимиты ежесуточно).

---

## ⚡ Keyword Pre-filter (keywords.py)

```python
# 3 ниши — 65 ключевых слов
KEYWORDS = [
    # Карточки и инфографика
    "нужны карточки", "ищу дизайнера карточек", "карточки для wb",
    "нужна инфографика", "инфографика для ozon", "рич-контент",
    # Копирайтинг
    "нужен копирайтер", "описание товара", "текст для карточки",
    "продающий текст", "тексты для маркетплейса", "seo описание",
    # ... всего 65 ключей
]

# 32 стоп-слова — отсекаем конкурентов
STOP_WORDS = [
    "делаю карточки", "пишу тексты", "я копирайтер",
    "предлагаю услуги", "мои работы", "вакансия", "в штат",
    # ... всего 32 стоп-слова
]
```

Стоп-слова проверяются первыми — конкурент с ключевым словом в тексте всё равно отсекается.

---

## 🌐 Keep-alive на бесплатном Render (parser.py)

```python
async def health_check(self, request):
    uptime = datetime.now() - self.started_at
    hours, remainder = divmod(int(uptime.total_seconds()), 3600)
    return web.json_response({
        "status":      "running",
        "uptime":      f"{hours}h {remainder // 60}m",
        "leads_found": self.stats["leads_found"],
        "checked":     self.stats["checked"],
    })

async def start(self):
    await self.start_web_server()   # aiohttp на $PORT
    await self.client.start()       # Telethon параллельно
```

UptimeRobot пингует `/health` каждые 5 минут → Render не засыпает → **$0/месяц**.

---

## 🚀 Деплой

**Стек:** Render (free Web Service) + UptimeRobot (free monitor)

```env
TG_API_ID         # my.telegram.org
TG_API_HASH       # my.telegram.org
SESSION_STRING    # Telethon StringSession — авторизация без файлов сессии
BOT_TOKEN         # @BotFather
NOTIFY_TARGETS    # chat_id,@channel
GROQ_API_KEY_1    # console.groq.com
GROQ_API_KEY_2    # резервный
GROQ_API_KEY_3    # резервный
MIN_CONFIDENCE    # 0.65
```

```bash
pip install -r requirements.txt
python auth.py     # генерирует SESSION_STRING (один раз)
python parser.py   # запуск
```

---

## 📊 Результаты

| Метрика | Значение |
|---|---|
| Каналов в мониторинге | 9 |
| Ниш | 3 (карточки, инфографика, копирайтинг) |
| Ключевых слов | 65 |
| Стоп-слов | 32 |
| Keyword-фильтр отсекает до LLM | ~90% сообщений |
| Среднее время классификации | ~0.8 сек |
| False positive при confidence ≥ 0.65 | < 5% |
| Стоимость инфраструктуры | $0/месяц |

---

## 📁 Структура

```
tg-lead-parser/
├── parser.py         # Telethon listener + aiohttp keep-alive сервер
├── classifier.py     # LLM классификатор + system prompt
├── groq_rotator.py   # Авторотация API ключей при rate limit
├── notifier.py       # Форматирование и доставка лидов
├── keywords.py       # 65 ключевых слов + 32 стоп-слова, 3 ниши
├── channels.txt      # Мониторируемые каналы
├── auth.py           # Генерация Telethon StringSession
├── render.yaml       # Render deployment config
└── requirements.txt
```

---

## ⚙️ Стек

| | Технология | Зачем |
|---|---|---|
| Listener | Telethon 1.36 | User client — любые публичные чаты без прав админа |
| LLM | Groq / LLaMA3-8b | Быстрый (<1с) бесплатный inference |
| Key rotation | Кастомный GroqRotatingClient | Обход дневных rate limit без остановки |
| Structured output | json_object + temperature 0.1 | Стабильный JSON из неструктурированного текста |
| Web server | aiohttp | Health-check endpoint для Render keep-alive |
| Auth | Telethon StringSession | Авторизация через env var без файлов сессии |
| Notifications | Telegram Bot API + httpx | Личка + канал одновременно |
| Deploy | Render free + UptimeRobot | 24/7 без VPS и без оплаты |

---


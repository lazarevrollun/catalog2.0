# Catalog Pricing Microservice — Архитектура

**Документ архитектуры для MVP (Amazon only)**

---

## Резюме

Микросервис **каталога ценообразования** — это автономная система расчёта финальной цены товаров для маркетплейса Amazon.

**Ключевые характеристики:**
- 📊 Синхронный расчёт: <100ms на запрос
- 🔄 Асинхронное обновление: каждые 12 часов через Cron
- 🏗️ Clean Architecture + DDD для maintainability
- 🔌 Интеграция с API поставщиков (OpenAPI)
- 📦 Маппинг товаров (rid ↔ supplier_id) через специализированный DataStore
- 🎯 MVP Scope: только Amazon, легко расширить на eBay

**Технологический стек:**
- PHP 8.2 · Laminas/Mezzio
- rollun-datastore (для работы с данными)
- rollun-callback (для Cron)
- rollun-openapi (для генерации API)

**Состав команды на разработку:** 1 backend-разработчик (3-4 недели)

---

## Видение системы

Микросервис **каталога ценообразования** вычисляет финальную цену товара для **Amazon** на основе:

- Цены от поставщика
- Коэффициента разметки Amazon
- Процента наценки
- Стоимости доставки

**Формула:** `FinalPrice = SupplierPrice × Coefficient + (SupplierPrice × Markup%) + ShippingCost`

**MVP Scope:** Только Amazon. eBay и другие маркетплейсы — будущие итерации.

---

## Архитектурные слои

Система построена на четырёх слоях:

```
┌─────────────────────────────────────────────────┐
│        Presentation (HTTP API)                  │
│  ↓ OpenAPI Handlers (маппинг DTO → Query/Cmd)  │
├─────────────────────────────────────────────────┤
│        Application (Orchestration)              │
│  ↓ Query & Command Handlers (бизнес-логика)    │
├─────────────────────────────────────────────────┤
│        Domain (Core Logic)                      │
│  ↓ Value Objects, Services, Ports               │
├─────────────────────────────────────────────────┤
│        Infrastructure (External Systems)        │
│  ↓ Repositories, Adapters, DataStore API       │
└─────────────────────────────────────────────────┘
```

### Правило зависимостей

Зависимости направлены **только внутрь** (к Domain):

```
Domain          ← ничего не зависит
  ↑
Application     ← зависит от Domain
  ↑
Infrastructure  ← зависит от Domain
  ↑
Presentation    ← зависит от Application
```

---

## Два сценария работы

### Сценарий 1: Синхронный (HTTP запрос → расчёт цены)

**Триггер:** Внешний запрос на получение цены товара для маркетплейса

**Поток:**
1. **Presentation:** Маркетплейс отправляет HTTP запрос с `rid`
2. **Presentation → Application:** Handler маппит OpenAPI DTO в Query объект
3. **Application:** QueryHandler оркестрирует получение данных:
   - Получить цену товара (из маппинга поставщиков)
   - Получить стоимость доставки для маркетплейса
   - Получить правила ценообразования (коэффициент, наценка)
4. **Domain:** PriceCalculatorService считает финальную цену
5. **Application → Presentation:** Возвращает результат маркетплейсу

**Характеристики:**
- ⏱️ Синхронный, <100ms
- 📊 Много запросов (каждый час пересчитываем)
- 🔄 Читает данные из БД (загруженные Cron-ом)
- 🎯 Критичный по скорости

**Взаимодействие:**
```
HTTP Request (rid)
       ↓
Presentation Handler (маппит DTO)
       ↓
Application Query Handler (читает из Infrastructure)
       ↓
Domain Service (считает цену)
       ↓
HTTP Response (price_cents)
```

---

### Сценарий 2: Асинхронный (Cron → обновление данных)

**Триггер:** Cron-процесс (каждые 12 часов)

**Поток:**
1. **Infrastructure:** Cron-callback инициирует процесс обновления
2. **Application:** CommandHandler оркестрирует процесс:
   - Получить список поставщиков
   - Для каждого поставщика: запросить текущие цены
3. **Infrastructure:** SupplierOpenApiAdapter вызывает API каждого поставщика
4. **Infrastructure:** Repository сохраняет цены в локальное хранилище (DataStore)
5. **Done:** Данные готовы к синхронным запросам

**Характеристики:**
- ⏱️ Асинхронный, ~30 сек для всех поставщиков
- 📦 Пакетная обработка
- 🔌 Вызывает внешние API поставщиков
- 🔄 Обновляет локальное хранилище

**Взаимодействие:**
```
Cron Trigger (каждые 12 часов)
       ↓
Infrastructure Callback
       ↓
Application Command Handler (управляет обновлением)
       ↓
Infrastructure Adapter (вызывает API поставщика)
       ↓
Supplier API Response
       ↓
Infrastructure Repository (сохраняет в DataStore)
       ↓
Done (данные готовы)
```

---

## Концепция маппинга товаров (RidSupplierMapping)

**Проблема:** У вас есть внутренние ID товаров (rid), а у каждого поставщика — свои ID

**Решение:** RidSupplierMappingDataStore — промежуточный слой связи

```
Ваша система          RidSupplierMappingDataStore       Поставщик
┌──────────────┐     ┌──────────────────────┐      ┌──────────────┐
│ rid = товар-1│────→│ rid: товар-1         │─────→│ supplier_id  │
│ (rollun_id)  │     │ supplier: supplier1  │      │ SKU-001      │
└──────────────┘     │ supplier_product_id: │      └──────────────┘
                     │   SKU-001            │
                     └──────────────────────┘
```

**Ключевые особенности RidSupplierMappingDataStore:**
- Специализированный DataStore (как в service-catalog)
- Связывает rid ↔ supplier_id ↔ supplier_product_id
- Отслеживает историю изменений
- Может иметь валидацию (чёрные списки и т.д.)
- Уведомляет другие сервисы при изменении

**Как это работает:**

1. **Синхронный запрос:**
   - Получить `rid` (rollun_id товара в вашей системе)
   - Найти маппинг: какой `supplier_id` у этого товара
   - Получить цену для `supplier_id` из локального хранилища
   - Вернуть цену

2. **Асинхронное обновление:**
   - Получить все маппинги для поставщика
   - Запросить цены для `supplier_id` из API
   - Сохранить цены связанные с `supplier_id`
   - При синхронном запросе маппинг свяжет их с `rid`

**Ключевая идея:** Маппинг отделяет нашу систему от поставщиков. Можно переподключить другого поставщика без изменения логики.

---

## Взаимодействие слоёв

### Domain Layer — сердце системы

**Отвечает за:** Правила бизнеса, вычисления, валидация

**Содержит:**
- **Value Objects:** Money, ProductId, Coefficient, Percentage
  - Неизменяемые (immutable)
  - Инкапсулируют валидацию
  - Не имеют ID
- **Entities:** Product (если потребуется)
  - Имеют идентичность
  - Могут меняться
- **Domain Services:** PriceCalculator
  - Бизнес-логика, которая не принадлежит одному Entity
  - Не имеют состояния
  - Используют Value Objects и Entities
- **Ports (интерфейсы):** Определяют контракты
  - `SupplierPriceProvider` — как получить цену поставщика
  - `ShippingCostCalculator` — как получить доставку
  - `PricingRulesRepository` — как получить правила
  - Не знают о конкретной реализации (DataStore, API и т.д.)

**Принцип:** Domain не зависит ни от чего. Если вы удалите всё, Domain всё ещё работает.

---

### Application Layer — оркестр

**Отвечает за:** Использовать Domain для реализации Use Cases

**Содержит:**
- **Query Handlers:** Для чтения данных
  - `CalculateAmazonPriceQueryHandler`
  - `CalculateEbayPriceQueryHandler`
  - Используют Ports для получения данных
  - Используют Domain Services для расчёта
  - Возвращают результат
- **Command Handlers:** Для изменения состояния
  - `UpdateSupplierPricesCommandHandler`
  - Используют Adapters для получения данных
  - Используют Repository для сохранения
- **DTO (Data Transfer Objects):** Контракты с внешним миром
  - Query DTO (что запрашиваем)
  - Result DTO (что возвращаем)
  - Command DTO (что изменяем)

**Принцип:** Application координирует, но не содержит бизнес-логику. Domain содержит логику.

---

### Infrastructure Layer — реальный мир

**Отвечает за:** Интеграция с внешними системами

**Содержит:**
- **Repositories:** Доступ к данным
  - Реализуют Ports из Domain
  - Используют DataStore API для чтения/записи
  - Маппят Domain объекты ↔ DataStore данные
- **Adapters:** Интеграция с внешними сервисами
  - Реализуют Ports из Domain
  - Вызывают внешние API (поставщики, доставка)
  - Трансформируют ответы в Domain объекты
- **Callbacks:** Асинхронные задачи
  - Запускаются Cron-ом
  - Создают Command объекты
  - Вызывают Command Handler'ы

**Принцип:** Всё конкретное (DataStore, HTTP клиент, Cron) находится здесь.

---

### Presentation Layer — фасад

**Отвечает за:** HTTP API, маршруты, маппинг DTO

**Содержит:**
- **HTTP Handlers:** Точки входа для запросов
  - Получают OpenAPI DTO из запроса
  - Маппят в Query/Command объекты
  - Вызывают Application Handler'ы
  - Маппят результат обратно в OpenAPI DTO
  - Возвращают HTTP ответ
- **Маршруты:** Определяют HTTP endpoints
  - OpenAPI маршруты (бизнес-операции)
  - DataStore API маршруты (CRUD для администрирования)

**Принцип:** Самый тонкий слой. Только маппинг и роутинг.

---

## Коммуникация между слоями

### Вверх (Query)

```
HTTP запрос
    ↓
Presentation Handler (маппит DTO в Query)
    ↓
Application QueryHandler (координирует)
    ↓
Infrastructure Repository (получает через Ports)
    ↓
Domain Service (считает через Value Objects)
    ↓
Результат вверх (тот же путь)
    ↓
HTTP ответ
```

### Вверх (Command)

```
Cron запуск
    ↓
Infrastructure Callback (создаёт Command)
    ↓
Application CommandHandler (координирует)
    ↓
Infrastructure Adapter (вызывает внешний API)
    ↓
Infrastructure Repository (сохраняет через Ports)
    ↓
Done
```

### Инверсия управления через Ports

```
Application         Domain
    ↓                ↑
используем          определяет контракт
Ports ───────────────┘

Infrastructure
    ↓
реализует Ports
```

Domain определяет, **что нужно** (Ports).
Application использует **что нужно** (Ports).
Infrastructure предоставляет **как это реализовать** (реализация Ports).

---

## API сервиса

### Уровень 1: OpenAPI (бизнес-операции) — MVP

**Примеры:**
- `POST /openapi/CatalogData/v1/price/amazon` — расчёт цены для Amazon (с rid)

**Характеристики:**
- Проходит через всю архитектуру (Domain → Application → Presentation)
- Содержит бизнес-логику и валидацию
- Контролируется OpenAPI манифестом
- Медленнее, но надёжнее

---

### Уровень 2: DataStore API (CRUD администрирования) — MVP

**Примеры:**
- `GET /api/datastore/supplier_prices` — получить все цены
- `PUT /api/datastore/pricing_rules/amazon` — обновить правила для Amazon
- `POST /api/datastore/rid_supplier_mapping` — добавить маппинг между rid и поставщиком
- `GET /api/datastore/shipping_costs` — получить стоимость доставки

**Характеристики:**
- Встроенный API (не пишем Handlers)
- Прямой доступ к данным (минует Application слой)
- Для администрирования и отладки
- Быстрый доступ, но без бизнес-логики

---

## Домен на словах

### Сущности (Entities)

- **Product** — товар в нашей системе (если потребуется)
  - Имеет `rid`
  - Может быть связан с несколькими поставщиками

### Value Objects

- **Money** — денежная сумма в центах USD
  - Валюта: всегда USD
  - Единица: центы (int)
  - Операции: add, multiply
  - Валидация: не может быть отрицательной
- **Rid** (RollunId) — идентификатор товара в нашей системе
- **SupplierId** — идентификатор поставщика
- **SupplierProductId** — айди товара в системе поставщика
- **Coefficient** — коэффициент ценообразования (1.0 = базовая цена)
- **Percentage** — процент наценки (0-100)

### Ports (контракты)

- **SupplierPriceProvider** — получить цену товара от поставщика
- **ShippingCostCalculator** — вычислить стоимость доставки
- **PricingRulesRepository** — получить правила ценообразования для маркетплейса
- **SupplierOpenApiAdapter** — интеграция с API поставщиков

### Services (Domain Services)

- **PriceCalculator** — вычисляет финальную цену
  - Входы: supplier_price, coefficient, markup, shipping_cost
  - Выход: final_price
  - Правила: Money × Coefficient + Money × Markup% + Money

### Use Cases (Application)

**Синхронные (Query) — MVP:**
- **CalculateAmazonPrice** — рассчитать цену для Amazon
  - Input: rid
  - Output: price_cents

**Асинхронные (Command) — MVP:**
- **UpdateSupplierPrices** — обновить цены от поставщиков
  - Input: supplier_id (опционально)
  - Output: none (только сохранение)
- **UpdateShippingCosts** — обновить стоимость доставки для Amazon
  - Input: none (обновляет всё)
  - Output: none (только сохранение)

**Будущие расширения:**
- CalculateEbayPrice — для eBay (Phase 2)
- Другие маркетплейсы — по необходимости

---

## Поток данных

### Синхронный поток (получение цены для Amazon) — MVP

```
1. Запрос: Amazon adapter → HTTP POST /openapi/CatalogData/v1/price/amazon
   Body: { rid: "товар-123" }

2. Presentation: Получить rid из OpenAPI DTO

3. Application: CalculateAmazonPriceQueryHandler:
   - Получить цену товара (через RidSupplierMappingDataStore → SupplierPricesDataStore)
   - Получить доставку для Amazon (через ShippingCostsDataStore)
   - Получить правила Amazon (через PricingRulesDataStore)

4. Domain: PriceCalculator.calculate():
   finalPrice = supplierPrice × coefficient + supplierPrice × markup% + shippingCost

5. Response: Вернуть { price_cents: 1500 } Amazon adapter'у

Время: ~50ms (всё из локального DataStore, никаких внешних вызовов)
```

### Асинхронный поток (обновление данных) — MVP

```
1. Cron: Запуск процесса (каждые 12 часов)

2. Infrastructure: Callback инициирует:
   - UpdateSupplierPricesCommand
   - UpdateShippingCostsCommand

3. Application: UpdateSupplierPricesCommandHandler:
   - Получить список поставщиков
   - Для каждого поставщика:
     a) Получить маппинги (rid → supplier_id) через RidSupplierMappingDataStore
     b) Получить цены через SupplierOpenApiAdapter
     c) Сохранить в SupplierPricesDataStore

4. Application: UpdateShippingCostsCommandHandler:
   - Получить стоимость доставки для Amazon
   - Сохранить в ShippingCostsDataStore

5. Done: Данные готовы для синхронных запросов через CalculateAmazonPrice

Время: ~30 сек (зависит от количества поставщиков и товаров)
```

---

## Принципы архитектуры

1. **Clean Architecture**: Слои независимы друг от друга
2. **DDD**: Domain содержит весь смысл системы
3. **Dependency Inversion**: Depend on abstractions, not concretions
4. **Single Responsibility**: Каждый слой делает одно
5. **Open/Closed**: Открыты для расширения (новые поставщики), закрыты для модификации

---

## Структура проекта

```
src/
├── Domain/                          ← Бизнес-логика (Domain-Driven Design)
│   ├── ValueObject/
│   │   ├── Money.php                  ← Денежная сумма USD
│   │   ├── Rid.php                    ← ID товара в нашей системе
│   │   ├── Coefficient.php            ← Коэффициент ценообразования
│   │   └── Percentage.php             ← Процент наценки
│   ├── Service/
│   │   └── PriceCalculator.php        ← Вычисление финальной цены
│   └── Port/
│       ├── SupplierPriceProvider.php  ← Интерфейс получения цены от поставщика
│       ├── ShippingCostCalculator.php ← Интерфейс получения доставки
│       ├── PricingRulesRepository.php ← Интерфейс получения правил Amazon
│       └── SupplierOpenApiAdapter.php ← Интерфейс вызова API поставщиков
│
├── Application/                     ← Use Cases (Бизнес-процессы)
│   ├── Query/
│   │   ├── CalculateAmazonPrice/
│   │   │   ├── CalculateAmazonPriceQuery.php          ← Запрос
│   │   │   ├── CalculateAmazonPriceQueryHandler.php   ← Обработчик
│   │   │   └── CalculateAmazonPriceResult.php         ← Результат
│   └── Command/
│       ├── UpdateSupplierPrices/
│       │   ├── UpdateSupplierPricesCommand.php        ← Команда
│       │   └── UpdateSupplierPricesCommandHandler.php ← Обработчик
│       └── UpdateShippingCosts/
│           ├── UpdateShippingCostsCommand.php         ← Команда
│           └── UpdateShippingCostsCommandHandler.php  ← Обработчик
│
├── Infrastructure/                  ← Интеграция с внешними системами
│   ├── Repository/
│   │   ├── RidSupplierMappingRepository.php  ← Маппинг товаров
│   │   ├── SupplierPricesRepository.php      ← Цены поставщиков
│   │   ├── ShippingCostsRepository.php       ← Доставка
│   │   └── PricingRulesRepository.php        ← Правила ценообразования
│   ├── Adapter/
│   │   ├── SupplierOpenApiAdapter.php        ← Вызов API поставщиков
│   │   └── ShippingCostAdapter.php           ← Вызов API доставки
│   ├── DataStore/
│   │   ├── RidSupplierMappingDataStore.php   ← Специализированный DataStore
│   │   └── ... (остальные DataStore'ы)
│   ├── Callback/
│   │   ├── UpdateSupplierPricesCallback.php  ← Cron для обновления цен
│   │   └── UpdateShippingCostsCallback.php   ← Cron для обновления доставки
│   └── DI/
│       └── ConfigProvider.php                 ← DI контейнер конфиг
│
└── Presentation/                    ← HTTP API
    ├── Handler/
    │   └── CalculateAmazonPriceHandler.php   ← HTTP Handler
    ├── Middleware/
    │   └── ... (если нужны)
    └── RoutesDelegator.php                   ← Роутинг

config/
├── autoload/
│   ├── datastore.global.php         ← DataStore конфиг
│   ├── services.global.php          ← DI конфиг
│   └── cron.global.php              ← Cron конфиг (Callbacks)
└── routes.php                        ← HTTP роуты

openapi/
└── CatalogData/
    └── v1.yaml                      ← OpenAPI манифест для генерации кода

tests/
├── Unit/
│   ├── Domain/                      ← Unit-тесты Domain Layer
│   └── Application/                 ← Unit-тесты Use Cases
└── Integration/
    ├── Infrastructure/              ← Integration-тесты DataStore
    └── E2E/                         ← E2E-тесты через HTTP
```

---

## Файлы для разработки (MVP)

### Domain Layer

| Файл | Назначение | Статус |
|------|-----------|--------|
| Money.php | Value Object для денег USD | Core |
| Rid.php | Value Object для ID товара | Core |
| Coefficient.php | Value Object для коэффициента | Core |
| Percentage.php | Value Object для процента | Core |
| PriceCalculator.php | Domain Service расчёта цены | Core |
| Ports (4 интерфейса) | Контракты для Infrastructure | Core |

### Application Layer

| Файл | Назначение | Статус |
|------|-----------|--------|
| CalculateAmazonPriceQuery.php | Query DTO | Core |
| CalculateAmazonPriceQueryHandler.php | Обработчик запроса | Core |
| CalculateAmazonPriceResult.php | Result DTO | Core |
| UpdateSupplierPricesCommand.php | Command DTO | Core |
| UpdateSupplierPricesCommandHandler.php | Обработчик команды | Core |
| UpdateShippingCostsCommand.php | Command DTO | Core |
| UpdateShippingCostsCommandHandler.php | Обработчик команды | Core |

### Infrastructure Layer

| Файл | Назначение | Статус |
|------|-----------|--------|
| RidSupplierMappingDataStore.php | Специализированный DataStore | Priority |
| SupplierPricesRepository.php | Repository для цен | Core |
| ShippingCostsRepository.php | Repository для доставки | Core |
| PricingRulesRepository.php | Repository для правил | Core |
| SupplierOpenApiAdapter.php | Адаптер к API поставщика | Core |
| ShippingCostAdapter.php | Адаптер к API доставки | Optional |
| UpdateSupplierPricesCallback.php | Cron для обновления цен | Core |
| UpdateShippingCostsCallback.php | Cron для обновления доставки | Core |
| ConfigProvider.php | DI контейнер | Core |

### Presentation Layer

| Файл | Назначение | Статус |
|------|-----------|--------|
| CalculateAmazonPriceHandler.php | HTTP Handler | Core |
| RoutesDelegator.php | Роутинг | Core |

### Configuration

| Файл | Назначение | Статус |
|------|-----------|--------|
| datastore.global.php | DataStore регистрация | Core |
| services.global.php | DI регистрация | Core |
| cron.global.php | Cron регистрация | Core |
| routes.php | HTTP роуты | Core |
| CatalogData/v1.yaml | OpenAPI манифест | Core |

---

## Предположения и зависимости

### Предположения MVP
- ✅ Amazon adapter уже интегрирован (может вызывать `/openapi/CatalogData/v1/price/amazon`)
- ✅ Есть как минимум один источник цен поставщика с OpenAPI
- ✅ Доставка — статическое значение или простой API
- ✅ Правила ценообразования (коэффициент + наценка) редко меняются

### Зависимости
- **rollun-openapi** — генерирует HTTP Handler'ы из манифеста ✅ (уже в проекте)
- **rollun-datastore** — для работы с данными ✅ (уже в проекте)
- **rollun-callback** — для Cron-задач ✅ (уже в проекте)
- **PHP 8.2+** ✅
- **MySQL** (или другая БД) для DataStore ✅

### Риски и смягчение
| Риск | Вероятность | Смягчение |
|------|------------|----------|
| Задержка в API поставщика при синхронном запросе | Средняя | Асинхронная загрузка данных в Cron, синхронный запрос читает из кэша |
| Неправильный маппинг rid → supplier_id | Средняя | Валидация в RidSupplierMappingDataStore, истории изменений |
| Сложность расширения на eBay | Низкая | Архитектура уже готова для новых маркетплейсов (добавить CalculateEbayPriceQuery) |
| Производительность при большом количестве товаров | Низкая | DataStore + индексы, Cron работает асинхронно |

---

## Roadmap развития

### Phase 1: MVP — Amazon only (текущий этап)
- ✅ Синхронный расчёт цены для Amazon через `/openapi/CatalogData/v1/price/amazon`
- ✅ Асинхронное обновление цен через Cron (каждые 12 часов)
- ✅ Маппинг товаров между rid и поставщиками
- ✅ Базовая стоимость доставки
- ✅ Правила ценообразования для Amazon (коэффициент + наценка)
- ✅ DataStore для управления данными

### Phase 2: Масштабирование (eBay + оптимизация)
- Добавить CalculateEbayPrice
- Добавить больше поставщиков
- История изменения цен
- Кэширование часто запрашиваемых товаров

### Phase 3: Интеллект
- Предсказание цен на основе тренда
- Динамическая наценка по спросу
- Интеграция с системой инвентаризации

---

## Следующие шаги

### Для утверждения архитектуры
1. ✅ Обсудить с TeamLead'ом (этот документ)
2. ✅ Утвердить структуру папок и файлов
3. ✅ Согласовать OpenAPI манифест (v1.yaml)
4. ✅ Определить API поставщиков (какие данные возвращают)

### Для начала разработки
1. **Scaffolding:** Создать структуру папок и пусто файлы
2. **Domain Layer:** Реализовать Value Objects и PriceCalculator
3. **Application Layer:** Реализовать Query/Command Handler'ы
4. **Infrastructure Layer:** Реализовать Repository'ы и Adapter'ы
5. **Presentation Layer:** Реализовать HTTP Handler и роуты
6. **Tests:** Unit, Integration, E2E
7. **Integration:** Тестирование с реальным Amazon adapter'ом

### Timeline (примерный, 1 разработчик)
- **Неделя 1:** Domain + Application Layer (Unit-тесты)
- **Неделя 2:** Infrastructure Layer (Integration-тесты)
- **Неделя 3:** Presentation Layer + E2E тесты
- **Неделя 4:** Интеграция, документация, code review

---

## Вопросы для обсуждения

1. **OpenAPI манифест:** Как выглядит ожидаемый format запроса/ответа?
2. **API поставщиков:** Какие они будут на MVP (точные endpoints)?
3. **Доставка:** Статическое значение или динамический API?
4. **Маппинг:** Как будут добавляться маппинги (вручную через DataStore API или импорт)?
5. **Кэширование:** Нужна ли дополнительная кэширование помимо DataStore?

---

**Статус документа:** 📋 Ready for review by TeamLead


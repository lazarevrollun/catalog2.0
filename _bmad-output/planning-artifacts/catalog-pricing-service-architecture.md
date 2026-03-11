# Catalog Pricing Microservice — Архитектура

**Стек:** PHP 8.2 · Laminas/Mezzio · Clean Architecture · DDD
**Дата:** 2026-03-11
**Автор:** Iliya

---

## Концепция

Микросервис каталога считает цену товара для маркетплейсов (Amazon, eBay) по формуле:

```
FinalPrice = SupplierPrice × Coefficient + (SupplierPrice × Markup%) + ShippingCost
```

- Валюта: только **USD** (хранится в центах как `int`)
- Цена поставщика получается из внешнего OpenAPI-сервиса в реальном времени
- Стоимость доставки — из отдельного OpenAPI-сервиса
- Для Amazon и eBay — **отдельные Use Cases** с разными коэффициентами/наценками

---

## Точка входа: rollun-openapi

Все внешние запросы к сервису идут через OpenAPI манифест.
`rollun-openapi` генерирует серверный и клиентский код из манифеста `openapi/CatalogData/v1.yaml`.

### Генерируемая структура

```
src/
└── CatalogData/                             ← имя модуля
    └── src/
        └── OpenAPI/
            └── V1/
                ├── Client/                  ← клиент для вызова НАШЕГО сервиса извне
                │   ├── Api/
                │   │   └── CatalogApi.php           # сгенерирован
                │   ├── Rest/
                │   │   ├── CalculateAmazonPrice.php # сгенерирован
                │   │   └── CalculateEbayPrice.php   # сгенерирован
                │   └── Configuration.php            # сгенерирован
                └── DTO/                     ← DTO по манифесту
                    ├── AmazonPriceRequest.php   # сгенерирован
                    ├── AmazonPriceResponse.php  # сгенерирован
                    ├── EbayPriceRequest.php     # сгенерирован
                    └── EbayPriceResponse.php    # сгенерирован
```

> `OpenAPI/` — **сгенерированный код**, не редактируется вручную. Regenerate при изменении манифеста.

### Поток входящего запроса

```
Внешний сервис (Amazon adapter, eBay adapter, ...)
        │
        │ POST http://catalog/openapi/CatalogData/v1/price/amazon
        │ Body: AmazonPriceRequest { product_id: "SKU-123" }
        ▼
[rollun-openapi: роутинг по манифесту]
        │ /openapi/CatalogData/v1/price/amazon → CalculateAmazonPriceHandler
        ▼ [Presentation]
CalculateAmazonPriceHandler
        │ маппит OpenAPI DTO → Query
        │ new CalculateAmazonPriceQuery($dto->product_id)
        ▼ [Application]
CalculateAmazonPriceQueryHandler
        ├── SupplierPriceProviderInterface  → SupplierOpenApiAdapter
        ├── ShippingCostCalculatorInterface → ShippingOpenApiAdapter
        ├── PricingRulesRepositoryInterface → DataStorePricingRulesRepository
        └── PriceCalculatorService [Domain]
                └── CalculateAmazonPriceResult
                        │
                        ▼ [Presentation]
                AmazonPriceResponse { price_cents: 1500 }   ← OpenAPI DTO
                        │
                        ▼
        JSON Response 200
```

### Handler: маппинг OpenAPI DTO ↔ Application

Handler живёт в Presentation и является точкой склейки между сгенерированным OpenAPI DTO и нашим CQRS слоем:

```php
// Presentation/Handler/CalculateAmazonPriceHandler.php
final class CalculateAmazonPriceHandler implements RequestHandlerInterface
{
    public function __construct(
        private CalculateAmazonPriceQueryHandler $handler,
    ) {}

    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        /** @var AmazonPriceRequest $dto — сгенерированный OpenAPI DTO */
        $dto = $request->getAttribute(AmazonPriceRequest::class);

        $result = $this->handler->handle(
            new CalculateAmazonPriceQuery($dto->product_id)
        );

        // маппим обратно в OpenAPI DTO
        $response = new AmazonPriceResponse();
        $response->price_cents = $result->price->amount;

        return new JsonResponse($response);
    }
}
```

### Полная структура папок с OpenAPI

```
src/
├── CatalogData/                             ← имя модуля (сгенерировано rollun-openapi)
│   └── src/
│       └── OpenAPI/
│           └── V1/
│               ├── Client/
│               │   ├── Api/
│               │   │   └── CatalogApi.php
│               │   ├── Rest/
│               │   │   ├── CalculateAmazonPrice.php
│               │   │   └── CalculateEbayPrice.php
│               │   └── Configuration.php
│               └── DTO/
│                   ├── AmazonPriceRequest.php
│                   ├── AmazonPriceResponse.php
│                   ├── EbayPriceRequest.php
│                   └── EbayPriceResponse.php
│
├── Domain/                                  ← наш код
│   ├── ValueObject/  ...
│   ├── Service/      ...
│   └── Port/         ...
│
├── Application/                             ← наш код
│   ├── Query/        ...
│   └── Command/      ...
│
├── Infrastructure/                          ← наш код
│   ├── Adapter/      ...
│   ├── Repository/   ...
│   └── DI/
│       └── ConfigProvider.php
│
└── Presentation/                            ← наш код, склейка с OpenAPI DTO
    ├── Handler/
    │   ├── CalculateAmazonPriceHandler.php
    │   └── CalculateEbayPriceHandler.php
    └── RoutesDelegator.php
```

---

## Слои архитектуры

| Слой | Содержимое | Зависимости |
|---|---|---|
| **Domain** | Entities, Value Objects, Domain Services, Port-интерфейсы | Никаких внешних |
| **Application** | Use Cases, Request/Response DTO, конфиги правил | Только Domain |
| **Infrastructure** | OpenAPI Adapters, DataStore Repositories, DI ConfigProvider | Domain + rollun-libs |
| **Presentation** | Mezzio Handlers, роутинг | Application |

Правило: **зависимости направлены только внутрь** (к Domain). Нарушение = ошибка в CI (Deptrac).

---

## Структура папок

```
src/
├── Domain/
│   ├── Entity/
│   │   └── Product.php
│   ├── ValueObject/
│   │   ├── Money.php
│   │   ├── ProductId.php
│   │   ├── PricingCoefficient.php
│   │   ├── Percentage.php
│   │   └── ShippingCost.php
│   ├── Service/
│   │   └── PriceCalculatorService.php
│   └── Port/
│       ├── SupplierPriceProviderInterface.php
│       ├── ShippingCostCalculatorInterface.php
│       └── PricingRulesRepositoryInterface.php
│
├── Application/
│   ├── Query/                                   # CQRS — Query side (read, без мутации)
│   │   ├── CalculateAmazonPrice/
│   │   │   ├── CalculateAmazonPriceQuery.php    # [Query DTO]
│   │   │   ├── CalculateAmazonPriceQueryHandler.php
│   │   │   └── CalculateAmazonPriceResult.php   # [QueryResultDTO]
│   │   └── CalculateEbayPrice/
│   │       ├── CalculateEbayPriceQuery.php      # [Query DTO]
│   │       ├── CalculateEbayPriceQueryHandler.php
│   │       └── CalculateEbayPriceResult.php     # [QueryResultDTO]
│   │
│   ├── Command/                                 # CQRS — Command side (мутация состояния)
│   │   └── UpdatePricingRules/
│   │       ├── UpdatePricingRulesCommand.php    # [Command DTO]
│   │       └── UpdatePricingRulesCommandHandler.php
│   │
│   └── Port/
│       ├── SupplierPriceProviderInterface.php
│       ├── ShippingCostCalculatorInterface.php
│       └── PricingRulesRepositoryInterface.php
│
├── Infrastructure/
│   ├── Adapter/
│   │   ├── Supplier/
│   │   │   └── SupplierOpenApiAdapter.php
│   │   └── Shipping/
│   │       └── ShippingOpenApiAdapter.php
│   ├── Repository/
│   │   └── DataStorePricingRulesRepository.php
│   └── DI/
│       └── ConfigProvider.php
│
└── Presentation/
    ├── Handler/
    │   ├── CalculateAmazonPriceHandler.php   # вызывает QueryHandler напрямую
    │   └── CalculateEbayPriceHandler.php
    └── RoutesDelegator.php

tests/
├── Unit/
│   ├── Domain/Service/PriceCalculatorServiceTest.php
│   ├── Domain/ValueObject/MoneyTest.php
│   ├── Application/CalculateAmazonPriceUseCaseTest.php
│   └── Application/CalculateEbayPriceUseCaseTest.php
└── Integration/
    └── Infrastructure/SupplierOpenApiAdapterTest.php
```

---

## Domain Layer

### Money (Value Object)

```php
final class Money
{
    private function __construct(
        public readonly int $amount,      // в центах
        public readonly string $currency
    ) {}

    public static function usd(int $cents): self
    {
        if ($cents < 0) {
            throw new \InvalidArgumentException('Amount cannot be negative');
        }

        return new self($cents, 'USD');
    }

    public function add(self $other): self
    {
        return new self($this->amount + $other->amount, $this->currency);
    }

    public function multiply(float $multiplier): self
    {
        return new self((int) round($this->amount * $multiplier), $this->currency);
    }

    public function equals(self $other): bool
    {
        return $this->amount === $other->amount && $this->currency === $other->currency;
    }
}
```

```php
final class PricingCoefficient
{
    public function __construct(public readonly float $value)
    {
        if ($value <= 0) {
            throw new \InvalidArgumentException('Coefficient must be positive');
        }
    }
}

final class Percentage
{
    public function __construct(public readonly float $value)
    {
        if ($value < 0 || $value > 100) {
            throw new \InvalidArgumentException('Percentage must be between 0 and 100');
        }
    }
}
```

### PriceCalculatorService

```php
final class PriceCalculatorService
{
    public function calculate(
        Money $supplierPrice,
        PricingCoefficient $coefficient,
        Percentage $markup,
        Money $shippingCost
    ): Money {
        $base      = $supplierPrice->multiply($coefficient->value);
        $markupAmt = $supplierPrice->multiply($markup->value / 100);

        return $base->add($markupAmt)->add($shippingCost);
    }
}
```

Нет зависимостей от фреймворка. Тестируется без моков.

### Ports (интерфейсы)

```php
// Domain/Port/SupplierPriceProviderInterface.php
interface SupplierPriceProviderInterface
{
    public function getPriceForProduct(ProductId $productId): Money;
}

// Domain/Port/ShippingCostCalculatorInterface.php
interface ShippingCostCalculatorInterface
{
    public function calculate(ProductId $productId, string $marketplace): Money;
}

// Domain/Port/PricingRulesRepositoryInterface.php
interface PricingRulesRepositoryInterface
{
    public function findByMarketplace(string $marketplace): PricingRules;
}
```

---

## Application Layer

### Use Case (пример Amazon)

```php
final class CalculateAmazonPriceUseCase
{
    public function __construct(
        private SupplierPriceProviderInterface $supplierProvider,
        private ShippingCostCalculatorInterface $shippingCalculator,
        private PricingRulesRepositoryInterface $pricingRules,
        private PriceCalculatorService $calculator
    ) {}

    public function execute(CalculateAmazonPriceRequest $request): CalculateAmazonPriceResponse
    {
        $supplierPrice = $this->supplierProvider->getPriceForProduct($request->productId);
        $shippingCost  = $this->shippingCalculator->calculate($request->productId, 'amazon');
        $rules         = $this->pricingRules->findByMarketplace('amazon');

        $finalPrice = $this->calculator->calculate(
            $supplierPrice,
            $rules->coefficient,
            $rules->markup,
            $shippingCost
        );

        return new CalculateAmazonPriceResponse($finalPrice);
    }
}
```

`CalculateEbayPriceUseCase` — идентичная структура, `marketplace = 'ebay'`.

---

## CQRS

Расчёт цены — это **чтение без мутации состояния**, поэтому это **Query**.
Обновление правил ценообразования — **мутация**, поэтому это **Command**.

```
Command → CommandHandler → изменяет состояние → void / ID
Query   → QueryHandler  → читает данные     → ResultDTO
```

### Query DTO

```php
// Application/Query/CalculateAmazonPrice/CalculateAmazonPriceQuery.php
final class CalculateAmazonPriceQuery
{
    public function __construct(
        public readonly string $productId,
    ) {}
}
```

### QueryHandler

```php
// Application/Query/CalculateAmazonPrice/CalculateAmazonPriceQueryHandler.php
final class CalculateAmazonPriceQueryHandler
{
    public function __construct(
        private SupplierPriceProviderInterface $supplierProvider,
        private ShippingCostCalculatorInterface $shippingCalculator,
        private PricingRulesRepositoryInterface $rulesRepository,
        private PriceCalculatorService $calculator,
    ) {}

    public function handle(CalculateAmazonPriceQuery $query): CalculateAmazonPriceResult
    {
        $productId     = new ProductId($query->productId);
        $supplierPrice = $this->supplierProvider->getPriceForProduct($productId);
        $shippingCost  = $this->shippingCalculator->calculate($productId, 'amazon');
        $rules         = $this->rulesRepository->findByMarketplace('amazon');

        $finalPrice = $this->calculator->calculate(
            $supplierPrice,
            $rules->coefficient,
            $rules->markup,
            $shippingCost,
        );

        return new CalculateAmazonPriceResult($finalPrice);
    }
}
```

### QueryResultDTO

```php
// Application/Query/CalculateAmazonPrice/CalculateAmazonPriceResult.php
final class CalculateAmazonPriceResult
{
    public function __construct(
        public readonly Money $price,
    ) {}
}
```

### Command DTO + CommandHandler

```php
// Application/Command/UpdatePricingRules/UpdatePricingRulesCommand.php
final class UpdatePricingRulesCommand
{
    public function __construct(
        public readonly string $marketplace,
        public readonly float  $coefficient,
        public readonly float  $markupPercent,
    ) {}
}

// Application/Command/UpdatePricingRules/UpdatePricingRulesCommandHandler.php
final class UpdatePricingRulesCommandHandler
{
    public function __construct(
        private PricingRulesRepositoryInterface $repository,
    ) {}

    public function handle(UpdatePricingRulesCommand $command): void
    {
        $this->repository->save(
            $command->marketplace,
            new PricingCoefficient($command->coefficient),
            new Percentage($command->markupPercent),
        );
    }
}
```

### Presentation: Handler вызывает QueryHandler напрямую

Шины нет — всё синхронно. Handler получает конкретный QueryHandler через DI.

```php
// Presentation/Handler/CalculateAmazonPriceHandler.php
final class CalculateAmazonPriceHandler implements RequestHandlerInterface
{
    public function __construct(
        private CalculateAmazonPriceQueryHandler $handler,
    ) {}

    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        $body   = $request->getParsedBody();
        $result = $this->handler->handle(
            new CalculateAmazonPriceQuery($body['product_id'])
        );

        return new JsonResponse(['price_cents' => $result->price->amount]);
    }
}
```

### Поток CQRS

```
POST /openapi/CatalogData/v1/price/amazon
        │
        ▼ [Presentation]
CalculateAmazonPriceHandler
        │ handler->handle(CalculateAmazonPriceQuery)   ← прямой вызов через DI
        ▼ [Application]
CalculateAmazonPriceQueryHandler::handle()
        ├── SupplierPriceProviderInterface  → SupplierOpenApiAdapter
        ├── ShippingCostCalculatorInterface → ShippingOpenApiAdapter
        ├── PricingRulesRepositoryInterface → DataStorePricingRulesRepository
        └── PriceCalculatorService [Domain]
                └── CalculateAmazonPriceResult { price: Money::usd(1500) }

POST /openapi/CatalogData/v1/pricing-rules
        │
        ▼ [Presentation]
UpdatePricingRulesHandler
        │ handler->handle(UpdatePricingRulesCommand)   ← прямой вызов через DI
        ▼ [Application]
UpdatePricingRulesCommandHandler::handle()
        └── PricingRulesRepositoryInterface → DataStorePricingRulesRepository → rollun-datastore
```

---

## Infrastructure Layer

### OpenAPI Adapter (поставщик)

```php
final class SupplierOpenApiAdapter implements SupplierPriceProviderInterface
{
    // baseUrl = 'http://supplier/openapi/SupplierPrices/v1'
    public function __construct(
        private readonly HttpClientInterface $httpClient,
        private readonly string $baseUrl
    ) {}

    public function getPriceForProduct(ProductId $productId): Money
    {
        $response = $this->httpClient->get("{$this->baseUrl}/prices/{$productId->value()}");
        $data     = json_decode($response->getBody()->getContents(), true);

        return Money::usd($data['amount_cents']);
    }
}
```

### OpenAPI Adapter (доставка)

```php
final class ShippingOpenApiAdapter implements ShippingCostCalculatorInterface
{
    // baseUrl = 'http://carriers/openapi/ShippingMethods/v1'
    public function __construct(
        private readonly HttpClientInterface $httpClient,
        private readonly string $baseUrl
    ) {}

    public function calculate(ProductId $productId, string $marketplace): Money
    {
        $response = $this->httpClient->post("{$this->baseUrl}/shipping", [
            'json' => [
                'productId'   => $productId->value(),
                'marketplace' => $marketplace,
            ],
        ]);
        $data = json_decode($response->getBody()->getContents(), true);

        return Money::usd($data['shippingCost']['amountCents']);
    }
}
```

### DataStore Repository (правила цен)

```php
final class DataStorePricingRulesRepository implements PricingRulesRepositoryInterface
{
    public function __construct(
        private readonly DataStoreInterface $dataStore  // rollun-datastore
    ) {}

    public function findByMarketplace(string $marketplace): PricingRules
    {
        $record = $this->dataStore->read($marketplace);

        return new PricingRules(
            coefficient: new PricingCoefficient($record['coefficient']),
            markup:      new Percentage($record['markup_percent'])
        );
    }
}
```

---

## Presentation Layer

### Handler

```php
final class CalculateAmazonPriceHandler implements RequestHandlerInterface
{
    public function __construct(
        private CalculateAmazonPriceUseCase $useCase
    ) {}

    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        $body     = $request->getParsedBody();
        $response = $this->useCase->execute(
            new CalculateAmazonPriceRequest(new ProductId($body['product_id']))
        );

        return new JsonResponse(['price_cents' => $response->price->amount]);
    }
}
```

### Роутинг

```php
// Бизнес-роуты (OpenAPI) → через полный стек Clean Architecture
$app->post('/openapi/CatalogData/v1/price/amazon', CalculateAmazonPriceHandler::class);
$app->post('/openapi/CatalogData/v1/price/ebay',   CalculateEbayPriceHandler::class);

// DataStore роут → напрямую через rollun-datastore, минуя Domain/Application
$app->route(
    '/api/datastore[/{resourceName}[/{id}]]',
    DataStoreApi::class,
    ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
    DataStoreApi::class
);
```

---

## URL-форматы в системе

```
# OpenAPI сервисы — бизнес-операции
http://{service}/openapi/{ServiceName}/v{N}/{endpoint}
http://catalog/openapi/CatalogData/v1/price/amazon
http://carriers/openapi/ShippingMethods/v1/shipping

# DataStore — CRUD операции с данными
http://{service}/api/datastore/{resourceName}[/{id}]
http://catalog/api/datastore/pricing-rules
http://catalog/api/datastore/pricing-rules/amazon
```

---

## Поток данных

```
POST http://catalog/openapi/CatalogData/v1/price/amazon
        │
        ▼ [Presentation]
CalculateAmazonPriceHandler
        │ new CalculateAmazonPriceRequest(productId)
        ▼ [Application]
CalculateAmazonPriceUseCase
        ├── SupplierPriceProviderInterface
        │       └── [Infrastructure] SupplierOpenApiAdapter
        │               └── GET http://supplier/openapi/SupplierPrices/v1/prices/{id}
        ├── ShippingCostCalculatorInterface
        │       └── [Infrastructure] ShippingOpenApiAdapter
        │               └── POST http://carriers/openapi/ShippingMethods/v1/shipping
        ├── PricingRulesRepositoryInterface
        │       └── [Infrastructure] DataStorePricingRulesRepository
        │               └── rollun-datastore → GET /api/datastore/pricing-rules/amazon
        └── PriceCalculatorService [Domain]
                └── Money × Coefficient + Percentage% + ShippingCost
                        │
                        ▼ [Presentation]
                JsonResponse { price_cents: 1500 }
```

---

## Архитектурный контроль (Deptrac)

```yaml
# deptrac.yaml
layers:
  - name: Domain
    collectors:
      - { type: directory, value: src/Domain/.* }
  - name: Application
    collectors:
      - { type: directory, value: src/Application/.* }
  - name: Infrastructure
    collectors:
      - { type: directory, value: src/Infrastructure/.* }
  - name: Presentation
    collectors:
      - { type: directory, value: src/Presentation/.* }
ruleset:
  Domain:         []               # ни от кого не зависит
  Application:    [Domain]         # только от Domain
  Infrastructure: [Domain]         # только от Domain
  Presentation:   [Application]    # только от Application
```

---

## Стратегия тестирования

```php
// Unit — Domain (без моков, без фреймворка)
$result = (new PriceCalculatorService())->calculate(
    Money::usd(1000),              // $10.00
    new PricingCoefficient(1.2),
    new Percentage(10.0),
    Money::usd(200)                // $2.00
);
// Ожидается: 1000*1.2 + 1000*0.1 + 200 = 1500 центов = $15.00

// Unit — Use Case (мок портов)
$supplierProvider = $this->createMock(SupplierPriceProviderInterface::class);
$supplierProvider->method('getPriceForProduct')->willReturn(Money::usd(1000));

$shippingCalc = $this->createMock(ShippingCostCalculatorInterface::class);
$shippingCalc->method('calculate')->willReturn(Money::usd(200));

// Integration — Infrastructure (реальный HTTP к тестовому окружению)
```

---

## Зависимости

```json
{
    "require": {
        "php": "^8.1",
        "mezzio/mezzio": "^3.18",
        "mezzio/mezzio-fastroute": "^3.12",
        "laminas/laminas-servicemanager": "^3.22",
        "laminas/laminas-diactoros": "^3.3",
        "rollun-com/rollun-datastore": "^11.0",
        "rollun-com/rollun-callback": "^8.0.0",
        "rollun-com/rollun-logger": "^7.7.2",
        "rollun-com/rollun-openapi": "^11.0.0"
    },
    "require-dev": {
        "phpunit/phpunit": "^11.0",
        "phpstan/phpstan": "^1.10",
        "qossmic/deptrac": "^2.0",
        "friendsofphp/php-cs-fixer": "^3.0"
    }
}
```

---

## Roadmap реализации

1. **Domain** — Value Objects + `PriceCalculatorService` + Port-интерфейсы → unit-тесты
2. **Application** — `CalculateAmazonPriceUseCase` + `CalculateEbayPriceUseCase` + DTO → unit-тесты с моками
3. **Infrastructure** — OpenAPI Adapters + DataStore Repository + `ConfigProvider` → интеграционные тесты
4. **Presentation** — Handlers + Routes → E2E через HTTP
5. **CI** — Deptrac + PHPStan level 9 + PHP CS Fixer + PHPUnit

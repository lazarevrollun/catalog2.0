# Catalog Pricing Microservice — Архитектура

**Стек:** PHP 8.2 · Laminas/Mezzio · Clean Architecture · DDD
**Дата:** 2026-03-13
**Автор:** Iliya

---

## Концепция

Микросервис каталога считает цену товара для маркетплейсов (Amazon, eBay) по формуле:

```
FinalPrice = SupplierPrice × Coefficient + (SupplierPrice × Markup%) + ShippingCost
```

### Характеристики

- **Валюта:** только USD (хранится в центах как `int`)
- **Цена поставщика:** загружается асинхронно Cron-процессом в БД (не в реальном времени)
- **Стоимость доставки:** также загружается Cron-процессом и хранится в БД
- **Для каждого маркетплейса** (Amazon, eBay) — свои коэффициенты и наценки, хранятся в БД
- **Два сценария работы:**
  - **Синхронный (HTTP):** расчёт цены по данным из БД → 1 сек
  - **Асинхронный (Cron):** обновление данных поставщиков в БД → каждые 12 часов

---

## Архитектурные слои

| Слой | Содержимое | Зависимости |
|---|---|---|
| **Domain** | Entities, Value Objects, Domain Services, Port-интерфейсы | Никаких внешних |
| **Application** | Use Cases, Query/Command Handlers, DTO | Только Domain |
| **Infrastructure** | Repositories, Adapters, DI ConfigProvider | Domain + PHP libs |
| **Presentation** | Mezzio Handlers, роутинг, OpenAPI маппинг | Application |

**Правило:** зависимости только внутрь (к Domain). Контроль через Deptrac.

---

## Точка входа: rollun-openapi

Все HTTP-запросы идут через OpenAPI манифест `openapi/CatalogData/v1.yaml`.

`rollun-openapi` генерирует код структуру:

```
src/CatalogData/
└── OpenAPI/V1/
    ├── Client/                      ← сгенерированный клиент
    │   ├── Api/CatalogApi.php
    │   ├── Rest/CalculateAmazonPrice.php
    │   ├── Rest/CalculateEbayPrice.php
    │   └── Configuration.php
    └── DTO/                         ← сгенерированные DTO
        ├── AmazonPriceRequest.php
        ├── AmazonPriceResponse.php
        ├── EbayPriceRequest.php
        └── EbayPriceResponse.php
```

> Не редактировать вручную. Regenerate при изменении манифеста.

---

## Синхронный сценарий: расчёт цены (HTTP → читаем из БД)

```
Маркетплейс (Amazon adapter)
        │
        │ POST http://catalog/openapi/CatalogData/v1/price/amazon
        │ { product_id: "SKU-123" }
        ▼
[rollun-openapi маппит → CalculateAmazonPriceHandler]
        ▼
CalculateAmazonPriceHandler [Presentation]
        │ new CalculateAmazonPriceQuery("SKU-123")
        ▼
CalculateAmazonPriceQueryHandler [Application]
        ├── PricingRulesRepository.findByMarketplace('amazon')  → БД таблица pricing_rules
        ├── SupplierPricesRepository.findByProductId('SKU-123') → БД таблица supplier_prices
        ├── ShippingCostsRepository.find('SKU-123', 'amazon')   → БД таблица shipping_costs
        └── PriceCalculatorService [Domain]
                └── Money × Coefficient + Markup% + ShippingCost
                        │
                        ▼ [Presentation]
                AmazonPriceResponse { price_cents: 1500 }
                        │
                        ▼
                JSON 200
```

### Handler маппинга OpenAPI DTO

```php
// Presentation/Handler/CalculateAmazonPriceHandler.php
final class CalculateAmazonPriceHandler implements RequestHandlerInterface
{
    public function __construct(
        private CalculateAmazonPriceQueryHandler $handler,
    ) {}

    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        /** @var AmazonPriceRequest $dto */
        $dto = $request->getAttribute(AmazonPriceRequest::class);

        $result = $this->handler->handle(
            new CalculateAmazonPriceQuery($dto->product_id)
        );

        $response = new AmazonPriceResponse();
        $response->price_cents = $result->price->amount;

        return new JsonResponse($response);
    }
}
```

---

## Асинхронный сценарий: обновление данных (Cron → записываем в БД)

Cron-процесс запускается отдельно (через `rollun-callback`), вне HTTP цикла:

```
Cron запуск (каждые 12 часов)
        │
        ▼
UpdateSupplierPricesCommand [Application]
        │ new UpdateSupplierPricesCommand()
        ▼
UpdateSupplierPricesCommandHandler [Application]
        ├── SupplierOpenApiAdapter [Infrastructure]
        │       └── GET http://supplier1/prices
        │       └── GET http://supplier2/prices
        │       → массив данных
        │
        └── SupplierPricesRepository [Infrastructure]
                └── insertOrUpdate(productId, price, supplierId)
                        │
                        ▼ БД таблица supplier_prices
                ✓ Завершено
```

### Пример Cron Command Handler

```php
// Application/Command/UpdateSupplierPrices/UpdateSupplierPricesCommand.php
final class UpdateSupplierPricesCommand
{
    public function __construct(
        public readonly string $supplierId = '',  // пусто = все поставщики
    ) {}
}

// Application/Command/UpdateSupplierPrices/UpdateSupplierPricesCommandHandler.php
final class UpdateSupplierPricesCommandHandler
{
    public function __construct(
        private SupplierOpenApiAdapterInterface $supplierAdapter,
        private SupplierPricesRepository $pricesRepository,
    ) {}

    public function handle(UpdateSupplierPricesCommand $command): void
    {
        $suppliers = $command->supplierId
            ? [$command->supplierId]
            : $this->pricesRepository->getAllSupplierIds();

        foreach ($suppliers as $supplierId) {
            try {
                $prices = $this->supplierAdapter->fetchPrices($supplierId);

                foreach ($prices as $productId => $priceCents) {
                    $this->pricesRepository->insertOrUpdate(
                        $productId,
                        Money::usd($priceCents),
                        $supplierId
                    );
                }
            } catch (SupplierApiException $e) {
                // логируем и продолжаем со следующего поставщика
                \error_log("Supplier $supplierId error: " . $e->getMessage());
            }
        }
    }
}
```

### Регистрация Cron (rollun-callback)

```php
// config/autoload/cron.local.php
return [
    'rollun-callback' => [
        'jobs' => [
            'update-supplier-prices' => [
                'callback' => UpdateSupplierPricesCallback::class,
                'schedule' => '0 */12 * * *',  // каждые 12 часов
            ],
            'update-shipping-costs' => [
                'callback' => UpdateShippingCostsCallback::class,
                'schedule' => '0 */12 * * *',  // каждые 12 часов
            ],
        ],
    ],
];

// Infrastructure/Callback/UpdateSupplierPricesCallback.php
final class UpdateSupplierPricesCallback implements CallbackInterface
{
    public function __invoke(): void
    {
        $handler = $this->container->get(UpdateSupplierPricesCommandHandler::class);
        $handler->handle(new UpdateSupplierPricesCommand());
    }
}
```

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
│   │   └── SupplierId.php
│   ├── Service/
│   │   └── PriceCalculatorService.php
│   └── Port/
│       ├── SupplierPricesRepositoryInterface.php
│       ├── ShippingCostsRepositoryInterface.php
│       ├── PricingRulesRepositoryInterface.php
│       └── SupplierOpenApiAdapterInterface.php
│
├── Application/
│   ├── Query/
│   │   ├── CalculateAmazonPrice/
│   │   │   ├── CalculateAmazonPriceQuery.php
│   │   │   ├── CalculateAmazonPriceQueryHandler.php
│   │   │   └── CalculateAmazonPriceResult.php
│   │   └── CalculateEbayPrice/
│   │       ├── CalculateEbayPriceQuery.php
│   │       ├── CalculateEbayPriceQueryHandler.php
│   │       └── CalculateEbayPriceResult.php
│   │
│   └── Command/
│       ├── UpdateSupplierPrices/
│       │   ├── UpdateSupplierPricesCommand.php
│       │   └── UpdateSupplierPricesCommandHandler.php
│       └── UpdateShippingCosts/
│           ├── UpdateShippingCostsCommand.php
│           └── UpdateShippingCostsCommandHandler.php
│
├── Infrastructure/
│   ├── Adapter/
│   │   ├── Supplier/
│   │   │   └── SupplierOpenApiAdapter.php
│   │   └── Shipping/
│   │       └── ShippingOpenApiAdapter.php
│   ├── Repository/
│   │   ├── SupplierPricesRepository.php
│   │   ├── ShippingCostsRepository.php
│   │   └── PricingRulesRepository.php
│   ├── Callback/
│   │   ├── UpdateSupplierPricesCallback.php
│   │   └── UpdateShippingCostsCallback.php
│   └── DI/
│       └── ConfigProvider.php
│
└── Presentation/
    ├── Handler/
    │   ├── CalculateAmazonPriceHandler.php
    │   └── CalculateEbayPriceHandler.php
    └── RoutesDelegator.php

tests/
├── Unit/
│   ├── Domain/Service/PriceCalculatorServiceTest.php
│   ├── Domain/ValueObject/MoneyTest.php
│   ├── Application/Query/CalculateAmazonPriceTest.php
│   └── Application/Command/UpdateSupplierPricesTest.php
└── Integration/
    ├── Infrastructure/SupplierOpenApiAdapterTest.php
    └── Infrastructure/Repository/SupplierPricesRepositoryTest.php
```

---

## Domain Layer

### Value Objects

```php
// Domain/ValueObject/Money.php
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

// Domain/ValueObject/ProductId.php
final class ProductId
{
    public function __construct(
        public readonly string $value
    ) {
        if (empty($value)) {
            throw new \InvalidArgumentException('Product ID cannot be empty');
        }
    }
}

// Domain/ValueObject/PricingCoefficient.php
final class PricingCoefficient
{
    public function __construct(
        public readonly float $value
    ) {
        if ($value <= 0) {
            throw new \InvalidArgumentException('Coefficient must be positive');
        }
    }
}

// Domain/ValueObject/Percentage.php
final class Percentage
{
    public function __construct(
        public readonly float $value
    ) {
        if ($value < 0 || $value > 100) {
            throw new \InvalidArgumentException('Percentage must be between 0 and 100');
        }
    }
}
```

### Domain Service

```php
// Domain/Service/PriceCalculatorService.php
final class PriceCalculatorService
{
    public function calculate(
        Money $supplierPrice,
        PricingCoefficient $coefficient,
        Percentage $markup,
        Money $shippingCost
    ): Money {
        // Base: SupplierPrice × Coefficient
        $base = $supplierPrice->multiply($coefficient->value);

        // Markup: SupplierPrice × Markup%
        $markupAmount = $supplierPrice->multiply($markup->value / 100);

        // Final: Base + Markup + Shipping
        return $base->add($markupAmount)->add($shippingCost);
    }
}
```

### Ports (интерфейсы)

```php
// Domain/Port/SupplierPricesRepositoryInterface.php
interface SupplierPricesRepositoryInterface
{
    public function findByProductId(ProductId $productId): Money;
    public function insertOrUpdate(ProductId $productId, Money $price, string $supplierId): void;
    public function getAllSupplierIds(): array;
}

// Domain/Port/ShippingCostsRepositoryInterface.php
interface ShippingCostsRepositoryInterface
{
    public function find(ProductId $productId, string $marketplace): Money;
    public function insertOrUpdate(ProductId $productId, string $marketplace, Money $cost): void;
}

// Domain/Port/PricingRulesRepositoryInterface.php
interface PricingRulesRepositoryInterface
{
    public function findByMarketplace(string $marketplace): PricingRules;
    public function save(string $marketplace, PricingCoefficient $coefficient, Percentage $markup): void;
}

// Domain/Port/SupplierOpenApiAdapterInterface.php
interface SupplierOpenApiAdapterInterface
{
    /**
     * Получить цены от поставщика
     * @return array ['supplier_product_id' => price_in_cents, ...]
     * Например: ['SKU-001' => 1000, 'SKU-002' => 2000]
     */
    public function fetchPrices(string $supplierId): array;
}
```

---

## Application Layer

### Query: Расчёт цены Amazon (синхронно из БД)

```php
// Application/Query/CalculateAmazonPrice/CalculateAmazonPriceQuery.php
final class CalculateAmazonPriceQuery
{
    public function __construct(
        public readonly string $productId,
    ) {}
}

// Application/Query/CalculateAmazonPrice/CalculateAmazonPriceQueryHandler.php
final class CalculateAmazonPriceQueryHandler
{
    public function __construct(
        private SupplierPricesRepositoryInterface $pricesRepository,
        private ShippingCostsRepositoryInterface $shippingRepository,
        private PricingRulesRepositoryInterface $rulesRepository,
        private PriceCalculatorService $calculator,
    ) {}

    public function handle(CalculateAmazonPriceQuery $query): CalculateAmazonPriceResult
    {
        $productId = new ProductId($query->productId);

        // Все данные читаются из БД (загруженные Cron-ом)
        $supplierPrice = $this->pricesRepository->findByProductId($productId);
        $shippingCost  = $this->shippingRepository->find($productId, 'amazon');
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

// Application/Query/CalculateAmazonPrice/CalculateAmazonPriceResult.php
final class CalculateAmazonPriceResult
{
    public function __construct(
        public readonly Money $price,
    ) {}
}
```

### Command: Обновление цен поставщика (асинхронно в БД)

```php
// Application/Command/UpdateSupplierPrices/UpdateSupplierPricesCommand.php
final class UpdateSupplierPricesCommand
{
    public function __construct(
        public readonly string $supplierId = '',  // пусто = обновить всех
    ) {}
}

// Application/Command/UpdateSupplierPrices/UpdateSupplierPricesCommandHandler.php
final class UpdateSupplierPricesCommandHandler
{
    public function __construct(
        private SupplierOpenApiAdapterInterface $supplierAdapter,
        private SupplierPricesRepositoryInterface $pricesRepository,
    ) {}

    /**
     * Обновляет сырые цены поставщика в БД
     * Вызывается Cron-процессом каждые 12 часов
     *
     * Процесс:
     * 1. Получить список товаров поставщика из маппинга
     * 2. Вызвать API поставщика для получения цен
     * 3. Сохранить цены в таблицу supplier_prices
     * 4. Система Query Handler потом получит лучшую цену через маппинг
     */
    public function handle(UpdateSupplierPricesCommand $command): void
    {
        $suppliers = $command->supplierId
            ? [$command->supplierId]
            : $this->pricesRepository->getAllSupplierIds();

        foreach ($suppliers as $supplierId) {
            try {
                // Получить все товары, которые этот поставщик может обслуживать
                $supplierProductIds = $this->pricesRepository->getSupplierProductIds($supplierId);

                if (empty($supplierProductIds)) {
                    \error_log("No products mapped for supplier $supplierId");
                    continue;
                }

                // Вызвать API поставщика для получения цен
                // API возвращает данные в формате:
                // ['supplier_product_id' => price_cents, ...]
                $prices = $this->supplierAdapter->fetchPrices($supplierId);

                // Сохранить цены в БД (таблица supplier_prices)
                foreach ($prices as $supplierProductId => $priceCents) {
                    $this->pricesRepository->insertOrUpdate(
                        $supplierId,
                        $supplierProductId,
                        Money::usd($priceCents)
                    );
                }

                \error_log("Successfully updated prices for supplier $supplierId: " . count($prices) . " items");
            } catch (\Exception $e) {
                \error_log("Failed to update supplier $supplierId: " . $e->getMessage());
            }
        }
    }
}
```

---

## Infrastructure Layer

### Repositories (прямой доступ к БД)

```php
// Infrastructure/Repository/SupplierPricesRepository.php
// Работает с сырыми ценами поставщиков
final class SupplierPricesRepository implements SupplierPricesRepositoryInterface
{
    public function __construct(
        private \PDO $pdo,
    ) {}

    /**
     * Получить лучшую цену товара из всех поставщиков через маппинг
     */
    public function findByProductId(ProductId $productId): Money
    {
        // SQL запрос идёт через маппинг:
        // product_supplier_mapping.product_id
        //   → product_supplier_mapping.supplier_id
        //   → supplier_prices.(supplier_id, supplier_product_id)
        $stmt = $this->pdo->prepare(
            'SELECT sp.price_cents
             FROM supplier_prices sp
             INNER JOIN product_supplier_mapping psm
               ON sp.supplier_id = psm.supplier_id
               AND sp.supplier_product_id = psm.supplier_product_id
             WHERE psm.product_id = ? AND psm.is_active = true
             ORDER BY sp.price_cents ASC
             LIMIT 1'
        );
        $stmt->execute([$productId->value]);

        $row = $stmt->fetch(\PDO::FETCH_ASSOC);
        if (!$row) {
            throw new \Exception("Price not found for product {$productId->value}");
        }

        return Money::usd((int) $row['price_cents']);
    }

    /**
     * Вставить или обновить цену поставщика
     * Вызывается Cron-процессом для обновления сырых данных
     */
    public function insertOrUpdate(
        string $supplierId,
        string $supplierProductId,
        Money $price
    ): void {
        $stmt = $this->pdo->prepare(
            'INSERT INTO supplier_prices (supplier_id, supplier_product_id, price_cents, updated_at)
             VALUES (?, ?, ?, NOW())
             ON DUPLICATE KEY UPDATE
             price_cents = VALUES(price_cents),
             updated_at = NOW()'
        );

        $stmt->execute([
            $supplierId,
            $supplierProductId,
            $price->amount,
        ]);
    }

    /**
     * Получить все активные поставщики
     */
    public function getAllSupplierIds(): array
    {
        $stmt = $this->pdo->query(
            'SELECT DISTINCT supplier_id FROM suppliers WHERE is_active = true'
        );
        return $stmt->fetchAll(\PDO::FETCH_COLUMN, 0);
    }

    /**
     * Получить все товары поставщика (для Cron при обновлении)
     */
    public function getSupplierProductIds(string $supplierId): array
    {
        $stmt = $this->pdo->prepare(
            'SELECT supplier_product_id FROM product_supplier_mapping WHERE supplier_id = ?'
        );
        $stmt->execute([$supplierId]);
        return $stmt->fetchAll(\PDO::FETCH_COLUMN, 0);
    }
}
```

### OpenAPI Adapters (синхронно вызывают внешние сервисы)

```php
// Infrastructure/Adapter/Supplier/SupplierOpenApiAdapter.php
final class SupplierOpenApiAdapter implements SupplierOpenApiAdapterInterface
{
    public function __construct(
        private HttpClientInterface $httpClient,
        private array $supplierUrls,  // конфиг ['supplier1' => 'http://...', ...]
    ) {}

    /**
     * Получить цены от поставщика через OpenAPI
     * API поставщика возвращает свои айди товаров (supplier_product_id)
     * Наша система через маппинг свяжет их с нашими product_id
     */
    public function fetchPrices(string $supplierId): array
    {
        $url = $this->supplierUrls[$supplierId] ?? null;
        if (!$url) {
            throw new \InvalidArgumentException("Unknown supplier: $supplierId");
        }

        try {
            $response = $this->httpClient->get("$url/prices");
            $data = \json_decode($response->getBody()->getContents(), true);

            // Трансформируем ответ поставщика в наш формат:
            // ['supplier_product_id' => price_cents]
            $prices = [];
            foreach ($data['items'] ?? [] as $item) {
                // Предполагаем, что API возвращает supplier_id как 'id' или 'sku'
                $supplierProductId = $item['id'] ?? $item['sku'] ?? null;
                if (!$supplierProductId) {
                    \error_log("Supplier API item missing id/sku: " . json_encode($item));
                    continue;
                }

                $priceCents = (int) ($item['price_cents'] ?? $item['price'] ?? 0);
                if ($priceCents <= 0) {
                    \error_log("Supplier API item has invalid price: $supplierProductId");
                    continue;
                }

                $prices[$supplierProductId] = $priceCents;
            }

            return $prices;
        } catch (\Exception $e) {
            throw new \RuntimeException(
                "Failed to fetch prices from supplier $supplierId: " . $e->getMessage(),
                0,
                $e
            );
        }
    }
}
```

---

## Presentation Layer

### Handler маппит OpenAPI DTO → Query/Command

```php
// Presentation/Handler/CalculateAmazonPriceHandler.php
final class CalculateAmazonPriceHandler implements RequestHandlerInterface
{
    public function __construct(
        private CalculateAmazonPriceQueryHandler $queryHandler,
    ) {}

    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        /** @var AmazonPriceRequest $dto */
        $dto = $request->getAttribute(AmazonPriceRequest::class);

        $result = $this->queryHandler->handle(
            new CalculateAmazonPriceQuery($dto->product_id)
        );

        $response = new AmazonPriceResponse();
        $response->price_cents = $result->price->amount;

        return new JsonResponse($response);
    }
}

// Presentation/Handler/CalculateEbayPriceHandler.php
final class CalculateEbayPriceHandler implements RequestHandlerInterface
{
    public function __construct(
        private CalculateEbayPriceQueryHandler $queryHandler,
    ) {}

    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        /** @var EbayPriceRequest $dto */
        $dto = $request->getAttribute(EbayPriceRequest::class);

        $result = $this->queryHandler->handle(
            new CalculateEbayPriceQuery($dto->product_id)
        );

        $response = new EbayPriceResponse();
        $response->price_cents = $result->price->amount;

        return new JsonResponse($response);
    }
}
```

### Роутинг

```php
// Presentation/RoutesDelegator.php
final class RoutesDelegator
{
    public function __invoke(Application $app, ContainerInterface $container): void
    {
        // Бизнес-эндпоинты (через OpenAPI)
        $app->post(
            '/openapi/CatalogData/v1/price/amazon',
            CalculateAmazonPriceHandler::class
        );
        $app->post(
            '/openapi/CatalogData/v1/price/ebay',
            CalculateEbayPriceHandler::class
        );
    }
}
```

---

## Схема БД

### Основные таблицы для работы с ценами

```sql
-- Маппинг товаров между вашей системой и поставщиками
-- Связывает ваш product_id (внутренний) с supplier_product_id (в системе поставщика)
CREATE TABLE product_supplier_mapping (
    id INT AUTO_INCREMENT PRIMARY KEY,
    product_id VARCHAR(100) NOT NULL,              -- ваш внутренний айди товара
    supplier_id VARCHAR(100) NOT NULL,             -- айди поставщика (Amazon, eBay, supplier1, ...)
    supplier_product_id VARCHAR(100) NOT NULL,     -- айди товара в системе поставщика
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_product_supplier (product_id, supplier_id),
    INDEX idx_supplier_product (supplier_id, supplier_product_id),
    INDEX idx_product (product_id)
);

-- Таблица цен поставщиков (обновляется асинхронно Cron)
-- Цена из исходного источника данных (поставщика)
CREATE TABLE supplier_prices (
    id INT AUTO_INCREMENT PRIMARY KEY,
    supplier_id VARCHAR(100) NOT NULL,
    supplier_product_id VARCHAR(100) NOT NULL,     -- айди товара в системе поставщика
    price_cents INT NOT NULL,                       -- цена в центах USD
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_supplier_product (supplier_id, supplier_product_id),
    INDEX idx_updated (updated_at)
);

-- Таблица итоговых цен по товарам (вычисляется из supplier_prices через маппинг)
-- Если товар есть у нескольких поставщиков, здесь хранится лучшая цена
CREATE TABLE product_prices (
    id INT AUTO_INCREMENT PRIMARY KEY,
    product_id VARCHAR(100) NOT NULL UNIQUE,
    best_supplier_id VARCHAR(100),                 -- какой поставщик дал лучшую цену
    best_price_cents INT NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_product (product_id),
    INDEX idx_updated (updated_at)
);

-- Таблица стоимости доставки (обновляется асинхронно Cron)
CREATE TABLE shipping_costs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    product_id VARCHAR(100) NOT NULL,
    marketplace VARCHAR(50) NOT NULL,
    cost_cents INT NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_product_marketplace (product_id, marketplace),
    INDEX idx_product (product_id)
);

-- Таблица правил ценообразования (редко обновляется)
CREATE TABLE pricing_rules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    marketplace VARCHAR(50) NOT NULL UNIQUE,
    coefficient DECIMAL(5, 2) NOT NULL,
    markup_percent DECIMAL(5, 2) NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_marketplace (marketplace)
);

-- Справочник поставщиков
CREATE TABLE suppliers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    supplier_id VARCHAR(100) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    api_url VARCHAR(500),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_active (is_active)
);
```

### Поток работы с маппингом

```
1. product_supplier_mapping (маппинг)
   product_id (ваш) → supplier_id → supplier_product_id (у поставщика)

2. supplier_prices (сырые цены от поставщика)
   supplier_id + supplier_product_id → price_cents

3. product_prices (итоговые цены, объединённые по вашим товарам)
   product_id → best_supplier_id + best_price_cents
```

---

## Управление маппингами товаров

Маппинг связывает ваши товары с товарами поставщиков. Без правильного маппинга система не сможет получить цены.

### Как установить маппинг

**Вариант 1: Вручную через API (INSERT в таблицу)**

```sql
-- Связать ваш товар product_id="INTERNAL-001"
-- с товаром поставщика supplier="supplier1" (айди "SKU-123")
INSERT INTO product_supplier_mapping (product_id, supplier_id, supplier_product_id, is_active)
VALUES ('INTERNAL-001', 'supplier1', 'SKU-123', true);

-- Можно один товар связать с несколькими поставщиками
INSERT INTO product_supplier_mapping (product_id, supplier_id, supplier_product_id, is_active)
VALUES ('INTERNAL-001', 'supplier2', 'PROD-456', true);
```

**Вариант 2: Через Application Layer (Command)**

```php
// Application/Command/CreateProductMapping/CreateProductMappingCommand.php
final class CreateProductMappingCommand
{
    public function __construct(
        public readonly string $productId,           // ваш айди
        public readonly string $supplierId,          // поставщик
        public readonly string $supplierProductId,   // айди у поставщика
    ) {}
}

// Application/Command/CreateProductMapping/CreateProductMappingCommandHandler.php
final class CreateProductMappingCommandHandler
{
    public function __construct(
        private ProductMappingRepository $repository,
    ) {}

    public function handle(CreateProductMappingCommand $command): void
    {
        $this->repository->create(
            $command->productId,
            $command->supplierId,
            $command->supplierProductId,
            true  // is_active
        );
    }
}
```

### Как получить цены после установки маппинга

```
1. Установить маппинг в product_supplier_mapping
2. Cron-процесс вызывает UpdateSupplierPricesCommand
3. Adapter получает цены от поставщика по supplier_product_id
4. Цены сохраняются в supplier_prices
5. Query Handler читает цены через маппинг:
   product_id → product_supplier_mapping → supplier_prices
6. Возвращает итоговую цену
```

---

## Поток данных

### Синхронный: HTTP запрос → расчёт цены

```
1. GET /openapi/CatalogData/v1/price/amazon?product_id=INTERNAL-001
   [Presentation]
2. CalculateAmazonPriceHandler маппит OpenAPI DTO
3. new CalculateAmazonPriceQuery('INTERNAL-001')
   [Application]
4. CalculateAmazonPriceQueryHandler::handle()
   - Читает из БД через маппинг:
     product_supplier_mapping WHERE product_id = 'INTERNAL-001'
     → получить supplier_product_id для поставщика
     → supplier_prices WHERE (supplier_id, supplier_product_id)
     → SELECT price_cents (лучшую цену)

   - Читает shipping_costs для 'INTERNAL-001' и 'amazon'
   - Читает pricing_rules для 'amazon'

   [Domain]
5. PriceCalculatorService::calculate()
   - finalPrice = price × coeff + price × markup% + shipping

   [Presentation]
6. Return AmazonPriceResponse { price_cents: 1500 }

⏱️ ~50ms (одна query с JOIN через маппинг)
```

### Асинхронный: Cron → обновление данных

```
1. Cron триггер (каждые 12 часов)
   [Infrastructure/Callback]
2. UpdateSupplierPricesCallback::__invoke()
   → создаёт UpdateSupplierPricesCommand
   [Application]
3. UpdateSupplierPricesCommandHandler::handle()
   - Получить список поставщиков
   - Для каждого поставщика:
     a) Получить supplier_product_ids из product_supplier_mapping
     b) Вызвать SupplierOpenApiAdapter::fetchPrices()
        GET http://supplier1.com/prices
        → {'SKU-123': 1000, 'SKU-124': 1500, ...}
     c) Сохранить в supplier_prices (supplier_id, supplier_product_id, price)
   [Infrastructure]
4. SupplierPricesRepository::insertOrUpdate()
   - INSERT OR UPDATE supplier_prices
   - (supplier_id='supplier1', supplier_product_id='SKU-123', price_cents=1000)
5. ✓ Данные готовы к использованию синхронными запросами

⏱️ ~30 секунд (для всех поставщиков)

Поток: supplier API → supplier_prices → (через маппинг) → Query Handler
```

---

## Стратегия тестирования

### Unit тесты (Domain)

```php
// tests/Unit/Domain/Service/PriceCalculatorServiceTest.php
final class PriceCalculatorServiceTest extends TestCase
{
    public function testCalculatePrice(): void
    {
        $service = new PriceCalculatorService();

        $result = $service->calculate(
            Money::usd(1000),              // $10.00
            new PricingCoefficient(1.2),   // 20% coefficient
            new Percentage(10.0),          // 10% markup
            Money::usd(200)                // $2.00 shipping
        );

        // 1000 * 1.2 + 1000 * 0.1 + 200 = 1500 центов
        $this->assertEquals(1500, $result->amount);
    }
}
```

### Unit тесты (Application)

```php
// tests/Unit/Application/Query/CalculateAmazonPriceTest.php
final class CalculateAmazonPriceTest extends TestCase
{
    public function testCalculatePrice(): void
    {
        $priceRepo = $this->createMock(SupplierPricesRepositoryInterface::class);
        $priceRepo->method('findByProductId')
            ->willReturn(Money::usd(1000));

        $shippingRepo = $this->createMock(ShippingCostsRepositoryInterface::class);
        $shippingRepo->method('find')
            ->willReturn(Money::usd(200));

        $rulesRepo = $this->createMock(PricingRulesRepositoryInterface::class);
        $rulesRepo->method('findByMarketplace')
            ->willReturn(new PricingRules(
                new PricingCoefficient(1.2),
                new Percentage(10.0)
            ));

        $handler = new CalculateAmazonPriceQueryHandler(
            $priceRepo,
            $shippingRepo,
            $rulesRepo,
            new PriceCalculatorService()
        );

        $result = $handler->handle(new CalculateAmazonPriceQuery('SKU-123'));

        $this->assertEquals(1500, $result->price->amount);
    }
}
```

### Integration тесты (Infrastructure)

```php
// tests/Integration/Infrastructure/SupplierPricesRepositoryTest.php
final class SupplierPricesRepositoryTest extends TestCase
{
    private \PDO $pdo;
    private SupplierPricesRepository $repository;

    protected function setUp(): void
    {
        $this->pdo = $this->getTestDatabase();
        $this->repository = new SupplierPricesRepository($this->pdo);
        $this->seedTestData();
    }

    private function seedTestData(): void
    {
        // Создать маппинг товара
        $this->pdo->exec("
            INSERT INTO product_supplier_mapping (product_id, supplier_id, supplier_product_id, is_active)
            VALUES ('INTERNAL-001', 'supplier1', 'SKU-123', true)
        ");
    }

    public function testInsertAndFindByProductId(): void
    {
        $supplierId = 'supplier1';
        $supplierProductId = 'SKU-123';
        $price = Money::usd(1000);

        // Вставить сырую цену поставщика
        $this->repository->insertOrUpdate($supplierId, $supplierProductId, $price);

        // Найти цену по нашему product_id (через маппинг)
        $productId = new ProductId('INTERNAL-001');
        $found = $this->repository->findByProductId($productId);

        $this->assertTrue($found->equals($price));
    }

    public function testFindByProductIdWithMultipleSuppliers(): void
    {
        // Создать маппинги для нескольких поставщиков
        $this->pdo->exec("
            INSERT INTO product_supplier_mapping (product_id, supplier_id, supplier_product_id, is_active)
            VALUES
            ('INTERNAL-001', 'supplier2', 'PROD-456', true)
        ");

        // Вставить цены от разных поставщиков
        $this->repository->insertOrUpdate('supplier1', 'SKU-123', Money::usd(1000));
        $this->repository->insertOrUpdate('supplier2', 'PROD-456', Money::usd(800));  // дешевле

        // Должна вернуться самая дешевая цена
        $productId = new ProductId('INTERNAL-001');
        $found = $this->repository->findByProductId($productId);

        $this->assertEquals(800, $found->amount);
    }
}
```

---

## Конфигурация DI (ConfigProvider)

```php
// Infrastructure/DI/ConfigProvider.php
final class ConfigProvider
{
    public function __invoke(): array
    {
        return [
            'dependencies' => [
                'factories' => [
                    // Domain Services
                    PriceCalculatorService::class => fn() => new PriceCalculatorService(),

                    // Query Handlers (синхронные)
                    CalculateAmazonPriceQueryHandler::class => fn($container) => new CalculateAmazonPriceQueryHandler(
                        $container->get(SupplierPricesRepository::class),
                        $container->get(ShippingCostsRepository::class),
                        $container->get(PricingRulesRepository::class),
                        $container->get(PriceCalculatorService::class),
                    ),

                    CalculateEbayPriceQueryHandler::class => fn($container) => new CalculateEbayPriceQueryHandler(
                        $container->get(SupplierPricesRepository::class),
                        $container->get(ShippingCostsRepository::class),
                        $container->get(PricingRulesRepository::class),
                        $container->get(PriceCalculatorService::class),
                    ),

                    // Command Handlers (асинхронные)
                    UpdateSupplierPricesCommandHandler::class => fn($container) => new UpdateSupplierPricesCommandHandler(
                        $container->get(SupplierOpenApiAdapter::class),
                        $container->get(SupplierPricesRepository::class),
                    ),

                    UpdateShippingCostsCommandHandler::class => fn($container) => new UpdateShippingCostsCommandHandler(
                        $container->get(ShippingOpenApiAdapter::class),
                        $container->get(ShippingCostsRepository::class),
                    ),

                    // Repositories
                    SupplierPricesRepository::class => fn($container) => new SupplierPricesRepository(
                        $container->get(\PDO::class)
                    ),
                    ShippingCostsRepository::class => fn($container) => new ShippingCostsRepository(
                        $container->get(\PDO::class)
                    ),
                    PricingRulesRepository::class => fn($container) => new PricingRulesRepository(
                        $container->get(\PDO::class)
                    ),

                    // Adapters
                    SupplierOpenApiAdapter::class => fn($container) => new SupplierOpenApiAdapter(
                        $container->get(HttpClientInterface::class),
                        $container->get('config')['suppliers'] ?? []
                    ),

                    // Presentation
                    CalculateAmazonPriceHandler::class => fn($container) => new CalculateAmazonPriceHandler(
                        $container->get(CalculateAmazonPriceQueryHandler::class),
                    ),
                ],

                'aliases' => [
                    SupplierPricesRepositoryInterface::class => SupplierPricesRepository::class,
                    ShippingCostsRepositoryInterface::class => ShippingCostsRepository::class,
                    PricingRulesRepositoryInterface::class => PricingRulesRepository::class,
                    SupplierOpenApiAdapterInterface::class => SupplierOpenApiAdapter::class,
                ],
            ],

            'suppliers' => [
                'supplier1' => 'https://api.supplier1.com/openapi/Prices/v1',
                'supplier2' => 'https://api.supplier2.com/openapi/Prices/v1',
            ],
        ];
    }
}
```

---

## Контроль архитектуры (Deptrac)

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

## Зависимости (composer.json)

```json
{
    "require": {
        "php": "^8.2",
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

1. **Domain Layer** → Value Objects + `PriceCalculatorService` + Port-интерфейсы
2. **Infrastructure Layer** → Repositories + OpenAPI Adapter + DI ConfigProvider
3. **Application Layer** → Query & Command Handlers
4. **Presentation Layer** → HTTP Handlers + OpenAPI маппинг + роутинг
5. **Cron Integration** → Callbacks для асинхронного обновления данных
6. **Tests** → Unit + Integration + E2E
7. **CI** → Deptrac + PHPStan + PHP CS Fixer + PHPUnit


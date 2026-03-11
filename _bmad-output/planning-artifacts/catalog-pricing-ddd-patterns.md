# Catalog Pricing Microservice — DDD Паттерны

**Стек:** PHP 8.1 · Laminas/Mezzio · Clean Architecture · DDD
**Дата:** 2026-03-11

> Файл описывает структуру папок и маппинг классов на DDD-паттерны.
> Только паттерны, релевантные для use case ценообразования.

---

## Структура папок

```
src/
│
├── Domain/
│   ├── ValueObject/
│   │   ├── Money.php                        # [Value Object]
│   │   ├── ProductId.php                    # [Value Object]
│   │   ├── PricingCoefficient.php           # [Value Object]
│   │   └── Percentage.php                   # [Value Object]
│   │
│   ├── Service/
│   │   └── PriceCalculatorService.php       # [Domain Service]
│   │
│   ├── Policy/
│   │   ├── AmazonPricingPolicy.php          # [Policy]
│   │   └── EbayPricingPolicy.php            # [Policy]
│   │
│   ├── Guard/
│   │   └── MinimumPriceGuard.php            # [Guard]
│   │
│   ├── Event/
│   │   └── PriceCalculated.php              # [Domain Event]
│   │
│   └── Exception/
│       ├── DomainException.php              # [DomainException]
│       └── InvariantViolationException.php  # [InvariantViolationException]
│
├── Application/
│   ├── UseCase/
│   │   ├── CalculateAmazonPrice/
│   │   │   ├── CalculateAmazonPriceUseCase.php   # [UseCase / Interactor]
│   │   │   ├── CalculateAmazonPriceCommand.php   # [Command DTO]
│   │   │   └── CalculateAmazonPriceResult.php    # [ResponseDTO]
│   │   └── CalculateEbayPrice/
│   │       ├── CalculateEbayPriceUseCase.php     # [UseCase / Interactor]
│   │       ├── CalculateEbayPriceCommand.php     # [Command DTO]
│   │       └── CalculateEbayPriceResult.php      # [ResponseDTO]
│   │
│   ├── Port/
│   │   ├── SupplierPricePort.php            # [GatewayPort / ApiClientPort]
│   │   ├── ShippingCostPort.php             # [GatewayPort / ApiClientPort]
│   │   └── PricingRulesRepositoryPort.php   # [RepositoryPort]
│   │
│   ├── Validator/
│   │   └── CalculatePriceValidator.php      # [InputValidator]
│   │
│   ├── Normalizer/
│   │   └── CalculatePriceNormalizer.php     # [RequestNormalizer / InputMapper]
│   │
│   └── Exception/
│       ├── ApplicationException.php         # [ApplicationException]
│       ├── ValidationException.php          # [ValidationException]
│       └── ResourceNotFoundException.php    # [ResourceNotFoundException]
│
├── Infrastructure/
│   ├── Http/
│   │   └── CalculatePriceController.php     # [Controller]
│   │
│   ├── Adapter/
│   │   ├── SupplierApiClientAdapter.php     # [ApiClientAdapter]  → реализует SupplierPricePort
│   │   └── ShippingApiClientAdapter.php     # [ApiClientAdapter]  → реализует ShippingCostPort
│   │
│   ├── Cache/
│   │   └── SupplierPriceCacheAdapter.php    # [CacheAdapter]  → декоратор над SupplierApiClientAdapter
│   │
│   ├── Translator/
│   │   ├── SupplierResponseTranslator.php   # [ACL Translator]
│   │   ├── ShippingResponseTranslator.php   # [ACL Translator]
│   │   └── HttpErrorTranslator.php          # [ErrorTranslator]
│   │
│   ├── DTO/
│   │   ├── SupplierPriceResponse.php        # [External DTO]
│   │   └── ShippingMethodsResponse.php      # [External DTO]
│   │
│   └── Exception/
│       ├── InfrastructureException.php      # [InfrastructureException]
│       └── ExternalSystemException.php      # [ExternalSystemException]
│
└── Persistence/
    ├── Repository/
    │   └── PricingRulesRepositoryFacade.php # [RepositoryFacade] → реализует PricingRulesRepositoryPort
    │
    └── Mapper/
        └── PricingRulesMapper.php           # [Decomposer / Mapper]
```

---

## Паттерны — зачем каждый

### Domain

**`[Policy]` — `AmazonPricingPolicy` / `EbayPricingPolicy`**
Инкапсулируют правила ценообразования конкретного маркетплейса:
коэффициент, процент наценки. `PriceCalculatorService` принимает Policy как параметр — не знает про Amazon/eBay.

```php
interface MarketplacePricingPolicy
{
    public function coefficient(): PricingCoefficient;
    public function markup(): Percentage;
}

final class AmazonPricingPolicy implements MarketplacePricingPolicy { ... }
final class EbayPricingPolicy implements MarketplacePricingPolicy { ... }
```

---

**`[Guard]` — `MinimumPriceGuard`**
Проверяет инвариант домена: итоговая цена не может быть ниже минимально допустимой.
Вызывается внутри `PriceCalculatorService` до возврата результата.

```php
final class MinimumPriceGuard
{
    public function check(Money $price, Money $minimum): void
    {
        if ($price->amount < $minimum->amount) {
            throw new InvariantViolationException(
                "Price {$price->amount} is below minimum {$minimum->amount}"
            );
        }
    }
}
```

---

**`[Domain Event]` — `PriceCalculated`**
Публикуется после успешного расчёта. Содержит `productId`, `marketplace`, `finalPrice`, `calculatedAt`.
В текущем use case используется для логирования / аудита.

```php
final class PriceCalculated
{
    public function __construct(
        public readonly ProductId $productId,
        public readonly string    $marketplace,
        public readonly Money     $finalPrice,
        public readonly \DateTimeImmutable $calculatedAt,
    ) {}
}
```

---

### Application

**`[Command DTO]` — `CalculateAmazonPriceCommand`**
Структурированный входной объект Use Case. Заменяет массив `$request->getParsedBody()`.

```php
final class CalculateAmazonPriceCommand
{
    public function __construct(
        public readonly string $productId,
    ) {}
}
```

---

**`[InputValidator]` — `CalculatePriceValidator`**
Проверяет команду до передачи в Use Case: `productId` не пустой, формат корректный.
Бросает `ValidationException` — не `DomainException`.

```php
final class CalculatePriceValidator
{
    public function validate(CalculateAmazonPriceCommand $command): void
    {
        if (empty(trim($command->productId))) {
            throw new ValidationException('productId is required');
        }
    }
}
```

---

**`[RequestNormalizer / InputMapper]` — `CalculatePriceNormalizer`**
Маппит сырой массив из HTTP-запроса в `Command`. Живёт на границе Presentation → Application.

```php
final class CalculatePriceNormalizer
{
    public function normalize(array $body): CalculateAmazonPriceCommand
    {
        return new CalculateAmazonPriceCommand(
            productId: (string) ($body['product_id'] ?? ''),
        );
    }
}
```

---

**`[GatewayPort / ApiClientPort]` — `SupplierPricePort` / `ShippingCostPort`**
Порты в Application слое. Domain не знает об этих портах — они Application-уровня,
потому что оркестрацию выполняет Use Case, а не Domain Service.

```php
interface SupplierPricePort
{
    public function getPriceForProduct(ProductId $productId): Money;
}

interface ShippingCostPort
{
    public function calculate(ProductId $productId, string $marketplace): Money;
}
```

---

**`[RepositoryPort]` — `PricingRulesRepositoryPort`**
Порт для получения правил ценообразования (коэффициент + наценка) по маркетплейсу.

```php
interface PricingRulesRepositoryPort
{
    public function findByMarketplace(string $marketplace): MarketplacePricingPolicy;
}
```

---

**Use Case — финальная сборка**

```php
final class CalculateAmazonPriceUseCase
{
    public function __construct(
        private SupplierPricePort            $supplierPort,
        private ShippingCostPort             $shippingPort,
        private PricingRulesRepositoryPort   $rulesRepo,
        private PriceCalculatorService       $calculator,
    ) {}

    public function execute(CalculateAmazonPriceCommand $command): CalculateAmazonPriceResult
    {
        $productId     = new ProductId($command->productId);
        $supplierPrice = $this->supplierPort->getPriceForProduct($productId);
        $shippingCost  = $this->shippingPort->calculate($productId, 'amazon');
        $policy        = $this->rulesRepo->findByMarketplace('amazon');  // → AmazonPricingPolicy

        $finalPrice = $this->calculator->calculate(
            $supplierPrice,
            $policy->coefficient(),
            $policy->markup(),
            $shippingCost,
        );

        return new CalculateAmazonPriceResult($finalPrice);
    }
}
```

---

### Infrastructure

**`[Controller]` — `CalculatePriceController`**
Тонкий Mezzio handler. Делегирует нормализацию → валидацию → use case.

```php
final class CalculatePriceController implements RequestHandlerInterface
{
    public function __construct(
        private CalculatePriceNormalizer      $normalizer,
        private CalculatePriceValidator       $validator,
        private CalculateAmazonPriceUseCase   $useCase,
    ) {}

    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        $command = $this->normalizer->normalize((array) $request->getParsedBody());
        $this->validator->validate($command);

        $result = $this->useCase->execute($command);

        return new JsonResponse(['price_cents' => $result->price->amount]);
    }
}
```

---

**`[ApiClientAdapter]` — `SupplierApiClientAdapter`**
Реализует `SupplierPricePort`. Вызывает OpenAPI-сервис поставщика.

```php
final class SupplierApiClientAdapter implements SupplierPricePort
{
    public function __construct(
        private readonly HttpClientInterface $httpClient,
        private readonly SupplierResponseTranslator $translator,
        private readonly string $baseUrl,
    ) {}

    public function getPriceForProduct(ProductId $productId): Money
    {
        try {
            $response = $this->httpClient->get("{$this->baseUrl}/prices/{$productId->value()}");
            $data     = json_decode($response->getBody()->getContents(), true);

            return $this->translator->toMoney($data);  // ACL Translator
        } catch (\Throwable $e) {
            throw new ExternalSystemException('Supplier API error', previous: $e);
        }
    }
}
```

---

**`[ACL Translator]` — `SupplierResponseTranslator`**
Переводит внешний контракт (External DTO) в доменный объект `Money`.
Изолирует домен от изменений API поставщика.

```php
final class SupplierResponseTranslator
{
    public function toMoney(array $data): Money
    {
        return Money::usd((int) $data['amount_cents']);
    }
}
```

---

**`[External DTO]` — `SupplierPriceResponse`**
Представляет сырой ответ от внешнего API. Живёт только в Infrastructure.

```php
final class SupplierPriceResponse
{
    public function __construct(
        public readonly int    $amount_cents,
        public readonly string $currency,
        public readonly string $product_id,
    ) {}
}
```

---

**`[CacheAdapter]` — `SupplierPriceCacheAdapter`**
Декоратор над `SupplierApiClientAdapter`. Domain и Application не знают о кэше.

```php
final class SupplierPriceCacheAdapter implements SupplierPricePort
{
    public function __construct(
        private SupplierPricePort  $inner,   // SupplierApiClientAdapter
        private CacheInterface     $cache,
        private int                $ttl = 30,
    ) {}

    public function getPriceForProduct(ProductId $productId): Money
    {
        $key = "supplier_price_{$productId->value()}";

        return $this->cache->get($key, function () use ($productId) {
            $price = $this->inner->getPriceForProduct($productId);
            return $price;
        }, $this->ttl);
    }
}
```

---

### Persistence

**`[RepositoryFacade]` — `PricingRulesRepositoryFacade`**
Реализует `PricingRulesRepositoryPort` через `rollun-datastore`.
Скрывает детали DataStoreInterface за доменным интерфейсом.

```php
final class PricingRulesRepositoryFacade implements PricingRulesRepositoryPort
{
    public function __construct(
        private readonly DataStoreInterface  $dataStore,
        private readonly PricingRulesMapper  $mapper,
    ) {}

    public function findByMarketplace(string $marketplace): MarketplacePricingPolicy
    {
        $record = $this->dataStore->read($marketplace);

        if ($record === null) {
            throw new ResourceNotFoundException("Pricing rules not found for: {$marketplace}");
        }

        return $this->mapper->toPolicy($record, $marketplace);
    }
}
```

---

**`[Decomposer / Mapper]` — `PricingRulesMapper`**
Маппит сырую запись из DataStore в доменный Policy-объект и обратно.

```php
final class PricingRulesMapper
{
    public function toPolicy(array $record, string $marketplace): MarketplacePricingPolicy
    {
        return match ($marketplace) {
            'amazon' => new AmazonPricingPolicy(
                new PricingCoefficient((float) $record['coefficient']),
                new Percentage((float) $record['markup_percent']),
            ),
            'ebay' => new EbayPricingPolicy(
                new PricingCoefficient((float) $record['coefficient']),
                new Percentage((float) $record['markup_percent']),
            ),
            default => throw new DomainException("Unknown marketplace: {$marketplace}"),
        };
    }
}
```

---

## Сводная таблица паттернов

| Класс | Паттерн | Слой |
|---|---|---|
| `Money`, `ProductId`, `PricingCoefficient`, `Percentage` | Value Object | Domain |
| `PriceCalculatorService` | Domain Service | Domain |
| `AmazonPricingPolicy`, `EbayPricingPolicy` | Policy | Domain |
| `MinimumPriceGuard` | Guard | Domain |
| `PriceCalculated` | Domain Event | Domain |
| `InvariantViolationException` | InvariantViolationException | Domain |
| `CalculateAmazonPriceUseCase`, `CalculateEbayPriceUseCase` | UseCase / Interactor | Application |
| `CalculateAmazonPriceCommand`, `CalculateEbayPriceCommand` | Command DTO | Application |
| `CalculateAmazonPriceResult`, `CalculateEbayPriceResult` | ResponseDTO | Application |
| `CalculatePriceValidator` | InputValidator | Application |
| `CalculatePriceNormalizer` | RequestNormalizer / InputMapper | Application |
| `SupplierPricePort`, `ShippingCostPort` | GatewayPort / ApiClientPort | Application |
| `PricingRulesRepositoryPort` | RepositoryPort | Application |
| `ValidationException`, `ResourceNotFoundException` | Exceptions | Application |
| `CalculatePriceController` | Controller | Infrastructure |
| `SupplierApiClientAdapter`, `ShippingApiClientAdapter` | ApiClientAdapter | Infrastructure |
| `SupplierPriceCacheAdapter` | CacheAdapter | Infrastructure |
| `SupplierResponseTranslator`, `ShippingResponseTranslator` | ACL Translator | Infrastructure |
| `HttpErrorTranslator` | ErrorTranslator | Infrastructure |
| `SupplierPriceResponse`, `ShippingMethodsResponse` | External DTO | Infrastructure |
| `ExternalSystemException` | ExternalSystemException | Infrastructure |
| `PricingRulesRepositoryFacade` | RepositoryFacade | Persistence |
| `PricingRulesMapper` | Decomposer / Mapper | Persistence |

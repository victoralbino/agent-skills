---
title: Domain Building Blocks
scope: Data Objects, Actions, Models, Enums, QueryBuilders, Collections, Events, Cross-Domain Communication, Tests
---

# Domain Building Blocks — Detailed Reference

## Table of Contents

1. [Data Objects (DTOs)](#data-objects-dtos)
2. [Actions](#actions)
3. [Models](#models)
4. [Custom Query Builders](#custom-query-builders)
5. [Custom Collections](#custom-collections)
6. [Enums with Behavior](#enums-with-behavior)
7. [Transitions as Actions](#transitions-as-actions)
8. [Events](#events)
9. [Cross-Domain Communication](#cross-domain-communication)
10. [Managing Domains](#managing-domains)
11. [Testing Domains](#testing-domains)

---

## Data Objects (DTOs)

Data Objects wrap unstructured data into typed classes. Use them **only when justified** —
not for every action.

### When to Use DTOs

| Situation | Use DTO? |
|---|---|
| Action receives data from multiple sources (request + config + computed) | Yes |
| Action is called from Controller, Job, Console, and tests | Yes |
| Action receives 5+ parameters | Yes |
| Controller passes 2-3 fields to an Action | No — use parameters |
| Action receives a single Model | No — pass the model |

### Implementation with Pure PHP

```php
namespace App\Domain\Invoice;

use Carbon\Carbon;

class InvoiceData
{
    public function __construct(
        public readonly string $number,
        public readonly string $customer_email,
        public readonly Carbon $due_date,
        /** @var InvoiceLineData[] */
        public readonly array $lines = [],
    ) {}

    /**
     * Creates from a validated request.
     * Pragmatic: mixes app layer knowledge into the domain,
     * but avoids a separate factory class in most cases.
     */
    public static function fromRequest(StoreInvoiceRequest $request): self
    {
        $validated = $request->validated();

        return new self(
            number: $validated['number'],
            customer_email: $validated['customer_email'],
            due_date: Carbon::parse($validated['due_date']),
            lines: array_map(
                fn (array $line) => new InvoiceLineData(...$line),
                $validated['lines'] ?? [],
            ),
        );
    }
}
```

### Simple Data Object (no nested objects)

When all fields are primitive types, the spread works directly:

```php
namespace App\Domain\Customer;

class CustomerData
{
    public function __construct(
        public readonly string $name,
        public readonly string $email,
    ) {}

    public static function fromRequest(StoreCustomerRequest $request): self
    {
        return new self(...$request->validated());
    }
}
```

### Without DTO — Direct Parameters

For simple actions, skip the DTO entirely:

```php
class MarkInvoiceAsPaidAction
{
    public function execute(Invoice $invoice): Invoice
    {
        // No DTO needed — single model parameter
    }
}

class UpdateCustomerEmailAction
{
    public function execute(Customer $customer, string $email): Customer
    {
        // No DTO needed — model + one field
    }
}
```

### Using in the Controller

```php
class InvoiceController
{
    public function store(StoreInvoiceRequest $request)
    {
        $data = InvoiceData::fromRequest($request);
        $invoice = app(CreateInvoiceAction::class)->execute($data);

        return new InvoiceResource($invoice);
    }
}
```

### Purist Alternative: Factory in the App Layer

If you want to keep the domain 100% clean from HTTP knowledge:

```php
// app/Http/Factories/InvoiceDataFactory.php
class InvoiceDataFactory
{
    public static function fromRequest(StoreInvoiceRequest $request): InvoiceData
    {
        return new InvoiceData(
            number: $request->validated('number'),
            customer_email: $request->validated('customer_email'),
            due_date: Carbon::parse($request->validated('due_date')),
            lines: array_map(
                fn (array $line) => new InvoiceLineData(...$line),
                $request->validated('lines', []),
            ),
        );
    }
}
```

---

## Actions

Actions are simple classes representing a single user story or business operation. They are the
primary place for business logic in the domain.

### Conventions

- **Suffix**: always `*Action` (e.g.: `CreateInvoiceAction`)
- **One public method**: `execute()` (not `handle` to avoid Laravel auto-injection, not `__invoke` for cleaner composition)
- **Dependencies**: via constructor (container DI)
- **Input**: via `execute()` parameters (context-specific data)
- **Location**: `app/Domain/{DomainName}/`

### Basic Action

```php
namespace App\Domain\Invoice;

class CreateInvoiceAction
{
    public function __construct(
        private CreateInvoiceLineAction $createInvoiceLineAction,
    ) {}

    public function execute(InvoiceData $data): Invoice
    {
        $invoice = Invoice::create([
            'number' => $data->number,
            'customer_email' => $data->customer_email,
            'due_date' => $data->due_date,
            'status' => InvoiceStatus::Pending,
        ]);

        foreach ($data->lines as $lineData) {
            $this->createInvoiceLineAction->execute($invoice, $lineData);
        }

        return $invoice->refresh();
    }
}
```

### Action Composition

Actions compose via constructor injection. Keep the dependency chain shallow:

```php
namespace App\Domain\Invoice;

class CreateInvoiceLineAction
{
    public function __construct(
        private VatCalculator $vatCalculator,
    ) {}

    public function execute(Invoice $invoice, InvoiceLineData $data): InvoiceLine
    {
        [$priceIncVat, $priceExVat] = $data->vat_included
            ? $this->vatCalculator->fromIncluded($data->price, $data->vat_percentage)
            : $this->vatCalculator->fromExcluded($data->price, $data->vat_percentage);

        return $invoice->invoiceLines()->create([
            'description' => $data->description,
            'item_price' => $data->price,
            'amount' => $data->amount,
            'total_price' => $data->amount * $priceIncVat,
            'total_price_excluding_vat' => $data->amount * $priceExVat,
        ]);
    }
}
```

### Dispatching as a Job

Create a normal job that calls the action:

```php
// app/Jobs/SendInvoiceMailJob.php
namespace App\Jobs;

use App\Domain\Invoice\Invoice;
use App\Domain\Invoice\SendInvoiceMailAction;

class SendInvoiceMailJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public Invoice $invoice) {}

    public function handle(SendInvoiceMailAction $action): void
    {
        $action->execute($this->invoice);
    }
}

// Dispatching:
SendInvoiceMailJob::dispatch($invoice);
```

---

## Models

Models should be lean — they represent database data and provide relationships, casts, and scopes.
Business logic goes elsewhere.

### What Belongs in the Model

```php
namespace App\Domain\Invoice;

use App\Domain\Customer\Customer;
use App\Domain\Payment\Payment;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Invoice extends Model
{
    protected $fillable = [
        'number', 'customer_email', 'due_date',
        'status', 'total_price', 'paid_at',
    ];

    protected $casts = [
        'due_date' => 'date',
        'paid_at' => 'datetime',
        'total_price' => 'integer',
        'status' => InvoiceStatus::class,
    ];

    // Relationships
    public function invoiceLines(): HasMany
    {
        return $this->hasMany(InvoiceLine::class);
    }

    public function customer(): BelongsTo
    {
        return $this->belongsTo(Customer::class);
    }

    public function payments(): HasMany
    {
        return $this->hasMany(Payment::class);
    }

    // Custom query builder
    public function newEloquentBuilder($query): InvoiceQueryBuilder
    {
        return new InvoiceQueryBuilder($query);
    }
}
```

### What Does NOT Belong in the Model

- Price calculations -> Actions
- PDF generation -> Actions
- Email sending -> Actions
- Complex conditional logic -> Enums with methods
- Query filtering beyond simple scopes -> Custom QueryBuilders

---

## Custom Query Builders

Extract query scopes into dedicated builder classes. This is a standard Eloquent mechanism:

```php
namespace App\Domain\Invoice;

use App\Domain\Customer\Customer;
use Illuminate\Database\Eloquent\Builder;

class InvoiceQueryBuilder extends Builder
{
    public function wherePaid(): self
    {
        return $this->where('status', InvoiceStatus::Paid);
    }

    public function whereOverdue(): self
    {
        return $this->where('due_date', '<', now())
            ->whereNot('status', InvoiceStatus::Paid);
    }

    public function whereForCustomer(Customer $customer): self
    {
        return $this->where('customer_id', $customer->id);
    }
}
```

Register in the model:

```php
public function newEloquentBuilder($query): InvoiceQueryBuilder
{
    return new InvoiceQueryBuilder($query);
}
```

Usage: `Invoice::query()->wherePaid()->whereOverdue()->get()`

---

## Custom Collections

When collection operations are repeated, extract them into a custom class:

```php
namespace App\Domain\Invoice;

use Illuminate\Database\Eloquent\Collection;

class InvoiceLineCollection extends Collection
{
    public function creditLines(): self
    {
        return $this->filter(fn (InvoiceLine $line) => $line->isCreditLine());
    }

    public function totalPrice(): int
    {
        return $this->sum('total_price');
    }

    public function totalPriceExcludingVat(): int
    {
        return $this->sum('total_price_excluding_vat');
    }
}
```

Register in the model:

```php
class InvoiceLine extends Model
{
    public function newCollection(array $models = []): InvoiceLineCollection
    {
        return new InvoiceLineCollection($models);
    }

    public function isCreditLine(): bool
    {
        return $this->total_price < 0;
    }
}
```

Usage:

```php
$invoice->invoiceLines->creditLines()->totalPrice();
```

---

## Enums with Behavior

Instead of the state pattern with abstract and concrete classes (which requires packages or lots of
boilerplate), use PHP 8.3+ enums with methods. Covers 80% of cases with zero overhead:

### Complete Enum

```php
namespace App\Domain\Invoice;

enum InvoiceStatus: string
{
    case Pending = 'pending';
    case Paid = 'paid';
    case Cancelled = 'cancelled';

    public function color(): string
    {
        return match ($this) {
            self::Pending => 'orange',
            self::Paid => 'green',
            self::Cancelled => 'red',
        };
    }

    public function label(): string
    {
        return match ($this) {
            self::Pending => 'Pending',
            self::Paid => 'Paid',
            self::Cancelled => 'Cancelled',
        };
    }

    public function canBeDeleted(): bool
    {
        return match ($this) {
            self::Pending, self::Cancelled => true,
            self::Paid => false,
        };
    }

    public function mustBePaid(): bool
    {
        return match ($this) {
            self::Pending => true,
            self::Paid, self::Cancelled => false,
        };
    }

    public function canTransitionTo(self $new): bool
    {
        return match ($this) {
            self::Pending => in_array($new, [self::Paid, self::Cancelled]),
            self::Paid => false,
            self::Cancelled => false,
        };
    }

    /**
     * Returns all statuses this one can transition to.
     * Useful for APIs that need to inform the frontend about available actions.
     */
    public function allowedTransitions(): array
    {
        return collect(self::cases())
            ->filter(fn (self $status) => $this->canTransitionTo($status))
            ->values()
            ->all();
    }
}
```

### In the Model

```php
class Invoice extends Model
{
    protected $casts = [
        'status' => InvoiceStatus::class,
    ];
}

// Usage — no if/else anywhere
$invoice->status->color();              // "orange"
$invoice->status->canBeDeleted();       // true
$invoice->status->allowedTransitions(); // [InvoiceStatus::Paid, InvoiceStatus::Cancelled]
```

### Enum for Types (no transitions)

Not every enum needs transitions. Types that never change also benefit:

```php
namespace App\Domain\Invoice;

enum InvoiceType: string
{
    case Debit = 'debit';
    case Credit = 'credit';

    public function mustBePaid(): bool
    {
        return match ($this) {
            self::Debit => true,
            self::Credit => false,
        };
    }
}
```

### When to Migrate to Full State Pattern

If an enum accumulates 5+ methods with complex logic that needs to access model data (not
just return static values), consider migrating to state classes. Warning signs:

- Enum methods that need to receive the model as a parameter
- Nested conditional logic inside `match` blocks
- Enum method tests becoming complex

---

## Transitions as Actions

Instead of separate `Transition` classes, use normal Actions. A transition is just another
business operation:

```php
namespace App\Domain\Invoice;

class MarkInvoiceAsPaidAction
{
    public function execute(Invoice $invoice): Invoice
    {
        if (! $invoice->status->canTransitionTo(InvoiceStatus::Paid)) {
            throw new InvalidTransitionException(
                from: $invoice->status,
                to: InvoiceStatus::Paid,
            );
        }

        $invoice->update([
            'status' => InvoiceStatus::Paid,
            'paid_at' => now(),
        ]);

        return $invoice->refresh();
    }
}

class CancelInvoiceAction
{
    public function execute(Invoice $invoice, string $reason = ''): Invoice
    {
        if (! $invoice->status->canTransitionTo(InvoiceStatus::Cancelled)) {
            throw new InvalidTransitionException(
                from: $invoice->status,
                to: InvoiceStatus::Cancelled,
            );
        }

        $invoice->update([
            'status' => InvoiceStatus::Cancelled,
            'cancelled_at' => now(),
            'cancellation_reason' => $reason,
        ]);

        return $invoice->refresh();
    }
}
```

### Transition Exception

```php
namespace App\Domain\Invoice;

class InvalidTransitionException extends \DomainException
{
    public function __construct(
        public readonly InvoiceStatus $from,
        public readonly InvoiceStatus $to,
    ) {
        parent::__construct(
            "Transition from '{$from->value}' to '{$to->value}' is not allowed."
        );
    }
}
```

---

## Events

Use events when multiple listeners need to react to the same action, or when the reaction
is a side effect that shouldn't couple domains.

### Custom Event

```php
namespace App\Domain\Invoice;

class InvoiceCreatedEvent
{
    public function __construct(
        public readonly Invoice $invoice,
    ) {}
}
```

### Dispatching in the Action

```php
class CreateInvoiceAction
{
    public function execute(InvoiceData $data): Invoice
    {
        $invoice = Invoice::create([...]);

        event(new InvoiceCreatedEvent($invoice));

        return $invoice;
    }
}
```

### Listener in the App Layer

```php
// app/Listeners/SendInvoiceCreatedNotification.php
namespace App\Listeners;

use App\Domain\Invoice\InvoiceCreatedEvent;

class SendInvoiceCreatedNotification
{
    public function handle(InvoiceCreatedEvent $event): void
    {
        $event->invoice->customer->notify(
            new InvoiceCreatedNotification($event->invoice)
        );
    }
}
```

Register in `EventServiceProvider` as usual.

---

## Cross-Domain Communication

Domains communicate pragmatically based on the **nature of the interaction**.

### Rules

| Situation | Approach |
|---|---|
| Read a model/enum from another domain | Direct import |
| Validate a precondition from another domain | Direct call |
| Orchestrate actions atomically (transaction) | Direct call |
| Side effect (notify, log, sync) | Event |
| Async reaction (can fail without affecting the flow) | Event |
| Multiple domains react to the same fact | Event |

### Direct Import (default)

The most common case: one domain uses models or data objects from another directly.

```php
namespace App\Domain\Invoice;

use App\Domain\Customer\Customer;

class CreateInvoiceAction
{
    public function execute(InvoiceData $data, Customer $customer): Invoice
    {
        if (! $customer->isActive()) {
            throw new \App\Domain\Customer\InactiveCustomerException();
        }

        return Invoice::create([
            'customer_id' => $customer->id,
            'number' => $data->number,
            'due_date' => $data->due_date,
        ]);
    }
}
```

### Cross-Domain Relationships

Models can have relationships with models from other domains normally:

```php
namespace App\Domain\Invoice;

use App\Domain\Customer\Customer;

class Invoice extends Model
{
    public function customer(): BelongsTo
    {
        return $this->belongsTo(Customer::class);
    }
}
```

### Orchestration (needs the result, must be atomic)

When multiple domains must succeed together, the **app layer** orchestrates:

```php
// app/Http/Controllers/Api/CheckoutController.php
class CheckoutController
{
    public function store(CheckoutRequest $request)
    {
        return DB::transaction(function () use ($request) {
            $invoice = app(CreateInvoiceAction::class)->execute($invoiceData);
            $payment = app(ProcessPaymentAction::class)->execute($invoice, $paymentData);

            return new CheckoutResource($invoice, $payment);
        });
    }
}
```

### Events (side effects)

```php
// Domain A dispatches
class CreateInvoiceAction
{
    public function execute(InvoiceData $data): Invoice
    {
        $invoice = Invoice::create([...]);
        event(new InvoiceCreatedEvent($invoice));

        return $invoice;
    }
}

// Domain B reacts via listener — no direct coupling
class ReconcilePaymentListener
{
    public function handle(InvoiceCreatedEvent $event): void
    {
        app(ReconcilePaymentAction::class)->execute($event->invoice);
    }
}
```

### What's Prohibited

- **Circular dependencies** — If A calls B and B calls A, extract to `Domain/Shared/` or use events
- **Side effects as direct calls** — If the caller doesn't need the result, use an event

### Domain/Shared

Use for genuinely cross-cutting concerns:

```
app/Domain/Shared/
├── Currency.php
├── LogActivityAction.php
└── ActivityLog.php
```

Keep `Shared/` small. If it grows too large, extract into its own domain.

---

## Managing Domains

### Identifying Domains

- Listen to how the business describes the project: "billing", "customer management", "bookings"
- Each high-level concept becomes a domain
- Domains can and should change over time — split when they grow too large
- Use a `Shared` domain for cross-cutting concerns — keep it small

### When to Add Subfolders

| Domain size | Structure |
|---|---|
| Up to ~15 files | Flat (all files at domain root) |
| 15+ files | Subfolders by type (Actions/, Models/, etc.) |
| 30+ files | Consider splitting into two domains |

### Real-World Examples

**Flare (error tracker)**: `Error/`, `Notification/`, `Project/`, `Subscription/`, `Team/`, `Shared/`

**Mailcoach (email platform)**: `Audience/`, `Automation/`, `Campaign/`, `TransactionalMail/`, `Shared/`

### When to Start

Don't force domains from the beginning. It's normal to start with the standard Laravel structure
and refactor into domains when directories become too large. IDEs like PhpStorm make it easy to
move namespaces quickly.

---

## Testing Domains

### Test Factories (custom, immutable)

```php
namespace Tests\Factories;

use App\Domain\Invoice\Invoice;
use App\Domain\Invoice\InvoiceStatus;
use Illuminate\Support\Str;

class InvoiceFactory
{
    private InvoiceStatus $status = InvoiceStatus::Pending;
    private ?string $expiresAt = null;
    private ?PaymentFactory $paymentFactory = null;

    public static function new(): self
    {
        return new self();
    }

    public function paid(PaymentFactory $paymentFactory = null): self
    {
        $clone = clone $this;
        $clone->status = InvoiceStatus::Paid;
        $clone->paymentFactory = $paymentFactory ?? PaymentFactory::new();
        return $clone;
    }

    public function cancelled(): self
    {
        $clone = clone $this;
        $clone->status = InvoiceStatus::Cancelled;
        return $clone;
    }

    public function expiresAt(string $date): self
    {
        $clone = clone $this;
        $clone->expiresAt = $date;
        return $clone;
    }

    public function create(array $extra = []): Invoice
    {
        $invoice = Invoice::create(array_merge([
            'number' => 'INV-' . Str::random(6),
            'status' => $this->status,
            'due_date' => $this->expiresAt ?? now()->addMonth()->format('Y-m-d'),
            'customer_email' => 'test@example.com',
            'total_price' => 0,
        ], $extra));

        if ($this->paymentFactory) {
            $this->paymentFactory->forInvoice($invoice)->create();
        }

        return $invoice;
    }
}
```

**Why immutable?** To reuse the same base factory without side effects:

```php
$base = InvoiceFactory::new()->expiresAt('2024-06-01');

$invoiceA = $base->paid()->create();    // paid, expires in June
$invoiceB = $base->create();            // pending, expires in June (not contaminated)
```

### Testing Actions (setup -> execute -> assert)

```php
it('creates an invoice with correct total', function () {
    $data = InvoiceDataFactory::new()
        ->addLine(InvoiceLineDataFactory::new()->withPrice(10_00)->withAmount(2))
        ->addLine(InvoiceLineDataFactory::new()->withPrice(5_00)->withAmount(1))
        ->create();

    $invoice = app(CreateInvoiceAction::class)->execute($data);

    expect($invoice)->toBeInstanceOf(Invoice::class);
    expect($invoice->total_price)->toBe(25_00);
    expect($invoice->invoiceLines)->toHaveCount(2);
    $this->assertDatabaseHas('invoices', ['id' => $invoice->id]);
});
```

### Mocking Composed Actions

```php
namespace Tests\Mocks;

use App\Domain\Pdf\GeneratePdfAction;

class MockGeneratePdfAction extends GeneratePdfAction
{
    public static function setUp(): void
    {
        app()->singleton(GeneratePdfAction::class, fn () => new self());
    }

    public function execute($toPdf): void
    {
        return; // Skips PDF generation in tests
    }
}

// In test setUp:
MockGeneratePdfAction::setUp();
```

### Testing Enums

```php
it('pending invoices can transition to paid', function () {
    expect(InvoiceStatus::Pending->canTransitionTo(InvoiceStatus::Paid))
        ->toBeTrue();
});

it('paid invoices cannot transition to anything', function () {
    expect(InvoiceStatus::Paid->allowedTransitions())
        ->toBeEmpty();
});

it('pending invoice color is orange', function () {
    expect(InvoiceStatus::Pending->color())
        ->toBe('orange');
});
```

### Testing Transitions (via Actions)

```php
it('marks an invoice as paid', function () {
    $invoice = InvoiceFactory::new()->create();

    $result = app(MarkInvoiceAsPaidAction::class)->execute($invoice);

    expect($result->status)->toBe(InvoiceStatus::Paid);
    expect($result->paid_at)->not->toBeNull();
});

it('cannot mark a cancelled invoice as paid', function () {
    $invoice = InvoiceFactory::new()->cancelled()->create();

    app(MarkInvoiceAsPaidAction::class)->execute($invoice);
})->throws(InvalidTransitionException::class);
```

### Testing QueryBuilders

```php
it('filters overdue invoices', function () {
    $overdue = InvoiceFactory::new()->create(['due_date' => now()->subWeek()]);
    $future = InvoiceFactory::new()->create(['due_date' => now()->addWeek()]);

    $results = Invoice::query()->whereOverdue()->pluck('id');

    expect($results)->toContain($overdue->id);
    expect($results)->not->toContain($future->id);
});
```

### Testing Custom Collections

```php
it('returns only credit lines', function () {
    $credit = InvoiceLineFactory::new()->withPrice(-100)->create();
    $debit = InvoiceLineFactory::new()->withPrice(100)->create();

    $collection = new InvoiceLineCollection([$credit, $debit]);

    expect($collection->creditLines())->toHaveCount(1);
    expect($collection->creditLines()->first()->is($credit))->toBeTrue();
});
```

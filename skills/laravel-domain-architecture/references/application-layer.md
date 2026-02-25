---
title: Application Layer
scope: Controllers, ViewModels (Blade and API), HTTP Queries, Jobs, Requests, Resources, API Versioning
---

# Application Layer — Detailed Reference

## Table of Contents

1. [Application Structure](#application-structure)
2. [Controllers](#controllers)
3. [ViewModels — Blade](#viewmodels--blade)
4. [ViewModels — API](#viewmodels--api)
5. [HTTP Query Builders](#http-query-builders)
6. [Jobs](#jobs)
7. [Requests and Validation](#requests-and-validation)
8. [Resources](#resources)
9. [API Versioning](#api-versioning)

---

## Application Structure

Each "application" is a separate entry point for the domain: Admin panel, API, Customer portal,
Console. Within each application, code is grouped by **business module**, not by technical type.

```
src/App/Api/
├── Invoices/                       ← One module per business concept
│   ├── Controllers/
│   │   ├── InvoicesController.php
│   │   └── InvoiceStatusController.php
│   ├── Requests/
│   │   └── StoreInvoiceRequest.php
│   ├── Resources/
│   │   ├── InvoiceResource.php
│   │   └── InvoiceLineResource.php
│   ├── ViewModels/
│   │   └── InvoiceDetailViewModel.php
│   ├── Queries/
│   │   └── InvoiceIndexQuery.php
│   └── Middleware/
│       └── EnsureValidInvoiceSettingsMiddleware.php
├── Customers/
│   └── ...
└── Support/
    ├── Controllers/
    │   └── BaseController.php
    └── Middleware/
        └── EnsureAuthenticated.php
```

Grouping by module avoids the problem of having a `Controllers/` directory with 100+ files.
When working on billing, everything you need is in one folder.

---

## Controllers

Controllers should be thin — they receive input, delegate to domain actions, and return output.
They belong to the app layer because they know about HTTP, but should not contain business logic.

### Thin Controller with Action

```php
namespace App\Api\Invoices\Controllers;

use Domain\Invoices\Actions\CreateInvoiceAction;
use Domain\Invoices\Data\InvoiceData;
use App\Api\Invoices\Requests\StoreInvoiceRequest;
use App\Api\Invoices\Resources\InvoiceResource;

class InvoicesController
{
    public function index(InvoiceIndexQuery $query)
    {
        return InvoiceResource::collection(
            $query->build()->paginate()
        );
    }

    public function store(StoreInvoiceRequest $request)
    {
        $data = InvoiceData::fromRequest($request);
        $invoice = app(CreateInvoiceAction::class)->execute($data);

        return new InvoiceResource($invoice);
    }

    public function show(Invoice $invoice)
    {
        $viewModel = new InvoiceDetailViewModel(
            invoice: $invoice->load(['invoiceLines', 'payments']),
            user: auth()->user(),
        );

        return response()->json($viewModel->toResponse());
    }
}
```

### Single Action Controller

For operations that are not CRUD, use invokable controllers:

```php
class MarkInvoiceAsPaidController
{
    public function __invoke(Invoice $invoice, MarkInvoiceAsPaidAction $action)
    {
        $invoice = $action->execute($invoice);

        return new InvoiceResource($invoice);
    }
}
```

---

## ViewModels — Blade

For applications with Blade, ViewModels implement `Arrayable` and group everything the view needs.

```php
namespace App\Admin\Invoices\ViewModels;

use Illuminate\Contracts\Support\Arrayable;

class InvoiceFormViewModel implements Arrayable
{
    public function __construct(
        private User $user,
        private ?Invoice $invoice = null,
    ) {}

    public function toArray(): array
    {
        return [
            'invoice' => $this->invoice ?? new Invoice(),
            'categories' => Category::allowedForUser($this->user)->get(),
            'statuses' => InvoiceStatus::cases(),
            'customers' => Customer::active()->get(),
        ];
    }
}
```

In the controller:

```php
public function create()
{
    return view('invoices.form', new InvoiceFormViewModel(current_user()));
}

public function edit(Invoice $invoice)
{
    return view('invoices.form', new InvoiceFormViewModel(current_user(), $invoice));
}
```

In Blade, use the variables directly (`$invoice`, `$categories`, `$statuses`).

---

## ViewModels — API

In API, ViewModels have a different role: **they compose JSON responses when the endpoint returns data
from multiple sources** (model + permissions + metadata + calculated data).

### When to Use ViewModel in API

| Situation | Use |
|-----------|-----|
| Response is 1 model or list | **Resource** directly |
| Response combines model + permissions + metadata | **ViewModel** |
| Same composition reused across multiple endpoints | **ViewModel** |

### Implementation

```php
namespace App\Api\Invoices\ViewModels;

use App\Api\Invoices\Resources\InvoiceResource;
use App\Api\Invoices\Resources\InvoiceLineResource;
use Domain\Invoices\Enums\InvoiceStatus;
use Domain\Invoices\Models\Invoice;
use App\Models\User;

class InvoiceDetailViewModel
{
    public function __construct(
        private Invoice $invoice,
        private User $user,
    ) {}

    public function toResponse(): array
    {
        return [
            'invoice' => InvoiceResource::make($this->invoice),
            'lines' => InvoiceLineResource::collection($this->invoice->invoiceLines),
            'permissions' => $this->permissions(),
            'payment_summary' => $this->paymentSummary(),
            'available_transitions' => $this->availableTransitions(),
        ];
    }

    private function permissions(): array
    {
        return [
            'can_edit' => $this->user->can('update', $this->invoice),
            'can_delete' => $this->user->can('delete', $this->invoice),
            'can_mark_as_paid' => $this->invoice->status->canTransitionTo(InvoiceStatus::Paid),
        ];
    }

    private function paymentSummary(): array
    {
        $totalPaid = $this->invoice->payments->sum('amount');

        return [
            'total_paid' => $totalPaid,
            'remaining' => $this->invoice->total_price - $totalPaid,
            'is_fully_paid' => $totalPaid >= $this->invoice->total_price,
        ];
    }

    private function availableTransitions(): array
    {
        return collect($this->invoice->status->allowedTransitions())
            ->map(fn (InvoiceStatus $status) => [
                'value' => $status->value,
                'label' => $status->label(),
            ])
            ->values()
            ->all();
    }
}
```

### In the Controller

```php
public function show(Invoice $invoice)
{
    $viewModel = new InvoiceDetailViewModel(
        invoice: $invoice->load(['invoiceLines', 'payments']),
        user: auth()->user(),
    );

    return response()->json($viewModel->toResponse());
}
```

### Alternative Naming

The name "ViewModel" can be confusing in an API context. Valid alternatives:

- `InvoiceDetailResponse`
- `InvoiceDetailComposer`
- `InvoiceDetailPresenter`

The concept is the same: a class that joins data from different sources into a cohesive structure.

---

## HTTP Query Builders

For listings with filters, sorting, and pagination, extract the query logic into a dedicated class.
No packages — use Eloquent's native `when()`:

### Dedicated Query Class

```php
namespace App\Api\Invoices\Queries;

use Domain\Invoices\Models\Invoice;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Http\Request;

class InvoiceIndexQuery
{
    public function __construct(
        private Request $request,
    ) {}

    public function build(): Builder
    {
        return Invoice::query()
            ->with(['customer', 'invoiceLines'])
            ->when(
                $this->request->get('status'),
                fn (Builder $q, string $status) => $q->where('status', $status),
            )
            ->when(
                $this->request->get('customer_id'),
                fn (Builder $q, int $id) => $q->where('customer_id', $id),
            )
            ->when(
                $this->request->get('search'),
                fn (Builder $q, string $search) => $q->where('number', 'like', "%{$search}%"),
            )
            ->when(
                $this->request->get('overdue'),
                fn (Builder $q) => $q->whereOverdue(), // uses the domain's custom QueryBuilder
            )
            ->orderBy(
                $this->request->get('sort', 'due_date'),
                $this->request->get('direction', 'desc'),
            );
    }
}
```

### Using in the Controller

```php
class InvoicesController
{
    public function index(InvoiceIndexQuery $query)
    {
        return InvoiceResource::collection(
            $query->build()->paginate()
        );
    }
}
```

The `Request` is automatically injected by the container. The controller stays clean.

### Reuse with Refinement

The same query can be refined by different controllers:

```php
class PaidInvoicesController
{
    public function index(InvoiceIndexQuery $query)
    {
        return InvoiceResource::collection(
            $query->build()
                ->wherePaid()  // method from the domain's InvoiceQueryBuilder
                ->paginate()
        );
    }
}
```

### With Joins for Complex Filters

```php
class InvoiceIndexQuery
{
    public function __construct(private Request $request) {}

    public function build(): Builder
    {
        return Invoice::query()
            ->join('customers', 'customers.id', '=', 'invoices.customer_id')
            ->select('invoices.*')
            ->when(
                $this->request->get('search'),
                fn (Builder $q, string $search) => $q->where(function (Builder $q) use ($search) {
                    $q->where('invoices.number', 'like', "%{$search}%")
                      ->orWhere('customers.name', 'like', "%{$search}%")
                      ->orWhere('customers.email', 'like', "%{$search}%");
                }),
            )
            ->orderBy(
                $this->request->get('sort', 'invoices.due_date'),
                $this->request->get('direction', 'desc'),
            );
    }
}
```

---

## Jobs

Jobs belong to the app layer because their responsibility is managing queue infrastructure
(retries, timeouts, middleware). Business logic stays in domain actions.

### Standard Job

```php
namespace App\Api\Invoices\Jobs;

use Domain\Invoices\Actions\SendInvoiceMailAction;
use Domain\Invoices\Models\Invoice;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class SendInvoiceMailJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $backoff = 60;

    public function __construct(
        public Invoice $invoice,
    ) {}

    public function handle(SendInvoiceMailAction $action): void
    {
        $action->execute($this->invoice);
    }
}
```

### Dispatching

```php
// In the action or controller:
SendInvoiceMailJob::dispatch($invoice);

// With delay:
SendInvoiceMailJob::dispatch($invoice)->delay(now()->addMinutes(5));
```

Use dedicated job classes only when you need specific configuration (`$tries`, `$timeout`,
`$backoff`, rate limiting, chaining). For the simple case of "run an action on the queue", a
generic 10-line job is enough.

---

## Requests and Validation

Use standard Laravel Form Requests in the app layer:

```php
namespace App\Api\Invoices\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StoreInvoiceRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', Invoice::class);
    }

    public function rules(): array
    {
        return [
            'number' => ['required', 'string', 'max:50', 'unique:invoices,number'],
            'customer_email' => ['required', 'email'],
            'due_date' => ['required', 'date', 'after:today'],
            'lines' => ['required', 'array', 'min:1'],
            'lines.*.description' => ['required', 'string', 'max:255'],
            'lines.*.amount' => ['required', 'integer', 'min:1'],
            'lines.*.price' => ['required', 'integer'],
            'lines.*.vat_percentage' => ['required', 'numeric', 'min:0', 'max:100'],
        ];
    }
}
```

The request validates and sanitizes. The domain Data Object receives already-validated data via
`InvoiceData::fromRequest($request)`.

---

## Resources

API Resources transform models/data for HTTP output. They live in the application module.

```php
namespace App\Api\Invoices\Resources;

use Illuminate\Http\Resources\Json\JsonResource;

class InvoiceResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'number' => $this->number,
            'status' => [
                'value' => $this->status->value,
                'label' => $this->status->label(),
                'color' => $this->status->color(),
            ],
            'total_price' => $this->total_price,
            'due_date' => $this->due_date->format('Y-m-d'),
            'paid_at' => $this->paid_at?->format('Y-m-d H:i:s'),
            'customer_email' => $this->customer_email,
            'lines' => InvoiceLineResource::collection(
                $this->whenLoaded('invoiceLines')
            ),
            'created_at' => $this->created_at->format('Y-m-d H:i:s'),
        ];
    }
}

class InvoiceLineResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'description' => $this->description,
            'amount' => $this->amount,
            'item_price' => $this->item_price,
            'total_price' => $this->total_price,
            'total_price_excluding_vat' => $this->total_price_excluding_vat,
        ];
    }
}
```

Resources map 1:1 with a model. ViewModels compose multiple resources for complex responses.

---

## API Versioning

The API is versioned **only at the application layer**. The domain layer is shared across all
versions — Actions, Models, and Enums don't change per version.

### Folder Structure

```
src/App/Api/
├── V1/
│   ├── Invoices/
│   │   ├── Controllers/
│   │   │   └── InvoicesController.php
│   │   ├── Requests/
│   │   │   └── StoreInvoiceRequest.php
│   │   ├── Resources/
│   │   │   └── InvoiceResource.php
│   │   ├── ViewModels/
│   │   │   └── InvoiceDetailViewModel.php
│   │   └── Queries/
│   │       └── InvoiceIndexQuery.php
│   └── Customers/
│       └── ...
├── V2/
│   └── Invoices/
│       ├── Controllers/
│       │   └── InvoicesController.php        ← New controller
│       ├── Requests/
│       │   └── StoreInvoiceRequest.php       ← Updated validation
│       └── Resources/
│           └── InvoiceResource.php           ← Resource with different fields
└── Support/
    ├── Controllers/
    │   └── BaseController.php
    └── Middleware/
        └── EnsureAuthenticated.php
```

### Routes per Version

Create separate route files for each version:

```php
// routes/api_v1.php
Route::prefix('v1')->group(function () {
    Route::apiResource('invoices', V1\Invoices\Controllers\InvoicesController::class);
    Route::apiResource('customers', V1\Customers\Controllers\CustomersController::class);
});

// routes/api_v2.php
Route::prefix('v2')->group(function () {
    Route::apiResource('invoices', V2\Invoices\Controllers\InvoicesController::class);
    // Customers didn't change — no need to replicate in V2
});
```

Register in `bootstrap/app.php` or `RouteServiceProvider`:

```php
// bootstrap/app.php (Laravel 11+)
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        api: __DIR__.'/../routes/api_v1.php',
        then: function () {
            Route::middleware('api')
                ->group(base_path('routes/api_v2.php'));
        },
    );
```

### Versioned Controller

The V2 controller uses the **same domain Actions**, but can have different mapping logic:

```php
namespace App\Api\V2\Invoices\Controllers;

use Domain\Invoices\Actions\CreateInvoiceAction;
use Domain\Invoices\Data\InvoiceData;
use App\Api\V2\Invoices\Requests\StoreInvoiceRequest;
use App\Api\V2\Invoices\Resources\InvoiceResource;

class InvoicesController
{
    public function store(StoreInvoiceRequest $request)
    {
        // V2 can have different validation and mapping
        $data = InvoiceData::fromRequest($request);
        $invoice = app(CreateInvoiceAction::class)->execute($data);

        // V2 Resource returns different fields
        return new InvoiceResource($invoice);
    }
}
```

### Versioned Resource

The main difference between versions is usually in Resources — fields added, removed,
or restructured:

```php
// V1 — original format
namespace App\Api\V1\Invoices\Resources;

class InvoiceResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'number' => $this->number,
            'status' => $this->status->value,         // V1: simple string
            'total_price' => $this->total_price,
            'due_date' => $this->due_date->format('Y-m-d'),
        ];
    }
}

// V2 — expanded format
namespace App\Api\V2\Invoices\Resources;

class InvoiceResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'number' => $this->number,
            'status' => [                              // V2: object with metadata
                'value' => $this->status->value,
                'label' => $this->status->label(),
                'color' => $this->status->color(),
            ],
            'pricing' => [                             // V2: grouped
                'total' => $this->total_price,
                'currency' => 'BRL',
            ],
            'due_date' => $this->due_date->toIso8601String(),  // V2: ISO 8601
            'created_at' => $this->created_at->toIso8601String(),
        ];
    }
}
```

### Versioning Rules

| Rule | Description |
|---|---|
| **Only version what changes** | If V2 only changes Invoices, don't copy Customers to V2 |
| **Domain is shared** | Actions, Models, Enums, QueryBuilders — same across all versions |
| **Deprecate, don't delete** | Keep V1 running as long as there are consumers |
| **Breaking changes = new version** | Field removal, type changes, response restructuring |
| **Additive changes = same version** | New optional fields, new endpoints, new query params |
| **Reuse what you can** | V2 can use Queries and ViewModels from V1 if they haven't changed |

### Reuse Between Versions

When a component doesn't change between versions, reuse it directly:

```php
namespace App\Api\V2\Invoices\Controllers;

// V2 uses the same Query from V1 — it didn't change
use App\Api\V1\Invoices\Queries\InvoiceIndexQuery;
use App\Api\V2\Invoices\Resources\InvoiceResource;

class InvoicesController
{
    public function index(InvoiceIndexQuery $query)
    {
        return InvoiceResource::collection(
            $query->build()->paginate()
        );
    }
}
```

### Deprecation Headers

Signal deprecation of older versions in controllers or middleware:

```php
// src/App/Api/Support/Middleware/DeprecatedApiVersion.php
class DeprecatedApiVersion
{
    public function handle(Request $request, Closure $next, string $sunset): Response
    {
        $response = $next($request);

        $response->headers->set('Deprecation', 'true');
        $response->headers->set('Sunset', $sunset);
        $response->headers->set('Link', '</api/v2>; rel="successor-version"');

        return $response;
    }
}

// routes/api_v1.php
Route::prefix('v1')
    ->middleware('deprecated:2026-06-01')
    ->group(function () {
        // ...
    });
```

### When to Create a New Version

- **Remove fields** from an existing response
- **Change types** (string → object, int → float)
- **Restructure** the response format (flat → nested)
- **Change behavior** of existing endpoints
- **Remove endpoints**

**Do NOT** create a new version to:

- Add optional fields to the response
- Create new endpoints
- Add optional query parameters
- Fix bugs

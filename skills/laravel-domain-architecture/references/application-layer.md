---
title: Application Layer
scope: Controllers, Requests, Resources, HTTP Queries, ViewModels (Blade and API), Jobs, API Versioning
---

# Application Layer — Detailed Reference

## Table of Contents

1. [Application Structure](#application-structure)
2. [Controllers](#controllers)
3. [HTTP Query Builders](#http-query-builders)
4. [Requests and Validation](#requests-and-validation)
5. [Resources](#resources)
6. [ViewModels — API](#viewmodels--api)
7. [ViewModels — Blade](#viewmodels--blade)
8. [Jobs](#jobs)
9. [API Versioning](#api-versioning)

---

## Application Structure

The application layer uses **standard Laravel structure**. No custom directories, no custom
namespaces. Group by domain within each folder only when it grows large.

### Starting Structure (small-medium project)

```
app/Http/
├── Controllers/
│   ├── Api/
│   │   ├── InvoiceController.php
│   │   ├── CustomerController.php
│   │   └── PaymentController.php
│   └── Web/
│       └── InvoiceController.php
├── Requests/
│   ├── StoreInvoiceRequest.php
│   ├── UpdateInvoiceRequest.php
│   └── StoreCustomerRequest.php
├── Resources/
│   ├── InvoiceResource.php
│   ├── InvoiceLineResource.php
│   └── CustomerResource.php
├── Queries/
│   └── InvoiceIndexQuery.php
├── ViewModels/
│   └── InvoiceDetailViewModel.php
└── Middleware/
```

### Grown Structure (large project)

When folders accumulate 15+ files, group by domain:

```
app/Http/
├── Controllers/
│   ├── Api/
│   │   ├── Invoice/
│   │   │   ├── InvoiceController.php
│   │   │   └── InvoiceStatusController.php
│   │   ├── Customer/
│   │   │   └── CustomerController.php
│   │   └── Payment/
│   │       └── PaymentController.php
│   └── Web/
│       └── InvoiceController.php
├── Requests/
│   ├── Invoice/
│   │   ├── StoreInvoiceRequest.php
│   │   └── UpdateInvoiceRequest.php
│   └── Customer/
│       └── StoreCustomerRequest.php
├── Resources/
│   ├── Invoice/
│   │   ├── InvoiceResource.php
│   │   └── InvoiceLineResource.php
│   └── Customer/
│       └── CustomerResource.php
├── Queries/
│   └── InvoiceIndexQuery.php
├── ViewModels/
│   └── InvoiceDetailViewModel.php
└── Middleware/
```

Grouping by domain avoids the problem of having a `Controllers/` directory with 100+ files.

---

## Controllers

Controllers should be thin — they receive input, delegate to domain actions, and return output.
They belong to the app layer because they know about HTTP, but should not contain business logic.

### Standard Controller

```php
namespace App\Http\Controllers\Api;

use App\Domain\Invoice\CreateInvoiceAction;
use App\Domain\Invoice\Invoice;
use App\Domain\Invoice\InvoiceData;
use App\Http\Queries\InvoiceIndexQuery;
use App\Http\Requests\StoreInvoiceRequest;
use App\Http\Resources\InvoiceResource;

class InvoiceController
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
        return new InvoiceResource(
            $invoice->load(['invoiceLines', 'payments'])
        );
    }
}
```

### Single Action Controller

For operations that are not CRUD, use invokable controllers:

```php
namespace App\Http\Controllers\Api;

use App\Domain\Invoice\Invoice;
use App\Domain\Invoice\MarkInvoiceAsPaidAction;
use App\Http\Resources\InvoiceResource;

class MarkInvoiceAsPaidController
{
    public function __invoke(Invoice $invoice, MarkInvoiceAsPaidAction $action)
    {
        return new InvoiceResource(
            $action->execute($invoice)
        );
    }
}
```

### Controller with ViewModel (complex response)

When the response combines data from multiple sources, use a ViewModel:

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

---

## HTTP Query Builders

For listings with filters, sorting, and pagination, extract the query logic into a dedicated class.
No packages — use Eloquent's native `when()`:

### Dedicated Query Class

```php
namespace App\Http\Queries;

use App\Domain\Invoice\Invoice;
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
class InvoiceController
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
class PaidInvoiceController
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

## Requests and Validation

Use standard Laravel Form Requests:

```php
namespace App\Http\Requests;

use App\Domain\Invoice\Invoice;
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

API Resources transform models/data for HTTP output:

```php
namespace App\Http\Resources;

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

For simple endpoints, the Resource is usually enough. Use ViewModels only when
composing data from multiple sources.

---

## ViewModels — API

Use ViewModels **only when the response combines data from multiple sources** (model + permissions +
metadata + calculated data). For single-model responses, use Resources directly.

### When to Use ViewModel in API

| Situation | Use |
|-----------|-----|
| Response is 1 model or list | **Resource** directly |
| Response combines model + permissions + metadata | **ViewModel** |
| Same composition reused across multiple endpoints | **ViewModel** |

### Implementation

```php
namespace App\Http\ViewModels;

use App\Domain\Invoice\Invoice;
use App\Domain\Invoice\InvoiceStatus;
use App\Http\Resources\InvoiceResource;
use App\Http\Resources\InvoiceLineResource;
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

---

## ViewModels — Blade

For applications with Blade, ViewModels implement `Arrayable` and group everything the view needs.

```php
namespace App\Http\ViewModels;

use App\Domain\Customer\Customer;
use App\Domain\Invoice\Invoice;
use App\Domain\Invoice\InvoiceStatus;
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

## Jobs

Jobs belong to the app layer because their responsibility is managing queue infrastructure
(retries, timeouts, middleware). Business logic stays in domain actions.

### Standard Job

```php
namespace App\Jobs;

use App\Domain\Invoice\Invoice;
use App\Domain\Invoice\SendInvoiceMailAction;
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

## API Versioning

**Don't version until you need to.** While there's only one version, no prefix is needed.

### Single Version (no prefix)

```
app/Http/Controllers/Api/InvoiceController.php
```

```php
// routes/api.php
Route::apiResource('invoices', InvoiceController::class);
```

### When V2 Is Needed

Only create version folders when the first breaking change happens:

```
app/Http/
├── Controllers/Api/
│   ├── V1/
│   │   └── InvoiceController.php
│   └── V2/
│       └── InvoiceController.php
├── Requests/
│   ├── V1/
│   │   └── StoreInvoiceRequest.php
│   └── V2/
│       └── StoreInvoiceRequest.php
└── Resources/
    ├── V1/
    │   └── InvoiceResource.php
    └── V2/
        └── InvoiceResource.php
```

### Routes per Version

Create separate route files for each version:

```php
// routes/api_v1.php
Route::prefix('v1')->group(function () {
    Route::apiResource('invoices', V1\InvoiceController::class);
    Route::apiResource('customers', CustomerController::class); // didn't change — no V1 prefix
});

// routes/api_v2.php
Route::prefix('v2')->group(function () {
    Route::apiResource('invoices', V2\InvoiceController::class);
    // Customers didn't change — no need to replicate in V2
});
```

Register in `bootstrap/app.php` (Laravel 11+):

```php
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
namespace App\Http\Controllers\Api\V2;

use App\Domain\Invoice\CreateInvoiceAction;
use App\Domain\Invoice\InvoiceData;
use App\Http\Requests\V2\StoreInvoiceRequest;
use App\Http\Resources\V2\InvoiceResource;

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

### Versioned Resource

The main difference between versions is usually in Resources:

```php
// V1 — original format
namespace App\Http\Resources\V1;

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
namespace App\Http\Resources\V2;

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

### Reuse Between Versions

When a component doesn't change between versions, reuse it directly:

```php
namespace App\Http\Controllers\Api\V2;

// V2 uses the same Query from before — it didn't change
use App\Http\Queries\InvoiceIndexQuery;
use App\Http\Resources\V2\InvoiceResource;

class InvoiceController
{
    public function index(InvoiceIndexQuery $query)
    {
        return InvoiceResource::collection(
            $query->build()->paginate()
        );
    }
}
```

### Versioning Rules

| Rule | Description |
|---|---|
| **Only version what changes** | Don't copy modules that didn't change |
| **Domain is shared** | Actions, Models, Enums — same across all versions |
| **Deprecate, don't delete** | Keep V1 running as long as there are consumers |
| **Breaking changes = new version** | Field removal, type changes, response restructuring |
| **Additive changes = same version** | New optional fields, new endpoints, new query params |
| **Reuse what you can** | V2 can use Queries and ViewModels from V1 if unchanged |

### When to Create a New Version

- **Remove fields** from an existing response
- **Change types** (string -> object, int -> float)
- **Restructure** the response format (flat -> nested)
- **Change behavior** of existing endpoints
- **Remove endpoints**

**Do NOT** create a new version to:

- Add optional fields to the response
- Create new endpoints
- Add optional query parameters
- Fix bugs

### Deprecation Headers

Signal deprecation of older versions in middleware:

```php
// app/Http/Middleware/DeprecatedApiVersion.php
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

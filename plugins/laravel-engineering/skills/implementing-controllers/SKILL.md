---
name: implementing-controllers
description: Use when creating HTTP controllers, defining routes, implementing API endpoints, setting up Inertia page rendering, or organizing controller concerns in a DDD bounded context.
---

## Overview

Controllers follow the ADR (Action-Domain-Responder) pattern — one action per invokable controller. They live in the Infrastructure layer, receive validated input from Form Requests, delegate to use cases, and return responses. Controllers never contain business logic.

---

## Invokable Controller (Standard Pattern)

### Web Controller (Inertia Rendering)

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Infrastructure\Http\Controllers;

use App\CustomerRelationshipManagement\Contacts\Application\FindContact;
use App\CustomerRelationshipManagement\Contacts\Domain\Contracts\ContactRepository;
use App\CustomerRelationshipManagement\Contacts\Domain\Exceptions\ContactNotFound;
use Inertia\Inertia;
use Inertia\Response;

class ShowContactController
{
    private FindContact $finder;

    public function __construct(ContactRepository $repository)
    {
        $this->finder = new FindContact($repository);
    }

    /**
     * @throws ContactNotFound
     */
    public function __invoke(string $id): Response
    {
        return Inertia::render('Contacts/Show', [
            'contact' => $this->finder->byId($id),
        ]);
    }
}
```

### Web Controller (Create/Update Action)

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Infrastructure\Http\Controllers;

use App\CustomerRelationshipManagement\Contacts\Application\CreateContact;
use App\CustomerRelationshipManagement\Contacts\Infrastructure\Http\Requests\ContactRequest;
use Illuminate\Http\RedirectResponse;

class CreateContactController
{
    public function __construct(private CreateContact $creator) {}

    public function __invoke(ContactRequest $request): RedirectResponse
    {
        $contact = $this->creator->create($request->validated());

        return redirect()
            ->route('contacts.show', $contact)
            ->banner('Contact created successfully.');
    }
}
```

### Web Controller (Delete Action)

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Infrastructure\Http\Controllers;

use App\CustomerRelationshipManagement\Contacts\Application\DeleteContact;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class DeleteContactController
{
    public function __construct(private DeleteContact $deleter) {}

    public function __invoke(Request $request, string $id): RedirectResponse
    {
        $this->deleter->delete($id);

        return redirect()
            ->route('contacts.index')
            ->banner('Contact deleted.');
    }
}
```

### API Controller (JSON Response)

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Infrastructure\Http\Controllers\API;

use App\CustomerRelationshipManagement\Contacts\Application\FindContact;
use App\CustomerRelationshipManagement\Contacts\Domain\Contracts\ContactRepository;
use App\CustomerRelationshipManagement\Contacts\Infrastructure\Http\Requests\SearchContactRequest;
use Illuminate\Http\JsonResponse;

class SearchContactController
{
    private FindContact $finder;

    public function __construct(ContactRepository $repository)
    {
        $this->finder = new FindContact($repository);
    }

    public function __invoke(SearchContactRequest $request): JsonResponse
    {
        return response()->json(
            $this->finder->byCriteria($request->criteria())
        );
    }
}
```

---

## Naming Convention

Controllers follow `{Verb}{Entity}Controller` with one action per class:

| Action | Controller Name | HTTP Method |
|---|---|---|
| List / Index | `ListContactsController` | GET |
| Show | `ShowContactController` | GET |
| Create (form) | `CreateContactController` | GET |
| Store | `StoreContactController` | POST |
| Edit (form) | `EditContactController` | GET |
| Update | `UpdateContactController` | PUT/PATCH |
| Delete | `DeleteContactController` | DELETE |
| Search (API) | `SearchContactController` | POST |

**Exception:** Simple CRUD tokens or similar trivial resources may use a traditional resource controller with `index`, `store`, `update`, `destroy` methods. This should be rare — the ADR pattern is the default.

---

## Directory Structure

```text
{Module}/Infrastructure/Http/
├── Controllers/
│   ├── ListContactsController.php
│   ├── ShowContactController.php
│   ├── CreateContactController.php
│   ├── UpdateContactController.php
│   ├── DeleteContactController.php
│   └── API/
│       ├── SearchContactController.php
│       └── ListContactsController.php
├── Requests/
│   ├── ContactRequest.php
│   └── SearchContactRequest.php
└── (routes loaded from context-level Shared)
```

Web controllers in `Controllers/`, API controllers in `Controllers/API/`. Both delegate to the same use cases.

---

## Route Organization

Routes are organized at the bounded context level, not per module:

```text
{BoundedContext}/Shared/Infrastructure/Http/Routes/
├── web.php
└── api.php
```

### Web Routes

```php
use App\CustomerRelationshipManagement\Contacts\Infrastructure\Http\Controllers\ListContactsController;
use App\CustomerRelationshipManagement\Contacts\Infrastructure\Http\Controllers\ShowContactController;
use App\CustomerRelationshipManagement\Contacts\Infrastructure\Http\Controllers\CreateContactController;
use App\CustomerRelationshipManagement\Contacts\Infrastructure\Http\Controllers\DeleteContactController;

Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/contacts', ListContactsController::class)->name('contacts.index');
    Route::get('/contacts/{id}', ShowContactController::class)->name('contacts.show');
    Route::post('/contacts', CreateContactController::class)->name('contacts.store');
    Route::delete('/contacts/{id}', DeleteContactController::class)->name('contacts.destroy');
});
```

### API Routes

```php
use App\CustomerRelationshipManagement\Contacts\Infrastructure\Http\Controllers\API\SearchContactController;

Route::middleware(['auth:sanctum'])->group(function () {
    Route::post('/contacts/search', SearchContactController::class);
});
```

### Loading Context Routes

The bounded context ServiceProvider loads routes during boot. See `configuring-bounded-contexts` for the full ServiceProvider setup.

---

## Base Controllers

The Shared Kernel provides minimal base controllers:

### Abstract Base Controller

```php
declare(strict_types=1);

namespace App\Shared\Infrastructure\Http\Controllers;

use Illuminate\Routing\Controller as BaseController;

abstract class Controller extends BaseController
{
    protected function localizedMarkdownPath(string $path): string
    {
        // Resolves localized markdown file paths for static pages
    }
}
```

### Inertia Base Controller

```php
declare(strict_types=1);

namespace App\Shared\Infrastructure\Http\Controllers;

use Inertia\Inertia;
use Inertia\Response;

abstract class InertiaController extends Controller
{
    protected function render(string $component, array $props = []): Response
    {
        return Inertia::render($component, $props);
    }
}
```

Base controllers provide utilities, not behavior. Most controllers don't need to extend them — only use when the helper methods are needed (e.g., static page controllers, Inertia-heavy controllers).

---

## Controller Concerns

Shared traits for sensitive operations:

```php
declare(strict_types=1);

namespace App\Shared\Infrastructure\Http\Controllers\Concerns;

trait ConfirmsPasswords
{
    // Password confirmation flow for sensitive actions (delete account, change password)
}

trait ConfirmsTwoFactorAuthentication
{
    // Validates 2FA state before allowing access
}
```

**Usage:**

```php
class DeleteAccountController
{
    use ConfirmsPasswords;

    public function __invoke(DeleteAccountRequest $request): RedirectResponse
    {
        $this->ensurePasswordConfirmed();

        $this->deleter->delete(id: $request->user()->id, owner: $request->user()->id);

        return redirect()->route('dashboard');
    }
}
```

**Placement:** `app/Shared/Infrastructure/Http/Controllers/Concerns/`

---

## Constructor Injection

Controllers inject dependencies via constructor — primarily use cases, occasionally repositories:

| Dependency | When |
|---|---|
| Use case | Always — controllers delegate to use cases |
| Repository | When the controller needs to compose a finder use case |
| EventBus | Rare — only if the controller publishes events directly (prefer use cases) |

**Standard pattern — inject use case directly:**

```php
public function __construct(private CreateContact $creator) {}
```

**Finder pattern — compose from repository:**

```php
public function __construct(ContactRepository $repository)
{
    $this->finder = new FindContact($repository);
}
```

This mirrors the use case composition pattern (see `implementing-use-cases`).

---

## Response Patterns

### Inertia Render (Page Display)

```php
return Inertia::render('Contacts/Show', [
    'contact' => $contact,
    'sessions' => $user->sessions(),
]);
```

### Redirect with Flash (After Mutation)

```php
return redirect()
    ->route('contacts.index')
    ->banner('Contact created successfully.');
```

Flash macros (`banner()`, `error()`) are registered in the Shared ServiceProvider:

```php
RedirectResponse::macro('banner', function (string $message) {
    return $this->with('flash.banner', $message);
});
```

### JSON Response (API)

```php
return response()->json($this->finder->byCriteria($request->criteria()));
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Business logic in controllers | Delegate to use cases; controllers orchestrate only |
| Validation in controllers | Use Form Request classes (see `implementing-form-requests`) |
| Multiple actions in one controller | One controller = one `__invoke()` method (ADR pattern) |
| Injecting repositories instead of use cases | Inject use cases; repositories are internal to use cases |
| Putting API controllers in the same directory as web | Separate into `Controllers/API/` subdirectory |
| Extending base controller unnecessarily | Only extend when you need the utility methods |
| Route definitions inside module directories | Routes live at bounded context level in `Shared/Infrastructure/Http/Routes/` |

---

## Pre-flight Checklist

- [ ] Controller class in `{Module}/Infrastructure/Http/Controllers/`?
- [ ] Named as `{Verb}{Entity}Controller`?
- [ ] Single `__invoke()` method (invokable pattern)?
- [ ] Constructor injects use cases (not repositories directly)?
- [ ] Form Request handles validation (not the controller)?
- [ ] Web controller returns Inertia render or redirect?
- [ ] API controller in `Controllers/API/` subdirectory?
- [ ] API controller returns JSON response?
- [ ] Routes defined in bounded context's `Shared/Infrastructure/Http/Routes/`?
- [ ] Controller concerns from Shared used for sensitive operations?

---

## Laravel Reference

This skill uses Laravel-specific features. Consult the official Laravel documentation (via Context7 or web search) for details on:

- `Illuminate\Routing\Controller` — base controller class
- Invokable controllers — single-action controllers with `__invoke()`
- `Inertia\Inertia::render()` — server-side rendering for Inertia.js
- `Illuminate\Http\RedirectResponse` — redirect responses with flash data
- `Illuminate\Http\JsonResponse` — JSON API responses
- Route registration — `Route::get()`, `Route::post()`, `Route::delete()`, middleware groups
- `Illuminate\Http\Request` — HTTP request handling
- `auth:sanctum` middleware — API authentication via Laravel Sanctum

---
name: implementing-form-requests
description: Use when creating form request validation classes, connecting domain validation rules to HTTP requests, building filter or search requests, or setting up criteria-based querying from the HTTP layer.
---

## Overview

Form Requests handle HTTP-layer validation and authorization. They live in Infrastructure and bridge Domain validation rules to the HTTP boundary. Every authenticated endpoint uses a Form Request — validation never lives in controllers.

---

## Standard Form Request

Reuses the `ValidationRules` trait from the Domain layer:

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Infrastructure\Http\Requests;

use App\CustomerRelationshipManagement\Contacts\Domain\ValidationRules;
use App\Shared\Infrastructure\Http\Requests\AuthenticatedRequest;

class ContactRequest extends AuthenticatedRequest
{
    use ValidationRules;
}
```

The `ValidationRules` trait provides the `rules()` method — the Form Request inherits it directly. No need to override `rules()` unless you need to merge additional HTTP-specific rules.

### With Augmented Data

Override `validated()` to inject context the client doesn't send:

```php
class ContactRequest extends AuthenticatedRequest
{
    use ValidationRules;

    public function validated($key = null, $default = null): array
    {
        return array_merge(parent::validated(), [
            'account_id' => $this->user()->currentAccount->id,
        ]);
    }
}
```

### With Inline Rules

For requests with simple, one-off validation that doesn't belong in the Domain:

```php
class UploadDocumentRequest extends AuthenticatedRequest
{
    public function rules(): array
    {
        return [
            'file' => ['required', 'file', 'mimes:pdf,jpg,png', 'max:10240'],
            'label' => ['nullable', 'string', 'max:255'],
        ];
    }
}
```

---

## Naming Convention

| Request Type | Naming | Example |
|---|---|---|
| Create / Update | `{Entity}Request` | `ContactRequest` |
| Specific action | `{Action}{Entity}Request` | `InviteUserToAccountRequest` |
| List / Filter | `Filter{Entity}Request` | `FilterContactRequest` |
| Search (API) | `Search{Entity}Request` | `SearchContactRequest` |
| Delete (with validation) | `Delete{Entity}Request` | `DeleteAccountRequest` |

Use `{Entity}Request` when the same rules apply to both create and update (the `ValidationRules` trait's `rules()` method accepts an optional entity ID to toggle unique constraints). Use `{Action}{Entity}Request` when the action has unique validation needs.

---

## Base Request Classes

The Shared Kernel provides two base classes:

### AuthenticatedRequest

```php
declare(strict_types=1);

namespace App\Shared\Infrastructure\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class AuthenticatedRequest extends FormRequest
{
    public function authorize(): bool
    {
        return auth()->check();
    }
}
```

All module-level form requests extend this class. Authentication is handled once — individual requests only override `authorize()` when they need additional ownership checks.

### SearchByCriteriaRequest

```php
declare(strict_types=1);

namespace App\Shared\Infrastructure\Http\Requests;

use ComplexHeart\Domain\Criteria\Criteria;

class SearchByCriteriaRequest extends AuthenticatedRequest
{
    public function rules(): array
    {
        return [
            'filters' => ['nullable', 'array'],
            'filters.*.field' => ['required_with:filters', 'string'],
            'filters.*.operator' => ['required_with:filters', 'string'],
            'filters.*.value' => ['required_with:filters'],
            'orderBy' => ['nullable', 'string'],
            'orderType' => ['nullable', 'string', 'in:asc,desc'],
            'page' => ['nullable', 'integer', 'min:1'],
            'perPage' => ['nullable', 'integer', 'min:1', 'max:100'],
        ];
    }

    public function criteria(): Criteria
    {
        return Criteria::fromValues(
            filters: $this->input('filters', []),
            orderBy: $this->input('orderBy'),
            orderType: $this->input('orderType', 'asc'),
            page: $this->integer('page', 1),
            perPage: $this->integer('perPage', 15),
        );
    }
}
```

**Placement:** `app/Shared/Infrastructure/Http/Requests/`

---

## Filter Requests

For web list endpoints with pagination and sorting:

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Infrastructure\Http\Requests;

use App\Shared\Infrastructure\Http\Requests\AuthenticatedRequest;

class FilterContactRequest extends AuthenticatedRequest
{
    public function rules(): array
    {
        return [
            'search' => ['nullable', 'string', 'max:255'],
            'sort' => ['nullable', 'string', 'in:name,email,created_at'],
            'direction' => ['nullable', 'string', 'in:asc,desc'],
            'per_page' => ['nullable', 'integer', 'min:1', 'max:100'],
        ];
    }
}
```

## Search Requests

For API search endpoints using the Criteria system:

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Infrastructure\Http\Requests;

use App\Shared\Infrastructure\Http\Requests\SearchByCriteriaRequest;

class SearchContactRequest extends SearchByCriteriaRequest {}
```

Most search requests are empty classes — the base handles all criteria validation. Override `rules()` only to add entity-specific constraints on top.

**Controller usage:**

```php
public function __invoke(SearchContactRequest $request): JsonResponse
{
    return response()->json(
        $this->finder->byCriteria($request->criteria())
    );
}
```

---

## Authorization

### Simple Authentication (Default)

Inherited from `AuthenticatedRequest` — no override needed:

```php
class ContactRequest extends AuthenticatedRequest
{
    use ValidationRules;
    // authorize() returns auth()->check() from base
}
```

### Ownership Verification

Override `authorize()` for resource-level permission checks:

```php
class UpdateAccountInformationRequest extends AuthenticatedRequest
{
    use ValidationRules;

    public function authorize(): bool
    {
        $account = Account::find($this->route('id'));

        return $account && $account->isOwner($this->user()->id);
    }
}
```

---

## Directory Structure

```text
{Module}/Infrastructure/Http/Requests/
├── ContactRequest.php              # Create/Update
├── FilterContactRequest.php        # Web list endpoint
├── SearchContactRequest.php        # API search endpoint
├── DeleteContactRequest.php        # Delete (if validation needed)
└── InviteContactRequest.php        # Specific action
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Validation logic in controllers | Move to Form Request classes |
| Duplicating domain rules in Form Request | Use `ValidationRules` trait from Domain layer |
| Extending `FormRequest` directly | Extend `AuthenticatedRequest` (handles auth check) |
| Putting form requests in Application or Domain | Form Requests are Infrastructure: `Infrastructure/Http/Requests/` |
| Custom query methods on search requests | Use `criteria()` from `SearchByCriteriaRequest` base |
| Not overriding `validated()` for context injection | Override when you need to add `account_id` or similar |

---

## Pre-flight Checklist

- [ ] Form Request in `{Module}/Infrastructure/Http/Requests/`?
- [ ] Extends `AuthenticatedRequest` (or `SearchByCriteriaRequest` for search)?
- [ ] Uses `ValidationRules` trait from Domain when rules are shared?
- [ ] Named as `{Entity}Request`, `Filter{Entity}Request`, or `Search{Entity}Request`?
- [ ] `authorize()` overridden only when ownership checks are needed?
- [ ] `validated()` overridden to inject context (e.g., `account_id`) when needed?
- [ ] Controller type-hints the Form Request (not `Request`)?

---

## Laravel Reference

This skill uses Laravel-specific features. Consult the official Laravel documentation (via Context7 or web search) for details on:

- `Illuminate\Foundation\Http\FormRequest` — base form request class with validation and authorization
- `authorize()` — form request authorization method
- `rules()` — validation rules definition
- `validated()` — retrieving validated input data
- `Illuminate\Support\Facades\Validator` — manual validator creation (used in use cases, not form requests)
- `complexheart/laravel-criteria` — Criteria pattern for domain-driven querying (see package docs)

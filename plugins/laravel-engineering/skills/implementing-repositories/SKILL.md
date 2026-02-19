---
name: implementing-repositories
description: Use when defining repository contracts, implementing Eloquent repositories, setting up criteria-based querying with field mapping, or binding repositories in a ServiceProvider.
---

## Overview

Repositories abstract persistence behind domain contracts. The contract interface lives in `Domain/Contracts/`; the Eloquent implementation lives in `Infrastructure/Persistence/`. Repositories handle persistence only — they never publish events (that's the use case's job).

---

## Repository Contract

Every aggregate root has a repository contract with four standard methods:

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Domain\Contracts;

use App\CustomerRelationshipManagement\Contacts\Domain\Contact;
use ComplexHeart\Domain\Criteria\Criteria;
use Illuminate\Pagination\LengthAwarePaginator;

interface ContactRepository
{
    public function find(string $id): ?Contact;

    public function match(Criteria $criteria): LengthAwarePaginator;

    public function save(Contact $contact): Contact;

    public function delete(Contact $contact): void;
}
```

**Rules:**

- Named `{Entity}Repository` — always an interface
- Four standard methods: `find`, `match`, `save`, `delete`
- `find()` returns nullable entity — never throws (the use case handles not-found logic)
- `match()` accepts `Criteria` and returns `LengthAwarePaginator` for paginated results
- `save()` returns the persisted entity
- `delete()` returns void
- No custom query methods (e.g., `findByEmail()`) — use `match()` with criteria instead
- Placement: `{Module}/Domain/Contracts/`

### Read-Only Contract

When an entity is a read projection or lookup-only, omit `save()` and `delete()`:

```php
interface ManagerRepository
{
    public function find(string $id): ?Manager;

    public function match(Criteria $criteria): LengthAwarePaginator;
}
```

---

## Eloquent Implementation

### Standard Implementation

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Infrastructure\Persistence;

use App\CustomerRelationshipManagement\Contacts\Domain\Contact;
use App\CustomerRelationshipManagement\Contacts\Domain\Contracts\ContactRepository;
use ComplexHeart\Domain\Criteria\Criteria;
use ComplexHeart\Infrastructure\Laravel\EloquentCriteriaParser;
use Illuminate\Pagination\LengthAwarePaginator;

class EloquentContactRepository implements ContactRepository
{
    public function find(string $id): ?Contact
    {
        return Contact::find($id);
    }

    public function match(Criteria $criteria): LengthAwarePaginator
    {
        $parser = new EloquentCriteriaParser([], true);

        return $parser->applyCriteria(Contact::query(), $criteria);
    }

    public function save(Contact $contact): Contact
    {
        $contact->save();

        return $contact;
    }

    public function delete(Contact $contact): void
    {
        $contact->delete();
    }
}
```

### With Eager Loading

Load relationships when needed for performance:

```php
public function find(string $id): ?Visa
{
    return Visa::with('applicant')->find($id);
}

public function match(Criteria $criteria): LengthAwarePaginator
{
    $parser = new EloquentCriteriaParser([], true);

    return $parser->applyCriteria(
        Visa::query()->with('applicant'),
        $criteria
    );
}
```

### With Field Mapping

Map domain field names to database column names in the criteria parser:

```php
public function match(Criteria $criteria): LengthAwarePaginator
{
    $parser = new EloquentCriteriaParser([
        'name' => 'first_name',
        'account' => 'account_id',
    ], true);

    return $parser->applyCriteria(Contact::query(), $criteria);
}
```

### With Cross-Table Joins

When filtering requires data from related tables:

```php
public function match(Criteria $criteria): LengthAwarePaginator
{
    $query = Visa::with('applicant')
        ->select('vm_visas.*')
        ->join('crm_contacts', 'vm_visas.applicant_id', '=', 'crm_contacts.id');

    $parser = new EloquentCriteriaParser([
        'id' => 'vm_visas.id',
        'account' => 'crm_contacts.account_id',
        'applicant' => 'vm_visas.applicant_id',
    ], true);

    return $parser->applyCriteria($query, $criteria);
}
```

### With Delete Cleanup

When deletion requires cleaning up related data:

```php
public function delete(User $user): void
{
    $user->deleteProfilePhoto();
    $user->tokens->each->delete();
    $user->delete();
}
```

```php
public function delete(CustomAttributeGroup $group): void
{
    $group->attributes()->delete();
    $group->delete();
}
```

### With Exception Translation

Convert database exceptions to domain exceptions:

```php
use App\CustomerRelationshipManagement\Documents\Domain\Exceptions\DocumentException;
use App\CustomerRelationshipManagement\Documents\Domain\Exceptions\InvalidOwnerReference;
use PDOException;

public function save(Document $document): Document
{
    try {
        $document->save();

        return $document;
    } catch (PDOException $e) {
        match ($e->getCode()) {
            '23000' => throw new InvalidOwnerReference('Invalid owner reference.'),
            default => throw new DocumentException("Unable to persist the document due to: {$e->getMessage()}"),
        };
    }
}
```

**Placement:** `{Module}/Infrastructure/Persistence/`

---

## Criteria System

The `match()` method uses `ComplexHeart\Domain\Criteria\Criteria` with `EloquentCriteriaParser` to translate domain queries into Eloquent queries.

### EloquentCriteriaParser

```php
$parser = new EloquentCriteriaParser(
    fieldMapping: ['domainField' => 'database_column'],
    paginate: true,
);

return $parser->applyCriteria($query, $criteria);
```

**Constructor parameters:**

| Parameter | Type | Purpose |
|---|---|---|
| `fieldMapping` | `array` | Maps domain field names to database column names |
| `paginate` | `bool` | `true` returns `LengthAwarePaginator`, `false` returns `Collection` |

**What the parser handles automatically:**
- Filtering (where clauses from criteria filters)
- Sorting (order by from criteria order)
- Pagination (page + per-page from criteria)

### Field Mapping Examples

```php
// No mapping needed — domain and database names match
new EloquentCriteriaParser([], true);

// Simple renames
new EloquentCriteriaParser([
    'name' => 'first_name',
    'account' => 'account_id',
], true);

// Qualified columns for joins
new EloquentCriteriaParser([
    'id' => 'vm_visas.id',
    'account' => 'crm_contacts.account_id',
], true);
```

---

## ServiceProvider Binding

Repository contracts are bound to implementations in the bounded context's ServiceProvider via the `$bindings` array:

```php
class CustomerRelationshipManagementServiceProvider extends ServiceProvider
{
    public array $bindings = [
        ContactRepository::class => EloquentContactRepository::class,
        ManagerRepository::class => EloquentManagerRepository::class,
        DocumentRepository::class => EloquentDocumentRepository::class,
    ];
}
```

Laravel resolves these automatically via constructor injection in use cases. See `configuring-bounded-contexts` for full ServiceProvider details.

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Publishing events in repositories | Repositories handle persistence only; use cases publish events |
| Adding custom query methods (`findByEmail()`) | Use `match()` with criteria — keeps the contract generic |
| Throwing not-found exceptions in `find()` | Return `null`; the use case decides how to handle missing entities |
| Putting contracts in Infrastructure | Contracts live in `Domain/Contracts/`, implementations in `Infrastructure/Persistence/` |
| Forgetting field mapping for renamed columns | Map domain field names to database columns in `EloquentCriteriaParser` |
| Missing eager loading on frequently accessed relations | Add `with()` to `find()` and `match()` queries as needed |
| Binding repositories in the wrong ServiceProvider | Bind in the owning bounded context's SP, not in `SharedServiceProvider` |

---

## Pre-flight Checklist

- [ ] Contract interface in `{Module}/Domain/Contracts/{Entity}Repository.php`?
- [ ] Four standard methods: `find`, `match`, `save`, `delete`?
- [ ] `find()` returns nullable entity (never throws)?
- [ ] `match()` uses `Criteria` and returns `LengthAwarePaginator`?
- [ ] Eloquent implementation in `{Module}/Infrastructure/Persistence/`?
- [ ] `EloquentCriteriaParser` configured with field mapping?
- [ ] No event publishing in repository methods?
- [ ] Delete method cleans up related data if needed?
- [ ] Contract bound in the context's ServiceProvider `$bindings`?

---

## Laravel Reference

This skill uses Laravel-specific features. Consult the official Laravel documentation (via Context7 or web search) for details on:

- `Illuminate\Pagination\LengthAwarePaginator` — paginated query results
- `Illuminate\Database\Eloquent\Builder` — Eloquent query builder (`query()`, `with()`, `find()`, `where()`)
- `Illuminate\Database\Eloquent\Model::save()` — persisting model state
- `complexheart/laravel-criteria` — Criteria pattern for domain-driven querying (see package docs)

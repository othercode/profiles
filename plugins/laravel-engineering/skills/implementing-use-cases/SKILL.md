---
name: implementing-use-cases
description: Use when creating application services, writing use case classes, implementing business operations, or composing use cases for create, find, update, and delete flows.
---

## Overview

Use cases are `final readonly class` entries in `{Module}/Application/`. Each class represents a single business operation. They orchestrate the flow: validate input → call domain methods → persist via repository → publish events.

---

## Use Case Types

### Create

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Application;

use App\CustomerRelationshipManagement\Contacts\Domain\Contact;
use App\CustomerRelationshipManagement\Contacts\Domain\Contracts\ContactRepository;
use App\CustomerRelationshipManagement\Contacts\Domain\ValidationRules;
use App\Shared\Domain\Contracts\ServiceBus\EventBus;
use Illuminate\Support\Facades\Validator;

final readonly class CreateContact
{
    use ValidationRules;

    public function __construct(
        private ContactRepository $repository,
        private EventBus $eventBus,
    ) {}

    public function create(array $input): Contact
    {
        $attributes = Validator::make($input, $this->rules())
            ->validate();

        $contact = $this->repository->save(Contact::new($attributes));
        $contact->publishDomainEvents($this->eventBus);

        return $contact;
    }
}
```

**Flow:** Validate → `Entity::new()` → repository save → publish events → return entity.

### Find

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Application;

use App\CustomerRelationshipManagement\Contacts\Domain\Contact;
use App\CustomerRelationshipManagement\Contacts\Domain\Contracts\ContactRepository;
use App\CustomerRelationshipManagement\Contacts\Domain\Exceptions\ContactNotFound;
use ComplexHeart\Domain\Criteria\Criteria;
use Illuminate\Pagination\LengthAwarePaginator;

final readonly class FindContact
{
    public function __construct(private ContactRepository $repository) {}

    /**
     * @throws ContactNotFound
     */
    public function byId(string $id): Contact
    {
        if (! $contact = $this->repository->find($id)) {
            throw new ContactNotFound(strtr('Contact {id} not found.', [
                '{id}' => $id,
            ]));
        }

        return $contact;
    }

    public function byCriteria(Criteria $criteria): LengthAwarePaginator
    {
        return $this->repository->match($criteria);
    }
}
```

**Rules:**

- `byId()` throws `{Entity}NotFound` — never returns null
- `byCriteria()` delegates to repository's `match()` — returns paginator
- No event bus needed — finders are read-only
- Other use cases compose finders to locate entities before operating on them

### Update

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Application;

use App\CustomerRelationshipManagement\Contacts\Domain\Contracts\ContactRepository;
use App\CustomerRelationshipManagement\Contacts\Domain\Exceptions\ContactNotFound;
use App\CustomerRelationshipManagement\Contacts\Domain\ValidationRules;
use App\Shared\Domain\Contracts\ServiceBus\EventBus;
use Illuminate\Support\Facades\Validator;

final readonly class UpdateContactInformation
{
    use ValidationRules;

    private FindContact $finder;

    public function __construct(
        private ContactRepository $repository,
        private EventBus $eventBus,
    ) {
        $this->finder = new FindContact($this->repository);
    }

    /**
     * @throws ContactNotFound
     */
    public function update(string $id, array $input): void
    {
        $contact = $this->finder->byId($id);

        Validator::make($input, $this->rules($contact->id))
            ->validate();

        if (isset($input['first_name'], $input['last_name'])) {
            $contact->updateName($input['first_name'], $input['last_name']);
        }

        if (isset($input['email'])) {
            $contact->updateEmail($input['email']);
        }

        $this->repository->save($contact);
        $contact->publishDomainEvents($this->eventBus);
    }
}
```

**Flow:** Find entity → validate → call domain methods → repository save → publish events.

**Rules:**

- Composes `Find{Entity}` to locate the entity
- Calls domain behavior methods on the entity (e.g., `updateName()`, `updateEmail()`) — never `forceFill()` directly
- Returns `void` for mutations (the caller already knows what was updated)
- Pass entity ID to `ValidationRules` methods to toggle unique rules for update vs create

### Delete

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Application;

use App\CustomerRelationshipManagement\Contacts\Domain\Contracts\ContactRepository;
use App\CustomerRelationshipManagement\Contacts\Domain\Exceptions\ContactNotFound;
use App\Shared\Domain\Contracts\ServiceBus\EventBus;

final readonly class DeleteContact
{
    private FindContact $finder;

    public function __construct(
        private ContactRepository $repository,
        private EventBus $eventBus,
    ) {
        $this->finder = new FindContact($this->repository);
    }

    public function delete(string $id): void
    {
        try {
            $contact = $this->finder->byId($id);
        } catch (ContactNotFound) {
            return;
        }

        $this->repository->delete($contact->toBeDeleted());
        $contact->publishDomainEvents($this->eventBus);
    }
}
```

**Flow:** Find entity (idempotent) → `toBeDeleted()` → repository delete → publish events.

**Rules:**

- Catches `{Entity}NotFound` and returns silently — deletion is **idempotent**
- Calls `toBeDeleted()` on the entity to register the deletion event
- Publishes events AFTER `$repository->delete()` (the entity object is still in memory)

---

## Naming Conventions

| Use Case Type | Naming | Method | Return |
|---|---|---|---|
| Create | `Create{Entity}` | `create(array $input)` | Entity |
| Find | `Find{Entity}` | `byId(string $id)`, `byCriteria(Criteria)` | Entity or Paginator |
| Update | `Update{Entity}Information` | `update(string $id, array $input)` | `void` |
| Delete | `Delete{Entity}` | `delete(string $id)` | `void` |
| State change | `Change{Entity}Status` | `change(string $id, string $status)` | Entity |

Use named methods matching the action — not `__invoke()`.

---

## Use Case Composition

Use cases compose other use cases by **instantiating** them in the constructor — not injecting via the container:

```php
final readonly class DeleteContact
{
    private FindContact $finder;

    public function __construct(
        private ContactRepository $repository,
        private EventBus $eventBus,
    ) {
        $this->finder = new FindContact($this->repository);
    }
}
```

**Why instantiate, not inject?** The finder depends on the same repository. Instantiating it keeps the dependency explicit and avoids unnecessary container bindings.

Common compositions:
- `Delete{Entity}` composes `Find{Entity}`
- `Update{Entity}` composes `Find{Entity}`
- `Accept{Action}` composes `Find{Entity}` + `Create{RelatedEntity}`

---

## Validation

Use cases validate input using `Validator::make()`, either with inline rules or the `ValidationRules` trait from the Domain layer:

### With ValidationRules Trait

```php
final readonly class CreateContact
{
    use ValidationRules;

    public function create(array $input): Contact
    {
        $attributes = Validator::make($input, $this->rules())
            ->validate();

        // ...
    }
}
```

### With Inline Rules

```php
public function create(array $input): Account
{
    $attributes = Validator::make($input, [
        'name' => ['required', 'string', 'max:255'],
    ])->validate();

    // ...
}
```

### With Error Bag (for form-based updates)

```php
public function update(User $user, array $input): void
{
    Validator::make($input, $this->rules($user->id))
        ->validateWithBag('updateProfileInformation');

    // ...
}
```

Prefer the `ValidationRules` trait when rules are shared between create and update flows. Use inline rules for simple, one-off validation.

---

## Authorization in Use Cases

When a use case requires ownership or permission checks, verify after finding the entity:

```php
public function update(string $id, string $owner, array $input): void
{
    $account = $this->finder->byId($id);

    if (! $account->isOwner($owner)) {
        throw new AccountOwnershipError('Only owners can update account information.');
    }

    // ... proceed with update
}
```

Authorization checks throw domain exceptions (`{Entity}OwnershipError`), not HTTP exceptions.

---

## Constructor Injection

Use cases receive dependencies via constructor injection:

| Dependency | When |
|---|---|
| `{Entity}Repository` | Always — every use case needs persistence |
| `EventBus` | When the use case creates, updates, or deletes entities |
| `ValidationRules` trait | When the use case validates input |

Finders and composed use cases are **instantiated**, not injected:

```php
public function __construct(
    private ContactRepository $repository,
    private EventBus $eventBus,
) {
    $this->finder = new FindContact($this->repository);
}
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Using `__invoke()` instead of named methods | Use `create()`, `delete()`, `byId()`, etc. |
| Returning null from finders | Throw `{Entity}NotFound` — never return null |
| Injecting finders via container | Instantiate in constructor: `new Find{Entity}($repository)` |
| Publishing events before persistence | Always publish AFTER `$repository->save()` or `$repository->delete()` |
| Calling `forceFill()` in use cases | Call domain behavior methods on the entity instead |
| Throwing on delete when entity not found | Catch `{Entity}NotFound` and return — deletion is idempotent |
| Putting business logic in use cases | Delegate to entity domain methods; use cases orchestrate |
| Missing `@throws` declarations | Declare all thrown exceptions in PHPDoc |

---

## Pre-flight Checklist

- [ ] Use case is `final readonly class`?
- [ ] Named method matches the action (`create()`, `delete()`, `byId()`)?
- [ ] Constructor injects repository + event bus?
- [ ] Finder composed via instantiation (not container injection)?
- [ ] Validation uses `Validator::make()` with rules trait or inline?
- [ ] Domain methods called on entity (not `forceFill()` in use case)?
- [ ] Events published after persistence?
- [ ] Delete is idempotent (catches not-found)?
- [ ] Authorization checks throw domain exceptions?
- [ ] Placement in `{Module}/Application/`?

---

## Laravel Reference

This skill uses Laravel-specific features. Consult the official Laravel documentation (via Context7 or web search) for details on:

- `Illuminate\Support\Facades\Validator` — manual validator creation with `Validator::make()`
- `Illuminate\Validation\ValidationException` — thrown when validation fails
- `validateWithBag()` — validation with named error bags for form-based flows
- `Illuminate\Pagination\LengthAwarePaginator` — paginated query results

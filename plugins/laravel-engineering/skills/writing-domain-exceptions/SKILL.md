---
name: writing-domain-exceptions
description: Use when creating domain exception classes, structuring exception hierarchies for a module, or adding application-layer exceptions for use case errors.
---

## Overview

Every module has a two-level exception tree rooted at `{Entity}Exception`. All exception classes are **empty-bodied** — naming alone conveys intent. Each module owns its own tree; never extend another module's exception base.

---

## Exception Hierarchy

```text
Exception (PHP built-in)
└── {Entity}Exception                    # Module root — always
    ├── {Entity}NotFound                 # Resource lookup failure — always
    ├── {Entity}OwnershipError           # Authorization violation — when needed
    ├── Invalid{Concept}Exception        # Validation/invariant failure — when needed
    ├── {Entity}AlreadyExists            # Uniqueness violation — when needed
    └── {Entity}{Action}Error            # Specific domain error — when needed
```

---

## Base Exception

One per module, extends `\Exception` directly:

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Domain\Exceptions;

use Exception;

class ContactException extends Exception {}
```

**Rules:**

- Named `{Entity}Exception` matching the aggregate root
- Extends `\Exception` — never another module's exception
- Empty body — no custom methods or properties
- Placement: `{Module}/Domain/Exceptions/`

---

## Standard Exception Types

All extend the module's base exception. All have empty bodies.

### Not-Found

```php
class ContactNotFound extends ContactException {}
```

Every module needs this. Thrown by finder use cases when the repository returns `null`.

### Ownership Error

```php
class ContactOwnershipError extends ContactException {}
```

Thrown by use cases that enforce resource ownership (e.g., "you can only manage your own contacts").

### Invalid / Validation

```php
class InvalidVisaSubclass extends VisaException {}
```

Thrown by value object constructors and entity methods when input violates domain invariants. Named `Invalid{Concept}Exception` or `Invalid{Concept}`.

### Already Exists

```php
class PackageAlreadyExists extends PackageException {}
```

Thrown when a uniqueness constraint is violated at the domain level.

### Domain-Specific Errors

```php
class DocumentStoreError extends DocumentException {}
```

```php
class AccountInvitationError extends AccountException {}
```

Named descriptively for the specific failure. Use `Error` or `Exception` suffix consistently within a module.

---

## Application-Layer Exceptions

Use case errors that don't belong to the domain live in `{Module}/Application/Exceptions/`:

```php
declare(strict_types=1);

namespace App\IdentityAndAccess\Users\Application\Exceptions;

use Exception;

class InvalidUserPassword extends Exception {}
```

**Rules:**

- Extend `\Exception` directly — they root their own small tree
- Same empty-body, naming-driven pattern as domain exceptions
- Use sparingly — most errors belong in the domain layer

---

## Throwing Exceptions

### In Value Objects (constructor validation)

```php
public function __construct(string $abn)
{
    $this->abn = str_replace(' ', '', $abn);

    if (! $this->isValidFormat()) {
        throw new InvalidABNException("Invalid ABN format {$this->abn}");
    }
}
```

### In Entities (business rule enforcement)

```php
public function addContact(array $attributes): self
{
    if ($this->contacts()->count() >= 10) {
        throw new CompanyException('Maximum 10 contacts allowed.');
    }

    $this->contacts()->create($attributes);

    return $this;
}
```

### In Use Cases (lookup + ownership)

```php
public function __invoke(string $id): Contact
{
    return Contact::find($id)
        ?? throw new ContactNotFound(strtr('Contact {id} not found.', ['{id}' => $id]));
}
```

### Exception Messages

- Use `strtr()` with token substitution for messages containing identifiers:
  ```php
  throw new AccountNotFound(strtr('Account {id} not found.', ['{id}' => $id]));
  ```
- Keep messages business-oriented and descriptive
- Always declare thrown exceptions in method docblocks:
  ```php
  /**
   * @throws ContactNotFound
   */
  ```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Adding methods or properties to exceptions | Keep all exception classes empty-bodied |
| Extending another module's base exception | Each module owns its own `{Entity}Exception` tree |
| Putting all exceptions under `\Exception` directly | Leaf exceptions extend the module's base, not `\Exception` |
| Domain validation errors in Application layer | `Invalid{Concept}` exceptions belong in `Domain/Exceptions/` |
| Generic exception messages without identifiers | Use `strtr()` with tokens for actionable messages |
| Missing `@throws` docblock declarations | Always declare thrown exceptions in PHPDoc |

---

## Pre-flight Checklist

- [ ] Base `{Entity}Exception` extends `\Exception` directly?
- [ ] `{Entity}NotFound` extends the module's base exception?
- [ ] All leaf exceptions have empty bodies?
- [ ] No cross-module exception inheritance?
- [ ] Application-layer exceptions in `Application/Exceptions/` if needed?
- [ ] All exceptions placed in `{Module}/Domain/Exceptions/`?
- [ ] `@throws` declarations on methods that throw?

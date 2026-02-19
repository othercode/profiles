---
name: writing-value-objects
description: Use when creating value objects, PHP enums, state classes, state transitions, or custom validation rule objects in a DDD bounded context.
---

## Overview

Value objects, enums, and state machines live in `{Module}/Domain/`. They encapsulate domain concepts with self-validating construction and immutable semantics. Enums model fixed sets, value objects model composite domain concepts, and state machines model lifecycle workflows.

---

## Value Objects

A value object is a `final readonly class` that validates its own invariants in the constructor. It is NOT an Eloquent model — it represents a domain concept with no identity.

### Simple Value Object

```php
declare(strict_types=1);

namespace App\CompanyRegistry\Shared\Domain;

use App\CompanyRegistry\Shared\Domain\Exceptions\InvalidPostCodeException;

final readonly class PostCode
{
    private string $value;

    private string $resolvedState;

    /**
     * @throws InvalidPostCodeException
     */
    public function __construct(string $postCode, ?string $state = null)
    {
        if (! is_numeric($postCode) || strlen($postCode) !== 4) {
            throw new InvalidPostCodeException('The postcode must be a 4-digit string number.');
        }

        $this->value = $postCode;
        $this->resolvedState = $this->resolveState($postCode, $state);
    }

    public function value(): string
    {
        return $this->value;
    }

    public function state(): string
    {
        return $this->resolvedState;
    }

    private function resolveState(string $postCode, ?string $state): string
    {
        // Domain logic to resolve state from postcode ranges
    }
}
```

### Value Object with Checksum Validation

```php
declare(strict_types=1);

namespace App\CompanyRegistry\Companies\Domain;

use App\CompanyRegistry\Companies\Domain\Exceptions\InvalidABNException;

final readonly class ABN
{
    private string $abn;

    /**
     * @throws InvalidABNException
     */
    public function __construct(string $abn)
    {
        $this->abn = str_replace(' ', '', $abn);

        if (! $this->isValidFormat()) {
            throw new InvalidABNException("Invalid ABN format {$this->abn}");
        }

        if (! $this->isValidChecksum()) {
            throw new InvalidABNException("Invalid ABN checksum {$this->abn}");
        }
    }

    public function __toString(): string
    {
        return $this->abn;
    }

    private function isValidFormat(): bool
    {
        return ctype_digit($this->abn) && strlen($this->abn) === 11;
    }

    private function isValidChecksum(): bool
    {
        // Algorithm-specific validation logic
    }
}
```

**Rules:**

- Always `final readonly class`
- Constructor validates invariants and throws a domain exception on failure
- Expose values via named accessors (`value()`, `state()`) — not public properties
- Implement `__toString()` when the VO has a natural string representation
- Validation logic is private — external code only sees the valid object or the exception
- Value objects are immutable — mutation methods return a **new instance**:

```php
public function increase(int $amount = 1): self
{
    return new self($this->value + $amount);
}
```

### Composite Value Object

When a value object composes multiple parts, use a private constructor with static factory methods:

```php
final readonly class ConsolidatedWorkId
{
    private function __construct(
        private string $industryId,
        private string $postcode,
        private ?string $area
    ) {}

    public static function fromComponents(string $industryId, string $postcode, ?string $area): self
    {
        return new self($industryId, $postcode, $area);
    }

    public static function parse(string $consolidatedId): self
    {
        // Parse string representation back to components
    }

    public function toString(): string
    {
        return "{$this->industryId}::{$this->postcode}::".($this->area ?? 'no-area');
    }
}
```

**Placement:** `{Module}/Domain/` or `{BoundedContext}/Shared/Domain/` when shared across modules within the same context.

---

## PHP Enums

All enums are **string-backed** and live in `{Module}/Domain/`.

### Basic Enum

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Domain;

enum ContactGender: string
{
    case MALE = 'male';
    case FEMALE = 'female';
}
```

### Enum with `label()` Method

```php
declare(strict_types=1);

namespace App\CompanyRegistry\Shared\Domain;

enum ContractType: string
{
    case FULL_TIME = 'full_time';
    case PART_TIME = 'part_time';
    case CASUAL = 'casual';
    case FIXED_TERM = 'fixed_term';

    public function label(): string
    {
        return match ($this) {
            self::FULL_TIME => 'Full-time Employment',
            self::PART_TIME => 'Part-time Employment',
            self::CASUAL => 'Casual Employment',
            self::FIXED_TERM => 'Fixed-term Contract',
        };
    }
}
```

### Enum with Static Factory and Semantic Helpers

```php
declare(strict_types=1);

namespace App\VisaManagement\Visas\Domain;

use App\VisaManagement\Visas\Domain\Exceptions\InvalidVisaSubclass;

enum Subclass: string
{
    case Student = '500';
    case WorkAndHoliday = '462';
    case WorkingHoliday = '417';

    /**
     * @throws InvalidVisaSubclass
     */
    public static function new(string $value): self
    {
        if (! $instance = self::tryFrom($value)) {
            throw new InvalidVisaSubclass("Invalid visa subclass $value");
        }

        return $instance;
    }

    /**
     * @return array<string>
     */
    public static function available(): array
    {
        return array_map(fn (self $case) => $case->value, self::cases());
    }

    public function isStudent(): bool
    {
        return $this === self::Student;
    }
}
```

**Rules:**

- Always `string`-backed — never `int`-backed or unit enums
- Case naming: prefer `SCREAMING_CASE` for constant-like values (`FULL_TIME`, `MALE`). `PascalCase` is acceptable for proper nouns or domain terms (`Student`, `WorkAndHoliday`)
- `label(): string` returns a human-readable display name using `match`
- `static new(string $value): self` wraps `tryFrom()` and throws a domain exception on invalid input — consistent with the entity `static new()` pattern
- `available(): array` returns all valid values as strings
- `is{CaseName}(): bool` helpers for semantic checks in domain logic
- Add methods as needed — enums are the right place for domain behavior tied to the set of values

---

## State Machines

State machines use `spatie/laravel-model-states`. The structure is: abstract base → concrete states → transition classes.

### Abstract State Base

```php
declare(strict_types=1);

namespace App\VisaManagement\Visas\Domain;

use Spatie\ModelStates\Exceptions\InvalidConfig;
use Spatie\ModelStates\State;
use Spatie\ModelStates\StateConfig;

abstract class Status extends State
{
    /**
     * @throws InvalidConfig
     */
    public static function config(): StateConfig
    {
        return parent::config()
            ->default(Granted::class)
            ->allowTransition(Granted::class, Active::class, ToActive::class)
            ->allowTransition(Granted::class, Expired::class, ToExpired::class)
            ->allowTransition(Active::class, Expired::class, ToExpired::class);
    }
}
```

**Rules:**

- Named `Status` (or a domain-specific name like `ApplicationStatus` if the module has multiple state machines)
- Extends `Spatie\ModelStates\State`
- `config()` defines the default state, all allowed transitions, and optionally the transition class for each
- Simple transitions (no side effects) omit the third parameter
- Complex transitions (with side effects) specify a `Transition` class

### Concrete State Classes

Each state is a minimal class with only a `$name` property:

```php
declare(strict_types=1);

namespace App\VisaManagement\Visas\Domain;

class Granted extends Status
{
    public static string $name = 'granted';
}
```

```php
class Active extends Status
{
    public static string $name = 'active';
}
```

```php
class Expired extends Status
{
    public static string $name = 'expired';
}
```

**Rules:**

- Extend the abstract state base — never `State` directly
- `$name` matches the database column value (lowercase, snake_case if multi-word)
- No business logic in state classes — they are markers only

### Transition Classes

Transitions encapsulate the logic that executes when a state change occurs:

```php
declare(strict_types=1);

namespace App\VisaManagement\Visas\Domain;

use Spatie\ModelStates\Transition;

class ToActive extends Transition
{
    public function __construct(private Visa $visa) {}

    public function handle(): Visa
    {
        $this->visa->status = new Active($this->visa);
        $this->visa->activated_at = now();
        $this->visa->save();

        return $this->visa;
    }
}
```

**Rules:**

- Named `To{TargetState}` or `{FromState}To{TargetState}` when disambiguation is needed
- Constructor receives the entity
- `handle()` sets the new state, performs side effects, saves, and returns the entity
- Transitions are the **one place** where calling `save()` is acceptable — the spatie framework requires it. Domain behavior methods on entities must NOT call `save()` (see `writing-domain-entities`)
- Throw domain exceptions for invalid transitions (see the `writing-domain-entities` skill for the entity-side `activate()` pattern that wraps `transitionTo()`)

### Entity Integration

Entities use the `HasStates` trait and cast the state column:

```php
use Spatie\ModelStates\HasStates;

class Visa extends Model
{
    use HasStates;

    protected function casts(): array
    {
        return [
            'status' => Status::class,
        ];
    }
}
```

The entity's domain behavior methods call `$this->status->transitionTo(TargetState::class)` — see the `writing-domain-entities` skill for that pattern.

### Directory Layout

```text
{Module}/Domain/
├── Status.php            # Abstract state base
├── Granted.php           # Concrete state
├── Active.php            # Concrete state
├── Expired.php           # Concrete state
├── ToActive.php          # Transition class
└── ToExpired.php         # Transition class
```

All state and transition classes live flat in `{Module}/Domain/` alongside the entity.

---

## Custom Validation Rule Objects

Validation rules wrap value object construction to bridge domain validation into Laravel's validation system:

```php
declare(strict_types=1);

namespace App\CompanyRegistry\Companies\Domain\Validators;

use App\CompanyRegistry\Companies\Domain\ABN;
use App\CompanyRegistry\Companies\Domain\Exceptions\InvalidABNException;
use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

class ABNRule implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        try {
            new ABN($value);
        } catch (InvalidABNException $e) {
            $fail('The :attribute must be a valid Australian Business Number (ABN).');
        }
    }
}
```

### Data-Aware Rule

When validation depends on other form fields, implement `DataAwareRule`:

```php
declare(strict_types=1);

namespace App\CompanyRegistry\Shared\Domain\Validators;

use App\CompanyRegistry\Shared\Domain\Exceptions\InvalidPostCodeException;
use App\CompanyRegistry\Shared\Domain\PostCode;
use Closure;
use Illuminate\Contracts\Validation\DataAwareRule;
use Illuminate\Contracts\Validation\ValidationRule;

class PostCodeRule implements DataAwareRule, ValidationRule
{
    protected array $data = [];

    public function setData(array $data): self
    {
        $this->data = $data;

        return $this;
    }

    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        try {
            new PostCode($value, $this->data['state'] ?? null);
        } catch (InvalidPostCodeException $e) {
            $fail($e->getMessage());
        }
    }
}
```

**Rules:**

- Delegate validation to the value object constructor — never duplicate validation logic
- Catch the domain exception and call `$fail()` with a user-friendly message
- Use `DataAwareRule` when the rule needs to reference other fields in the request
- Placement: `{Module}/Domain/Validators/` or `{BoundedContext}/Shared/Domain/Validators/` — always use `Validators/` as the subdirectory name
- Used in `ValidationRules` traits (see `writing-domain-entities`) and Form Requests (see `writing-form-requests`)

---

## Naming Conventions

| Artifact | Convention | Examples |
|---|---|---|
| Value object | Descriptive noun | `PostCode`, `ABN`, `ConsolidatedWorkId` |
| Enum | Descriptive noun | `ContractType`, `ContactGender`, `Subclass` |
| State base | `Status` or `{Context}Status` | `Status`, `ApplicationStatus` |
| Concrete state | Past participle or adjective | `Granted`, `Active`, `Expired`, `Draft` |
| Transition | `To{State}` or `{From}To{To}` | `ToActive`, `DraftToApplied` |
| Validation rule | `{Concept}Rule` | `PostCodeRule`, `ABNRule`, `Adult` |

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Using `int`-backed enums | Always use `string`-backed enums |
| Public properties on value objects | Expose via named accessor methods (`value()`, `state()`) |
| Mutating a value object | Return a new instance from mutation methods |
| Duplicating VO logic in validation rules | Delegate to the value object constructor |
| Concrete state extending `State` directly | Extend the module's abstract state base class |
| Business logic in concrete state classes | Keep states as markers; put logic in transition classes |
| State/transition classes in a subdirectory | Keep flat in `{Module}/Domain/` alongside the entity |
| Enum without `static new()` factory | Add `static new()` that wraps `tryFrom()` + throws domain exception |

---

## Pre-flight Checklist

- [ ] Value objects are `final readonly class`?
- [ ] Constructor validates invariants and throws domain exceptions?
- [ ] Enums are string-backed with `label()` for display?
- [ ] Enums have `static new()` factory wrapping `tryFrom()`?
- [ ] State base class defines `config()` with default + transitions?
- [ ] Concrete states only have `$name` property?
- [ ] Transition classes handle side effects in `handle()`?
- [ ] Entity uses `HasStates` trait and casts state column?
- [ ] Custom validation rules delegate to value object constructors?
- [ ] All artifacts placed in `{Module}/Domain/`?

---

## Laravel Reference

This skill uses Laravel-specific features. Consult the official Laravel documentation (via Context7 or web search) for details on:

- `Illuminate\Contracts\Validation\ValidationRule` — custom validation rule interface
- `Illuminate\Contracts\Validation\DataAwareRule` — validation rules that access other form fields
- `Rule::enum()` — built-in enum validation for request validation
- `spatie/laravel-model-states` — state machine package for Eloquent models (see package docs)

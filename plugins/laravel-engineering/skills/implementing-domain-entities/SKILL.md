---
name: implementing-domain-entities
description: Use when creating aggregate root entities, child entities, read projections, or adding domain behavior methods to Eloquent models in a DDD bounded context.
---

## Overview

Every aggregate root is an Eloquent model enriched with domain behavior. Entities live in `{Module}/Domain/`, use a static `new()` constructor that registers a creation event, and expose mutation methods that register domain events before returning `$this`.

---

## Aggregate Root Template

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Domain;

use App\CustomerRelationshipManagement\Contacts\Domain\Events\ContactCreated;
use App\CustomerRelationshipManagement\Contacts\Domain\Events\ContactDeleted;
use App\Shared\Domain\HasDomainEvents;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

/**
 * @property string $id
 * @property string $first_name
 * @property string $last_name
 * @property string $email
 * @property string $phone
 *
 * @method static ContactFactory factory()
 */
class Contact extends Model
{
    use HasDomainEvents;
    use HasFactory;
    use HasUuids;

    protected $table = 'crm_contacts';

    protected $fillable = [
        'first_name',
        'last_name',
        'email',
        'phone',
    ];

    public static function new(array $attributes = []): self
    {
        $contact = new self($attributes);
        $contact->registerDomainEvent(new ContactCreated($contact));

        return $contact;
    }

    public function toBeDeleted(): self
    {
        $this->registerDomainEvent(new ContactDeleted($this->id));

        return $this;
    }

    protected function casts(): array
    {
        return [
            'gender' => ContactGender::class,
        ];
    }

    protected static function newFactory(): ContactFactory
    {
        return new ContactFactory();
    }
}
```

Key conventions visible in this template:

- **`@property` docblock** lists all attributes with types — enables IDE completion
- **`@method static` docblock** declares the factory return type
- **`$table`** uses the bounded context prefix (`crm_`)
- **`casts()`** is a method (not `$casts` property) — current Laravel convention
- **`newFactory()`** points to the co-located factory class explicitly

---

## Static Constructor `new()`

Every aggregate root provides a static `new()` that replaces direct `new Model()` calls:

```php
public static function new(array $attributes = []): self
{
    $entity = new self($attributes);
    $entity->registerDomainEvent(new EntityCreated($entity));

    return $entity;
}
```

**Rules:**

- Always the first public method in the class body (after `newFactory()`)
- Always registers the `{Entity}Created` domain event
- Returns `self` — never void
- Seed/lookup entities (e.g., `Industry`) may omit the event registration

**Variants:**

```php
// Guard in constructor — throws before creating
public static function new(array $attributes, Collection $requiredRelation): self
{
    if ($requiredRelation->isEmpty()) {
        throw new NoMatchingItemException('At least one item is required.');
    }

    $entity = new self($attributes);
    $entity->registerDomainEvent(new EntityCreated($entity));

    return $entity;
}

// Second named constructor for a special creation path
public static function newAsServiceUser(array $attributes = []): self
{
    $attributes['email_verified_at'] = now();
    $attributes['password'] = bcrypt(Str::random(32));

    $user = new self($attributes);
    $user->registerDomainEvent(new UserCreated($user));

    return $user;
}
```

---

## Deletion Method

Every aggregate root that raises events provides `toBeDeleted()`:

```php
public function toBeDeleted(): self
{
    $this->registerDomainEvent(new EntityDeleted($this->id));

    return $this;
}
```

**Note:** The event payload is the entity's `$id` (string), not the entity object — because the entity is about to be deleted. This differs from creation events which pass the full entity.

---

## Domain Behavior Methods

Mutation methods follow a consistent pattern — `forceFill()` + register event + return `$this`:

```php
public function updateName(string $first, string $last): self
{
    $this->forceFill([
        'first_name' => $first,
        'last_name' => $last,
    ]);

    $this->registerDomainEvent(new ContactNameUpdated($this));

    return $this;
}
```

**Rules:**

- Named as `update{Attribute}()` or `{verb}()` (e.g., `verify()`, `activate()`, `expire()`)
- Use `forceFill()` for attribute mutation — avoids mass-assignment guard issues on internal operations
- Always register a domain event describing what changed
- Always return `$this` for fluent chaining
- Never call `save()` — persistence is the use case's responsibility

**State transitions** (when using `spatie/laravel-model-states`):

```php
public function activate(): self
{
    try {
        $this->status->transitionTo(Active::class);
    } catch (CouldNotPerformTransition $e) {
        throw new VisaException($e->getMessage());
    }

    $this->registerDomainEvent(new VisaActivated($this));

    return $this;
}
```

Wrap Spatie's `transitionTo()` in a try/catch and convert `CouldNotPerformTransition` into a domain exception. See the `implementing-value-objects` skill for state machine setup.

**Computed attributes** — use Laravel's `Attribute` class for derived values:

```php
protected function name(): Attribute
{
    return Attribute::get(fn () => trim("{$this->first_name} {$this->last_name}"));
}
```

**Lifecycle hooks** — use `booted()` to set default values on model events:

```php
protected static function booted(): void
{
    static::creating(function (Visa $visa) {
        if ($visa->subclass !== Subclass::Student) {
            $visa->expire_at = now()->addYear();
        }
    });
}
```

Consult the Laravel documentation for full details on `Attribute` accessors and model lifecycle events (`boot()` / `booted()`).

---

## Child Entities

Child entities live inside the parent module's `Domain/` directory — never in their own module:

```text
Companies/Domain/Company.php            # Aggregate root
Companies/Domain/CompanyContact.php     # Child entity
```

Child entity characteristics:

- Use `HasFactory` + `HasUuids` (usually)
- Do **not** use `HasDomainEvents` — the parent aggregate manages events
- Do **not** have `static new()` — they are created through the aggregate root's methods
- May have their own named constructor (e.g., `Invitation::forUser(string $email, string $name, string $role): self`)

**Aggregate root manages child lifecycle:**

```php
// In Company.php (aggregate root)
public function addContact(array $attributes): self
{
    if ($this->contacts()->count() >= 10) {
        throw new CompanyException('Maximum 10 contacts allowed.');
    }

    $this->contacts()->create($attributes);

    return $this;
}
```

---

## Read Projections

A read projection is an Eloquent model that queries another aggregate's table through a different domain lens. It has no write behavior. Use this when a bounded context needs to read data owned by another context without importing its domain classes.

```php
declare(strict_types=1);

namespace App\{ConsumerContext}\{Module}\Domain;

use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;

class {ProjectionName} extends Model
{
    use HasUuids;

    protected $table = '{owner_prefix}_{table}';   // Another aggregate's table

    protected $hidden = [/* fields this context should not expose */];

    protected static function booted(): void
    {
        static::addGlobalScope('{scope-name}', function ($query) {
            $query->where(/* filter to relevant rows */);
        });
    }
}
```

**Characteristics:**

- `$table` points to another aggregate's table (e.g., an `Applicant` projection in VisaManagement reading `crm_contacts`)
- Restricted `$hidden` set — exposes only what this context needs
- Optional global scope in `booted()` to filter to relevant rows
- No `HasDomainEvents`, no `HasFactory`, no `static new()`
- No mutation methods — read-only by design

**Immutable projection** — when writes must be physically blocked:

```php
protected static function booted(): void
{
    static::addGlobalScope('read-only', fn ($q) => $q->where('type', 'system'));
    static::saving(fn () => false);
    static::creating(fn () => false);
    static::updating(fn () => false);
    static::deleting(fn () => false);
}
```

---

## ValidationRules Trait

Domain validation rules are defined as a trait in `{Module}/Domain/` and consumed by Infrastructure Request classes:

```php
// Domain layer — defines the rules
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Domain;

use Illuminate\Validation\Rule;

trait ValidationRules
{
    public function rules(?string $contact = null): array
    {
        $unique = $contact
            ? Rule::unique(Contact::class)->ignore($contact)
            : Rule::unique(Contact::class);

        return [
            'first_name' => ['required', 'string', 'max:64'],
            'last_name'  => ['required', 'string', 'max:64'],
            'email'      => ['required', 'string', 'email', 'max:255', $unique],
            'gender'     => [Rule::enum(ContactGender::class)],
        ];
    }
}
```

**Rules:**

- Nullable entity ID parameter (`?string $contact = null`) toggles the unique rule for create vs. update
- Use `Rule::enum()` to validate backed enums
- Advanced traits split into multiple methods: `rules()` for create, `updateRules()` for update
- Naming: `ValidationRules` when one per module, `{Entity}ValidationRules` when multiple exist in the same context
- Custom rule objects (e.g., `new PostCodeRule`) wrap value object validation — see `implementing-value-objects`

The Infrastructure layer consumes the trait in Form Request classes — see `implementing-form-requests`.

---

## Trait Reference

Aggregate roots compose behavior through traits from the Shared Kernel. Use as needed — don't apply all to every entity:

| Trait | Source | Purpose | Details |
|---|---|---|---|
| `HasDomainEvents` | `Shared/Domain/` | Event registration + publishing | See `implementing-domain-events` |
| `HasFactory` | Eloquent | Test factory support | See `implementing-factories` |
| `HasUuids` | Eloquent | UUID primary keys | Standard on all entities |
| `HasStates` | `spatie/laravel-model-states` | State machine support | See `implementing-value-objects` |
| `HasProfilePhoto` | `Shared/Domain/` | Avatar management | Jetstream stack |
| `Notifiable` | Eloquent | Mail/notification channels | Consult Laravel docs |
| `IsExtensible` | `Shared/Domain/` | Custom attribute support | Marker trait |
| `CanGenerateIdentifiers` | `Shared/Application/` | UUID generation in use cases | Application layer, not entities |

**Note:** `CanGenerateIdentifiers` lives in the Application layer — it is used by use case classes, not by entities directly.

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Calling `save()` inside a domain method | Domain methods mutate state; persistence is the use case's job |
| Passing entity object in deletion event | Pass `$this->id` (string) — the entity is about to be deleted |
| Child entity in its own module | Keep inside parent module's `Domain/` directory |
| Read projection with `HasDomainEvents` | Read projections are read-only — no events, no `static new()` |
| Using `$casts` property instead of method | Use `protected function casts(): array` (method form) |
| Missing `@property` docblock | Always declare all attributes with types for IDE support |
| `forceFill()` without returning `$this` | All behavior methods return `$this` for fluent chaining |
| Adding `CanGenerateIdentifiers` to entity | It's an Application-layer trait for use cases, not for models |

---

## Pre-flight Checklist

- [ ] Entity extends `Model` (or `Authenticatable` for User)?
- [ ] Uses `HasDomainEvents` + `HasFactory` + `HasUuids`?
- [ ] `$table` uses bounded context prefix?
- [ ] `static new()` creates entity AND registers `Created` event?
- [ ] `toBeDeleted()` registers `Deleted` event with entity ID?
- [ ] Domain behavior methods use `forceFill()` + register event + return `$this`?
- [ ] `@property` docblock lists all attributes?
- [ ] `casts()` is a method, not a property?
- [ ] `newFactory()` points to named factory class?
- [ ] Child entities have no `HasDomainEvents` and no `static new()`?
- [ ] Read projections have `$table` override, global scope, and no write behavior?
- [ ] `ValidationRules` trait defined if entity has validation needs?

---

## Laravel Reference

This skill uses Laravel-specific features. Consult the official Laravel documentation (via Context7 or web search) for details on:

- `Illuminate\Database\Eloquent\Model` — base Eloquent model, mass assignment, `$fillable` / `$guarded`
- `HasFactory`, `HasUuids` — Eloquent traits for factories and UUID primary keys
- `Notifiable` — notification channel support (mail, SMS, Slack)
- `boot()` / `booted()` — model lifecycle hooks and event callbacks
- `forceFill()` — bypass mass assignment guards for internal mutations
- `Attribute` — computed/virtual accessors and mutators
- `Rule::unique()`, `Rule::enum()` — validation rule builders

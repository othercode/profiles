---
name: implementing-factories
description: Use when creating Eloquent model factories for testing or seeding, setting up the EloquentFactory base class, or ensuring domain events fire during factory creation.
---

## Overview

Factories extend a custom `EloquentFactory` base class that calls `Model::new()` instead of `new Model()`. This ensures domain events are registered even during test and seed data creation. Factories are co-located with their entities in the Domain layer.

---

## EloquentFactory Base Class

```php
declare(strict_types=1);

namespace App\Shared\Infrastructure\Persistence;

use Illuminate\Database\Eloquent\Factories\Factory;

abstract class EloquentFactory extends Factory
{
    protected function newModel(array $attributes = [])
    {
        return $this->modelName()::new($attributes);
    }
}
```

Overrides `newModel()` to route through the entity's static `::new()` constructor, which registers the `Created` domain event. Without this, factory-created entities would skip event registration.

**Placement:** `app/Shared/Infrastructure/Persistence/EloquentFactory.php`

---

## Factory Implementation

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Domain\Factories;

use App\CustomerRelationshipManagement\Contacts\Domain\Contact;
use App\Shared\Infrastructure\Persistence\EloquentFactory;

class ContactFactory extends EloquentFactory
{
    protected $model = Contact::class;

    public function definition(): array
    {
        return [
            'first_name' => $this->faker->firstName(),
            'last_name' => $this->faker->lastName(),
            'email' => $this->faker->unique()->safeEmail(),
            'phone' => $this->faker->phoneNumber(),
        ];
    }
}
```

### With State Methods

```php
class VisaFactory extends EloquentFactory
{
    protected $model = Visa::class;

    public function definition(): array
    {
        return [
            'applicant_id' => Contact::factory(),
            'subclass' => VisaSubclass::StudentVisa,
            'submitted_at' => now(),
        ];
    }

    public function granted(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => 'granted',
            'granted_at' => now(),
        ]);
    }

    public function expired(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => 'expired',
            'expired_at' => now()->subMonth(),
        ]);
    }
}
```

---

## Entity Integration

Entities override `newFactory()` to point to the co-located factory:

```php
class Contact extends Model
{
    use HasDomainEvents;
    use HasFactory;

    protected static function newFactory(): ContactFactory
    {
        return ContactFactory::new();
    }
}
```

This allows `Contact::factory()` to resolve the domain-level factory instead of Laravel's default `database/factories/` location.

---

## Placement

Two conventions exist across codebases:

| Convention | Path | Used By |
|---|---|---|
| Domain co-location | `{Module}/Domain/Factories/{Entity}Factory.php` | AnotherYear, IcedLatte |
| Infrastructure | `{Module}/Infrastructure/Persistence/{Entity}Factory.php` | Laravel DDD base |

Prefer Domain co-location — factories describe how to build valid domain entities, making them a Domain concern.

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Extending `Illuminate\...\Factory` directly | Extend `EloquentFactory` to ensure `::new()` is called |
| Factory in `database/factories/` | Co-locate with entity in `Domain/Factories/` |
| Missing `newFactory()` override on entity | Required for `Entity::factory()` to find the co-located factory |
| Using `new Model()` in factory | `EloquentFactory` handles this — the base class calls `::new()` |

---

## Pre-flight Checklist

- [ ] Factory extends `EloquentFactory` (not `Illuminate\...\Factory`)?
- [ ] `$model` property set to the entity class?
- [ ] `definition()` returns valid attribute array?
- [ ] Entity overrides `newFactory()` to return the co-located factory?
- [ ] Factory placed in `{Module}/Domain/Factories/`?
- [ ] State methods use `$this->state()` for scenario variations?

---

## Laravel Reference

This skill uses Laravel-specific features. Consult the official Laravel documentation (via Context7 or web search) for details on:

- `Illuminate\Database\Eloquent\Factories\Factory` — base factory class
- `Illuminate\Database\Eloquent\Factories\HasFactory` — trait enabling `Model::factory()`
- `definition()` — defining default model attributes
- `state()` — defining factory state variations
- `afterCreating()` — post-creation hooks for relationship setup
- `Faker\Generator` — fake data generation via `$this->faker`

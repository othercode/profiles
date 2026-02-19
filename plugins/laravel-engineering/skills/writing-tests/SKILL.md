---
name: writing-tests
description: Use when writing tests, creating test files, naming test functions, choosing between Feature and Unit tests, using factories, mocking dependencies, or asserting HTTP responses in a Laravel DDD project.
---

## The Rule

**Every endpoint gets a Feature test. Every use case gets a Unit test.** No code ships without tests that prove it works. Tests are not optional, not "nice to have", not "we'll add them later." They are part of the deliverable.

---

## Framework & Tooling

| Tool | Version | Purpose |
|---|---|---|
| Pest PHP | ^3.0 | Primary test framework (wrapper over PHPUnit) |
| pest-plugin-laravel | — | Laravel-specific assertions and helpers |
| pest-plugin-faker | — | Faker integration via `$this->faker` |
| Mockery | ^1.6 | Mock objects and spies |
| Larastan | ^2.9 | Static analysis (PHPStan level 5) |
| Laravel Pint | ^1.13 | Code formatting (PSR-12) |

**Run tests:** `vendor/bin/pest`
**Run static analysis:** `vendor/bin/phpstan analyse`
**Run formatter:** `vendor/bin/pint`

---

## Test Directory Structure

```text
tests/
├── Pest.php                              # Pest configuration
├── TestCase.php                          # Base test class
├── Feature/
│   ├── {BoundedContext}/
│   │   └── {Module}/
│   │       ├── Create{Entity}Test.php
│   │       ├── Update{Entity}Test.php
│   │       ├── Delete{Entity}Test.php
│   │       ├── List{Entity}Test.php
│   │       └── Search{Entity}Test.php
└── Unit/
    ├── {BoundedContext}/
    │   └── {Module}/
    │       ├── Create{Entity}Test.php
    │       ├── Find{Entity}Test.php
    │       ├── Update{Entity}Test.php
    │       ├── Delete{Entity}Test.php
    │       └── {Entity}Test.php
```

Tests mirror the domain structure — organized by bounded context and module, not by technical concern.

---

## Feature vs Unit — Where Does This Test Go?

| What You're Testing | Type | Directory |
|---|---|---|
| HTTP endpoint (controller + request + response) | Feature | `tests/Feature/` |
| Full request cycle with database | Feature | `tests/Feature/` |
| Authentication and authorization flow | Feature | `tests/Feature/` |
| Inertia page rendering | Feature | `tests/Feature/` |
| API JSON response structure | Feature | `tests/Feature/` |
| Use case logic (isolated) | Unit | `tests/Unit/` |
| Domain entity behavior methods | Unit | `tests/Unit/` |
| Value object validation | Unit | `tests/Unit/` |
| State transitions | Unit | `tests/Unit/` |
| Repository query logic | Unit | `tests/Unit/` |
| Domain event registration | Unit | `tests/Unit/` |

**The line is simple:** If it touches HTTP or the database, it's Feature. If it tests isolated logic, it's Unit.

---

## Test Naming

### File Naming

One test file per subject. File name matches the action being tested:

| Artifact | Test File |
|---|---|
| `CreateContactController` | `CreateContactTest.php` |
| `DeleteContact` (use case) | `DeleteContactTest.php` |
| `Contact` (entity methods) | `ContactTest.php` |
| `EloquentContactRepository` | `ContactRepositoryTest.php` |

### Method Naming

Use Pest's `it()` syntax or `test_` prefix. Methods describe **what should happen**:

```php
// Pest it() syntax
it('creates a contact with valid data', function () { });
it('throws ContactNotFound when id does not exist', function () { });
it('returns 422 when email is missing', function () { });
it('deletes the contact and redirects', function () { });
it('does not delete when user is not owner', function () { });

// Alternative: test_ prefix
test('it creates a contact with valid data', function () { });
```

**Rules:**

- Describe the expected behavior, not the implementation
- Include the failure scenario in the name when testing negative cases
- Stay consistent within a project — pick one style and use it everywhere

---

## Feature Test Patterns

### Web Endpoint (Inertia)

```php
use App\CustomerRelationshipManagement\Contacts\Domain\Contact;
use App\IdentityAndAccess\Users\Domain\User;

use function Pest\Laravel\actingAs;

it('displays the contact list page', function () {
    $user = User::factory()->create();
    Contact::factory()->count(3)->create();

    actingAs($user)
        ->get('/contacts')
        ->assertOk()
        ->assertInertia(fn ($page) => $page
            ->component('Contacts/Index')
            ->has('contacts', 3)
        );
});
```

### Web Endpoint (Create + Redirect)

```php
it('creates a contact and redirects to show page', function () {
    $user = User::factory()->create();

    actingAs($user)
        ->post('/contacts', [
            'first_name' => 'John',
            'last_name' => 'Doe',
            'email' => 'john@example.com',
        ])
        ->assertRedirect();

    $this->assertDatabaseHas('crm_contacts', [
        'email' => 'john@example.com',
    ]);
});
```

### Web Endpoint (Validation Failure)

```php
it('returns validation errors when email is missing', function () {
    $user = User::factory()->create();

    actingAs($user)
        ->post('/contacts', [
            'first_name' => 'John',
        ])
        ->assertSessionHasErrors(['email']);
});
```

### Web Endpoint (Delete)

```php
it('deletes the contact and redirects', function () {
    $user = User::factory()->create();
    $contact = Contact::factory()->create();

    actingAs($user)
        ->delete("/contacts/{$contact->id}")
        ->assertRedirect('/contacts');

    $this->assertDatabaseMissing('crm_contacts', [
        'id' => $contact->id,
    ]);
});
```

### Web Endpoint (Unauthorized)

```php
it('redirects unauthenticated users to login', function () {
    $this->get('/contacts')
        ->assertRedirect('/login');
});
```

### API Endpoint (JSON)

```php
use function Pest\Laravel\actingAs;

it('returns contacts matching search criteria', function () {
    $user = User::factory()->create();
    Contact::factory()->count(5)->create();

    actingAs($user)
        ->postJson('/api/contacts/search', [
            'filters' => [
                ['field' => 'email', 'operator' => 'CONTAINS', 'value' => 'example'],
            ],
            'perPage' => 10,
        ])
        ->assertOk()
        ->assertJsonStructure([
            'data' => [['id', 'first_name', 'last_name', 'email']],
        ]);
});
```

---

## Unit Test Patterns

### Use Case — Create

```php
use App\CustomerRelationshipManagement\Contacts\Application\CreateContact;
use App\CustomerRelationshipManagement\Contacts\Domain\Contact;
use App\CustomerRelationshipManagement\Contacts\Domain\Contracts\ContactRepository;
use App\Shared\Domain\Contracts\ServiceBus\EventBus;

it('creates and persists a contact', function () {
    $repository = Mockery::mock(ContactRepository::class);
    $eventBus = Mockery::mock(EventBus::class);

    $repository->shouldReceive('save')
        ->once()
        ->andReturnUsing(fn (Contact $c) => $c);

    $eventBus->shouldReceive('publish')->once();

    $creator = new CreateContact($repository, $eventBus);
    $contact = $creator->create([
        'first_name' => 'John',
        'last_name' => 'Doe',
        'email' => 'john@example.com',
    ]);

    expect($contact)->toBeInstanceOf(Contact::class);
});
```

### Use Case — Find (Success)

```php
use App\CustomerRelationshipManagement\Contacts\Application\FindContact;
use App\CustomerRelationshipManagement\Contacts\Domain\Contact;
use App\CustomerRelationshipManagement\Contacts\Domain\Contracts\ContactRepository;

it('returns the contact when found', function () {
    $contact = Contact::factory()->make(['id' => 'uuid-123']);
    $repository = Mockery::mock(ContactRepository::class);

    $repository->shouldReceive('find')
        ->with('uuid-123')
        ->andReturn($contact);

    $finder = new FindContact($repository);

    expect($finder->byId('uuid-123'))->toBe($contact);
});
```

### Use Case — Find (Not Found)

```php
use App\CustomerRelationshipManagement\Contacts\Application\FindContact;
use App\CustomerRelationshipManagement\Contacts\Domain\Contracts\ContactRepository;
use App\CustomerRelationshipManagement\Contacts\Domain\Exceptions\ContactNotFound;

it('throws ContactNotFound when contact does not exist', function () {
    $repository = Mockery::mock(ContactRepository::class);

    $repository->shouldReceive('find')
        ->with('missing-id')
        ->andReturnNull();

    $finder = new FindContact($repository);
    $finder->byId('missing-id');
})->throws(ContactNotFound::class);
```

### Use Case — Delete (Idempotent)

```php
it('silently returns when contact does not exist', function () {
    $repository = Mockery::mock(ContactRepository::class);
    $eventBus = Mockery::mock(EventBus::class);

    $repository->shouldReceive('find')
        ->with('missing-id')
        ->andReturnNull();

    $repository->shouldNotReceive('delete');
    $eventBus->shouldNotReceive('publish');

    $deleter = new DeleteContact($repository, $eventBus);
    $deleter->delete('missing-id');
});
```

### Entity — Domain Behavior

```php
use App\CustomerRelationshipManagement\Contacts\Domain\Contact;
use App\CustomerRelationshipManagement\Contacts\Domain\Events\ContactNameUpdated;
use App\Shared\Domain\Contracts\ServiceBus\EventBus;

it('registers a name updated event when name changes', function () {
    $contact = Contact::factory()->make([
        'first_name' => 'John',
        'last_name' => 'Doe',
    ]);

    $contact->updateName('Jane', 'Smith');

    $eventBus = Mockery::mock(EventBus::class);
    $eventBus->shouldReceive('publish')
        ->once()
        ->withArgs(fn ($event) => $event instanceof ContactNameUpdated);

    $contact->publishDomainEvents($eventBus);
});
```

### Value Object — Validation

```php
use App\Shared\Domain\ValueObjects\ABN;
use App\Shared\Domain\Exceptions\InvalidABNException;

it('creates a valid ABN', function () {
    $abn = new ABN('51 824 753 556');

    expect($abn->value())->toBe('51824753556');
});

it('throws on invalid ABN format', function () {
    new ABN('invalid');
})->throws(InvalidABNException::class);
```

---

## Database Handling

### RefreshDatabase (Default)

```php
// tests/Pest.php
uses(Illuminate\Foundation\Testing\RefreshDatabase::class)->in('Feature');
```

Applied globally to all Feature tests via `Pest.php`. Runs migrations before each test and rolls back after.

### In-Memory SQLite (Fast)

```xml
<!-- phpunit.xml -->
<env name="DB_CONNECTION" value="sqlite"/>
<env name="DB_DATABASE" value=":memory:"/>
```

---

## Mocking Strategy

### Prefer Laravel Fakes Over Mockery

For framework services, use Laravel's built-in fakes:

```php
// Events
Event::fake();
// ... execute code ...
Event::assertDispatched(ContactCreated::class);
Event::assertNotDispatched(ContactDeleted::class);

// Mail
Mail::fake();
// ... execute code ...
Mail::assertSent(UserInvitationMail::class, fn ($mail) =>
    $mail->hasTo('user@example.com')
);

// Queue
Queue::fake();
// ... execute code ...
Queue::assertPushed(SendEmailVerification::class);

// Storage
Storage::fake('documents');
// ... execute code ...
Storage::disk('documents')->assertExists('file.pdf');
```

### Use Mockery For Domain Contracts

For repository and event bus contracts in Unit tests:

```php
$repository = Mockery::mock(ContactRepository::class);
$repository->shouldReceive('find')->with('id')->andReturn($contact);
$repository->shouldReceive('save')->once()->andReturnUsing(fn ($c) => $c);

$eventBus = Mockery::mock(EventBus::class);
$eventBus->shouldReceive('publish')->once();
```

### Never Mock What You Don't Own

- Mock **contracts** (interfaces) — `ContactRepository`, `EventBus`
- Never mock **concrete classes** (Eloquent models, Laravel internals)
- Never mock the **class under test**

---

## Factory Usage in Tests

### Creating Entities

```php
// Single entity
$contact = Contact::factory()->create();

// With specific attributes
$contact = Contact::factory()->create([
    'email' => 'specific@example.com',
]);

// Multiple entities
$contacts = Contact::factory()->count(5)->create();

// With state
$visa = Visa::factory()->granted()->create();

// In-memory only (no database)
$contact = Contact::factory()->make();
```

### Relationship Factories

```php
// Entity with relationship
$contact = Contact::factory()
    ->for(Account::factory())
    ->create();

// Entity with multiple related entities
$account = Account::factory()
    ->has(Contact::factory()->count(3))
    ->create();
```

Factories extend `EloquentFactory` which calls `Model::new()` — domain events are registered automatically. See `implementing-factories` for the full factory setup.

---

## Pest Configuration

### tests/Pest.php

```php
<?php

use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

uses(TestCase::class)->in('Feature', 'Unit');
uses(RefreshDatabase::class)->in('Feature');
```

### Pest Expectations (Prefer Over PHPUnit Assertions)

```php
// Pest expect() syntax — preferred
expect($contact)->toBeInstanceOf(Contact::class);
expect($contact->email)->toBe('john@example.com');
expect($contacts)->toHaveCount(3);
expect($result)->toBeNull();
expect($result)->not->toBeNull();
expect(fn () => $finder->byId('x'))->toThrow(ContactNotFound::class);

// PHPUnit assertions — acceptable but less idiomatic in Pest
$this->assertInstanceOf(Contact::class, $contact);
$this->assertEquals('john@example.com', $contact->email);
$this->assertDatabaseHas('crm_contacts', ['id' => $contact->id]);
```

Use `expect()` for value assertions. Use `$this->assert*` for database and HTTP assertions (which don't have Pest equivalents).

---

## What to Test — Minimum Coverage

### For Every Controller (Feature Tests)

- [ ] Successful action (happy path)
- [ ] Validation failure (missing/invalid input)
- [ ] Unauthenticated access (redirects to login)
- [ ] Unauthorized access (forbidden when not owner)
- [ ] Not-found response (invalid ID)

### For Every Use Case (Unit Tests)

- [ ] Successful execution (happy path)
- [ ] Not-found exception (for find/update/delete)
- [ ] Validation failure (invalid input)
- [ ] Domain events published after persistence
- [ ] Idempotent deletion (no exception on missing entity)

### For Every Entity (Unit Tests)

- [ ] Domain behavior methods register correct events
- [ ] `static new()` registers `Created` event
- [ ] `toBeDeleted()` registers `Deleted` event
- [ ] State transitions follow allowed paths
- [ ] Invalid transitions throw domain exceptions

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| No tests for an endpoint | Every endpoint gets a Feature test — no exceptions |
| Testing implementation details instead of behavior | Test WHAT happens, not HOW it happens internally |
| Using `Mockery::mock(ConcreteClass::class)` | Mock interfaces (contracts), not concrete classes |
| Missing validation failure tests | Always test that invalid input returns errors |
| Missing authentication test | Always test unauthenticated access redirects |
| Sharing state between tests | Each test creates its own data — no shared fixtures |
| Testing private methods directly | Test through public API; private methods are implementation |
| Skipping domain event assertions in unit tests | Always verify events are registered after domain operations |
| Using `$this->withoutExceptionHandling()` everywhere | Only use when debugging — remove before committing |

---

## Red Flags — STOP and Fix

- A controller with no corresponding Feature test file
- A use case with no corresponding Unit test file
- A test that passes without any assertions
- A test name that doesn't describe the expected behavior
- Mockery used on a concrete Eloquent model
- `RefreshDatabase` used in a Unit test (it shouldn't touch the database)
- Tests that depend on execution order

---

## Pre-flight Checklist

- [ ] Every controller has a Feature test?
- [ ] Every use case has a Unit test?
- [ ] Tests use Pest syntax (`it()` or `test()`)?
- [ ] Feature tests use `actingAs()` for authentication?
- [ ] Unit tests mock contracts (interfaces), not concrete classes?
- [ ] Database assertions use `assertDatabaseHas` / `assertDatabaseMissing`?
- [ ] Domain event assertions verify correct events are registered?
- [ ] Validation failure tests check `assertSessionHasErrors` or 422 response?
- [ ] Test files mirror the bounded context directory structure?
- [ ] `RefreshDatabase` applied to Feature tests only (via `Pest.php`)?

---

## Laravel Reference

This skill uses Laravel-specific features. Consult the official Laravel documentation (via Context7 or web search) for details on:

- `Pest PHP` — testing framework (pestphp.com)
- `Illuminate\Foundation\Testing\RefreshDatabase` — database reset per test
- `actingAs()` — authenticate a user for testing
- `assertInertia()` — Inertia.js response assertions
- `assertJson()`, `assertJsonStructure()` — JSON API response assertions
- `assertDatabaseHas()`, `assertDatabaseMissing()` — database state assertions
- `assertSessionHasErrors()` — validation error assertions
- `Event::fake()`, `Mail::fake()`, `Queue::fake()` — Laravel service fakes
- `Mockery` — mock object framework (mockery.io)

---
name: implementing-domain-events
description: Use when defining domain events, setting up the event bus contract, adding event support to entities, implementing the event bus adapter, or understanding the event publishing flow.
---

## Overview

Domain events decouple side effects from domain logic. Entities **register** events during domain methods; use cases **publish** them after persistence. The Domain layer depends only on contracts (`Event`, `EventBus`); the Infrastructure layer provides the Laravel adapter.

---

## Shared Kernel Contracts

### Event Marker Interface

```php
declare(strict_types=1);

namespace App\Shared\Domain\Contracts\ServiceBus;

interface Event {}
```

A pure marker interface — no methods. All domain events implement this contract.

### EventBus Contract

```php
declare(strict_types=1);

namespace App\Shared\Domain\Contracts\ServiceBus;

interface EventBus
{
    public function publish(Event ...$events): void;
}
```

Single method accepting variadic `Event` arguments. The Domain layer depends on this contract only.

**Placement:** `app/Shared/Domain/Contracts/ServiceBus/`

---

## HasDomainEvents Trait

The trait that gives entities event registration and publishing capabilities:

```php
declare(strict_types=1);

namespace App\Shared\Domain;

use App\Shared\Domain\Contracts\ServiceBus\Event;
use App\Shared\Domain\Contracts\ServiceBus\EventBus;

trait HasDomainEvents
{
    /**
     * @var Event[]
     */
    private array $domainEvents = [];

    final public function publishDomainEvents(EventBus $eventBus): void
    {
        $eventBus->publish(...$this->domainEvents);
        $this->domainEvents = [];
    }

    final protected function registerDomainEvent(Event $domainEvent): void
    {
        $this->domainEvents[] = $domainEvent;
    }
}
```

**Key design decisions:**

- `registerDomainEvent()` is `final protected` — only the entity itself can register events
- `publishDomainEvents()` is `final public` — use cases call this after persistence
- Events are **cleared after publishing** — prevents double-publishing
- Uses variadic unpacking: `$eventBus->publish(...$this->domainEvents)`

**Placement:** `app/Shared/Domain/HasDomainEvents.php`

---

## Domain Event Classes

### Event with Entity Payload

The standard pattern — event carries the full entity for listeners to access:

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Domain\Events;

use App\CustomerRelationshipManagement\Contacts\Domain\Contact;
use App\Shared\Domain\Contracts\ServiceBus\Event;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Queue\SerializesModels;

final class ContactCreated implements Event
{
    use InteractsWithSockets;
    use SerializesModels;

    public function __construct(public Contact $contact) {}
}
```

### Event with Scalar ID Payload

Used for deletion events — the entity is about to be destroyed, so pass only the ID:

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Domain\Events;

use App\Shared\Domain\Contracts\ServiceBus\Event;
use Illuminate\Broadcasting\InteractsWithSockets;

final class ContactDeleted implements Event
{
    use InteractsWithSockets;

    /**
     * @param string $contact The deleted contact uuid
     */
    public function __construct(public string $contact) {}
}
```

**Rules:**

- Always `final class` — events are immutable facts
- Always `implements Event` (the Shared Kernel marker interface)
- Constructor uses **promoted public properties** for the payload
- Entity-carrying events use `SerializesModels` — enables queue serialization of Eloquent models
- Deletion events use `InteractsWithSockets` only (no `SerializesModels` — there's no model to serialize)
- Naming: `{Entity}{PastTenseVerb}` — e.g., `ContactCreated`, `VisaActivated`, `UserDeleted`, `DocumentUploaded`
- Placement: `{Module}/Domain/Events/`

### Payload Guidelines

| Event Type | Payload | Traits |
|---|---|---|
| Created | Full entity | `InteractsWithSockets` + `SerializesModels` |
| Updated / State change | Full entity | `InteractsWithSockets` + `SerializesModels` |
| Deleted | Scalar ID (`string`) | `InteractsWithSockets` only |

Scalar ID variation also works for events where the listener only needs the identifier (e.g., `AccountInvitationAccepted` carrying `string $invitationId`).

---

## Event Registration on Entities

Events are registered during domain methods — never published directly by entities.

### In `static new()` — Creation

```php
public static function new(array $attributes = []): self
{
    $contact = new self($attributes);
    $contact->registerDomainEvent(new ContactCreated($contact));

    return $contact;
}
```

### In `toBeDeleted()` — Deletion

```php
public function toBeDeleted(): self
{
    $this->registerDomainEvent(new ContactDeleted($this->id));

    return $this;
}
```

### In Domain Methods — Updates and State Transitions

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

### Conditional Registration

Register events only when state actually changes:

```php
public function updateEmail(string $email): self
{
    if ($email !== $this->email) {
        $this->forceFill(['email' => $email]);
        $this->registerDomainEvent(new UserEmailUpdated($this));
    }

    return $this;
}
```

See the `implementing-domain-entities` skill for the full entity patterns.

---

## Event Publishing in Use Cases

Use cases publish events **after persistence** — this ensures events reflect committed state:

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement\Contacts\Application;

use App\CustomerRelationshipManagement\Contacts\Domain\Contact;
use App\CustomerRelationshipManagement\Contacts\Domain\Contracts\ContactRepository;
use App\Shared\Domain\Contracts\ServiceBus\EventBus;

final readonly class CreateContact
{
    public function __construct(
        private ContactRepository $repository,
        private EventBus $eventBus,
    ) {}

    public function __invoke(array $attributes): Contact
    {
        $contact = $this->repository->save(Contact::new($attributes));
        $contact->publishDomainEvents($this->eventBus);

        return $contact;
    }
}
```

**The publishing flow:**

1. Entity registers events during domain methods (`registerDomainEvent()`)
2. Use case persists the entity via repository (`$repository->save()`)
3. Use case publishes events (`$entity->publishDomainEvents($this->eventBus)`)
4. `IlluminateEventBus` dispatches events through Laravel's `event()` helper
5. Registered listeners handle side effects (see `implementing-event-handlers`)

**Rules:**

- Always publish **after** `$repository->save()` — never before
- Use cases inject `EventBus` via constructor
- The entity's `publishDomainEvents()` clears the event queue after dispatching
- For deletion: publish before calling `$repository->delete()` (the entity still exists at that point)

---

## IlluminateEventBus Adapter

The Infrastructure adapter that bridges domain events to Laravel's event system:

```php
declare(strict_types=1);

namespace App\Shared\Infrastructure\ServiceBus;

use App\Shared\Domain\Contracts\ServiceBus\Event;
use App\Shared\Domain\Contracts\ServiceBus\EventBus;

class IlluminateEventBus implements EventBus
{
    public function publish(Event ...$events): void
    {
        if (empty($events)) {
            return;
        }

        event(...$events);
    }
}
```

**Key points:**

- Thin adapter — delegates to Laravel's `event()` helper
- Guards against empty event arrays
- **Framework coupling boundary:** The Domain depends on `Event` + `EventBus` contracts only. `IlluminateEventBus` is Infrastructure. This keeps the Domain pure
- Bound in `SharedServiceProvider` via `$bindings` array (see `configuring-bounded-contexts`):

```php
public array $bindings = [
    EventBus::class => IlluminateEventBus::class,
];
```

**Placement:** `app/Shared/Infrastructure/ServiceBus/IlluminateEventBus.php`

---

## Complete Event Flow

```text
Entity                    Use Case                 Infrastructure
──────                    ────────                 ──────────────
registerDomainEvent()  →  $repository->save()   →  Database write
                          publishDomainEvents()  →  IlluminateEventBus::publish()
                                                 →  Laravel event() helper
                                                 →  Registered listeners
                                                 →  domainEvents[] cleared
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Publishing events before persistence | Always call `publishDomainEvents()` AFTER `$repository->save()` |
| Entity calling `publishDomainEvents()` on itself | Publishing is the use case's responsibility, not the entity's |
| Passing full entity in deletion events | Pass `$this->id` (string) — the entity is about to be deleted |
| Missing `SerializesModels` on entity-carrying events | Required for queue serialization of Eloquent models |
| Adding `SerializesModels` to deletion events | Deletion events carry scalar IDs, not models — omit it |
| Importing `IlluminateEventBus` in Domain layer | Domain depends on `EventBus` contract only |
| Registering events in Infrastructure code | Events are registered in Domain methods and published by Application use cases |

---

## Pre-flight Checklist

- [ ] `Event` marker interface exists in `Shared/Domain/Contracts/ServiceBus/`?
- [ ] `EventBus` contract defines `publish(Event ...$events): void`?
- [ ] `HasDomainEvents` trait in `Shared/Domain/` with `registerDomainEvent()` + `publishDomainEvents()`?
- [ ] Domain events are `final class` implementing `Event`?
- [ ] Entity-carrying events use `SerializesModels`?
- [ ] Deletion events carry scalar ID, not entity?
- [ ] Events named as `{Entity}{PastTenseVerb}`?
- [ ] Events placed in `{Module}/Domain/Events/`?
- [ ] Use cases publish events AFTER persistence?
- [ ] `IlluminateEventBus` adapter bound in `SharedServiceProvider`?

---

## Laravel Reference

This skill uses Laravel-specific features. Consult the official Laravel documentation (via Context7 or web search) for details on:

- `event()` helper — dispatching events through Laravel's event system
- `Illuminate\Queue\SerializesModels` — queue-safe serialization of Eloquent models in events
- `Illuminate\Broadcasting\InteractsWithSockets` — WebSocket broadcasting support for events
- `Event::listen()` — registering event listeners in service providers
- `ShouldQueue` — marking event listeners for async processing

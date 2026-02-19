---
name: writing-event-handlers
description: Use when creating event handler classes, handling domain events in the application layer, setting up cross-context event listeners, or configuring queued event processing.
---

## Overview

Event handlers react to domain events by triggering side effects — sending emails, creating related entities, updating other aggregates. They live in the Application layer and are registered in the bounded context's ServiceProvider.

---

## Handler Structure

### Sending Notifications

```php
declare(strict_types=1);

namespace App\IdentityAndAccess\Users\Application\EventHandlers;

use App\IdentityAndAccess\Users\Application\FindUser;
use App\IdentityAndAccess\Users\Domain\Contracts\UserRepository;
use App\IdentityAndAccess\Users\Domain\Events\UserEmailUpdated;
use App\IdentityAndAccess\Users\Domain\Exceptions\UserNotFound;
use Illuminate\Contracts\Queue\ShouldQueue;

final readonly class SendUserEmailVerification implements ShouldQueue
{
    private FindUser $finder;

    public function __construct(UserRepository $repository)
    {
        $this->finder = new FindUser($repository);
    }

    /**
     * @throws UserNotFound
     */
    public function handle(UserEmailUpdated $event): void
    {
        $this->finder
            ->byId($event->user->id)
            ->sendEmailVerificationNotification();
    }
}
```

### Sending Mail

```php
declare(strict_types=1);

namespace App\IdentityAndAccess\Users\Application\EventHandlers;

use App\IdentityAndAccess\Accounts\Domain\Events\AccountInvitationCreated;
use App\IdentityAndAccess\Users\Infrastructure\Mail\UserInvitationMail;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Support\Facades\Mail;

class SendUserInvitationOnAccountInvitationCreated implements ShouldQueue
{
    public function handle(AccountInvitationCreated $event): void
    {
        Mail::to($event->invitation->email)->send(new UserInvitationMail(
            accountId: $event->invitation->account->id,
            accountName: $event->invitation->account->name,
            invitationId: $event->invitation->id,
        ));
    }
}
```

### Creating Related Entities

```php
declare(strict_types=1);

namespace App\IdentityAndAccess\Accounts\Application\EventHandlers;

use App\IdentityAndAccess\Accounts\Application\CreateAccount;
use App\IdentityAndAccess\Accounts\Domain\Contracts\AccountRepository;
use App\Shared\Domain\Contracts\ServiceBus\EventBus;
use Illuminate\Auth\Events\Registered;

final readonly class CreateAccountOnUserRegistered
{
    private CreateAccount $creator;

    public function __construct(AccountRepository $repository, EventBus $eventBus)
    {
        $this->creator = new CreateAccount($repository, $eventBus);
    }

    public function handle(Registered $event): void
    {
        $this->creator
            ->create(['name' => "{$event->user->name}'s Account"])
            ->addUser($event->user->id, 'owner');
    }
}
```

### Deleting Related Entities

```php
declare(strict_types=1);

namespace App\IdentityAndAccess\Accounts\Application\EventHandlers;

use App\IdentityAndAccess\Accounts\Application\DeleteAccount;
use App\IdentityAndAccess\Accounts\Domain\Contracts\AccountRepository;
use App\IdentityAndAccess\Users\Domain\Events\UserDeleted;
use App\Shared\Domain\Contracts\ServiceBus\EventBus;

class DeleteAccountOnUserDeleted
{
    private DeleteAccount $deleter;

    public function __construct(AccountRepository $repository, EventBus $eventBus)
    {
        $this->deleter = new DeleteAccount($repository, $eventBus);
    }

    public function handle(UserDeleted $event): void
    {
        $this->deleter->delete(id: $event->user, owner: $event->user);
    }
}
```

---

## Naming Convention

Handlers follow the pattern `{Action}On{Event}` or a descriptive action name:

| Handler | Listens To | Action |
|---|---|---|
| `SendUserEmailVerification` | `UserEmailUpdated` | Send verification email |
| `SendUserInvitationOnAccountInvitationCreated` | `AccountInvitationCreated` | Send invitation mail |
| `CreateAccountOnUserRegistered` | `Registered` | Create default account |
| `DeleteAccountOnUserDeleted` | `UserDeleted` | Delete associated account |
| `UpdatePersonalAccountNameOnUserNameUpdated` | `UserNameUpdated` | Sync account name |

When the action is self-explanatory (e.g., `SendUserEmailVerification`), the `On{Event}` suffix can be omitted. Use the full `{Action}On{Event}` form when disambiguation is needed.

---

## Queued vs Synchronous

Implement `ShouldQueue` for handlers that perform I/O operations:

| Should Queue | Synchronous |
|---|---|
| Sending emails / notifications | Creating related entities |
| Calling external APIs | Updating aggregates in the same transaction |
| Heavy processing | Lightweight in-memory operations |

```php
// Queued — I/O operation
final readonly class SendUserEmailVerification implements ShouldQueue
{
    public function handle(UserEmailUpdated $event): void { /* ... */ }
}

// Synchronous — entity creation should happen immediately
final readonly class CreateAccountOnUserRegistered
{
    public function handle(Registered $event): void { /* ... */ }
}
```

---

## Cross-Context Handlers

Handlers can listen to events from other bounded contexts. This is the primary mechanism for inter-context communication:

```text
Accounts Context (publishes)          Users Context (handles)
─────────────────────────            ────────────────────────
AccountInvitationCreated  ────────→  SendUserInvitationOnAccountInvitationCreated

Users Context (publishes)             Accounts Context (handles)
─────────────────────────            ────────────────────────
UserDeleted               ────────→  DeleteAccountOnUserDeleted
UserNameUpdated           ────────→  UpdatePersonalAccountNameOnUserNameUpdated
```

**Rules:**

- The handler imports the event class from the other context's `Domain/Events/`
- The handler lives in its own context's `Application/EventHandlers/`
- Registration happens in the **handler's** context ServiceProvider (not the event-producing context)

---

## ServiceProvider Registration

Register event → handler mappings in the bounded context's `$events` array:

```php
class IdentityAndAccessServiceProvider extends ServiceProvider
{
    protected array $events = [
        UserEmailUpdated::class => SendUserEmailVerification::class,
        AccountInvitationCreated::class => SendUserInvitationOnAccountInvitationCreated::class,
        Registered::class => CreateAccountOnUserRegistered::class,
    ];
}
```

The base ServiceProvider's `bootEvents()` registers each mapping via `Event::listen()`. See `configuring-bounded-contexts` for full details.

---

## Handler Dependencies

Handlers inject dependencies via constructor and compose use cases the same way use cases compose finders:

| Dependency | Pattern |
|---|---|
| Repository | Injected via constructor |
| Use case | Instantiated in constructor: `new CreateAccount($repository, $eventBus)` |
| Finder | Instantiated in constructor: `new FindUser($repository)` |
| EventBus | Injected when the handler publishes its own events |
| Mail / external services | Used directly in `handle()` via facades |

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Putting handlers in Domain layer | Handlers live in `Application/EventHandlers/` |
| Registering handlers in the event-producing context's SP | Register in the **handler's** context SP |
| Missing `ShouldQueue` on mail/notification handlers | Always queue I/O-heavy handlers |
| Injecting use cases via container | Instantiate in constructor (same as use case composition) |
| Handler doing too much | Delegate to use cases; handlers orchestrate, don't implement |
| Registering in Laravel's `EventServiceProvider` | Use the bounded context SP's `$events` array |

---

## Pre-flight Checklist

- [ ] Handler class in `{Module}/Application/EventHandlers/`?
- [ ] `handle()` method accepts the typed event class?
- [ ] Named as `{Action}On{Event}` or descriptive action?
- [ ] Implements `ShouldQueue` for I/O operations?
- [ ] Registered in the handler's context ServiceProvider `$events` array?
- [ ] Cross-context events imported from the producing context's `Domain/Events/`?
- [ ] Dependencies injected via constructor, use cases instantiated?

---

## Laravel Reference

This skill uses Laravel-specific features. Consult the official Laravel documentation (via Context7 or web search) for details on:

- `Illuminate\Contracts\Queue\ShouldQueue` — marking listeners for async queue processing
- `Illuminate\Support\Facades\Event` — event registration via `Event::listen()`
- `Illuminate\Support\Facades\Mail` — sending mail from handlers
- `Illuminate\Auth\Events\Registered` — Laravel's built-in user registration event

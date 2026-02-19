---
name: configuring-bounded-contexts
description: Use when writing a ServiceProvider for a bounded context, registering repository bindings or event listeners, setting up context-level routing, loading module-level migrations, or configuring boot-time behavior.
---

## Overview

Every bounded context has a dedicated ServiceProvider that wires its dependencies into Laravel's container. All context ServiceProviders extend a base `ServiceProvider` from the Shared Kernel that provides common booting for events and commands.

---

## Base ServiceProvider

The abstract base class in `app/Shared/Infrastructure/ServiceProvider.php` eliminates boilerplate across all context providers:

```php
declare(strict_types=1);

namespace App\Shared\Infrastructure;

use Illuminate\Support\Facades\Event;
use Illuminate\Support\ServiceProvider as BaseServiceProvider;

abstract class ServiceProvider extends BaseServiceProvider
{
    public array $bindings = [];

    protected array $events = [];

    protected array $commands = [];

    public function boot(): void
    {
        $this->bootEvents();
        $this->bootCommands();
    }

    protected function bootEvents(): void
    {
        foreach ($this->events as $event => $listener) {
            Event::listen($event, $listener);
        }
    }

    protected function bootCommands(): void
    {
        if ($this->app->runningInConsole()) {
            $this->commands($this->commands);
        }
    }
}
```

Every context ServiceProvider extends this base — never `Illuminate\Support\ServiceProvider` directly.

---

## Context ServiceProvider

### Minimal Template

A new bounded context ServiceProvider only declares what it needs:

```php
declare(strict_types=1);

namespace App\CustomerRelationshipManagement;

use App\CustomerRelationshipManagement\Contacts\Domain\Contracts\ContactRepository;
use App\CustomerRelationshipManagement\Contacts\Infrastructure\Console\Commands\ContactsBirthdayNotificationCommand;
use App\CustomerRelationshipManagement\Contacts\Infrastructure\Persistence\EloquentContactRepository;
use App\CustomerRelationshipManagement\Documents\Domain\Contracts\DocumentRepository;
use App\CustomerRelationshipManagement\Documents\Infrastructure\Persistence\EloquentDocumentRepository;
use App\CustomerRelationshipManagement\Managers\Domain\Contracts\ManagerRepository;
use App\CustomerRelationshipManagement\Managers\Infrastructure\Persistence\EloquentManagerRepository;
use App\Shared\Infrastructure\ServiceProvider;

class CustomerRelationshipManagementServiceProvider extends ServiceProvider
{
    public array $bindings = [
        ContactRepository::class => EloquentContactRepository::class,
        ManagerRepository::class => EloquentManagerRepository::class,
        DocumentRepository::class => EloquentDocumentRepository::class,
    ];

    protected array $commands = [
        ContactsBirthdayNotificationCommand::class,
    ];
}
```

No `boot()` override needed — the base class handles event and command booting automatically.

### With Routing and Migrations

Override `boot()` when the context needs routes or module-level migrations:

```php
use Illuminate\Support\Facades\Route;
use App\Shared\Infrastructure\ServiceProvider;

class CompanyRegistryServiceProvider extends ServiceProvider
{
    public array $bindings = [
        IndustryRepository::class => EloquentIndustryRepository::class,
        CompanyRepository::class => EloquentCompanyRepository::class,
        JobOfferRepository::class => EloquentJobOfferRepository::class,
    ];

    public function boot(): void
    {
        $this->loadMigrationsFrom([
            __DIR__.'/Companies/Infrastructure/Persistence/Migrations',
            __DIR__.'/Industries/Infrastructure/Persistence/Migrations',
            __DIR__.'/JobOffers/Infrastructure/Persistence/Migrations',
        ]);

        Route::middleware('web')
            ->group(__DIR__.'/Shared/Infrastructure/Http/Routes/web.php');

        Route::middleware('api')
            ->prefix('api')
            ->group(__DIR__.'/Shared/Infrastructure/Http/Routes/api.php');

        $this->bootEvents();
        $this->bootCommands();
    }
}
```

**Critical:** When overriding `boot()`, always call `$this->bootEvents()` and `$this->bootCommands()` at the end.

### With Custom Registration

Use `register()` for singletons or bindings that need constructor arguments — anything that doesn't fit the `$bindings` array:

```php
class AnalyticsAndReportingServiceProvider extends ServiceProvider
{
    public array $bindings = [
        ReportRepository::class => EloquentReportRepository::class,
    ];

    public function register(): void
    {
        $this->app->singleton(ReportGenerator::class, function ($app) {
            return new ReportGenerator(
                $app->make(ReportRepository::class),
                config('analytics.default_format'),
            );
        });
    }
}
```

---

## Three Declarative Arrays

### `$bindings` — Repository Contracts

Maps domain repository interfaces to their Eloquent implementations. Laravel resolves these automatically via constructor injection.

```php
public array $bindings = [
    ContactRepository::class => EloquentContactRepository::class,
    DocumentRepository::class => EloquentDocumentRepository::class,
];
```

### `$events` — Domain Event Listeners

Maps domain events to their application-layer handlers. The base `bootEvents()` registers them via `Event::listen()`.

```php
protected array $events = [
    UserEmailUpdated::class => SendUserEmailVerification::class,
    AccountInvitationCreated::class => SendUserInvitationOnAccountInvitationCreated::class,
    Registered::class => CreateAccountOnUserRegistered::class,
];
```

### `$commands` — Artisan Commands

Lists console commands for registration. The base `bootCommands()` registers them only when running in console.

```php
protected array $commands = [
    ContactsBirthdayNotificationCommand::class,
    ExpireVisasCommand::class,
];
```

---

## Registration

All context ServiceProviders are registered in `bootstrap/providers.php`:

```php
return [
    App\Shared\SharedServiceProvider::class,
    App\ExtensibilityAndCustomisation\ExtensibilityAndCustomisationServiceProvider::class,
    App\IdentityAndAccess\IdentityAndAccessServiceProvider::class,
    App\CustomerRelationshipManagement\CustomerRelationshipManagementServiceProvider::class,
    App\VisaManagement\VisaManagementServiceProvider::class,
    App\AnalyticsAndReporting\AnalyticsAndReportingServiceProvider::class,
];
```

**Order matters:** `SharedServiceProvider` must be **first** — it provides the `EventBus` binding, Eloquent strict mode, and global configuration used by all other contexts.

---

## SharedServiceProvider

The `SharedServiceProvider` has unique responsibilities beyond normal context SPs:

```php
class SharedServiceProvider extends ServiceProvider
{
    public array $bindings = [
        EventBus::class => IlluminateEventBus::class,   // Domain EventBus contract
    ];

    public function boot(): void
    {
        Model::shouldBeStrict();                         // Prevent lazy loading globally
        $this->extendRedirectResponses();                // Jetstream/Inertia stack
        $this->bootInertia();                            // Jetstream/Inertia stack
        $this->bootEvents();
        $this->bootCommands();
    }
}
```

---

## Context-Level Routes

Define route files in `{BC}/Shared/Infrastructure/Http/Routes/` and load them in `boot()` as shown in the "With Routing and Migrations" example above. Controllers are invokable — pass the class directly to the route definition.

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Extending `Illuminate\Support\ServiceProvider` directly | Extend `App\Shared\Infrastructure\ServiceProvider` |
| Registering events in Laravel's `EventServiceProvider` | Use the context SP's `$events` array |
| Putting routes in global `routes/web.php` | Use context-level `{BC}/Shared/Infrastructure/Http/Routes/` |
| Forgetting `bootEvents()` / `bootCommands()` in `boot()` override | Always call both when overriding `boot()` |
| Registering `SharedServiceProvider` last | Must be **first** in `bootstrap/providers.php` |
| Putting all migrations in `database/migrations/` | Use `loadMigrationsFrom()` for module-level migrations |

---

## Pre-flight Checklist

- [ ] SP extends `App\Shared\Infrastructure\ServiceProvider`?
- [ ] `$bindings` maps all repository contracts to implementations?
- [ ] `$events` maps all domain events to handlers?
- [ ] `$commands` lists all Artisan commands?
- [ ] Registered in `bootstrap/providers.php` (after `SharedServiceProvider`)?
- [ ] Context-level routes loaded in `boot()` if needed?
- [ ] Module-level migrations loaded with `loadMigrationsFrom()` if needed?
- [ ] `bootEvents()` and `bootCommands()` called in any `boot()` override?

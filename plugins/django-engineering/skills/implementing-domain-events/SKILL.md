---
name: implementing-domain-events
description: Use when implementing domain events, adding event handlers, integrating event publishing into Django services, or testing event-based functionality.
---

# Domain Event Bus Implementation Guide

## Overview

This skill provides guidance for implementing domain event bus architecture in Python projects. Use this when you need to:

- Create new domain events
- Add event handlers for existing events
- Integrate event publishing into services
- Test event-based functionality

## Core Principle

> **Events are REGISTERED when they conceptually happen (in the Domain), but PUBLISHED at the end of Service/Use Case execution (after transaction commits).**

This separation ensures:
- Domain purity (aggregates don't know about infrastructure)
- Transactional safety (no orphan events on rollback)
- Clear responsibilities per layer

## Architecture Summary

**Pattern:** Publisher Domain → Event Bus → Consumer Domain (one-way dependency)

**Benefits:**
- **Decoupling**: Publishers don't know about consumers
- **Extensibility**: New handlers subscribe without changing publishers
- **Async by design**: Events map to task queues (Celery, etc.)
- **Auditability**: Clear trace of triggers

## Layer Responsibilities

```
┌─────────────────────────────────────────────────────────────────┐
│                    AGGREGATE (Domain Layer)                      │
│  Events registered HERE - when they conceptually happen          │
│                                                                  │
│  - Factory method registers CreatedEvent on instantiation        │
│  - Business methods register UpdatedEvent on state changes       │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                REPOSITORY (Infrastructure Layer)                 │
│  Pure persistence - NO event logic                               │
│                                                                  │
│  - Calls domain factory/methods (which register events)          │
│  - save() only - NO EventBus dependency                          │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 SERVICE (Application Layer)                      │
│  Orchestration - publishes events after commit                   │
│                                                                  │
│  - Injected: Repository + EventBus                               │
│  - Calls repository (events already registered by domain)        │
│  - transaction.on_commit(aggregate.publish_domain_events)        │
└──────────────────────────────────────────────────────────────────┘
```

| Layer | Responsibility | EventBus? | Event Registration? |
|-------|---------------|-----------|---------------------|
| **Aggregate** | Register events when they happen | No | YES |
| **Repository** | Pure persistence only | NO | NO |
| **Service** | Orchestrate + publish after commit | YES | NO |

## Package Structure

```
<project>/shared/event_bus/
├── __init__.py          # Public exports
├── contracts.py         # DomainEvent, EventBus, HasDomainEventsMixin
├── adapters.py          # InMemorySynchronousEventBus, CeleryAsyncEventBus
├── registry.py          # EVENT_CLASSES, EVENTS_MAP, register_event, register_handler
├── provider.py          # get_event_bus()
└── tasks.py             # Async task for event processing (Celery/other)

<app>/
├── events.py            # Domain-specific events
└── event_handlers.py    # Event handlers
```

## Core Contracts

### DomainEvent Base Class

```python
from abc import ABC
from dataclasses import dataclass, field, fields, asdict
from datetime import datetime, UTC
from typing import Type, TypeVar, Any
from uuid import UUID, uuid4

E = TypeVar("E", bound="DomainEvent")

@dataclass(frozen=True, kw_only=True)
class DomainEvent(ABC):
    id: UUID = field(default_factory=uuid4)
    occurred_on: datetime = field(default_factory=lambda: datetime.now(UTC))

    def to_dict(self) -> dict[str, Any]:
        return asdict(self)

    @classmethod
    def from_dict(cls: Type[E], data: dict[str, Any]) -> E:
        field_names = {f.name for f in fields(cls)}
        return cls(**{k: v for k, v in data.items() if k in field_names})

    def event_type(self) -> str:
        return self.__class__.__name__
```

### EventBus Interface

```python
from abc import ABC, abstractmethod

class EventBus(ABC):
    @abstractmethod
    def publish(self, *domain_events: DomainEvent) -> None:
        pass
```

### HasDomainEventsMixin

```python
from collections import deque

class HasDomainEventsMixin:
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._domain_events: deque[DomainEvent] = deque()

    def register_domain_event(self, event: DomainEvent) -> None:
        self._domain_events.append(event)

    def publish_domain_events(self, event_bus: EventBus) -> None:
        event_bus.publish(*list(self._domain_events))
        self._domain_events.clear()
```

## Creating Domain Events

### Step 1: Define the Event Class

Location: `<app>/events.py`

```python
from dataclasses import dataclass
from uuid import UUID
from <project>.shared.event_bus.contracts import DomainEvent
from <project>.shared.event_bus.registry import register_event

@register_event
@dataclass(frozen=True, kw_only=True)
class EntityCreatedEvent(DomainEvent):
    entity_type: str
    entity_id: UUID

@register_event
@dataclass(frozen=True, kw_only=True)
class EntityUpdatedEvent(DomainEvent):
    entity_type: str
    entity_id: UUID
    changed_fields: tuple[str, ...]  # Use tuple for immutability
```

**Rules:**
- Always use `@register_event` decorator
- Always use `frozen=True` (immutability)
- Always use `kw_only=True` (explicit field names)
- Keep payload minimal - IDs preferred over full objects

### Step 2: Register Event in Aggregate

```python
from <project>.shared.event_bus.contracts import HasDomainEventsMixin

class MyAggregate(HasDomainEventsMixin):

    @classmethod
    def create(cls, **kwargs) -> "MyAggregate":
        """Factory method - registers CreatedEvent."""
        instance = cls(**kwargs)
        instance.register_domain_event(
            EntityCreatedEvent(
                entity_type=cls.__name__,
                entity_id=instance.id
            )
        )
        return instance

    def apply_changes(self, **kwargs) -> tuple[str, ...]:
        """Business method - registers UpdatedEvent if changed."""
        changed = []
        for field, value in kwargs.items():
            if getattr(self, field, None) != value:
                setattr(self, field, value)
                changed.append(field)

        if changed:
            self.register_domain_event(
                EntityUpdatedEvent(
                    entity_type=self.__class__.__name__,
                    entity_id=self.id,
                    changed_fields=tuple(changed)
                )
            )
        return tuple(changed)
```

## Event Registry

```python
from typing import Type, Callable

EventName = str
EventHandler = Callable[[DomainEvent], None]

EVENT_CLASSES: dict[EventName, Type[DomainEvent]] = {}
EVENTS_MAP: dict[EventName, list[EventHandler]] = {}

def register_event(event_cls: Type[DomainEvent]) -> Type[DomainEvent]:
    EVENT_CLASSES[event_cls.__name__] = event_cls
    return event_cls

def register_handler(event_type: str, handler: EventHandler) -> None:
    if event_type not in EVENTS_MAP:
        EVENTS_MAP[event_type] = []
    EVENTS_MAP[event_type].append(handler)
```

## Event Bus Adapters

### Synchronous (Testing)

```python
class InMemorySynchronousEventBus(EventBus):
    def __init__(self, events_map: dict[EventName, list[EventHandler]]):
        self.events_map = events_map

    def publish(self, *domain_events: DomainEvent) -> None:
        for event in domain_events:
            for handler in self.events_map.get(event.event_type(), []):
                handler(event)
```

### Asynchronous (Production - Celery)

```python
class CeleryAsyncEventBus(EventBus):
    def publish(self, *domain_events: DomainEvent) -> None:
        for event in domain_events:
            from <project>.shared.event_bus.tasks import process_domain_event
            process_domain_event.delay(
                event_type=event.event_type(),
                event_data=event.to_dict(),
            )
```

### Celery Task

```python
from celery import shared_task

@shared_task(
    bind=True,
    max_retries=3,
    default_retry_delay=60,
    autoretry_for=(Exception,),
    retry_backoff=True,
)
def process_domain_event(self, event_type: str, event_data: dict) -> None:
    from <project>.shared.event_bus.registry import EVENT_CLASSES, EVENTS_MAP

    event_cls = EVENT_CLASSES.get(event_type)
    if not event_cls:
        return

    event = event_cls.from_dict(event_data)

    for handler in EVENTS_MAP.get(event_type, []):
        handler(event)
```

## Creating Event Handlers

Location: `<consumer-app>/event_handlers.py`

```python
from <project>.shared.event_bus.registry import register_handler
from <publisher-app>.events import EntityCreatedEvent

def handle_entity_created(event: EntityCreatedEvent) -> None:
    # Trigger downstream processing, notifications, etc.
    pass

# Register at module level
register_handler("EntityCreatedEvent", handle_entity_created)
```

**Handler Guidelines:**
- Keep handlers focused on one responsibility
- Use async tasks for heavy processing
- Handle exceptions gracefully (will retry via task queue)
- Import handler module at application startup

## Service Integration

### Repository Pattern (Pure Persistence)

```python
class MyRepository:
    def save(self, entity: MyAggregate) -> MyAggregate:
        entity.save()  # Just persistence, no events
        return entity

    def upsert(self, identifier: str, data: dict) -> tuple[MyAggregate, bool]:
        existing = self.find_by_identifier(identifier)

        if existing:
            existing.apply_changes(**data)  # Domain registers event
            existing.save()
            return existing, False
        else:
            entity = MyAggregate.create(**data)  # Domain registers event
            entity.save()
            return entity, True
```

### Service Pattern (Orchestration)

```python
class MyService:
    def __init__(self, repo: MyRepository, event_bus: EventBus) -> None:
        self._repo = repo
        self._event_bus = event_bus

    def execute(self, data: dict) -> MyAggregate:
        # Repository calls domain methods (events registered there)
        entity, created = self._repo.upsert(data["id"], data)

        # Publish AFTER successful persistence
        entity.publish_domain_events(self._event_bus)

        return entity
```

### Django Integration (with transaction safety)

```python
from django.db import transaction

class DjangoService:
    def execute(self, data: dict) -> MyAggregate:
        entity, created = self._repo.upsert(data["id"], data)

        # Publish ONLY after transaction commits
        transaction.on_commit(
            lambda e=entity: e.publish_domain_events(self._event_bus)
        )

        return entity
```

### DI Container (dependency-injector)

```python
from dependency_injector import containers, providers

class Container(containers.DeclarativeContainer):
    event_bus = providers.Singleton(get_event_bus)

    my_service = providers.Factory(
        MyService,
        repo=my_repo,
        event_bus=event_bus,
    )
```

## Event Bus Provider

```python
def get_event_bus(driver: str | None = None) -> EventBus:
    if driver is None:
        driver = os.getenv("EVENT_BUS_DRIVER", "celery")

    if driver == "in-memory":
        from .adapters import InMemorySynchronousEventBus
        from .registry import EVENTS_MAP
        return InMemorySynchronousEventBus(EVENTS_MAP)

    from .adapters import CeleryAsyncEventBus
    return CeleryAsyncEventBus()
```

## Event Flow Sequence

```
1. API/Webhook receives request
          │
          ▼
2. Service calls Repository
          │
          ▼
3. Repository calls Aggregate methods
   └─► aggregate.register_domain_event(Event)
          │
          ▼
4. aggregate.save() → Database
          │
          ▼
5. After commit: aggregate.publish_domain_events()
          │
          ▼
6. EventBus.publish() → Task Queue
          │
          ▼
7. process_domain_event task executes
          │
          ▼
8. Handler functions process event
```

## DO and DON'T Rules

### DO

1. Register events in Aggregate factory methods (`create()`)
2. Register events in Aggregate business methods (`apply_changes()`, `resolve()`)
3. Use `transaction.on_commit()` to publish events after persistence
4. Keep Repository free of EventBus dependency
5. Inject EventBus into Service layer only
6. Use immutable events (`frozen=True`)

### DON'T

1. Register events in Service layer
2. Register events in Repository
3. Publish events before transaction commits
4. Inject EventBus into Repository
5. Use framework signals (Django signals) for domain events
6. Store mutable objects in event payloads

## Testing

### Test Event Dispatch

```python
def test_event_bus_dispatches_to_handlers():
    handled_events = []

    def capture_handler(event):
        handled_events.append(event)

    events_map = {"EntityCreatedEvent": [capture_handler]}
    bus = InMemorySynchronousEventBus(events_map)

    event = EntityCreatedEvent(entity_type="Order", entity_id=uuid4())
    bus.publish(event)

    assert len(handled_events) == 1
    assert handled_events[0] == event
```

### Test Event Serialization

```python
def test_event_serialization_roundtrip():
    event = EntityCreatedEvent(entity_type="Order", entity_id=uuid4())
    data = event.to_dict()
    restored = EntityCreatedEvent.from_dict(data)

    assert restored.entity_type == event.entity_type
    assert restored.entity_id == event.entity_id
```

## Configuration

```python
# Production
EVENT_BUS_DRIVER = "celery"

# Testing
EVENT_BUS_DRIVER = "in-memory"
```

## Why This Pattern?

| Concern | Solution |
|---------|----------|
| **When to register** | At the moment event conceptually happens (domain) |
| **When to publish** | After transaction commits (service via on_commit) |
| **Transactional safety** | `on_commit` ensures no orphan events on rollback |
| **Single Responsibility** | Repository = persistence, Service = orchestration |
| **Domain purity** | Domain owns its events, doesn't know about publishing |

## Checklist for New Events

- [ ] Define event as frozen dataclass with `@register_event`
- [ ] Add `HasDomainEventsMixin` to aggregate (if not already)
- [ ] Register event in aggregate factory (`create()`) or business method
- [ ] Ensure repository calls domain methods (not direct field assignment)
- [ ] Add `transaction.on_commit(aggregate.publish_domain_events)` in service
- [ ] Register handlers via `register_handler()`
- [ ] Write tests for event registration and handler execution

## Quick Reference

| Task | Location | Key Function |
|------|----------|--------------|
| Create event | `<app>/events.py` | `@register_event @dataclass` |
| Create handler | `<consumer>/event_handlers.py` | `register_handler()` |
| Publish in service | Service method | `entity.publish_domain_events(event_bus)` |
| Get event bus | DI/Factory | `get_event_bus()` |
| Test sync | Test file | `InMemorySynchronousEventBus` |

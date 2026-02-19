---
name: commenting-code
description: Use when writing, reviewing, or modifying PHP code in a Laravel DDD project. Governs when to add comments, when to remove them, PHPDoc usage, and type declaration practices.
---

## The Rule

**Every comment must earn its place.** If the information is already in the code (method name, type signature, class name), the comment is noise — delete it. If the comment explains WHY something exists, documents a non-obvious side effect, or declares a thrown exception — keep it.

---

## Type Declarations

Use PHP 8.2+ native types on ALL function signatures. No exceptions.

```php
// Correct — native types everywhere
public function find(string $id): ?Contact { }
public function save(Contact $contact): Contact { }
public function delete(Contact $contact): void { }
public function match(Criteria $criteria): LengthAwarePaginator { }
```

**`readonly` properties do NOT need `@var` annotations** — the type is visible in the constructor signature:

```php
// Correct — no @var needed
final readonly class CreateContact
{
    public function __construct(
        private ContactRepository $repository,
        private EventBus $eventBus,
    ) {}
}
```

---

## When PHPDoc IS Required

### 1. `@throws` — Always Declare Thrown Exceptions

```php
/**
 * @throws ContactNotFound
 */
public function byId(string $id): Contact
{
    if (! $contact = $this->repository->find($id)) {
        throw new ContactNotFound(strtr('Contact {id} not found.', ['{id}' => $id]));
    }

    return $contact;
}
```

**This is non-negotiable.** PHP does not enforce checked exceptions — `@throws` is the only way IDEs and consumers know what to expect. Omitting `@throws` is a bug.

### 2. `@property` on Eloquent Models

Eloquent model attributes cannot be declared as native PHP properties. Use `@property` and `@method` docblocks on the class:

```php
/**
 * @property string $id
 * @property string $first_name
 * @property string $last_name
 * @property string $email
 * @property Carbon $created_at
 * @method static ContactFactory factory()
 */
class Contact extends Model { }
```

### 3. `@var` for Typed Array Contents

When an array holds a specific type that native `array` cannot express:

```php
/**
 * @var Event[]
 */
private array $domainEvents = [];
```

### 4. Scalar ID Payload Clarification

When an event carries a scalar ID instead of an entity, document what it represents:

```php
/**
 * @param string $contact The deleted contact uuid
 */
public function __construct(public string $contact) {}
```

---

## When PHPDoc is FORBIDDEN

### Never Add `@param` or `@return` When Native Types Exist

```php
// WRONG — redundant, delete the entire docblock
/**
 * @param string $id
 * @return Contact|null
 */
public function find(string $id): ?Contact { }

// CORRECT — signature says everything
public function find(string $id): ?Contact { }
```

### Never Add Docstrings That Repeat the Method Name

```php
// WRONG — "Create a contact" is just the method name in prose
/**
 * Create a contact.
 */
public function create(array $input): Contact { }

// CORRECT — no docblock at all
public function create(array $input): Contact { }
```

### Never Add Class-Level Docstrings That Repeat the Class Name

```php
// WRONG — "Handler for sending user email verification" is just the class name
/**
 * Handler for sending user email verification.
 */
final readonly class SendUserEmailVerification implements ShouldQueue { }

// CORRECT — no docblock
final readonly class SendUserEmailVerification implements ShouldQueue { }
```

### Never Add `@var` to Constructor-Promoted Properties

```php
// WRONG — type is already in the constructor
/**
 * @var ContactRepository
 */
private ContactRepository $repository;

// CORRECT — type is in the signature, nothing else needed
public function __construct(private ContactRepository $repository) {}
```

---

## Inline Comments

### Keep: WHY Comments

```php
// Events must be published AFTER persistence to ensure committed state
$contact->publishDomainEvents($this->eventBus);

// Deletion is idempotent — missing entity is not an error
try {
    $contact = $this->finder->byId($id);
} catch (ContactNotFound) {
    return;
}

// Transitions call save() internally — spatie framework requirement
$this->status->transitionTo(Granted::class);
```

### Keep: Side-Effect Warnings

```php
// Also clears the domain events queue after dispatching
$eventBus->publish(...$this->domainEvents);
$this->domainEvents = [];
```

### Keep: Non-Obvious Business Rule

```php
// Only register event when email actually changes (avoid duplicate events)
if ($email !== $this->email) {
    $this->forceFill(['email' => $email]);
    $this->registerDomainEvent(new UserEmailUpdated($this));
}
```

### Delete: WHAT Comments

```php
// WRONG — all of these restate the code
$this->name = $name;           // Set the name
$contact->save();              // Save the contact
return redirect('/dashboard'); // Redirect to dashboard
$contact->delete();            // Delete the contact
```

### Delete: Visual Separators

```php
// WRONG — never use separator comments
// ========================================
// Static Constructors
// ========================================

// --- Relationships ---

// =====================
// Domain Methods
// =====================
```

---

## Comments in Exception Classes

Exception classes are **always empty-bodied**. Never add docstrings to them:

```php
// CORRECT — empty body, no docstring
class ContactNotFound extends ContactException {}

// WRONG — docstring adds nothing
/**
 * Exception thrown when a contact is not found.
 */
class ContactNotFound extends ContactException {}
```

---

## Comments in Tests

- Test method names are self-documenting — do not add docstrings to test methods
- Comment only non-obvious test setup or surprising assertions
- Do not add `// Arrange`, `// Act`, `// Assert` section labels unless the test is unusually complex

```php
// WRONG — test name already says this
/** @test */
public function it_throws_not_found_when_contact_does_not_exist(): void
{
    // Act & Assert  ← REMOVE these labels
    $this->expectException(ContactNotFound::class);
    $finder->byId('nonexistent-id');
}

// CORRECT — no docstring, no section labels
/** @test */
public function it_throws_not_found_when_contact_does_not_exist(): void
{
    $this->expectException(ContactNotFound::class);
    $finder->byId('nonexistent-id');
}
```

---

## Contract Interfaces

Interface-level docblocks document the **contract purpose**, not individual methods:

```php
// CORRECT — documents the contract, not the method
/**
 * Persistence contract for Contact aggregate root.
 */
interface ContactRepository
{
    public function find(string $id): ?Contact;
    public function match(Criteria $criteria): LengthAwarePaginator;
    public function save(Contact $contact): Contact;
    public function delete(Contact $contact): void;
}
```

Individual interface methods do NOT get docblocks — the method signature is the documentation. Add `@throws` only on the implementing class if it throws.

---

## Quick Decision Table

| Situation | Action |
|---|---|
| Method has native type signature | No `@param` / `@return` |
| Method throws an exception | Add `@throws` — always |
| Eloquent model attributes | Add `@property` docblock on class |
| Array holds typed items | Add `@var Type[]` |
| Event carries scalar ID | Add `@param` with description |
| `readonly` property in constructor | No `@var` |
| Comment explains WHY | Keep |
| Comment explains WHAT | Delete |
| Comment is a visual separator | Delete |
| Exception class body | Always empty — no docstring |
| Test method | No docstring — name is documentation |
| Interface-level purpose | Add class-level docblock |
| Interface method | No docblock — signature is enough |

---

## Red Flags — STOP and Remove

If you catch yourself writing any of these, delete it immediately:

- A docblock that starts with the method/class name in different words
- `@param` or `@return` tags when native types are already in the signature
- `// Set the...`, `// Get the...`, `// Create a...`, `// Delete the...`
- A section separator made of `=`, `-`, or `*` characters
- `// Arrange`, `// Act`, `// Assert` in a test shorter than 15 lines
- A docblock on an empty-bodied exception class
- `@var` on a constructor-promoted `readonly` property

---

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "PHPDoc improves IDE support" | Native types give IDEs everything they need. Redundant PHPDoc is noise. |
| "It's good practice to document all public methods" | Self-documenting code + `@throws` is the practice. Restating signatures is not. |
| "I added it for completeness" | Completeness means every comment adds value. Redundant comments subtract value. |
| "The team might not understand the method" | If the method name + types aren't clear, rename the method — don't paper over it with a comment. |
| "I'll add section separators for readability" | If your class needs separators, it's too large. Extract a class. |
| "Tests should be commented for clarity" | Test names ARE the documentation. A well-named test needs zero comments. |

---

## Pre-flight Checklist

- [ ] All function signatures have native PHP type declarations?
- [ ] No `@param` / `@return` duplicating native types?
- [ ] `@throws` declared on every method that throws?
- [ ] `@property` docblock on Eloquent models?
- [ ] No docstrings restating method or class names?
- [ ] No visual separator comments?
- [ ] No comments explaining WHAT (only WHY)?
- [ ] No docstrings on exception classes?
- [ ] No `@var` on constructor-promoted properties?
- [ ] Test methods have no docstrings?

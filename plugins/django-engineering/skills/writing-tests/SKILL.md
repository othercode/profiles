---
name: writing-tests
description: Use when writing tests, creating test files, naming test functions, adding fixtures, or using pytest markers in Django projects.
---

## Contents

- Test Location & File Naming
- Function Naming
- Style Rules (CRITICAL)
- No ORM in Tests
- Typed Fixtures
- Pytest Markers
- Test Templates
- Conftest Organization
- Test Data
- Common Assertions
- Pre-flight Checklist

---

## Test Creation Guidelines

### 1. Test Location & File Naming

Tests are **colocated** with the code they test, using `*_test.py` suffix.

```
# Pattern
module/feature.py → module/feature_test.py

# Examples
proxysubscription/plan_change.py → proxysubscription/plan_change_test.py
accountsuspension/actions.py → accountsuspension/actions_test.py
userprofile/handlers.py → userprofile/handlers_test.py
```

### 2. Function Naming

```
test_<subject>_should[_not]_<expectation>[_when_<scenario>]
```

| Part                    | Required | Description             |
| ----------------------- | -------- | ----------------------- |
| `test_`                 | Yes      | Pytest discovery prefix |
| `<subject>`             | Yes      | What is being tested    |
| `should` / `should_not` | Yes      | Expected behavior       |
| `<expectation>`         | Yes      | What should happen      |
| `when_<scenario>`       | No       | Condition or context    |

**Examples:**

```python
test_user_should_be_created_when_email_valid
test_subscription_should_not_renew_when_payment_fails
test_plan_upgrade_should_calculate_correct_credits
test_parser_should_return_none_when_input_empty
```

### 3. Style Rules (CRITICAL)

```python
# ❌ NEVER
class TestUserService:                    # No class-based tests
    def test_create(self): ...

def test_user():
    """Test user."""                      # No docstrings in tests

def test_user(user_service, data):        # No untyped fixtures
    assert user.name == "John"  # Check   # No obvious comments

# ✅ CORRECT
def test_user_should_be_created_when_email_valid(
    user_service: CreateUserService,
    sample_user_data: dict[str, Any],
):
    result = user_service.create(sample_user_data)
    assert result.id is not None
```

### 4. No ORM in Tests

Use service layer instead of `.objects.*` calls.

```python
# ❌ Bad - direct ORM usage
User.objects.create(email="test@example.com")
Plan.objects.filter(user=user).first()
AccountVerificationFlow.objects.create(user=user, state="inflow")

# ✅ Good - service layer
user_service.create_user(email="test@example.com")
plan_repository.find_by_user(user=user)
verification_service.start_flow(user=user)
```

### 5. Typed Fixtures (REQUIRED)

All fixture parameters must have type annotations.

```python
# ❌ Bad - untyped
def test_something(user, subscription, mock_repo):
    ...

# ✅ Good - fully typed
def test_something(
    user: User,
    subscription: SubscriptionPlan,
    mock_repo: MagicMock,
):
    ...
```

**Common fixture types:**

```python
def test_example(
    user: User,                            # Domain objects
    subscription: SubscriptionPlan,
    mock_payment_service: MagicMock,       # Mocks
    mocker: MockerFixture,                 # pytest-mock
    sample_payload: dict[str, Any],        # Test data
    mailoutbox: list[tuple],               # Email capture
    db: None,                              # Django DB access marker
):
    ...
```

### 6. Pytest Markers

| Marker                     | When to Use                                                   |
| -------------------------- | ------------------------------------------------------------- |
| `@pytest.mark.django_db`   | Test needs database access                                    |
| `@pytest.mark.small`       | Fast test with minimal dependencies - use for quick iteration |
| `@pytest.mark.stripe`      | Test involves Stripe API mocking/integration                  |
| `@pytest.mark.flaky`       | Known unreliable test (use sparingly, prefer fixing)          |
| `@pytest.mark.skip`        | Temporarily disabled (include reason)                         |
| `@pytest.mark.parametrize` | Data-driven test with multiple inputs                         |

**Examples:**

```python
@pytest.mark.django_db
def test_user_should_be_persisted(user_service: UserService):
    ...

@pytest.mark.small
def test_parser_should_handle_empty_input(parser: Parser):
    # No DB, no external deps - runs fast
    ...

@pytest.mark.django_db
@pytest.mark.stripe
def test_subscription_should_charge_card(payment_service: PaymentService):
    ...

@pytest.mark.flaky(reruns=3, reason="Redis connection sometimes drops")
@pytest.mark.django_db
def test_cache_should_invalidate(cache_service: CacheService):
    ...

@pytest.mark.parametrize("status,expected", [
    ("active", Status.ACTIVE),
    ("pending", Status.PENDING),
    ("unknown", Status.PENDING),
])
def test_parse_status_should_map_correctly(
    parser: StatusParser,
    status: str,
    expected: Status,
):
    assert parser.parse(status) == expected
```

### 7. Test Templates

**Unit Test (no DB):**

```python
@pytest.mark.small
def test_calculator_should_apply_discount_when_eligible(
    mock_pricing_repo: MagicMock,
):
    mock_pricing_repo.get_discount_rate.return_value = Decimal("0.1")
    calculator = PriceCalculator(pricing_repo=mock_pricing_repo)

    result = calculator.calculate(base_price=Decimal("100"))

    assert result == Decimal("90")
    mock_pricing_repo.get_discount_rate.assert_called_once()
```

**Integration Test (with DB):**

```python
@pytest.mark.django_db
def test_user_should_be_created_with_default_settings(
    user_service: CreateUserService,
    user_repository: UserRepository,
):
    email = f"test_{uuid4().hex[:8]}@example.com"

    user = user_service.create(email=email)

    persisted = user_repository.get_by_id(user_id=user.id)
    assert persisted.email == email
    assert persisted.settings.notifications_enabled is True
```

### 8. Conftest Organization

```
proxysubscription/
├── conftest.py              # Module-level shared fixtures
├── plan_change.py
├── plan_change_test.py
├── pricing.py
└── pricing_test.py

shared/
├── conftest.py              # Cross-module fixtures (event_bus, di_container)
```

**Module conftest.py example:**

```python
import pytest
from django.contrib.auth.models import User

@pytest.fixture
def customer() -> User:
    # Use service layer, not ORM
    return user_service.create_test_user(
        email="customer@example.com",
        is_staff=False,
    )

@pytest.fixture
def mock_payment_gateway() -> MagicMock:
    return MagicMock(spec=PaymentGateway)
```

### 9. Test Data

```python
# ✅ Use UUIDs/timestamps for uniqueness
email = f"user_{uuid4().hex[:8]}@example.com"
username = f"test_user_{timezone.now().timestamp()}"

# ✅ Minimal data - only what test needs
user_data = {"email": email, "name": "Test User"}

# ✅ Descriptive variable names
valid_subscription_data = {...}
expired_subscription_data = {...}
```

### 10. Common Assertions

| Pattern             | Example                                            |
| ------------------- | -------------------------------------------------- |
| Equality            | `assert result.status == "active"`                 |
| Identity            | `assert result is not None`                        |
| Collection contains | `assert "admin" in user.roles`                     |
| Collection length   | `assert len(results) == 3`                         |
| Exception raised    | `with pytest.raises(ValueError, match="invalid"):` |
| Mock called         | `mock_service.save.assert_called_once()`           |
| Mock called with    | `mock_service.save.assert_called_with(user=user)`  |
| Mock not called     | `mock_notifier.send.assert_not_called()`           |

### Pre-flight Checklist

- [ ] File named `*_test.py` and colocated with module?
- [ ] Function name follows `test_<subject>_should_<expectation>` pattern?
- [ ] All fixture parameters typed?
- [ ] No class-based tests?
- [ ] No docstrings or obvious comments?
- [ ] Using service layer instead of ORM?
- [ ] Correct markers (`django_db`, `small`, `stripe`)?
- [ ] Test data uses unique identifiers?

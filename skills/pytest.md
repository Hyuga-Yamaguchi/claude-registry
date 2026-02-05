# Pytest Best Practices

A comprehensive guide for improving test code quality with key decision points and important principles.

- [Pytest Best Practices](#pytest-best-practices)
  - [Test Function Naming](#test-function-naming)
  - [Given-When-Then Pattern](#given-when-then-pattern)
  - [Meaningful Assertions](#meaningful-assertions)
  - [Mock Usage Guidelines](#mock-usage-guidelines)
  - [Realistic Test Data](#realistic-test-data)
  - [Test Design Principles](#test-design-principles)
    - [Avoid `if` Statements in Tests](#avoid-if-statements-in-tests)
    - [Focus on Critical and Complex Business Logic](#focus-on-critical-and-complex-business-logic)
  - [Coverage Strategy](#coverage-strategy)
  - [Code Style](#code-style)

## Test Function Naming

**Use descriptive names that clearly communicate what is being tested.** Follow your project's naming conventions.

```python
# Good: Descriptive English names
def test_user_registration_with_invalid_email_returns_validation_error():
    pass

def test_order_total_for_domestic_shipping_includes_tax():
    pass

def test_expired_session_redirects_to_login_page():
    pass

def test_insufficient_balance_prevents_withdrawal():
    pass
```

**Bad examples:**
```python
def test_user():  # Unclear what aspect is being tested
    pass

def test_order_1():  # Sequential numbers provide no meaning
    pass

def test_it_works():  # Too vague
    pass
```

## Given-When-Then Pattern

Structure tests using the **Given-When-Then** pattern:

```python
def test_withdrawal_succeeds_when_balance_is_sufficient():
    # Given (preconditions)
    account = BankAccount(balance=100)

    # When (action)
    result = account.withdraw(30)

    # Then (expected outcome)
    assert result is True
    assert account.balance == 70
```

Use fixtures for complex setup:

```python
@pytest.fixture
def account_with_balance():
    """Given: An account with sufficient funds"""
    return BankAccount(balance=100)

def test_withdrawal_succeeds_when_balance_is_sufficient(account_with_balance):
    # When
    result = account_with_balance.withdraw(30)

    # Then
    assert result is True
    assert account_with_balance.balance == 70
```

## Meaningful Assertions

**Avoid weak assertions:**

```python
# Bad: Not verifying actual behavior
def test_get_users_weak_assertions():
    users = get_users()
    assert users is not None  # Too weak
    assert users  # Only checking truthiness
    assert len(users) == 3  # Only checking count, not content

# Good: Verify actual content
def test_get_users_verifies_content():
    users = get_users()

    assert users == [
        User(id=1, name="John Doe", email="john@example.com"),
        User(id=2, name="Jane Smith", email="jane@example.com"),
        User(id=3, name="Bob Johnson", email="bob@example.com"),
    ]
```

**Compare whole objects instead of individual properties:**

```python
# Bad: Property-by-property comparison (verbose and error-prone)
def test_create_user_property_comparison():
    user = create_user("Alice", "alice@example.com")
    assert user.name == "Alice"
    assert user.email == "alice@example.com"
    assert user.is_active is True
    assert user.role == "member"

# Good: Whole object comparison
def test_create_user_object_comparison():
    user = create_user("Alice", "alice@example.com")
    expected = User(
        name="Alice",
        email="alice@example.com",
        is_active=True,
        role="member"
    )
    assert user == expected
```

**For Pydantic models, use `model_dump()` for better diff output:**

```python
def test_create_order_pydantic():
    result = create_order(items=["apple", "banana"])
    expected = Order(
        items=["apple", "banana"],
        status="pending",
        total=150
    )
    # Better diff output on failure
    assert result.model_dump() == expected.model_dump()
```

## Mock Usage Guidelines

**Principle: Minimize mocking. Avoid it when possible.**

When necessary, limit mocking to external dependencies (APIs, database connections). Use real implementations for internal logic.

```python
# Bad: Over-mocking internal components
def test_excessive_mocking(mocker):
    mocker.patch("myapp.services.user_service.validate_email")
    mocker.patch("myapp.services.user_service.hash_password")
    mocker.patch("myapp.services.user_service.generate_id")

    result = create_user("alice@example.com", "password")
    # What are we actually testing here?

# Good: Mock only external dependencies
def test_appropriate_mocking(mocker):
    # Mock only actual external dependencies
    mock_db = mocker.patch("myapp.services.user_service.database")
    mock_db.save.return_value = True

    result = create_user("alice@example.com", "password")

    # All internal logic (validation, hashing, etc.) runs for real
    assert result.email == "alice@example.com"
    assert result.password_hash != "password"
```

**When to mock:**
- External APIs and network calls
- Database connections (or use test databases)
- Time-dependent operations (`datetime.now()`)
- Third-party services

**When NOT to mock:**
- Internal business logic
- Data transformations
- Validation functions
- Pure functions

## Realistic Test Data

**Use production-like data. Avoid placeholder data.**

```python
# Bad: Unrealistic placeholder data
def test_create_order_bad_data():
    order = create_order(
        customer_name="test",
        email="test@test.com",
        items=[{"name": "foo", "price": 123}],
        address="bar"
    )

# Good: Production-like realistic data
def test_create_order_realistic_data():
    order = create_order(
        customer_name="John Smith",
        email="john.smith@example.com",
        items=[
            {"name": "MacBook Pro 14-inch", "price": 2499},
            {"name": "USB-C Cable", "price": 25}
        ],
        address="123 Main Street, San Francisco, CA 94102"
    )
```

**Factory fixtures for efficient realistic data creation:**

```python
@pytest.fixture
def create_user():
    """Factory for creating realistic user data"""
    counter = 0
    names = ["John Doe", "Jane Smith", "Bob Johnson"]

    def _create(name: str | None = None, email: str | None = None, **kwargs):
        nonlocal counter
        counter += 1
        return User(
            name=name or names[counter % len(names)],
            email=email or f"user{counter}@example.com",
            phone=kwargs.get("phone", "+1-555-0100"),
            **kwargs
        )

    return _create
```

## Test Design Principles

### Avoid `if` Statements in Tests

Branching in tests indicates a design problem. Use parametrization instead.

```python
# Bad: Conditional logic in test
def test_with_branching(user_type):
    user = create_user(type=user_type)
    if user_type == "admin":
        assert user.can_delete is True
    else:
        assert user.can_delete is False

# Good: Parametrize to eliminate branching
@pytest.mark.parametrize("user_type,expected_can_delete", [
    ("admin", True),
    ("member", False),
    ("guest", False),
], ids=["admin_can_delete", "member_cannot_delete", "guest_cannot_delete"])
def test_user_permissions(user_type, expected_can_delete):
    user = create_user(type=user_type)
    assert user.can_delete is expected_can_delete
```

### Focus on Critical and Complex Business Logic

Prioritize testing error-prone areas over trivial cases.

```python
# Low value: Testing obvious getter
def test_get_user_name():
    user = User(name="Alice")
    assert user.name == "Alice"  # This is trivial

# High value: Testing complex business rules
def test_discount_calculation_for_loyal_customer_with_bulk_order():
    """
    Loyal customers (1+ years) get 10% discount.
    Bulk orders (10+ items) get additional 5% discount.
    Discounts are multiplicative, not additive.
    """
    # Given
    customer = Customer(membership_years=2)
    order = Order(items=[Item(price=1000)] * 15)

    # When
    discount = calculate_discount(customer, order)

    # Then: 10% × 5% = 14.5% (multiplicative), not 15% (additive)
    assert discount == pytest.approx(0.145)

# High value: Testing edge cases and error handling
def test_out_of_stock_product_raises_appropriate_error():
    """Verify proper error handling when product is unavailable"""
    with pytest.raises(OutOfStockError) as exc_info:
        create_order(items=[{"sku": "DISCONTINUED-001", "quantity": 1}])

    assert exc_info.value.sku == "DISCONTINUED-001"
    assert "out of stock" in str(exc_info.value).lower()
```

## Coverage Strategy

**Set quantitative targets (e.g., 80%), but don't mandate 100%.**

```toml
# pyproject.toml
[tool.coverage.report]
fail_under = 80  # Realistic goal
```

**Not worth testing for coverage:**

```python
# 1. Simple data classes
@dataclass
class Config:
    host: str
    port: int

# 2. Framework boilerplate
if __name__ == "__main__":  # pragma: no cover
    main()
```

**Coverage priorities:**

```python
# High priority coverage:
# - Business logic and calculations
# - Data validation and transformation
# - User-impacting error handling paths
# - Integration points

# Low priority coverage:
# - Logging statements
# - Debug utilities
# - Admin/maintenance scripts
```

**Focus on whether critical logic is tested, not just coverage numbers.**

## Code Style

**Follow the existing test style in your project.**

Check these aspects:
- Class-based vs function-based tests
- Fixture naming conventions
- Test function naming language and format
- Import ordering
- Parametrization style

For new projects, choose a consistent style. For existing projects, match the established conventions.

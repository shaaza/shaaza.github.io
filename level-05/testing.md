---
layout: page
title: "Testing: Unit, Integration, Property-Based, Mutation"
description: "Why testing is non-negotiable — especially with AI-generated code. From unit tests to mutation testing."
permalink: /level-05/testing/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-05/">Level 0.5</a> / Testing'
---

## Why Testing Is Non-Negotiable

Testing has always been important. With AI-generated code, it is **critical**.

When a human writes code, they build a mental model of how it works. They understand the edge cases they considered and the ones they skipped. When an AI agent generates code, *you have no such model*. The code might look correct. It might pass a quick glance. But without tests, you have no evidence that it actually works.

> "Code without tests is broken by default. AI-generated code without tests is broken by assumption."

Testing is your primary tool for **verifying** that generated code does what it should, **catching** regressions when code changes, and **specifying** behavior before code is written.

---

## Types of Tests

### Unit Tests

Test individual functions or methods in isolation. The foundation of every test suite.

```python
# Function to test
def calculate_discount(price: float, discount_pct: float) -> float:
    """Apply a percentage discount to a price. Returns the discounted price."""
    if discount_pct < 0 or discount_pct > 100:
        raise ValueError("Discount must be between 0 and 100")
    return price * (1 - discount_pct / 100)

# Unit tests
def test_calculate_discount_basic():
    assert calculate_discount(100, 10) == 90.0

def test_calculate_discount_zero():
    assert calculate_discount(100, 0) == 100.0

def test_calculate_discount_full():
    assert calculate_discount(100, 100) == 0.0

def test_calculate_discount_invalid():
    with pytest.raises(ValueError):
        calculate_discount(100, -5)
    with pytest.raises(ValueError):
        calculate_discount(100, 150)
```

**Properties of good unit tests:**
- **Fast** — milliseconds, not seconds
- **Isolated** — no database, no network, no file system
- **Deterministic** — same result every time
- **Focused** — one assertion per test (ideally)

### Integration Tests

Test that multiple modules work together correctly. Slower than unit tests, but catch interface mismatches.

```python
# Integration test: does the order pipeline work end-to-end?
def test_order_pipeline():
    # Set up
    db = create_test_database()
    user = create_test_user(db)
    product = create_test_product(db, price=50.00, stock=10)

    # Act: create an order through the full pipeline
    order = OrderService(db).create_order(
        user_id=user.id,
        items=[{"product_id": product.id, "quantity": 2}]
    )

    # Assert: order was created correctly
    assert order.total == 100.00
    assert order.status == "pending"

    # Assert: stock was decremented
    updated_product = db.get_product(product.id)
    assert updated_product.stock == 8
```

### End-to-End (E2E) Tests

Test the entire system from the user's perspective. Slowest but highest confidence.

```python
# E2E test using a browser automation tool
def test_user_can_complete_purchase():
    browser.goto("/")
    browser.click("Sign In")
    browser.fill("email", "test@example.com")
    browser.fill("password", "secret")
    browser.click("Submit")

    browser.click("Add to Cart")
    browser.click("Checkout")
    browser.fill("card_number", "4242424242424242")
    browser.click("Pay Now")

    assert browser.text_content(".confirmation") == "Order confirmed!"
```

### Property-Based Tests

Instead of writing specific examples, describe **properties** that should always hold. The testing framework generates hundreds of random inputs.

```python
from hypothesis import given, strategies as st

# Property: reversing a list twice gives the original list
@given(st.lists(st.integers()))
def test_reverse_reverse_identity(xs):
    assert list(reversed(list(reversed(xs)))) == xs

# Property: sorting is idempotent (sorting a sorted list changes nothing)
@given(st.lists(st.integers()))
def test_sort_idempotent(xs):
    sorted_once = sorted(xs)
    sorted_twice = sorted(sorted_once)
    assert sorted_once == sorted_twice

# Property: sorted output has same length as input
@given(st.lists(st.integers()))
def test_sort_preserves_length(xs):
    assert len(sorted(xs)) == len(xs)

# Property: every element in sorted output is <= the next
@given(st.lists(st.integers(), min_size=2))
def test_sort_is_ordered(xs):
    result = sorted(xs)
    for i in range(len(result) - 1):
        assert result[i] <= result[i + 1]

# Property: sorted output is a permutation of input
@given(st.lists(st.integers()))
def test_sort_is_permutation(xs):
    from collections import Counter
    assert Counter(sorted(xs)) == Counter(xs)
```

**Why property-based testing matters:** You specify *what* should be true, not *which specific inputs to check*. The framework finds edge cases you would never think of — empty lists, lists with duplicates, very large numbers, negative numbers.

### Fuzz Testing

Feed random, malformed, or unexpected input to find crashes, hangs, and security vulnerabilities.

```python
# Fuzz test a JSON parser
import atheris
import json

def fuzz_json_parser(data):
    try:
        json.loads(data)
    except (json.JSONDecodeError, UnicodeDecodeError):
        pass  # Expected — invalid input should raise, not crash
    # If we get here without an exception, the parser handled it
    # If the process crashes (segfault, etc.), that's a bug!

atheris.Setup(sys.argv, fuzz_json_parser)
atheris.Fuzz()
```

Fuzz testing is how Google found thousands of bugs in Chrome, and how security researchers find vulnerabilities in parsers, codecs, and protocol implementations.

### Mutation Testing

The test that tests your tests. **Mutation testing** makes small changes (mutations) to your source code and checks whether your tests catch them.

```python
# Original code
def is_adult(age: int) -> bool:
    return age >= 18

# Mutations the tool generates:
# Mutation 1: age > 18   (changed >= to >)
# Mutation 2: age >= 17  (changed 18 to 17)
# Mutation 3: age >= 19  (changed 18 to 19)
# Mutation 4: age <= 18  (changed >= to <=)
# Mutation 5: return True (replaced expression with constant)

# If your tests pass with a mutation, that's a SURVIVING mutant.
# Surviving mutants mean your tests are too weak.
```

If changing `>=` to `>` does not cause any test to fail, then you are missing a test for the boundary case `age == 18`. Mutation testing measures the **quality** of your test suite, not just its coverage.

### Contract and Snapshot Tests

**Contract tests** verify that module interfaces match their specification:

```python
# Contract: the UserService must implement these methods
def test_user_service_contract():
    service = UserService()
    assert hasattr(service, 'create')
    assert hasattr(service, 'find')
    assert hasattr(service, 'delete')

    # create must return a User with an id
    user = service.create(name="Alice", email="alice@example.com")
    assert hasattr(user, 'id')
    assert user.name == "Alice"
```

**Snapshot tests** capture output and flag any changes:

```python
def test_api_response_format(snapshot):
    response = client.get("/api/users/1")
    snapshot.assert_match(response.json())
    # First run: saves the response as the "snapshot"
    # Subsequent runs: fails if the response changes
```

---

## Testing Types Comparison Table

| Test Type | Speed | Scope | Catches | Typical Count |
|-----------|-------|-------|---------|--------------|
| **Unit** | Milliseconds | Single function | Logic errors, edge cases | Hundreds to thousands |
| **Integration** | Seconds | Multiple modules | Interface mismatches, wiring bugs | Dozens to hundreds |
| **E2E** | Minutes | Entire system | User-facing workflow failures | Tens |
| **Property-based** | Seconds | Single function, many inputs | Unexpected edge cases, invariant violations | Tens (generates thousands of cases) |
| **Fuzz** | Hours | Input parsing, boundaries | Crashes, security vulnerabilities, hangs | Few (runs millions of inputs) |
| **Mutation** | Minutes | Test suite quality | Weak or missing test assertions | Meta-test (runs against all your tests) |
| **Contract** | Milliseconds | Module interfaces | API drift, breaking changes | Per interface |
| **Snapshot** | Milliseconds | Output format | Unintended output changes | Per endpoint/component |

---

## The Test Pyramid

```
        /\
       /  \       E2E Tests (few, slow, high confidence)
      /    \
     /------\
    /        \    Integration Tests (moderate number, moderate speed)
   /          \
  /------------\
 /              \  Unit Tests (many, fast, focused)
/________________\
```

Most of your tests should be unit tests. Fewer integration tests. Even fewer E2E tests. This gives you fast feedback (unit tests catch most bugs in seconds) with high confidence (E2E tests verify the system works as a whole).

---

## Testing Strategies for AI-Generated Code

AI-generated code demands specific testing strategies. Here are the key approaches:

### 1. Test-First, Generate-Second

Write the tests **before** asking the AI to generate code. The tests become the specification.

```python
# Step 1: Write tests (human)
def test_parse_csv_basic():
    result = parse_csv("name,age\nAlice,30\nBob,25")
    assert result == [{"name": "Alice", "age": "30"},
                      {"name": "Bob", "age": "25"}]

def test_parse_csv_empty():
    result = parse_csv("")
    assert result == []

def test_parse_csv_quoted_fields():
    result = parse_csv('name,bio\nAlice,"likes, commas"')
    assert result == [{"name": "Alice", "bio": "likes, commas"}]

def test_parse_csv_newline_in_quotes():
    result = parse_csv('name,bio\nAlice,"line1\nline2"')
    assert result == [{"name": "Alice", "bio": "line1\nline2"}]

# Step 2: Give tests to AI agent as specification
# "Implement parse_csv so that all these tests pass."

# Step 3: Run the tests against the generated code
# If they pass: the code meets your specification
# If they fail: the agent needs to try again
```

### 2. Property-Based Testing for AI Code

Property-based tests are especially powerful for AI-generated code because they test *invariants* rather than *specific examples*. The AI cannot just memorize the test cases.

```python
from hypothesis import given, strategies as st

# Properties for a sort function — regardless of how it's implemented
@given(st.lists(st.integers()))
def test_ai_sort_preserves_elements(xs):
    result = ai_generated_sort(xs)
    assert sorted(result) == sorted(xs)  # Same elements

@given(st.lists(st.integers()))
def test_ai_sort_is_ordered(xs):
    result = ai_generated_sort(xs)
    assert all(result[i] <= result[i+1] for i in range(len(result)-1))

@given(st.lists(st.integers()))
def test_ai_sort_is_stable(xs):
    # For equal elements, original order is preserved
    indexed = list(enumerate(xs))
    result = ai_generated_sort_stable(indexed, key=lambda x: x[1])
    for i in range(len(result) - 1):
        if result[i][1] == result[i+1][1]:
            assert result[i][0] < result[i+1][0]
```

### 3. Mutation Testing as a Quality Gate

Run mutation testing on AI-generated test suites. If the AI wrote your tests, mutation testing tells you whether those tests are actually thorough.

```bash
# Using mutmut (Python)
$ mutmut run --paths-to-mutate=src/ --tests-dir=tests/

# Results:
# 150 mutants generated
# 142 killed (tests caught the mutation)
# 8 survived (tests are too weak here)
# Mutation score: 94.7%

# Investigate survivors:
$ mutmut show 23
# Mutant 23: Changed 'x >= 0' to 'x > 0' in validation.py:15
# This means: no test checks the boundary case x == 0
```

### 4. Regression Suites

Maintain a growing suite of tests that capture every bug found in AI-generated code. Every time the AI produces buggy code, add a test for that specific bug.

```python
# Regression test: AI once generated code that failed on empty input
def test_regression_empty_input():
    """Regression: v1 agent code crashed on empty input (issue #42)"""
    assert process_data([]) == []

# Regression test: AI once generated code with off-by-one error
def test_regression_off_by_one():
    """Regression: v2 agent code missed last element (issue #57)"""
    assert process_data([1, 2, 3]) == [2, 4, 6]  # All three elements
```

---

## Testing in Practice: A Worked Example

Let us test a function that calculates shipping costs:

```python
# Specification:
# - Orders under $25: $5.99 shipping
# - Orders $25-$99.99: $3.99 shipping
# - Orders $100+: free shipping
# - International: add $15 surcharge
# - Oversized items: add $10 surcharge

def calculate_shipping(
    subtotal: float,
    is_international: bool = False,
    has_oversized: bool = False
) -> float:
    if subtotal >= 100:
        base = 0.0
    elif subtotal >= 25:
        base = 3.99
    else:
        base = 5.99

    if is_international:
        base += 15.00
    if has_oversized:
        base += 10.00

    return round(base, 2)
```

```python
# Unit tests: cover every branch
class TestCalculateShipping:
    def test_small_order(self):
        assert calculate_shipping(10.00) == 5.99

    def test_medium_order(self):
        assert calculate_shipping(50.00) == 3.99

    def test_large_order(self):
        assert calculate_shipping(150.00) == 0.00

    # Boundary tests
    def test_boundary_25(self):
        assert calculate_shipping(25.00) == 3.99

    def test_boundary_just_under_25(self):
        assert calculate_shipping(24.99) == 5.99

    def test_boundary_100(self):
        assert calculate_shipping(100.00) == 0.00

    def test_boundary_just_under_100(self):
        assert calculate_shipping(99.99) == 3.99

    # Surcharges
    def test_international(self):
        assert calculate_shipping(50.00, is_international=True) == 18.99

    def test_oversized(self):
        assert calculate_shipping(50.00, has_oversized=True) == 13.99

    def test_both_surcharges(self):
        assert calculate_shipping(50.00, is_international=True,
                                  has_oversized=True) == 28.99

    def test_free_shipping_with_surcharges(self):
        assert calculate_shipping(150.00, is_international=True,
                                  has_oversized=True) == 25.00
```

```python
# Property-based tests: describe invariants
@given(st.floats(min_value=0, max_value=10000))
def test_shipping_never_negative(subtotal):
    result = calculate_shipping(subtotal)
    assert result >= 0

@given(st.floats(min_value=0, max_value=10000))
def test_international_always_more(subtotal):
    domestic = calculate_shipping(subtotal)
    international = calculate_shipping(subtotal, is_international=True)
    assert international >= domestic + 14.99  # At least $15 more (floating point)

@given(st.floats(min_value=100, max_value=10000))
def test_free_shipping_threshold(subtotal):
    result = calculate_shipping(subtotal)
    assert result == 0.0  # Orders >= $100 always free (before surcharges)
```

---

## Key Takeaways

1. **Testing is verification.** Without tests, you are guessing. With AI-generated code, you are guessing blindly.

2. **Different test types catch different bugs.** Unit tests catch logic errors. Integration tests catch wiring errors. Property-based tests catch edge cases. Mutation tests catch weak tests.

3. **Test-first is the AI-era default.** Write tests before generating code. Tests are the specification.

4. **Property-based tests are your best friend.** They describe *what should be true* rather than *what specific outputs to expect*. They find edge cases you would never think of.

5. **Mutation testing measures test quality.** Code coverage tells you which lines are executed. Mutation testing tells you whether your tests would actually catch bugs in those lines.

---

<p style="font-size: 0.9rem;">
  <a href="/level-05/modularity/">← Modularity</a> · Next: <a href="/level-05/formal-methods/">Formal Methods: CSP, Z, TLA+ & Industry Case Studies →</a>
</p>

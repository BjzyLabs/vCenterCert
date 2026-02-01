# Testing Standards

Comprehensive testing ensures code quality, reduces bugs, and enables confident deployments.

## Test-Driven Development (Mandatory)

**All development must follow the TDD cycle:**

1. **Red (Write Test)**: Create a failing test that defines the expected behavior.
    - *Why?* Proven capability gap, clear acceptance criteria.
2. **Green (Implement)**: Write the minimal amount of code to make the test pass.
    - *Why?* Avoids over-engineering, ensures focus.
3. **Refactor**: Improve code quality, naming, and structure while keeping tests green.
    - *Why?* Maintains maintainability without breaking functionality.

**Test Types by Context (Implementation Mapping)**

| Context | Tool | Method | File/Pattern |
| :--- | :--- | :--- | :--- |
| **Ansible Roles** | Molecule | `verify.yml` | `roles/<name>/molecule/default/verify.yml` |
| **Python Functions** | pytest | Fixtures | `tests/test_*.py` |
| **AWX Playbooks** | AWX/Ansible | `operation=test` | `playbooks/*-manage.yml` |
| **Infrastructure** | Ansible | Verification Tasks | Post-tasks in playbooks |

**Rule**: Never skip writing tests. If a feature is worth building, it is worth testing.
**Intent**: Tests are the executable documentation of the system.


## Testing Pyramid

```
     E2E (10%)
    Integration (20%)
    Unit Tests (70%)
```

### Unit Tests (70%)
- Fast execution (< 1 second)
- No external dependencies
- Single responsibility focus
- Deterministic results

### Integration Tests (20%)
- Test component interactions
- May use test databases
- Verify API contracts
- Isolated from external services (mocked)

### End-to-End Tests (10%)
- Complete user workflows
- Minimal but critical coverage
- Real-world scenarios only

## Test Organization

```
project/
├── src/
│   ├── app/
│   ├── services/
│   └── models/
└── tests/
    ├── unit/
    ├── integration/
    ├── e2e/
    └── fixtures/
```

## Naming Conventions

- **Files**: `test_*.py` or `*_test.py`
- **Classes**: `Test*`
- **Methods**: `test_<function>_<scenario>_<expectation>`

**Examples:**
```python
test_calculate_tax_with_valid_inputs_returns_correct_amount()
test_create_user_with_duplicate_email_raises_validation_error()
test_login_with_invalid_credentials_returns_401()
```

## Test Structure (Arrange-Act-Assert)

```python
def test_user_creation_sets_email():
    # Arrange - Set up test data
    user_data = {"email": "test@example.com", "name": "Test User"}
    
    # Act - Execute the function
    user = create_user(**user_data)
    
    # Assert - Verify the result
    assert user.email == "test@example.com"
```

## Mocking Strategy

Mock external dependencies:
- API calls
- Database operations (in unit tests)
- File system operations
- Time-dependent operations

```python
from unittest.mock import patch

@patch('myapp.services.external_api')
def test_service_with_mocked_api(mock_api):
    mock_api.fetch.return_value = {"id": 123}
    
    result = my_service.process()
    
    assert result.id == 123
    mock_api.fetch.assert_called_once()
```

## Test Data Management

Use fixtures and factories for reusable test data:

```python
@pytest.fixture
def sample_user(db):
    user = User(email="test@example.com", name="Test User")
    db.add(user)
    db.commit()
    return user

# Usage
def test_user_lookup(sample_user):
    result = find_user(sample_user.email)
    assert result.id == sample_user.id
```

## Common Patterns

### Testing Error Conditions
```python
def test_invalid_email_raises_validation_error():
    with pytest.raises(ValidationError, match="valid email"):
        validate_email("invalid-email")
```

### Parameterized Tests
```python
@pytest.mark.parametrize("input,expected", [
    (10, 20),
    (5, 10),
    (0, 0),
])
def test_double(input, expected):
    assert double(input) == expected
```

## Coverage Guidelines

- **Target**: 80-90% line coverage
- **Focus**: Critical business logic
- **Avoid**: 100% coverage requirement (diminishing returns)
- **Principle**: Quality over quantity

## Linting Configuration

### pytest.ini
```ini
[pytest]
testpaths = tests
python_files = test_*.py *_test.py
python_classes = Test*
python_functions = test_*
addopts = --verbose --tb=short --cov=src --cov-fail-under=80
```

### Jest Configuration
```javascript
module.exports = {
  testEnvironment: 'node',
  roots: ['<rootDir>/src', '<rootDir>/tests'],
  collectCoverageFrom: ['src/**/*.js', '!src/**/*.d.ts'],
  coverageThreshold: {
    global: { lines: 80, functions: 80, branches: 80 }
  }
};
```

## CI/CD Integration

```yaml
# .github/workflows/test.yml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
      - run: pip install -r requirements-test.txt
      - run: pytest tests/ -v --cov=src
      - run: pylint src tests
```

## Best Practices Checklist

✅ **Do This:**
- Write tests during development
- Use clear, descriptive test names
- Follow Arrange-Act-Assert pattern
- Keep tests independent and isolated
- Test both success and error cases
- Mock external dependencies
- Maintain test data isolation
- Regular test suite cleanup

❌ **Avoid This:**
- Writing tests after everything is done
- Testing implementation details
- Overly complex test setups
- Sharing state between tests
- Ignoring flaky/slow tests
- Chasing high coverage numbers
- Testing everything through UI
- Neglecting test maintenance

## Performance Testing

```python
def test_operation_completes_within_threshold():
    start = time.time()
    result = expensive_operation()
    duration = time.time() - start
    
    assert duration < 0.1  # 100ms threshold
    assert result is not None
```

## Test Maintenance

- Review test data regularly
- Remove obsolete tests
- Refactor duplicate test code
- Monitor test execution times
- Track and fix flaky tests
- Update for new requirements

---

*Good tests are your safety net. Invest in them early.*


# Pytest Basics

Here are the **Pytest Basics** you need to know to write and run tests effectively.

### 1. Naming Rules (Discovery)
Pytest relies on naming conventions to automatically find your tests. If you don't name them correctly, they won't run.
*   **Files**: Must start with `test_` (e.g., `test_login.py`) or end with `_test.py`.
*   **Functions**: Must start with `test_` (e.g., `def test_password_hash():`).
*   **Classes**: Must start with `Test` (e.g., `class TestUserRoutes:`).

### 2. The `assert` Statement
In standard Python, `assert` is rarely used, but in Pytest, it's everything. You use it to check if a condition is true.
```python
assert 5 == 5          # Pass
assert 5 == 6          # Fail
assert "hello" in "hello world"  # Pass
```
*Tip: Pytest analyzes your asserts to give you detailed error messages, showing exactly what differed.*

### 3. Running Tests (CLI Commands)
You run tests from the terminal. Here are the most useful commands:
*   `pytest`: Runs all tests in the current directory and subdirectories.
*   **`pytest -v`**: **Verbose** mode. Shows each test name and whether it passed/failed.
*   **`pytest -s`**: **Show output**. By default, Pytest hides `print()` statements. Use this if you debugging with print.
*   `pytest -k "login"`: **Keyword** search. Only runs tests with "login" in their name (e.g., `test_login_success`).
*   `pytest tests/test_utils.py`: Runs only the tests in that specific file.
*   `pytest --last-failed`: Runs only the tests that failed in the last run (great for fixing bugs).

### 4. Parametrization (Running one test with multiple inputs)
This is a superpower of Pytest. Instead of writing 3 separate tests for "add", "subtract", and "multiply", you write one test and feed it a list of inputs.

**Example**:
```python
import pytest

# This runs the test 3 times with different inputs
@pytest.mark.parametrize("input_password, expected_result", [
    ("password123", True),
    ("wrongpass", False),
    ("", False)
])
def test_password_validation(input_password, expected_result):
    result = is_valid(input_password)
    assert result == expected_result
```

### 5. Markers (Tagging Tests)
You can tag tests to group them or change how they run.
*   `@pytest.mark.skip(reason="Not finished yet")`: Skips this test.
*   `@pytest.mark.xfail`: "Expected Fail". Use this if you know a test will fail but you don't want it to break the build (e.g., testing a bug you haven't fixed yet).

### Summary Cheat Sheet
| Concept | Code/Command | Description |
| :--- | :--- | :--- |
| **Check** | `assert x == y` | Fails the test if not equal. |
| **Fixture** | `@pytest.fixture` | Setup code (like creating a database connection). |
| **Run All** | `pytest` | Finds and runs all `test_*.py` files. |
| **Debug** | `pytest -s` | Shows `print()` output. |
| **Filter** | `pytest -k "name"` | Runs only tests matching "name". |

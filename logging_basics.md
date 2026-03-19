# Python Logging Basics

Logging is the art of recording events while your application runs. It's crucial for debugging and monitoring, especially in production where you can't see the console.

### 1. Why Log instead of Print?
*   **Levels**: You can categorize messages (e.g., "Just info" vs "Critical Crash").
*   **Destinations**: Send logs to a file, the console, an email, or a centralized server simultaneously.
*   **Metadata**: Automatically record timestamps, filenames, and line numbers.
*   **Control**: You can turn off "Debug" logs in production without changing code.

### 2. The 5 Levels of Severity
Each message has a level. You configure your logger to ignore anything below a certain level.

1.  **DEBUG** (`logging.debug()`): Detailed info for diagnosing problems. (e.g., "Variable x = 5")
2.  **INFO** (`logging.info()`): Confirmation that things are working as expected. (e.g., "User logged in")
3.  **WARNING** (`logging.warning()`): Something unexpected happened, but the app is still running. (e.g., "Disk space low")
4.  **ERROR** (`logging.error()`): The app failed to perform a specific function. (e.g., "Database connection failed")
5.  **CRITICAL** (`logging.critical()`): The app is crashing. (e.g., "Out of memory")

### 3. Basic Configuration
The simplest way to start is `logging.basicConfig`.

```python
import logging

# Configure the logger
logging.basicConfig(
    level=logging.INFO,  # Only show INFO and above (WARNING, ERROR, CRITICAL)
    format='%(asctime)s - %(levelname)s - %(message)s',
    filename='app.log'   # Write to a file instead of console
)

logging.debug("This won't show")
logging.info("This will show")
logging.error("This will definitely show")
```

### 4. Advanced: Loggers, Handlers, and Formatters
For real apps, you don't just use the root logger. You create your own.

*   **Logger**: The entry point. Always name it `__name__` so it tracks the module name.
    ```python
    logger = logging.getLogger(__name__)
    ```
*   **Handler**: Where the logs go.
    *   `StreamHandler`: Console (Terminal).
    *   `FileHandler`: A text file.
    *   `RotatingFileHandler`: Splits files when they get too big.
*   **Formatter**: How the log looks (Text structure).

### 5. Example: Professional Setup
Here is how you typically set it up in a `main.py` or configuration file:

```python
import logging
import sys

# 1. Create Logger
logger = logging.getLogger("my_app")
logger.setLevel(logging.DEBUG)

# 2. Create Handlers
console_handler = logging.StreamHandler(sys.stdout)
file_handler = logging.FileHandler("app.log")

# 3. Create Formatters
formatter = logging.Formatter('%(asctime)s | %(name)s | %(levelname)s | %(message)s')
console_handler.setFormatter(formatter)
file_handler.setFormatter(formatter)

# 4. Add Handlers to Logger
logger.addHandler(console_handler)
logger.addHandler(file_handler)

# Usage
logger.info("Application starting...")
```

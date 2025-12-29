# F04: Logging Infrastructure - Pre-Implementation Documentation

## 1. Implementation Overview

### 1.1 Core Components

This feature implements structured logging infrastructure:

1. **Logger** - Main logging interface with level methods
2. **JsonFormatter** - Structured JSON log formatter
3. **TextFormatter** - Human-readable console formatter
4. **Qt Message Handler** - Integration with Qt framework logging

### 1.2 Key Design Patterns

- **Facade Pattern**: Logger wraps Python logging module
- **Strategy Pattern**: Pluggable formatters
- **Singleton Pattern**: Global logger instance

---

## 2. File Structure

```
src/
├── infrastructure/
│   └── logging/
│       ├── __init__.py
│       ├── logger.py            # Logger class
│       ├── formatters.py        # JsonFormatter, TextFormatter
│       ├── handlers.py          # Qt message handler
│       └── config.py            # Logging configuration
│
tests/
├── unit/
│   └── infrastructure/
│       └── logging/
│           ├── test_logger.py
│           ├── test_formatters.py
│           └── test_qt_handler.py
│
└── integration/
    └── test_logging.py
```

---

## 3. Component Specifications

### 3.1 Logger

**File**: `src/infrastructure/logging/logger.py`

```python
import logging
import logging.handlers
from pathlib import Path
from typing import Any, Optional
from datetime import datetime


class Logger:
    """Application logger with structured logging"""

    def __init__(self, name: str = "JTS"):
        """
        Initialize logger

        Args:
            name: Logger name (default: "JTS")
        """
        self._logger = logging.getLogger(name)
        self._logger.setLevel(logging.DEBUG)
        self._logger.propagate = False  # Don't propagate to root logger

    @classmethod
    def create_default(cls, log_dir: Optional[Path] = None) -> 'Logger':
        """
        Create logger with default configuration

        Args:
            log_dir: Optional log directory (default: ~/.jts/logs)

        Returns:
            Configured logger instance
        """
        logger = cls()

        # Determine log directory
        if log_dir is None:
            log_dir = Path.home() / ".jts" / "logs"

        log_dir.mkdir(parents=True, exist_ok=True)

        # Add rotating file handler (JSON format)
        log_file = log_dir / "jts.log"
        file_handler = logging.handlers.RotatingFileHandler(
            str(log_file),
            maxBytes=10 * 1024 * 1024,  # 10MB
            backupCount=5,
            encoding='utf-8'
        )
        file_handler.setLevel(logging.DEBUG)
        file_handler.setFormatter(JsonFormatter())
        logger._logger.addHandler(file_handler)

        # Add console handler (text format)
        console_handler = logging.StreamHandler()
        console_handler.setLevel(logging.INFO)
        console_handler.setFormatter(TextFormatter())
        logger._logger.addHandler(console_handler)

        return logger

    def debug(self, message: str, **kwargs: Any) -> None:
        """
        Log debug message

        Args:
            message: Log message
            **kwargs: Additional context fields
        """
        self._log(logging.DEBUG, message, **kwargs)

    def info(self, message: str, **kwargs: Any) -> None:
        """
        Log info message

        Args:
            message: Log message
            **kwargs: Additional context fields
        """
        self._log(logging.INFO, message, **kwargs)

    def warning(self, message: str, **kwargs: Any) -> None:
        """
        Log warning message

        Args:
            message: Log message
            **kwargs: Additional context fields
        """
        self._log(logging.WARNING, message, **kwargs)

    def error(self, message: str, **kwargs: Any) -> None:
        """
        Log error message

        Args:
            message: Log message
            **kwargs: Additional context fields
        """
        self._log(logging.ERROR, message, **kwargs)

    def critical(self, message: str, **kwargs: Any) -> None:
        """
        Log critical message

        Args:
            message: Log message
            **kwargs: Additional context fields
        """
        self._log(logging.CRITICAL, message, **kwargs)

    def exception(
        self,
        message: str,
        exc_info: Optional[Exception] = None,
        **kwargs: Any
    ) -> None:
        """
        Log exception with traceback

        Args:
            message: Log message
            exc_info: Exception instance (default: current exception)
            **kwargs: Additional context fields
        """
        self._log(logging.ERROR, message, exc_info=exc_info or True, **kwargs)

    def _log(self, level: int, message: str, **kwargs: Any) -> None:
        """
        Internal logging method with context

        Args:
            level: Log level (logging.DEBUG, etc.)
            message: Log message
            **kwargs: Additional context (stored in extra)
        """
        # Extract exc_info if present
        exc_info = kwargs.pop('exc_info', None)

        # Store remaining kwargs as context
        extra = {"context": kwargs} if kwargs else {}

        # Log with extra context
        self._logger.log(level, message, extra=extra, exc_info=exc_info)

    def get_log_file(self) -> Optional[Path]:
        """
        Get log file path

        Returns:
            Path to log file, or None if not configured
        """
        for handler in self._logger.handlers:
            if isinstance(handler, logging.handlers.RotatingFileHandler):
                return Path(handler.baseFilename)
        return None
```

---

### 3.2 Formatters

**File**: `src/infrastructure/logging/formatters.py`

```python
import logging
import json
from datetime import datetime
from typing import Any


class JsonFormatter(logging.Formatter):
    """JSON log formatter for structured logging"""

    def format(self, record: logging.LogRecord) -> str:
        """
        Format log record as JSON

        Args:
            record: Log record to format

        Returns:
            JSON string
        """
        # Build log data dict
        log_data = {
            "timestamp": datetime.fromtimestamp(record.created).isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno,
            "thread": record.thread,
            "thread_name": record.threadName,
        }

        # Add exception info if present
        if record.exc_info:
            log_data["exception"] = self.formatException(record.exc_info)

        # Add extra context if present
        if hasattr(record, "context"):
            log_data["context"] = record.context

        return json.dumps(log_data, ensure_ascii=False, default=str)


class TextFormatter(logging.Formatter):
    """Human-readable text formatter for console output"""

    # ANSI color codes
    COLORS = {
        'DEBUG': '\033[36m',      # Cyan
        'INFO': '\033[32m',       # Green
        'WARNING': '\033[33m',    # Yellow
        'ERROR': '\033[31m',      # Red
        'CRITICAL': '\033[35m',   # Magenta
        'RESET': '\033[0m',       # Reset
    }

    def __init__(self, use_colors: bool = True):
        """
        Initialize text formatter

        Args:
            use_colors: Enable colored output (default: True)
        """
        super().__init__(
            fmt="%(asctime)s [%(levelname)s] %(name)s - %(message)s",
            datefmt="%Y-%m-%d %H:%M:%S"
        )
        self._use_colors = use_colors

    def format(self, record: logging.LogRecord) -> str:
        """
        Format log record as colored text

        Args:
            record: Log record to format

        Returns:
            Formatted string
        """
        # Format base message
        formatted = super().format(record)

        # Add colors if enabled
        if self._use_colors:
            color = self.COLORS.get(record.levelname, '')
            reset = self.COLORS['RESET']
            formatted = f"{color}{formatted}{reset}"

        # Add context if present
        if hasattr(record, "context") and record.context:
            context_str = " | ".join(f"{k}={v}" for k, v in record.context.items())
            formatted += f" | {context_str}"

        return formatted
```

---

### 3.3 Qt Message Handler

**File**: `src/infrastructure/logging/handlers.py`

```python
from PySide6.QtCore import QtMsgType, qInstallMessageHandler, QtMessageHandler
from typing import Optional
from .logger import Logger


# Global logger instance for Qt messages
_qt_logger: Optional[Logger] = None


def qt_message_handler(
    msg_type: QtMsgType,
    context,
    message: str
) -> None:
    """
    Handle Qt framework messages

    Args:
        msg_type: Qt message type
        context: Qt message context
        message: Message text
    """
    global _qt_logger

    if _qt_logger is None:
        # Fallback if logger not initialized
        print(f"[Qt] {message}")
        return

    # Map Qt message types to logging levels
    type_map = {
        QtMsgType.QtDebugMsg: _qt_logger.debug,
        QtMsgType.QtInfoMsg: _qt_logger.info,
        QtMsgType.QtWarningMsg: _qt_logger.warning,
        QtMsgType.QtCriticalMsg: _qt_logger.error,
        QtMsgType.QtFatalMsg: _qt_logger.critical,
    }

    # Get appropriate logging function
    log_func = type_map.get(msg_type, _qt_logger.info)

    # Log with context
    log_func(
        f"[Qt] {message}",
        file=context.file if context.file else "unknown",
        line=context.line if context.line else 0,
        function=context.function if context.function else "unknown"
    )


def install_qt_message_handler(logger: Logger) -> None:
    """
    Install Qt message handler

    Args:
        logger: Logger instance for Qt messages
    """
    global _qt_logger
    _qt_logger = logger
    qInstallMessageHandler(qt_message_handler)


def uninstall_qt_message_handler() -> None:
    """Uninstall Qt message handler (restore default)"""
    global _qt_logger
    _qt_logger = None
    qInstallMessageHandler(None)
```

---

## 4. Implementation Order

### Phase 1: Core Logger (2 hours)
1. Implement `Logger` class with level methods
2. Implement `create_default()` with file and console handlers
3. Add rotating file handler configuration

### Phase 2: Formatters (1 hour)
4. Implement `JsonFormatter` for structured logs
5. Implement `TextFormatter` for console output
6. Add color support to TextFormatter

### Phase 3: Qt Integration (1 hour)
7. Implement `qt_message_handler()`
8. Implement `install_qt_message_handler()`
9. Test Qt message capture

### Phase 4: Testing (2 hours)
10. Write unit tests for Logger
11. Write unit tests for formatters
12. Write integration tests for file creation and rotation

---

## 5. Testing Strategy

### 5.1 Unit Tests

**File**: `tests/unit/infrastructure/logging/test_logger.py`

```python
import logging
import pytest


class TestLogger:
    """Unit tests for Logger"""

    def test_debug_logs_message(self, caplog):
        """debug() logs message at DEBUG level"""
        from infrastructure.logging.logger import Logger

        logger = Logger("test")
        logger._logger.setLevel(logging.DEBUG)

        with caplog.at_level(logging.DEBUG):
            logger.debug("test debug message")

        assert "test debug message" in caplog.text
        assert caplog.records[0].levelname == "DEBUG"

    def test_info_logs_message(self, caplog):
        """info() logs message at INFO level"""
        from infrastructure.logging.logger import Logger

        logger = Logger("test")

        with caplog.at_level(logging.INFO):
            logger.info("test info message")

        assert "test info message" in caplog.text
        assert caplog.records[0].levelname == "INFO"

    def test_exception_includes_traceback(self, caplog):
        """exception() includes traceback"""
        from infrastructure.logging.logger import Logger

        logger = Logger("test")

        try:
            raise ValueError("test error")
        except ValueError as e:
            with caplog.at_level(logging.ERROR):
                logger.exception("error occurred", exc_info=e)

        assert "error occurred" in caplog.text
        assert "ValueError: test error" in caplog.text
        assert "Traceback" in caplog.text

    def test_log_with_context(self, caplog):
        """Log methods accept context kwargs"""
        from infrastructure.logging.logger import Logger

        logger = Logger("test")

        with caplog.at_level(logging.INFO):
            logger.info("test message", user_id=123, action="login")

        record = caplog.records[0]
        assert hasattr(record, "context")
        assert record.context["user_id"] == 123
        assert record.context["action"] == "login"

    def test_create_default_creates_log_file(self, tmp_path):
        """create_default() creates log file"""
        from infrastructure.logging.logger import Logger

        log_dir = tmp_path / "logs"
        logger = Logger.create_default(log_dir=log_dir)

        logger.info("test message")

        log_file = log_dir / "jts.log"
        assert log_file.exists()
        assert log_file.read_text(encoding="utf-8")

    def test_get_log_file_returns_path(self, tmp_path):
        """get_log_file() returns log file path"""
        from infrastructure.logging.logger import Logger

        log_dir = tmp_path / "logs"
        logger = Logger.create_default(log_dir=log_dir)

        log_file = logger.get_log_file()

        assert log_file is not None
        assert log_file.name == "jts.log"
```

**File**: `tests/unit/infrastructure/logging/test_formatters.py`

```python
import logging
import json
import sys


class TestJsonFormatter:
    """Unit tests for JsonFormatter"""

    def test_format_returns_valid_json(self):
        """format() returns valid JSON string"""
        from infrastructure.logging.formatters import JsonFormatter

        formatter = JsonFormatter()
        record = logging.LogRecord(
            name="test",
            level=logging.INFO,
            pathname="test.py",
            lineno=10,
            msg="test message",
            args=(),
            exc_info=None
        )

        result = formatter.format(record)
        data = json.loads(result)

        assert data["message"] == "test message"
        assert data["level"] == "INFO"
        assert "timestamp" in data
        assert data["module"] == "test"
        assert data["line"] == 10

    def test_format_includes_exception(self):
        """format() includes exception info"""
        from infrastructure.logging.formatters import JsonFormatter

        formatter = JsonFormatter()

        try:
            raise ValueError("test error")
        except ValueError:
            exc_info = sys.exc_info()

        record = logging.LogRecord(
            name="test",
            level=logging.ERROR,
            pathname="test.py",
            lineno=10,
            msg="error occurred",
            args=(),
            exc_info=exc_info
        )

        result = formatter.format(record)
        data = json.loads(result)

        assert "exception" in data
        assert "ValueError: test error" in data["exception"]
        assert "Traceback" in data["exception"]

    def test_format_includes_context(self):
        """format() includes extra context"""
        from infrastructure.logging.formatters import JsonFormatter

        formatter = JsonFormatter()
        record = logging.LogRecord(
            name="test",
            level=logging.INFO,
            pathname="test.py",
            lineno=10,
            msg="test message",
            args=(),
            exc_info=None
        )
        record.context = {"user_id": 123, "action": "login"}

        result = formatter.format(record)
        data = json.loads(result)

        assert "context" in data
        assert data["context"]["user_id"] == 123
        assert data["context"]["action"] == "login"


class TestTextFormatter:
    """Unit tests for TextFormatter"""

    def test_format_returns_readable_text(self):
        """format() returns human-readable text"""
        from infrastructure.logging.formatters import TextFormatter

        formatter = TextFormatter(use_colors=False)
        record = logging.LogRecord(
            name="test",
            level=logging.INFO,
            pathname="test.py",
            lineno=10,
            msg="test message",
            args=(),
            exc_info=None
        )

        result = formatter.format(record)

        assert "test message" in result
        assert "[INFO]" in result
        assert "test" in result

    def test_format_with_colors(self):
        """format() includes ANSI colors when enabled"""
        from infrastructure.logging.formatters import TextFormatter

        formatter = TextFormatter(use_colors=True)
        record = logging.LogRecord(
            name="test",
            level=logging.ERROR,
            pathname="test.py",
            lineno=10,
            msg="test message",
            args=(),
            exc_info=None
        )

        result = formatter.format(record)

        # Check for ANSI escape codes
        assert "\033[" in result  # Color code
        assert "\033[0m" in result  # Reset code
```

### 5.2 Integration Tests

**File**: `tests/integration/test_logging.py`

```python
def test_log_rotation(tmp_path):
    """Log file rotates at size limit"""
    from infrastructure.logging.logger import Logger

    log_dir = tmp_path / "logs"
    logger = Logger.create_default(log_dir=log_dir)

    # Write logs until rotation occurs
    for i in range(10000):
        logger.info(f"Log message {i} with padding text" * 10)

    # Check for rotated files
    log_files = list(log_dir.glob("jts.log*"))
    assert len(log_files) > 1  # Original + at least one backup
```

---

## 6. Usage Examples

### 6.1 Basic Logging

```python
from infrastructure.logging.logger import Logger

logger = Logger.create_default()

logger.debug("Debug information", variable=value)
logger.info("Application started")
logger.warning("Deprecated API usage", api="old_method")
logger.error("Failed to connect", host="broker.com", port=20001)
logger.critical("Database connection lost")
```

### 6.2 Exception Logging

```python
try:
    risky_operation()
except Exception as e:
    logger.exception("Operation failed", operation="risky_operation", exc_info=e)
```

### 6.3 Structured Context

```python
logger.info(
    "User login successful",
    user_id=user.id,
    username=user.name,
    ip_address=request.ip,
    timestamp=datetime.now()
)
```

---

## 7. Common Pitfalls & Solutions

### 7.1 Logs Not Appearing

**Problem**: Log messages not showing up
**Cause**: Log level too high (e.g., file handler at ERROR, but logging INFO)
**Solution**: Check handler log levels in `create_default()`

### 7.2 Unicode Encoding Errors

**Problem**: Korean characters cause encoding errors
**Cause**: Wrong file encoding
**Solution**: Ensure `encoding='utf-8'` in RotatingFileHandler

### 7.3 Log File Not Rotating

**Problem**: Log file grows without rotating
**Cause**: File handler not configured correctly
**Solution**: Verify `maxBytes` and `backupCount` parameters

### 7.4 Performance Issues

**Problem**: Logging slows down application
**Cause**: Too many log messages or synchronous file I/O
**Solution**:
- Use appropriate log levels (avoid excessive DEBUG logs)
- Consider using QueueHandler for async logging

---

## 8. Log Levels Guidelines

| Level | When to Use | Example |
|-------|-------------|---------|
| **DEBUG** | Detailed diagnostic info | "Connecting to broker at 192.168.1.1:20001" |
| **INFO** | General informational | "Application started", "User logged in" |
| **WARNING** | Warning, app continues | "API rate limit approaching (90/100)" |
| **ERROR** | Error, feature may fail | "Failed to fetch portfolio data" |
| **CRITICAL** | Critical error, may crash | "Database connection lost" |

---

## 9. Performance Considerations

- **File I/O**: Rotating file handler is synchronous (~1-5ms per write)
- **JSON Serialization**: ~0.1ms per log with JsonFormatter
- **Target**: < 10ms per log message end-to-end

**Optimization**:
- Use QueueHandler for async logging in production
- Reduce log frequency for high-frequency events
- Use appropriate log levels

---

## 10. References

### 10.1 Documentation

- [Python Logging](https://docs.python.org/3/library/logging.html)
- [RotatingFileHandler](https://docs.python.org/3/library/logging.handlers.html#rotatingfilehandler)
- [Qt Message Handling](https://doc.qt.io/qtforpython/PySide6/QtCore/qInstallMessageHandler.html)

### 10.2 Related Specs

- E01 Epic Specification
- F03 DI Container (Logger as singleton)
- F05 Error Handling (Logger for error logging)

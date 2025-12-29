# F05: Error Handling - Pre-Implementation Documentation

## 1. Implementation Overview

### 1.1 Core Components

This feature implements centralized error handling:

1. **Exception Classes** - Hierarchy of application-specific exceptions
2. **ErrorHandler** - Central error handling service
3. **Error Dialog** - User-friendly error display
4. **Exception Hook** - Global unhandled exception catcher

### 1.2 Key Design Patterns

- **Exception Hierarchy**: Domain-specific exception types
- **Strategy Pattern**: Recovery strategies for errors
- **Template Method**: Standard error handling flow

---

## 2. File Structure

```
src/
├── domain/
│   └── exceptions/
│       ├── __init__.py
│       ├── base.py              # ApplicationError base class
│       ├── validation.py        # ValidationError
│       ├── broker.py            # Broker-related errors
│       └── data.py              # Data-related errors
│
├── infrastructure/
│   └── error_handling/
│       ├── __init__.py
│       ├── error_handler.py    # ErrorHandler class
│       └── exception_hook.py   # Global exception hook
│
└── presentation/
    └── dialogs/
        └── error_dialog.py     # Error display UI
│
tests/
├── unit/
│   ├── domain/
│   │   └── exceptions/
│   │       └── test_exceptions.py
│   └── infrastructure/
│       └── error_handling/
│           ├── test_error_handler.py
│           └── test_exception_hook.py
│
└── integration/
    └── test_error_handling.py
```

---

## 3. Component Specifications

### 3.1 Base Exception Classes

**File**: `src/domain/exceptions/base.py`

```python
from typing import Any, Optional


class ApplicationError(Exception):
    """Base class for all application errors"""

    def __init__(
        self,
        message: str,
        user_message: Optional[str] = None,
        recoverable: bool = True,
        **context: Any
    ):
        """
        Initialize application error

        Args:
            message: Technical error message (for logs)
            user_message: User-friendly message (for display)
            recoverable: Whether application can continue
            **context: Additional context data
        """
        super().__init__(message)
        self.user_message = user_message or message
        self.recoverable = recoverable
        self.context = context

    def __str__(self) -> str:
        """String representation includes context"""
        if self.context:
            context_str = ", ".join(f"{k}={v}" for k, v in self.context.items())
            return f"{super().__str__()} ({context_str})"
        return super().__str__()
```

**File**: `src/domain/exceptions/validation.py`

```python
from .base import ApplicationError


class ValidationError(ApplicationError):
    """Validation error (user input)"""

    def __init__(self, field: str, message: str, value: Any = None):
        """
        Initialize validation error

        Args:
            field: Field name that failed validation
            message: Validation error message
            value: Invalid value (optional)
        """
        super().__init__(
            message=f"Validation failed for field '{field}': {message}",
            user_message=f"입력 오류: {message}",
            recoverable=True,
            field=field,
            value=value
        )


class ConfigurationError(ApplicationError):
    """Configuration error (settings, environment)"""

    def __init__(self, setting: str, message: str):
        """
        Initialize configuration error

        Args:
            setting: Setting name
            message: Error message
        """
        super().__init__(
            message=f"Configuration error for '{setting}': {message}",
            user_message=f"설정 오류: {message}\n설정을 확인해주세요.",
            recoverable=True,
            setting=setting
        )
```

**File**: `src/domain/exceptions/broker.py`

```python
from .base import ApplicationError


class BrokerError(ApplicationError):
    """Base class for broker-related errors"""
    pass


class BrokerConnectionError(BrokerError):
    """Broker connection failed"""

    def __init__(self, broker: str, reason: str):
        """
        Initialize broker connection error

        Args:
            broker: Broker name
            reason: Connection failure reason
        """
        super().__init__(
            message=f"Failed to connect to {broker}: {reason}",
            user_message=f"증권사 연결 실패: {broker}\n잠시 후 다시 시도해주세요.",
            recoverable=True,
            broker=broker,
            reason=reason
        )


class BrokerAuthenticationError(BrokerError):
    """Broker authentication failed"""

    def __init__(self, broker: str):
        """
        Initialize authentication error

        Args:
            broker: Broker name
        """
        super().__init__(
            message=f"Authentication failed for {broker}",
            user_message=f"인증 실패: {broker}\nAPI 키를 확인해주세요.",
            recoverable=True,
            broker=broker
        )


class BrokerRateLimitError(BrokerError):
    """Broker rate limit exceeded"""

    def __init__(self, broker: str, retry_after: int):
        """
        Initialize rate limit error

        Args:
            broker: Broker name
            retry_after: Seconds until retry allowed
        """
        super().__init__(
            message=f"Rate limit exceeded for {broker}",
            user_message=f"요청 한도 초과: {broker}\n{retry_after}초 후 다시 시도해주세요.",
            recoverable=True,
            broker=broker,
            retry_after=retry_after
        )
```

**File**: `src/domain/exceptions/data.py`

```python
from .base import ApplicationError


class DataError(ApplicationError):
    """Base class for data-related errors"""
    pass


class DataNotFoundError(DataError):
    """Requested data not found"""

    def __init__(self, data_type: str, identifier: str):
        """
        Initialize data not found error

        Args:
            data_type: Type of data (e.g., "portfolio", "stock")
            identifier: Data identifier (e.g., stock code)
        """
        super().__init__(
            message=f"{data_type} not found: {identifier}",
            user_message=f"데이터를 찾을 수 없습니다: {identifier}",
            recoverable=True,
            data_type=data_type,
            identifier=identifier
        )


class DataFormatError(DataError):
    """Data format is invalid"""

    def __init__(self, expected: str, received: str):
        """
        Initialize data format error

        Args:
            expected: Expected format
            received: Received format
        """
        super().__init__(
            message=f"Invalid data format. Expected: {expected}, Received: {received}",
            user_message="데이터 형식이 올바르지 않습니다.",
            recoverable=True,
            expected=expected,
            received=received
        )
```

---

### 3.2 Error Handler

**File**: `src/infrastructure/error_handling/error_handler.py`

```python
from typing import Callable, Optional, Any
from PySide6.QtWidgets import QMessageBox, QWidget
from domain.exceptions.base import ApplicationError
from infrastructure.logging.logger import Logger


class ErrorHandler:
    """Centralized error handling service"""

    def __init__(self, logger: Logger, parent: Optional[QWidget] = None):
        """
        Initialize error handler

        Args:
            logger: Logger instance
            parent: Parent widget for dialogs
        """
        self._logger = logger
        self._parent = parent
        self._recovery_strategies: dict[type, Callable[[Exception], bool]] = {}

    def handle(self, error: Exception) -> bool:
        """
        Handle error with logging and user notification

        Args:
            error: Exception to handle

        Returns:
            True if error was recovered, False otherwise
        """
        # Log error
        self._log_error(error)

        # Show user message
        self._show_error_dialog(error)

        # Attempt recovery
        return self._attempt_recovery(error)

    def register_recovery_strategy(
        self,
        error_type: type,
        strategy: Callable[[Exception], bool]
    ) -> None:
        """
        Register recovery strategy for error type

        Args:
            error_type: Exception class to handle
            strategy: Recovery function (returns True if recovered)
        """
        self._recovery_strategies[error_type] = strategy

    def set_parent(self, parent: QWidget) -> None:
        """
        Set parent widget for dialogs

        Args:
            parent: Parent widget
        """
        self._parent = parent

    def _log_error(self, error: Exception) -> None:
        """
        Log error with context

        Args:
            error: Exception to log
        """
        if isinstance(error, ApplicationError):
            self._logger.error(
                str(error),
                error_type=type(error).__name__,
                user_message=error.user_message,
                recoverable=error.recoverable,
                **error.context
            )
        else:
            self._logger.exception(
                "Unhandled exception",
                exc_info=error,
                error_type=type(error).__name__
            )

    def _show_error_dialog(self, error: Exception) -> None:
        """
        Display error dialog to user

        Args:
            error: Exception to display
        """
        if isinstance(error, ApplicationError):
            title = "오류" if error.recoverable else "심각한 오류"
            message = error.user_message
            icon = QMessageBox.Warning if error.recoverable else QMessageBox.Critical
        else:
            title = "예상치 못한 오류"
            message = (
                "알 수 없는 오류가 발생했습니다.\n"
                "로그 파일을 확인하거나 개발자에게 문의해주세요."
            )
            icon = QMessageBox.Critical

        dialog = QMessageBox(icon, title, message, parent=self._parent)
        dialog.exec()

    def _attempt_recovery(self, error: Exception) -> bool:
        """
        Attempt to recover from error using registered strategy

        Args:
            error: Exception to recover from

        Returns:
            True if recovery succeeded, False otherwise
        """
        error_type = type(error)

        # Check for exact type match
        if error_type in self._recovery_strategies:
            return self._try_strategy(error, self._recovery_strategies[error_type])

        # Check for parent class matches
        for registered_type, strategy in self._recovery_strategies.items():
            if isinstance(error, registered_type):
                return self._try_strategy(error, strategy)

        return False

    def _try_strategy(self, error: Exception, strategy: Callable) -> bool:
        """
        Try recovery strategy with error handling

        Args:
            error: Original error
            strategy: Recovery function

        Returns:
            True if recovery succeeded, False otherwise
        """
        try:
            return strategy(error)
        except Exception as recovery_error:
            self._logger.error(
                "Recovery strategy failed",
                original_error=str(error),
                recovery_error=str(recovery_error),
                error_type=type(error).__name__
            )
            return False


# Global error handler instance
_error_handler: Optional[ErrorHandler] = None


def get_error_handler() -> ErrorHandler:
    """
    Get global error handler instance

    Returns:
        Global error handler

    Raises:
        RuntimeError: If error handler not initialized
    """
    if _error_handler is None:
        raise RuntimeError("Error handler not initialized. Call set_error_handler() first.")
    return _error_handler


def set_error_handler(handler: ErrorHandler) -> None:
    """
    Set global error handler instance

    Args:
        handler: ErrorHandler instance
    """
    global _error_handler
    _error_handler = handler
```

---

### 3.3 Exception Hook

**File**: `src/infrastructure/error_handling/exception_hook.py`

```python
import sys
import traceback
from typing import Type
from infrastructure.logging.logger import Logger
from .error_handler import get_error_handler


def exception_hook(
    exc_type: Type[BaseException],
    exc_value: BaseException,
    exc_traceback
) -> None:
    """
    Global exception hook for unhandled exceptions

    Args:
        exc_type: Exception type
        exc_value: Exception instance
        exc_traceback: Exception traceback
    """
    # Don't handle keyboard interrupt
    if issubclass(exc_type, KeyboardInterrupt):
        sys.__excepthook__(exc_type, exc_value, exc_traceback)
        return

    # Format traceback
    tb_lines = traceback.format_exception(exc_type, exc_value, exc_traceback)
    tb_text = "".join(tb_lines)

    # Log error
    logger = Logger.create_default()
    logger.critical(
        "Unhandled exception",
        exception_type=exc_type.__name__,
        traceback=tb_text
    )

    # Try to show error dialog
    try:
        handler = get_error_handler()
        handler.handle(exc_value)
    except RuntimeError:
        # Error handler not initialized, print to console
        print(tb_text, file=sys.stderr)


def install_exception_hook() -> None:
    """Install global exception hook"""
    sys.excepthook = exception_hook


def uninstall_exception_hook() -> None:
    """Uninstall global exception hook (restore default)"""
    sys.excepthook = sys.__excepthook__
```

---

## 4. Implementation Order

### Phase 1: Exception Classes (2 hours)
1. Implement `ApplicationError` base class
2. Implement `ValidationError` and `ConfigurationError`
3. Implement broker exceptions (`BrokerError`, `BrokerConnectionError`, etc.)
4. Implement data exceptions (`DataError`, `DataNotFoundError`, etc.)

### Phase 2: Error Handler (2-3 hours)
5. Implement `ErrorHandler` class with logging
6. Implement error dialog display
7. Implement recovery strategy registration
8. Add parent widget management

### Phase 3: Exception Hook (1 hour)
9. Implement global exception hook
10. Install hook in application bootstrap
11. Test unhandled exception catching

### Phase 4: Testing (2 hours)
12. Write unit tests for exception classes
13. Write unit tests for ErrorHandler
14. Write integration tests for exception hook

---

## 5. Testing Strategy

### 5.1 Unit Tests

**File**: `tests/unit/domain/exceptions/test_exceptions.py`

```python
import pytest
from domain.exceptions.base import ApplicationError
from domain.exceptions.validation import ValidationError, ConfigurationError
from domain.exceptions.broker import BrokerConnectionError, BrokerAuthenticationError
from domain.exceptions.data import DataNotFoundError, DataFormatError


class TestApplicationError:
    """Unit tests for ApplicationError"""

    def test_stores_technical_message(self):
        """ApplicationError stores technical message"""
        error = ApplicationError("Technical error message")
        assert str(error) == "Technical error message"

    def test_has_user_message(self):
        """ApplicationError has user-friendly message"""
        error = ApplicationError(
            "Technical error",
            user_message="User-friendly error"
        )
        assert error.user_message == "User-friendly error"

    def test_defaults_user_message_to_technical(self):
        """User message defaults to technical message"""
        error = ApplicationError("Error message")
        assert error.user_message == "Error message"

    def test_stores_context(self):
        """ApplicationError stores context data"""
        error = ApplicationError(
            "Error",
            user_id=123,
            action="login"
        )
        assert error.context["user_id"] == 123
        assert error.context["action"] == "login"

    def test_str_includes_context(self):
        """String representation includes context"""
        error = ApplicationError("Error", field="email", value="invalid")
        error_str = str(error)
        assert "field=email" in error_str
        assert "value=invalid" in error_str


class TestValidationError:
    """Unit tests for ValidationError"""

    def test_includes_field_name(self):
        """ValidationError includes field name"""
        error = ValidationError("email", "Invalid format")
        assert "email" in str(error)
        assert error.context["field"] == "email"

    def test_is_recoverable(self):
        """ValidationError is marked as recoverable"""
        error = ValidationError("field", "message")
        assert error.recoverable is True

    def test_has_korean_user_message(self):
        """ValidationError has Korean user message"""
        error = ValidationError("email", "잘못된 형식")
        assert "입력 오류" in error.user_message


class TestBrokerConnectionError:
    """Unit tests for BrokerConnectionError"""

    def test_includes_broker_and_reason(self):
        """BrokerConnectionError includes broker and reason"""
        error = BrokerConnectionError("eBEST", "Connection timeout")
        assert error.context["broker"] == "eBEST"
        assert error.context["reason"] == "Connection timeout"

    def test_has_korean_user_message(self):
        """BrokerConnectionError has Korean user message"""
        error = BrokerConnectionError("eBEST", "Timeout")
        assert "증권사 연결 실패" in error.user_message
        assert "eBEST" in error.user_message
```

**File**: `tests/unit/infrastructure/error_handling/test_error_handler.py`

```python
from unittest.mock import Mock, patch
import pytest
from infrastructure.error_handling.error_handler import ErrorHandler
from domain.exceptions.base import ApplicationError
from domain.exceptions.validation import ValidationError


class TestErrorHandler:
    """Unit tests for ErrorHandler"""

    def test_handle_logs_error(self):
        """handle() logs error"""
        mock_logger = Mock()
        handler = ErrorHandler(mock_logger)

        error = ApplicationError("Test error")
        handler.handle(error)

        mock_logger.error.assert_called_once()

    def test_handle_logs_exception_for_non_application_error(self):
        """handle() logs exception for non-ApplicationError"""
        mock_logger = Mock()
        handler = ErrorHandler(mock_logger)

        error = ValueError("Test error")
        handler.handle(error)

        mock_logger.exception.assert_called_once()

    @patch('infrastructure.error_handling.error_handler.QMessageBox')
    def test_handle_shows_error_dialog(self, mock_messagebox):
        """handle() shows error dialog"""
        handler = ErrorHandler(Mock())

        error = ApplicationError("Test error", user_message="User message")
        handler.handle(error)

        # Verify QMessageBox was created and shown
        assert mock_messagebox.called

    def test_register_and_call_recovery_strategy(self):
        """Recovery strategy is called for registered error type"""
        handler = ErrorHandler(Mock())
        mock_strategy = Mock(return_value=True)

        handler.register_recovery_strategy(ValidationError, mock_strategy)

        error = ValidationError("field", "message")
        result = handler.handle(error)

        mock_strategy.assert_called_once_with(error)
        assert result is True

    def test_recovery_strategy_inheritance(self):
        """Recovery strategy matches parent exception class"""
        handler = ErrorHandler(Mock())
        mock_strategy = Mock(return_value=True)

        # Register strategy for ApplicationError (parent class)
        handler.register_recovery_strategy(ApplicationError, mock_strategy)

        # Handle ValidationError (child class)
        error = ValidationError("field", "message")
        result = handler.handle(error)

        mock_strategy.assert_called_once()
        assert result is True

    def test_handle_returns_false_without_recovery(self):
        """handle() returns False when no recovery strategy"""
        handler = ErrorHandler(Mock())

        error = ApplicationError("Test error")
        result = handler.handle(error)

        assert result is False

    def test_failed_recovery_strategy_returns_false(self):
        """Failed recovery strategy returns False"""
        handler = ErrorHandler(Mock())
        mock_strategy = Mock(side_effect=Exception("Recovery failed"))

        handler.register_recovery_strategy(ApplicationError, mock_strategy)

        error = ApplicationError("Test error")
        result = handler.handle(error)

        assert result is False
```

### 5.2 Integration Tests

**File**: `tests/integration/test_error_handling.py`

```python
import sys
from infrastructure.error_handling.exception_hook import (
    exception_hook,
    install_exception_hook,
    uninstall_exception_hook
)


def test_exception_hook_catches_unhandled_exception(caplog):
    """Exception hook catches and logs unhandled exceptions"""
    try:
        raise ValueError("Test unhandled error")
    except ValueError:
        exc_info = sys.exc_info()
        exception_hook(*exc_info)

    assert "Unhandled exception" in caplog.text
    assert "ValueError" in caplog.text


def test_install_and_uninstall_exception_hook():
    """Exception hook can be installed and uninstalled"""
    original_hook = sys.excepthook

    install_exception_hook()
    assert sys.excepthook != original_hook

    uninstall_exception_hook()
    assert sys.excepthook == original_hook
```

---

## 6. Usage Examples

### 6.1 Raising Custom Exceptions

```python
from domain.exceptions.validation import ValidationError
from domain.exceptions.broker import BrokerConnectionError

# Validation error
if not email.contains("@"):
    raise ValidationError("email", "이메일 형식이 올바르지 않습니다", value=email)

# Broker connection error
try:
    connect_to_broker()
except ConnectionError as e:
    raise BrokerConnectionError("eBEST", str(e))
```

### 6.2 Handling Errors

```python
from infrastructure.error_handling.error_handler import get_error_handler

try:
    risky_operation()
except ApplicationError as e:
    # Error handler logs and shows dialog
    recovered = get_error_handler().handle(e)
    if not recovered:
        # Take fallback action
        use_cached_data()
```

### 6.3 Recovery Strategies

```python
def retry_connection(error: BrokerConnectionError) -> bool:
    """Recovery strategy for connection errors"""
    try:
        time.sleep(error.context.get("retry_after", 5))
        reconnect()
        return True
    except Exception:
        return False

# Register recovery strategy
error_handler.register_recovery_strategy(
    BrokerConnectionError,
    retry_connection
)
```

---

## 7. Common Pitfalls & Solutions

### 7.1 Error Dialog Not Showing

**Problem**: Error dialog doesn't appear
**Cause**: Parent widget not set
**Solution**: Call `error_handler.set_parent(main_window)` after creating main window

### 7.2 Recovery Strategy Not Called

**Problem**: Recovery strategy registered but not executed
**Cause**: Exception type mismatch
**Solution**: Check inheritance hierarchy, register for parent class if needed

### 7.3 Korean Characters Garbled

**Problem**: Korean error messages display incorrectly
**Cause**: Encoding issues
**Solution**: Ensure source files are UTF-8 encoded

---

## 8. Error Message Guidelines

### 8.1 Technical Messages (for logs)

- Use English for consistency
- Include specific details (URLs, IDs, parameters)
- Include context variables
- Be descriptive for debugging

**Example**: `"Failed to connect to broker 'eBEST' at hts.ebest.co.kr:20001: Connection timeout after 30s"`

### 8.2 User Messages (for dialogs)

- Use Korean as primary language
- Be concise and actionable
- Avoid technical jargon
- Suggest next steps when possible

**Example**: `"증권사 연결 실패: eBEST\n잠시 후 다시 시도해주세요."`

---

## 9. Performance Considerations

- **Error handling overhead**: < 10ms per error
- **Dialog display**: Blocks UI until dismissed (intentional)
- **Logging overhead**: ~1-5ms (file I/O)

**Note**: Error handling should NOT be on hot paths

---

## 10. References

### 10.1 Documentation

- [Python Exception Handling](https://docs.python.org/3/tutorial/errors.html)
- [QMessageBox](https://doc.qt.io/qtforpython/PySide6/QtWidgets/QMessageBox.html)
- [sys.excepthook](https://docs.python.org/3/library/sys.html#sys.excepthook)

### 10.2 Related Specs

- E01 Epic Specification
- F04 Logging Infrastructure (error logging)
- F03 DI Container (ErrorHandler as singleton)

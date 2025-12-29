# F03: Dependency Injection Container - Pre-Implementation Documentation

## 1. Implementation Overview

### 1.1 Core Components

This feature implements a lightweight dependency injection container:

1. **Container** - Service registration and resolution
2. **Lifetime** - Service lifetime management (Singleton, Transient)
3. **Bootstrap** - Application service registration

### 1.2 Key Design Patterns

- **Service Locator Pattern**: Centralized service registry
- **Factory Pattern**: Factory functions for service creation
- **Singleton Pattern**: Single instance management

---

## 2. File Structure

```
src/
├── infrastructure/
│   └── di/
│       ├── __init__.py
│       ├── container.py          # Container class
│       ├── lifetimes.py         # Lifetime enum
│       └── exceptions.py        # DI exceptions
│
└── application/
    └── bootstrap.py             # Service registration
│
tests/
├── unit/
│   └── infrastructure/
│       └── di/
│           ├── test_container.py
│           └── test_lifetimes.py
│
└── integration/
    └── test_bootstrap.py
```

---

## 3. Component Specifications

### 3.1 Lifetime Enum

**File**: `src/infrastructure/di/lifetimes.py`

```python
from enum import Enum


class Lifetime(Enum):
    """Service lifetime strategies"""

    SINGLETON = "singleton"
    """Single instance shared across all resolutions"""

    TRANSIENT = "transient"
    """New instance created for each resolution"""
```

---

### 3.2 DI Exceptions

**File**: `src/infrastructure/di/exceptions.py`

```python
class DIError(Exception):
    """Base exception for DI errors"""
    pass


class ServiceNotRegisteredError(DIError):
    """Service not found in container"""

    def __init__(self, service_type: type):
        super().__init__(
            f"Service '{service_type.__name__}' is not registered in the container. "
            f"Please register it using container.register() before resolving."
        )
        self.service_type = service_type


class ServiceAlreadyRegisteredError(DIError):
    """Service already registered in container"""

    def __init__(self, service_type: type):
        super().__init__(
            f"Service '{service_type.__name__}' is already registered. "
            f"Use container.replace() to replace existing registration."
        )
        self.service_type = service_type


class CircularDependencyError(DIError):
    """Circular dependency detected"""

    def __init__(self, dependency_chain: list[type]):
        chain_str = " -> ".join(t.__name__ for t in dependency_chain)
        super().__init__(
            f"Circular dependency detected: {chain_str}"
        )
        self.dependency_chain = dependency_chain
```

---

### 3.3 Container

**File**: `src/infrastructure/di/container.py`

```python
from typing import Any, Callable, TypeVar, Generic, Optional
from .lifetimes import Lifetime
from .exceptions import (
    ServiceNotRegisteredError,
    ServiceAlreadyRegisteredError,
    CircularDependencyError
)

T = TypeVar('T')


class Container:
    """Lightweight dependency injection container"""

    def __init__(self):
        """Initialize empty container"""
        self._registrations: dict[type, tuple[Callable, Lifetime]] = {}
        self._singletons: dict[type, Any] = {}
        self._resolution_stack: list[type] = []

    def register(
        self,
        interface: type[T],
        implementation: Callable[[], T],
        lifetime: Lifetime = Lifetime.SINGLETON
    ) -> 'Container':
        """
        Register service with factory function

        Args:
            interface: Service interface (type)
            implementation: Factory function returning instance
            lifetime: Service lifetime (SINGLETON or TRANSIENT)

        Returns:
            Self for chaining

        Raises:
            ServiceAlreadyRegisteredError: If service already registered
        """
        if interface in self._registrations:
            raise ServiceAlreadyRegisteredError(interface)

        self._registrations[interface] = (implementation, lifetime)
        return self

    def register_instance(self, interface: type[T], instance: T) -> 'Container':
        """
        Register pre-created singleton instance

        Args:
            interface: Service interface (type)
            instance: Pre-created instance

        Returns:
            Self for chaining

        Raises:
            ServiceAlreadyRegisteredError: If service already registered
        """
        if interface in self._registrations:
            raise ServiceAlreadyRegisteredError(interface)

        self._singletons[interface] = instance
        self._registrations[interface] = (lambda: instance, Lifetime.SINGLETON)
        return self

    def replace(
        self,
        interface: type[T],
        implementation: Callable[[], T],
        lifetime: Lifetime = Lifetime.SINGLETON
    ) -> 'Container':
        """
        Replace existing registration (useful for testing)

        Args:
            interface: Service interface (type)
            implementation: Factory function returning instance
            lifetime: Service lifetime

        Returns:
            Self for chaining
        """
        # Remove existing singleton if present
        if interface in self._singletons:
            del self._singletons[interface]

        self._registrations[interface] = (implementation, lifetime)
        return self

    def resolve(self, interface: type[T]) -> T:
        """
        Resolve service by interface

        Args:
            interface: Service interface (type)

        Returns:
            Service instance

        Raises:
            ServiceNotRegisteredError: If service not registered
            CircularDependencyError: If circular dependency detected
        """
        if interface not in self._registrations:
            raise ServiceNotRegisteredError(interface)

        # Check for circular dependency
        if interface in self._resolution_stack:
            self._resolution_stack.append(interface)
            raise CircularDependencyError(self._resolution_stack)

        factory, lifetime = self._registrations[interface]

        if lifetime == Lifetime.SINGLETON:
            # Return cached singleton or create new one
            if interface not in self._singletons:
                self._resolution_stack.append(interface)
                try:
                    self._singletons[interface] = factory()
                finally:
                    self._resolution_stack.pop()

            return self._singletons[interface]
        else:
            # Transient: create new instance each time
            self._resolution_stack.append(interface)
            try:
                return factory()
            finally:
                self._resolution_stack.pop()

    def has(self, interface: type) -> bool:
        """
        Check if service is registered

        Args:
            interface: Service interface (type)

        Returns:
            True if registered, False otherwise
        """
        return interface in self._registrations

    def clear(self) -> None:
        """Clear all registrations (useful for testing)"""
        self._registrations.clear()
        self._singletons.clear()
        self._resolution_stack.clear()

    def get_registered_services(self) -> list[type]:
        """Get list of all registered service types"""
        return list(self._registrations.keys())
```

---

### 3.4 Bootstrap

**File**: `src/application/bootstrap.py`

```python
from infrastructure.di.container import Container, Lifetime
from infrastructure.config.settings_service import SettingsService
from infrastructure.logging.logger import Logger
from infrastructure.error_handling.error_handler import ErrorHandler


def bootstrap_container() -> Container:
    """
    Register all application services

    Returns:
        Configured container with all services registered
    """
    container = Container()

    # Infrastructure services (singletons)
    container.register(
        SettingsService,
        lambda: SettingsService(),
        Lifetime.SINGLETON
    )

    container.register(
        Logger,
        lambda: Logger.create_default(),
        Lifetime.SINGLETON
    )

    container.register(
        ErrorHandler,
        lambda: ErrorHandler(
            logger=container.resolve(Logger)
        ),
        Lifetime.SINGLETON
    )

    # Application services will be added here as features are implemented
    # container.register(
    #     PortfolioService,
    #     lambda: PortfolioService(
    #         settings=container.resolve(SettingsService),
    #         logger=container.resolve(Logger)
    #     ),
    #     Lifetime.SINGLETON
    # )

    return container
```

---

## 4. Implementation Order

### Phase 1: Core Container (2-3 hours)
1. Implement `Lifetime` enum
2. Implement `DIError` exception classes
3. Implement `Container` class with register/resolve
4. Add singleton lifetime support

### Phase 2: Advanced Features (2 hours)
5. Add transient lifetime support
6. Implement `register_instance()` for pre-created instances
7. Add circular dependency detection
8. Implement `replace()` for testing support

### Phase 3: Bootstrap (1 hour)
9. Create `bootstrap.py` module
10. Register core infrastructure services

### Phase 4: Testing (2 hours)
11. Write comprehensive unit tests
12. Write integration tests for bootstrap
13. Test circular dependency detection

---

## 5. Testing Strategy

### 5.1 Unit Tests

**File**: `tests/unit/infrastructure/di/test_container.py`

```python
import pytest
from infrastructure.di.container import Container, Lifetime
from infrastructure.di.exceptions import (
    ServiceNotRegisteredError,
    ServiceAlreadyRegisteredError,
    CircularDependencyError
)


class DummyService:
    """Test service"""
    pass


class AnotherService:
    """Another test service"""

    def __init__(self, dummy: DummyService):
        self.dummy = dummy


class TestContainer:
    """Unit tests for Container"""

    def test_register_and_resolve_singleton(self):
        """Singleton returns same instance"""
        container = Container()
        container.register(DummyService, DummyService, Lifetime.SINGLETON)

        instance1 = container.resolve(DummyService)
        instance2 = container.resolve(DummyService)

        assert instance1 is instance2

    def test_register_and_resolve_transient(self):
        """Transient returns new instance each time"""
        container = Container()
        container.register(DummyService, DummyService, Lifetime.TRANSIENT)

        instance1 = container.resolve(DummyService)
        instance2 = container.resolve(DummyService)

        assert instance1 is not instance2
        assert isinstance(instance1, DummyService)
        assert isinstance(instance2, DummyService)

    def test_resolve_unregistered_raises_error(self):
        """Resolving unregistered service raises ServiceNotRegisteredError"""
        container = Container()

        with pytest.raises(ServiceNotRegisteredError) as exc_info:
            container.resolve(DummyService)

        assert "DummyService" in str(exc_info.value)

    def test_register_duplicate_raises_error(self):
        """Registering duplicate service raises ServiceAlreadyRegisteredError"""
        container = Container()
        container.register(DummyService, DummyService)

        with pytest.raises(ServiceAlreadyRegisteredError) as exc_info:
            container.register(DummyService, DummyService)

        assert "DummyService" in str(exc_info.value)

    def test_register_instance(self):
        """Pre-created instance can be registered"""
        container = Container()
        instance = DummyService()

        container.register_instance(DummyService, instance)
        resolved = container.resolve(DummyService)

        assert resolved is instance

    def test_replace_existing_registration(self):
        """replace() overwrites existing registration"""
        container = Container()
        container.register(DummyService, DummyService)

        instance1 = container.resolve(DummyService)

        # Replace registration
        container.replace(DummyService, DummyService)
        instance2 = container.resolve(DummyService)

        assert instance1 is not instance2

    def test_has_returns_true_for_registered(self):
        """has() returns True for registered service"""
        container = Container()
        container.register(DummyService, DummyService)

        assert container.has(DummyService) is True

    def test_has_returns_false_for_unregistered(self):
        """has() returns False for unregistered service"""
        container = Container()

        assert container.has(DummyService) is False

    def test_clear_removes_all_registrations(self):
        """clear() removes all services"""
        container = Container()
        container.register(DummyService, DummyService)

        container.clear()

        assert container.has(DummyService) is False

    def test_get_registered_services(self):
        """get_registered_services() returns all registered types"""
        container = Container()
        container.register(DummyService, DummyService)
        container.register(AnotherService, AnotherService)

        services = container.get_registered_services()

        assert DummyService in services
        assert AnotherService in services
        assert len(services) == 2

    def test_register_returns_self_for_chaining(self):
        """register() returns self for method chaining"""
        container = Container()

        result = container.register(DummyService, DummyService)

        assert result is container

    def test_circular_dependency_detection(self):
        """Circular dependency raises CircularDependencyError"""
        container = Container()

        # Service A depends on Service B
        class ServiceA:
            def __init__(self):
                self.b = container.resolve(ServiceB)

        # Service B depends on Service A (circular!)
        class ServiceB:
            def __init__(self):
                self.a = container.resolve(ServiceA)

        container.register(ServiceA, ServiceA)
        container.register(ServiceB, ServiceB)

        with pytest.raises(CircularDependencyError) as exc_info:
            container.resolve(ServiceA)

        assert "ServiceA" in str(exc_info.value)
        assert "ServiceB" in str(exc_info.value)
```

### 5.2 Integration Tests

**File**: `tests/integration/test_bootstrap.py`

```python
from application.bootstrap import bootstrap_container
from infrastructure.config.settings_service import SettingsService
from infrastructure.logging.logger import Logger
from infrastructure.error_handling.error_handler import ErrorHandler


def test_bootstrap_registers_all_core_services():
    """Bootstrap registers all core infrastructure services"""
    container = bootstrap_container()

    # Verify all core services are registered
    assert container.has(SettingsService)
    assert container.has(Logger)
    assert container.has(ErrorHandler)


def test_bootstrap_services_are_singletons():
    """Bootstrap services are singletons"""
    container = bootstrap_container()

    settings1 = container.resolve(SettingsService)
    settings2 = container.resolve(SettingsService)

    assert settings1 is settings2


def test_bootstrap_services_can_be_resolved():
    """All bootstrap services can be resolved without errors"""
    container = bootstrap_container()

    # Should not raise any errors
    settings = container.resolve(SettingsService)
    logger = container.resolve(Logger)
    error_handler = container.resolve(ErrorHandler)

    assert isinstance(settings, SettingsService)
    assert isinstance(logger, Logger)
    assert isinstance(error_handler, ErrorHandler)
```

---

## 6. Usage Examples

### 6.1 Basic Registration and Resolution

```python
from infrastructure.di.container import Container, Lifetime


# Create container
container = Container()

# Register service
container.register(
    SettingsService,
    lambda: SettingsService(),
    Lifetime.SINGLETON
)

# Resolve service
settings = container.resolve(SettingsService)
```

### 6.2 Method Chaining

```python
container = Container()
container.register(SettingsService, SettingsService) \
         .register(Logger, Logger.create_default) \
         .register(ErrorHandler, lambda: ErrorHandler(container.resolve(Logger)))
```

### 6.3 Constructor Injection

```python
class MainWindow(QMainWindow):
    """Main window with dependency injection"""

    def __init__(self, container: Container):
        super().__init__()
        self._settings = container.resolve(SettingsService)
        self._logger = container.resolve(Logger)
        self._error_handler = container.resolve(ErrorHandler)
```

### 6.4 Testing with Mocks

```python
def test_main_window_uses_settings():
    """MainWindow uses injected SettingsService"""
    # Arrange
    container = Container()
    mock_settings = Mock(spec=SettingsService)
    container.register_instance(SettingsService, mock_settings)

    # Act
    window = MainWindow(container)

    # Assert
    assert window._settings is mock_settings
```

---

## 7. Common Pitfalls & Solutions

### 7.1 Circular Dependencies

**Problem**: Service A depends on Service B, which depends on Service A
**Detection**: CircularDependencyError is raised
**Solution**: Refactor to eliminate circular dependency:
- Extract shared functionality to a third service
- Use lazy initialization
- Use events/signals instead of direct dependencies

### 7.2 Missing Registration

**Problem**: ServiceNotRegisteredError when resolving
**Cause**: Service not registered in bootstrap
**Solution**: Add registration to `bootstrap_container()`

### 7.3 Singleton Not Shared

**Problem**: Multiple instances of "singleton" service
**Cause**: Registered with `Lifetime.TRANSIENT` instead of `Lifetime.SINGLETON`
**Solution**: Verify lifetime parameter in `register()`

### 7.4 Factory Function Errors

**Problem**: Exception when resolving service
**Cause**: Factory function has errors or missing dependencies
**Solution**: Test factory function in isolation

---

## 8. Design Decisions

### 8.1 Why Not Use Dependency Injector Library?

**Decision**: Implement custom container instead of using `dependency_injector` or similar

**Rationale**:
- Simple requirements (< 200 LOC implementation)
- No need for advanced features (auto-wiring, scopes)
- Zero external dependencies
- Educational value
- Full control over behavior

### 8.2 Why Service Locator Pattern?

**Decision**: Use service locator instead of pure dependency injection

**Rationale**:
- Simpler to understand and implement
- Explicit service resolution
- Works well with Qt's object lifecycle
- Easier to debug (explicit dependencies)

**Trade-off**: Less compile-time safety, but acceptable for Python

### 8.3 Why Support Transient Lifetime?

**Decision**: Support both singleton and transient lifetimes

**Rationale**:
- Some services should not be shared (e.g., scoped operations)
- Flexibility for future use cases
- Minimal implementation complexity

---

## 9. Performance Considerations

- **Resolution Time**: O(1) lookup for singletons, O(1) + factory time for transients
- **Memory**: One dict for registrations, one for singleton instances
- **Overhead**: Negligible (< 1ms per resolution)

**Optimization**: Use `register_instance()` for expensive initialization

---

## 10. References

### 10.1 Patterns

- [Dependency Injection](https://martinfowler.com/articles/injection.html)
- [Service Locator](https://martinfowler.com/articles/injection.html#UsingAServiceLocator)

### 10.2 Related Specs

- E01 Epic Specification
- F02 Settings Management (registered in container)
- F04 Logging (registered in container)
- F05 Error Handling (registered in container)

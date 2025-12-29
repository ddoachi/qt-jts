# F02: Settings Management - Pre-Implementation Documentation

## 1. Implementation Overview

### 1.1 Core Components

This feature implements three main components:

1. **SettingsService** - QSettings wrapper with type safety
2. **EncryptionService** - Encryption/decryption for sensitive data
3. **SettingsSchema** - Centralized settings definition

### 1.2 Key Design Patterns

- **Facade Pattern**: SettingsService simplifies QSettings API
- **Strategy Pattern**: Encryption strategy for sensitive settings
- **Registry Pattern**: SettingsSchema centralizes all settings

---

## 2. File Structure

```
src/
├── infrastructure/
│   └── config/
│       ├── __init__.py
│       ├── settings_service.py      # SettingsService class
│       ├── encryption.py            # EncryptionService class
│       └── settings_schema.py       # SettingsSchema definitions
│
└── application/
    └── services/
        └── settings_manager.py      # Application-level facade (optional)
│
tests/
├── unit/
│   └── infrastructure/
│       └── config/
│           ├── test_settings_service.py
│           ├── test_encryption.py
│           └── test_settings_schema.py
│
└── integration/
    └── test_settings_persistence.py
```

---

## 3. Component Specifications

### 3.1 EncryptionService (Implement First)

**File**: `src/infrastructure/config/encryption.py`

**Dependencies**:
```bash
poetry add cryptography
```

**Class Implementation**:
```python
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2
import base64
import platform
from typing import Optional


class EncryptionService:
    """Encrypt/decrypt sensitive settings using machine-specific key"""

    def __init__(self, custom_key: Optional[bytes] = None):
        """
        Initialize encryption service

        Args:
            custom_key: Optional custom key (for testing)
        """
        self._key = custom_key or self._derive_key()
        self._cipher = Fernet(self._key)

    def _derive_key(self) -> bytes:
        """
        Derive encryption key from machine-specific data

        Uses platform.node() as salt for machine-specific key derivation.
        Not cryptographically perfect, but prevents casual inspection.

        Returns:
            32-byte encryption key
        """
        machine_id = platform.node().encode('utf-8')

        # Use PBKDF2 for key derivation
        kdf = PBKDF2(
            algorithm=hashes.SHA256(),
            length=32,
            salt=machine_id,
            iterations=100000,
        )

        # Derive key from static password + machine ID
        derived_key = kdf.derive(b"JTS_SETTINGS_ENCRYPTION_KEY_V1")
        return base64.urlsafe_b64encode(derived_key)

    def encrypt(self, plaintext: str) -> str:
        """
        Encrypt string

        Args:
            plaintext: String to encrypt

        Returns:
            Encrypted string (base64 encoded)
        """
        if not plaintext:
            return ""

        encrypted_bytes = self._cipher.encrypt(plaintext.encode('utf-8'))
        return encrypted_bytes.decode('utf-8')

    def decrypt(self, encrypted: str) -> str:
        """
        Decrypt string

        Args:
            encrypted: Encrypted string

        Returns:
            Decrypted plaintext, or empty string on failure
        """
        if not encrypted:
            return ""

        try:
            decrypted_bytes = self._cipher.decrypt(encrypted.encode('utf-8'))
            return decrypted_bytes.decode('utf-8')
        except Exception:
            # Return empty string on decryption failure
            # (e.g., corrupted data, wrong key)
            return ""
```

**Test Coverage**:
```python
# tests/unit/infrastructure/config/test_encryption.py

def test_encrypt_produces_different_output():
    """Encrypted text differs from plaintext"""

def test_decrypt_reverses_encryption():
    """Decrypt(Encrypt(x)) == x"""

def test_decrypt_invalid_returns_empty():
    """Decrypting invalid data returns empty string"""

def test_empty_string_handling():
    """Empty strings are handled gracefully"""

def test_unicode_support():
    """Korean and other unicode characters work"""

def test_different_machines_have_different_keys():
    """Key derivation uses machine-specific data"""
    # Mock platform.node() to test
```

---

### 3.2 SettingsService

**File**: `src/infrastructure/config/settings_service.py`

**Class Implementation**:
```python
from typing import Any, Optional, TypeVar
from pathlib import Path
from PySide6.QtCore import QSettings
from .encryption import EncryptionService

T = TypeVar('T')


class SettingsService:
    """Cross-platform settings management with type safety"""

    def __init__(
        self,
        organization: str = "JoohanTech",
        application: str = "JTS",
        encryption_service: Optional[EncryptionService] = None
    ):
        """
        Initialize settings service

        Args:
            organization: Organization name (for storage path)
            application: Application name (for storage path)
            encryption_service: Optional custom encryption service
        """
        self._settings = QSettings(organization, application)
        self._encryption = encryption_service or EncryptionService()

    def get(
        self,
        key: str,
        default: Any = None,
        value_type: type = str
    ) -> Any:
        """
        Get setting with type coercion

        Args:
            key: Setting key (e.g., "general/language")
            default: Default value if key not found
            value_type: Expected type (str, int, float, bool, Path)

        Returns:
            Setting value coerced to value_type
        """
        value = self._settings.value(key, default)

        # Return default if value is None
        if value is None:
            return default

        # Type coercion
        return self._coerce_type(value, value_type)

    def set(self, key: str, value: Any) -> None:
        """
        Set setting value

        Args:
            key: Setting key
            value: Value to store
        """
        self._settings.setValue(key, value)
        self._settings.sync()  # Ensure immediate persistence

    def get_encrypted(self, key: str, default: str = "") -> str:
        """
        Get encrypted setting (e.g., API key)

        Args:
            key: Setting key
            default: Default value if not found

        Returns:
            Decrypted value
        """
        encrypted_value = self._settings.value(key)
        if not encrypted_value:
            return default

        return self._encryption.decrypt(encrypted_value)

    def set_encrypted(self, key: str, value: str) -> None:
        """
        Set encrypted setting

        Args:
            key: Setting key
            value: Plaintext value to encrypt and store
        """
        encrypted_value = self._encryption.encrypt(value)
        self._settings.setValue(key, encrypted_value)
        self._settings.sync()

    def has(self, key: str) -> bool:
        """Check if setting exists"""
        return self._settings.contains(key)

    def remove(self, key: str) -> None:
        """Remove setting"""
        self._settings.remove(key)
        self._settings.sync()

    def clear(self) -> None:
        """Clear all settings"""
        self._settings.clear()
        self._settings.sync()

    def get_all_keys(self) -> list[str]:
        """Get all setting keys"""
        return self._settings.allKeys()

    def _coerce_type(self, value: Any, value_type: type) -> Any:
        """
        Coerce value to specified type

        Args:
            value: Raw value from QSettings
            value_type: Target type

        Returns:
            Value coerced to value_type
        """
        if value_type == bool:
            # QSettings stores bools as strings
            return value in [True, "true", "True", 1, "1"]
        elif value_type == int:
            return int(value)
        elif value_type == float:
            return float(value)
        elif value_type == Path:
            return Path(value)
        elif value_type == list:
            # QSettings returns lists as-is
            return value if isinstance(value, list) else [value]
        else:
            return str(value)
```

**Test Coverage**:
```python
# tests/unit/infrastructure/config/test_settings_service.py

def test_get_returns_default_for_missing_key():
    """Missing key returns default value"""

def test_set_and_get_roundtrip():
    """Set value can be retrieved"""

def test_get_with_type_coercion_int():
    """Type coercion works for int"""

def test_get_with_type_coercion_bool():
    """Type coercion works for bool"""

def test_get_encrypted_decrypts_value():
    """Encrypted value is decrypted on retrieval"""

def test_encrypted_value_not_plaintext_in_storage():
    """Encrypted value is not readable in raw storage"""

def test_has_returns_true_for_existing():
    """has() returns True for existing key"""

def test_remove_deletes_key():
    """remove() deletes setting"""

def test_clear_removes_all():
    """clear() removes all settings"""
```

---

### 3.3 SettingsSchema

**File**: `src/infrastructure/config/settings_schema.py`

**Class Implementation**:
```python
from dataclasses import dataclass
from typing import Any, Optional, Callable
from pathlib import Path


@dataclass(frozen=True)
class SettingDefinition:
    """Metadata for a setting"""

    key: str
    default: Any
    value_type: type
    encrypted: bool = False
    description: str = ""
    validator: Optional[Callable[[Any], bool]] = None


class SettingsSchema:
    """Centralized definition of all application settings"""

    # General Settings
    GENERAL_LANGUAGE = SettingDefinition(
        key="general/language",
        default="ko",
        value_type=str,
        description="UI language (ko, en)"
    )

    GENERAL_THEME = SettingDefinition(
        key="general/theme",
        default="light",
        value_type=str,
        description="UI theme (light, dark)"
    )

    GENERAL_STARTUP_VIEW = SettingDefinition(
        key="general/startup_view",
        default="dashboard",
        value_type=str,
        description="Default view on startup"
    )

    # Broker Settings
    BROKER_EBEST_URL = SettingDefinition(
        key="broker/ebest/url",
        default="hts.ebest.co.kr:20001",
        value_type=str,
        description="eBEST connection URL"
    )

    BROKER_EBEST_APP_KEY = SettingDefinition(
        key="broker/ebest/app_key",
        default="",
        value_type=str,
        encrypted=True,
        description="eBEST app key (encrypted)"
    )

    BROKER_EBEST_APP_SECRET = SettingDefinition(
        key="broker/ebest/app_secret",
        default="",
        value_type=str,
        encrypted=True,
        description="eBEST app secret (encrypted)"
    )

    # UI Settings
    UI_MAIN_WINDOW_GEOMETRY = SettingDefinition(
        key="ui/main_window/geometry",
        default=None,
        value_type=bytes,
        description="Main window geometry (QByteArray)"
    )

    UI_MAIN_WINDOW_STATE = SettingDefinition(
        key="ui/main_window/state",
        default=None,
        value_type=bytes,
        description="Main window state (QByteArray)"
    )

    # Trading Settings
    TRADING_DEFAULT_QUANTITY = SettingDefinition(
        key="trading/default_quantity",
        default=10,
        value_type=int,
        description="Default order quantity"
    )

    TRADING_MAX_POSITION_SIZE = SettingDefinition(
        key="trading/max_position_size",
        default=1000000,
        value_type=int,
        description="Maximum position size (KRW)"
    )

    # Data Settings
    DATA_CACHE_DIR = SettingDefinition(
        key="data/cache_dir",
        default=Path.home() / ".jts" / "cache",
        value_type=Path,
        description="Data cache directory"
    )

    DATA_UPDATE_INTERVAL = SettingDefinition(
        key="data/update_interval",
        default=60,
        value_type=int,
        description="Data update interval (seconds)"
    )

    @classmethod
    def get_all_definitions(cls) -> list[SettingDefinition]:
        """Get all setting definitions"""
        return [
            value for name, value in vars(cls).items()
            if isinstance(value, SettingDefinition)
        ]

    @classmethod
    def get_definition(cls, key: str) -> Optional[SettingDefinition]:
        """Get setting definition by key"""
        for definition in cls.get_all_definitions():
            if definition.key == key:
                return definition
        return None
```

**Usage Example**:
```python
# Application code
settings = SettingsService()

# Get with schema definition
definition = SettingsSchema.GENERAL_LANGUAGE
language = settings.get(
    definition.key,
    definition.default,
    definition.value_type
)

# Set encrypted
settings.set_encrypted(
    SettingsSchema.BROKER_EBEST_APP_KEY.key,
    "my_api_key"
)
```

---

## 4. Implementation Order

### Phase 1: Encryption Foundation (2-3 hours)
1. Implement `EncryptionService` with key derivation
2. Write unit tests for encryption/decryption
3. Test with various inputs (empty, unicode, long strings)

### Phase 2: Settings Service (3-4 hours)
4. Implement `SettingsService` with QSettings wrapper
5. Add type coercion logic
6. Implement encrypted get/set methods
7. Write comprehensive unit tests

### Phase 3: Settings Schema (1-2 hours)
8. Define `SettingDefinition` dataclass
9. Define all application settings in `SettingsSchema`
10. Add helper methods for definition lookup

### Phase 4: Integration Testing (2 hours)
11. Test settings persistence across app restarts
12. Test encrypted values are not readable in storage
13. Test cross-platform compatibility (if possible)

### Phase 5: Optional Enhancements (2-3 hours)
14. Create settings dialog UI
15. Add settings import/export
16. Add settings validation

---

## 5. Testing Strategy

### 5.1 Unit Tests

**EncryptionService**:
```python
def test_encrypt_produces_different_output()
def test_decrypt_reverses_encryption()
def test_decrypt_invalid_returns_empty()
def test_unicode_support()
def test_empty_string_handling()
```

**SettingsService**:
```python
def test_get_returns_default_for_missing_key()
def test_set_and_get_roundtrip()
def test_type_coercion_bool()
def test_type_coercion_int()
def test_type_coercion_float()
def test_type_coercion_path()
def test_get_encrypted_decrypts()
def test_set_encrypted_not_readable()
def test_has_existing_key()
def test_remove_deletes()
def test_clear_removes_all()
```

### 5.2 Integration Tests

```python
def test_settings_persist_across_restarts(tmp_path):
    """Settings persist when service is recreated"""

def test_encrypted_settings_survive_restart(tmp_path):
    """Encrypted settings persist and decrypt correctly"""

def test_qsettings_file_location():
    """Verify QSettings uses correct platform path"""
```

### 5.3 Manual Testing

- [ ] Set various setting types (str, int, bool, Path)
- [ ] Verify settings persist across app restart
- [ ] Check settings file location (platform-specific)
- [ ] Verify encrypted values are not plaintext in storage
- [ ] Test with Korean characters
- [ ] Test with empty values
- [ ] Test remove and clear operations

---

## 6. Platform-Specific Storage

### 6.1 Storage Locations

**Linux**:
- `~/.config/JoohanTech/JTS.conf` (INI format)

**macOS**:
- `~/Library/Preferences/com.joohantech.jts.plist` (plist format)

**Windows**:
- `HKEY_CURRENT_USER\Software\JoohanTech\JTS` (Registry)

### 6.2 Testing Storage Locations

```python
import sys
from PySide6.QtCore import QSettings

def test_settings_location():
    """Verify settings file location"""
    settings = QSettings("JoohanTech", "JTS")

    if sys.platform == "linux":
        assert "/.config/JoohanTech/JTS.conf" in settings.fileName()
    elif sys.platform == "darwin":
        assert "/Library/Preferences/com.joohantech.jts.plist" in settings.fileName()
    elif sys.platform == "win32":
        # Windows uses registry, fileName() returns empty
        assert settings.organizationName() == "JoohanTech"
```

---

## 7. Common Pitfalls & Solutions

### 7.1 QSettings Type Coercion Issues

**Problem**: QSettings returns strings for all values
**Solution**: Always use `value_type` parameter in `get()` method

### 7.2 Encrypted Values Not Decrypting

**Problem**: Encrypted value returns empty string
**Cause**: Decryption failure (key mismatch, corrupted data)
**Solution**: Log decryption failures, handle gracefully

### 7.3 Settings Not Persisting

**Problem**: Settings lost on restart
**Cause**: Forgot to call `sync()` after `setValue()`
**Solution**: Always call `sync()` in `set()` method

### 7.4 Boolean Settings Always True

**Problem**: Boolean settings return True even when set to False
**Cause**: QSettings stores "false" as string, which is truthy
**Solution**: Use explicit boolean coercion: `value in [True, "true", "True", 1, "1"]`

---

## 8. Security Considerations

### 8.1 Encryption Limitations

- **Machine-specific key**: Encrypted values only work on same machine
- **Not cryptographically secure**: Prevents casual inspection, not targeted attacks
- **Key stored in code**: Key derivation uses static password + machine ID

### 8.2 Production Recommendations

For production deployment, consider:
- Use OS keychain/credential manager for API keys
- Implement user password-based key derivation
- Add tamper detection for settings file
- Use HTTPS for any remote settings sync

---

## 9. Performance Considerations

- **Settings access**: < 10ms for get/set operations
- **Encryption/decryption**: < 50ms per operation
- **Sync overhead**: QSettings.sync() is I/O bound (~1-5ms)

**Optimization**:
- Cache frequently accessed settings in memory
- Batch settings writes, sync once
- Use lazy initialization for SettingsService

---

## 10. References

### 10.1 Documentation

- [QSettings](https://doc.qt.io/qtforpython/PySide6/QtCore/QSettings.html)
- [Cryptography Library](https://cryptography.io/en/latest/)
- [PBKDF2 Key Derivation](https://cryptography.io/en/latest/hazmat/primitives/key-derivation-functions/#cryptography.hazmat.primitives.kdf.pbkdf2.PBKDF2)

### 10.2 Related Specs

- E01 Epic Specification
- F01 Main Window (uses settings for geometry)
- F03 DI Container (SettingsService as singleton)

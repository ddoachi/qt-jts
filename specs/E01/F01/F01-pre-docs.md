# F01: Main Window & Navigation - Pre-Implementation Documentation

## 1. Implementation Overview

### 1.1 Core Components

This feature implements three main components:

1. **MainWindow** - Central QMainWindow managing view lifecycle
2. **NavigationBar** - Custom widget with navigation buttons
3. **StatusBar** - Status indicators for connection and system state

### 1.2 Key Design Patterns

- **Stacked Widget Pattern**: View switching without destroying widgets
- **Signal-Slot Communication**: Decoupled navigation event handling
- **Registry Pattern**: Dynamic view registration for extensibility

---

## 2. File Structure

```
src/
├── presentation/
│   └── main_window/
│       ├── __init__.py
│       ├── main_window.py          # MainWindow class
│       ├── navigation_bar.py       # NavigationBar widget
│       ├── status_bar.py          # StatusBar widget
│       └── view_registry.py       # View management helper
│
tests/
├── unit/
│   └── presentation/
│       └── main_window/
│           ├── test_main_window.py
│           ├── test_navigation_bar.py
│           └── test_status_bar.py
│
└── integration/
    └── test_navigation_flow.py
```

---

## 3. Component Specifications

### 3.1 MainWindow

**File**: `src/presentation/main_window/main_window.py`

**Class Signature**:
```python
from PySide6.QtWidgets import QMainWindow, QStackedWidget
from PySide6.QtCore import QSettings
from infrastructure.di.container import Container


class MainWindow(QMainWindow):
    """Main application window with navigation"""

    def __init__(self, container: Container):
        """
        Initialize main window

        Args:
            container: DI container for service resolution
        """
        super().__init__()
        self._container = container
        self._settings = container.resolve(SettingsService)
        self._stack = QStackedWidget()
        self._nav_bar: NavigationBar = None
        self._status_bar: StatusBar = None
        self._views: dict[str, QWidget] = {}

        self._setup_ui()
        self._restore_geometry()
```

**Key Methods**:
```python
def _setup_ui(self) -> None:
    """Setup UI components"""
    # Set window properties
    # Create navigation bar
    # Create stacked widget
    # Create status bar
    # Setup layout

def register_view(self, view_id: str, view: QWidget, nav_label: str, nav_icon: str) -> None:
    """
    Register view for navigation

    Args:
        view_id: Unique view identifier
        view: Widget to display
        nav_label: Label for navigation button
        nav_icon: Icon for navigation button
    """

def navigate_to(self, view_id: str) -> None:
    """
    Navigate to specified view

    Args:
        view_id: View identifier

    Raises:
        ValueError: If view not registered
    """

def _restore_geometry(self) -> None:
    """Restore window geometry from settings"""

def _save_geometry(self) -> None:
    """Save window geometry to settings"""

def closeEvent(self, event: QCloseEvent) -> None:
    """Handle window close event"""
```

**Test Coverage Points**:
- View registration adds to stack
- Navigate switches current widget
- Invalid view raises ValueError
- Geometry is saved on close
- Geometry is restored on startup

---

### 3.2 NavigationBar

**File**: `src/presentation/main_window/navigation_bar.py`

**Class Signature**:
```python
from PySide6.QtWidgets import QWidget, QPushButton, QHBoxLayout
from PySide6.QtCore import Signal


class NavigationBar(QWidget):
    """Navigation bar with view selection buttons"""

    navigation_requested = Signal(str)  # view_id

    def __init__(self):
        super().__init__()
        self._buttons: dict[str, QPushButton] = {}
        self._active_view: str = None
        self._setup_ui()
```

**Key Methods**:
```python
def _setup_ui(self) -> None:
    """Setup navigation bar layout"""

def add_navigation_item(self, view_id: str, label: str, icon: str) -> None:
    """
    Add navigation button

    Args:
        view_id: Unique view identifier
        label: Button label
        icon: Button icon (emoji or icon path)
    """

def set_active(self, view_id: str) -> None:
    """
    Set active view (visual highlight)

    Args:
        view_id: View to highlight
    """

def _on_button_clicked(self, view_id: str) -> None:
    """Handle button click"""
```

**Styling**:
```css
/* Active button style */
QPushButton[active="true"] {
    background-color: #0078d4;
    color: white;
    border-left: 3px solid #005a9e;
}

/* Inactive button style */
QPushButton {
    background-color: transparent;
    border: none;
    padding: 10px 20px;
    text-align: left;
}

QPushButton:hover {
    background-color: #e5e5e5;
}
```

**Test Coverage Points**:
- Adding item creates button
- Button click emits signal with correct view_id
- Active view is visually highlighted
- Only one button is active at a time

---

### 3.3 StatusBar

**File**: `src/presentation/main_window/status_bar.py`

**Class Signature**:
```python
from PySide6.QtWidgets import QStatusBar, QLabel
from PySide6.QtCore import Qt


class StatusBar(QStatusBar):
    """Application status bar with indicators"""

    def __init__(self):
        super().__init__()
        self._connection_label = QLabel()
        self._rate_limit_label = QLabel()
        self._message_label = QLabel()
        self._setup_ui()
```

**Key Methods**:
```python
def _setup_ui(self) -> None:
    """Setup status bar widgets"""

def set_connection_status(self, connected: bool, broker: str = "") -> None:
    """
    Update connection status

    Args:
        connected: Connection state
        broker: Broker name
    """

def set_rate_limit(self, remaining: int, limit: int) -> None:
    """
    Update rate limit indicator

    Args:
        remaining: Remaining requests
        limit: Total limit
    """

def show_message(self, message: str, timeout: int = 5000) -> None:
    """
    Show temporary message

    Args:
        message: Message text
        timeout: Display duration in ms
    """
```

**Test Coverage Points**:
- Connection status updates label
- Rate limit shows correct format
- Messages display with timeout
- Indicators have correct colors (green/red for connection)

---

## 4. Implementation Order

### Phase 1: Core Structure (Day 1)
1. Create `MainWindow` skeleton
2. Setup `QStackedWidget` central widget
3. Implement basic window properties

### Phase 2: Navigation (Day 1)
4. Create `NavigationBar` widget
5. Implement button creation and layout
6. Wire navigation signals to MainWindow

### Phase 3: View Management (Day 2)
7. Implement view registration
8. Implement view switching
9. Add error handling for invalid views

### Phase 4: Persistence (Day 2)
10. Add geometry save/restore
11. Integrate with SettingsService

### Phase 5: Status Bar (Day 2)
12. Create `StatusBar` widget
13. Add connection and rate limit indicators

### Phase 6: Polish (Day 3)
14. Add styling and icons
15. Add keyboard shortcuts
16. Performance optimization

---

## 5. Testing Strategy

### 5.1 Unit Tests

**MainWindow Tests**:
```python
# tests/unit/presentation/main_window/test_main_window.py

def test_register_view_adds_to_stack():
    """View registration adds widget to stack"""

def test_navigate_to_switches_view():
    """Navigation switches active view"""

def test_navigate_to_invalid_raises_error():
    """Invalid view raises ValueError"""

def test_geometry_restored_on_startup(mocker):
    """Window geometry restored from settings"""
```

**NavigationBar Tests**:
```python
# tests/unit/presentation/main_window/test_navigation_bar.py

def test_add_navigation_item_creates_button():
    """Adding item creates navigation button"""

def test_button_click_emits_signal(qtbot):
    """Clicking button emits navigation signal"""

def test_set_active_highlights_button():
    """Active view button is highlighted"""
```

### 5.2 Integration Tests

```python
# tests/integration/test_navigation_flow.py

def test_end_to_end_navigation(qtbot):
    """Complete navigation flow from button to view"""

def test_geometry_persists_across_restart(tmp_path):
    """Window geometry persists when recreated"""
```

### 5.3 Manual Testing Checklist

- [ ] Window launches at expected size
- [ ] Navigation buttons switch views correctly
- [ ] Active button is visually distinct
- [ ] Window geometry persists across restarts
- [ ] Status bar displays connection status
- [ ] Application closes gracefully
- [ ] Keyboard shortcuts work (if implemented)
- [ ] UI is responsive on resize

---

## 6. Integration Points

### 6.1 Dependencies

**Required Services**:
- `Container` (F03): Dependency injection
- `SettingsService` (F02): Geometry persistence

**Initialization Order**:
1. Container must be initialized
2. SettingsService must be registered
3. MainWindow can be created

### 6.2 Event Flow

```
User clicks navigation button
    ↓
NavigationBar emits navigation_requested(view_id)
    ↓
MainWindow receives signal
    ↓
MainWindow.navigate_to(view_id)
    ↓
QStackedWidget switches current widget
    ↓
NavigationBar.set_active(view_id) updates highlight
```

---

## 7. Common Pitfalls & Solutions

### 7.1 View Not Showing After Registration

**Problem**: View registered but not displayed
**Cause**: Forgot to add view to stacked widget
**Solution**: Ensure `_stack.addWidget(view)` is called in `register_view()`

### 7.2 Navigation Signal Not Received

**Problem**: Button click doesn't navigate
**Cause**: Signal not connected to slot
**Solution**: Verify `navigation_requested.connect(navigate_to)` in `__init__`

### 7.3 Geometry Not Persisting

**Problem**: Window size resets on restart
**Cause**: Settings not synced before close
**Solution**: Call `_settings.set()` and ensure `sync()` in `closeEvent()`

### 7.4 Multiple Active Buttons

**Problem**: Multiple navigation buttons highlighted
**Cause**: Not clearing previous active state
**Solution**: Loop through all buttons and set `active=False` before setting new active

---

## 8. Performance Considerations

### 8.1 View Initialization

- **Lazy Loading**: Only create views when first accessed
- **Singleton Views**: Reuse view instances instead of recreating
- **Background Initialization**: Load heavy views in background thread

### 8.2 Navigation Performance

- **Target**: < 100ms view switch time
- **Optimization**: Pre-instantiate views during idle time
- **Monitoring**: Add timing logs to measure switch duration

---

## 9. Accessibility

### 9.1 Keyboard Navigation

- Tab order: Navigation buttons → View content
- Enter/Space: Activate navigation button
- Ctrl+1-9: Quick navigation to views

### 9.2 Screen Reader Support

- Navigation buttons have accessible names
- Status bar updates are announced
- View changes are announced

---

## 10. References

### 10.1 Qt Documentation

- [QMainWindow](https://doc.qt.io/qtforpython/PySide6/QtWidgets/QMainWindow.html)
- [QStackedWidget](https://doc.qt.io/qtforpython/PySide6/QtWidgets/QStackedWidget.html)
- [QPushButton](https://doc.qt.io/qtforpython/PySide6/QtWidgets/QPushButton.html)
- [QSettings](https://doc.qt.io/qtforpython/PySide6/QtCore/QSettings.html)

### 10.2 Design Patterns

- Stacked Widget Pattern for view management
- Signal-Slot for decoupled communication
- Registry Pattern for dynamic view registration

### 10.3 Related Specs

- E01 Epic Specification
- F02 Settings Management (geometry persistence)
- F03 DI Container (dependency injection)

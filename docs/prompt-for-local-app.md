# Prompt: Split PRD into Epic Specs for Local App

## Context

I'm building **Joohan Trading System** as a **local desktop application** using **PySide6** (Qt for Python).

The PRD is located at: `docs/prd.md`

## Platform Decision Required

Before splitting into epics, I need to decide the **primary development platform**:

### Option A: Windows-First
**Pros:**
- Creon and Kiwoom brokers require Windows (COM/DLL-based APIs)
- Most Korean retail traders use Windows
- Single platform simplifies testing

**Cons:**
- Development environment setup more complex
- Cross-platform later requires significant refactoring
- Linux/Mac users excluded

### Option B: Linux-First (with Windows for broker integration)
**Pros:**
- Better development experience (CLI tools, Docker, etc.)
- Easier CI/CD setup
- Core logic can be cross-platform

**Cons:**
- Must run Windows VM/machine for broker testing
- Creon/Kiwoom integration requires Windows-specific code paths
- Two environments to maintain during development

### Option C: Cross-Platform from Start
**Pros:**
- Maximum flexibility
- Can use KIS (REST API) on any platform
- Future-proof

**Cons:**
- More complex architecture
- Broker abstraction layer needed upfront
- Slower initial development

**My Choice:** [DECIDE BEFORE PROCEEDING]

---

## Request

Based on the PRD at `docs/prd.md`, split the product into **epic specs** following this structure:

```
specs/
├── E01/
│   └── E01.spec.md             # Application Framework
├── E02/
│   └── E02.spec.md             # Storage Layer
├── E03/
│   └── E03.spec.md             # Broker Integration
├── ...
```

## Requirements for Epic Splitting

1. **Read the PRD thoroughly** (`docs/prd.md`)

2. **Create epics based on PRD sections:**
   - Section 5.0: Broker & Account Management
   - Section 5.1: Historical Data Collection
   - Section 5.2: Market Scanner
   - Section 5.3: Pattern Discovery
   - Section 5.4: Strategy Builder
   - Section 5.5: Backtesting
   - Section 5.6: Strategy Optimization
   - Section 5.7: Paper Trading
   - Section 5.8: Live Trading

3. **Add foundation epics not in PRD:**
   - Application Framework (main window, navigation, settings)
   - Storage Layer (local database for candles, formulas, results)
   - DSL Parser & Evaluator (formula language engine)

4. **For each epic spec, include:**
   - Overview and purpose
   - Dependencies on other epics
   - Tasks breakdown
   - UI mockups (from PRD or new)
   - Acceptance criteria
   - Platform-specific notes (Windows vs cross-platform)

5. **Consider PySide6 specifics:**
   - Use Qt signals/slots for inter-component communication
   - QThread for background operations
   - QSettings for configuration persistence
   - PyQtGraph or similar for charts

6. **Consider broker platform requirements:**
   - KIS: REST API (cross-platform)
   - Creon: COM automation (Windows only)
   - Kiwoom: DLL/OCX (Windows only)

## Suggested Epic Structure

| Epic | Name | Platform | Dependencies |
|------|------|----------|--------------|
| E01 | Application Framework | Cross-platform | None |
| E02 | Storage Layer | Cross-platform | E01 |
| E03 | Broker Integration | Windows (Creon/Kiwoom), Cross (KIS) | E01, E02 |
| E04 | Historical Data Collection | Depends on E03 | E02, E03 |
| E05 | DSL Parser & Evaluator | Cross-platform | E02 |
| E06 | Processing Engine | Cross-platform | E02, E05 |
| E07 | Market Scanner | Cross-platform | E04, E05, E06 |
| E08 | Pattern Discovery | Cross-platform | E07 |
| E09 | Strategy Builder | Cross-platform | E05, E08 |
| E10 | Backtesting | Cross-platform | E04, E06, E09 |
| E11 | Strategy Optimization | Cross-platform | E10 |
| E12 | Paper Trading | Depends on E03 | E03, E06, E09 |
| E13 | Live Trading | Windows (Creon/Kiwoom) | E03, E09, E12 |

## Technology Stack

| Component | Technology |
|-----------|------------|
| UI Framework | PySide6 (Qt 6) |
| Charts | PyQtGraph |
| Data Processing | pandas, numpy |
| Indicators | TA-Lib, pandas-ta |
| Backtesting | vectorbt (optional) |
| Local Database | DuckDB or SQLite |
| Broker (KIS) | REST API (aiohttp) |
| Broker (Creon) | pywin32 (COM) |
| Broker (Kiwoom) | pywin32 (DLL/OCX) |

## Output

Generate complete epic specs with:
1. Epic specs: `specs/E{XX}/E{XX}.spec.md` (e.g., `specs/E01/E01.spec.md`)
2. Feature specs within epics: `specs/E{XX}/F{YY}/E{XX}-F{YY}.spec.md`
3. Dependency graph showing build order
4. Estimated hours per epic
5. Platform requirements per epic

## Notes

- Focus on **Windows-first** if Creon/Kiwoom are essential
- Keep core logic (DSL, backtesting, indicators) **platform-agnostic**
- UI screens should match PRD mockups
- Rate limiters must be per-broker as specified in PRD

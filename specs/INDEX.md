# Joohan Trading System - Epic Specifications Index

## Overview

This document provides an index of all epic specifications for the Qt-based desktop trading application.

**Platform**: Linux-first development with gRPC integration to Windows broker servers
**Framework**: PySide6 (Qt for Python)
**Architecture**: DDD (Domain-Driven Design) with TDD (Test-Driven Development)

---

## Epic Summary

| Epic | Name | Status | Platform | Dependencies |
|------|------|--------|----------|--------------|
| [E01](E01/E01.spec.md) | Application Framework | Draft | Cross-platform | None |
| [E02](E02/E02.spec.md) | Storage Layer | Draft | Cross-platform | E01 |
| [E03](E03/E03.spec.md) | Broker Integration | Draft | Cross-platform (gRPC) | E01, E02 |
| [E04](E04/E04.spec.md) | Historical Data Collection | Draft | Cross-platform | E02, E03 |
| [E05](E05/E05.spec.md) | DSL Parser & Evaluator | Draft | Cross-platform | E02 |
| [E06](E06/E06.spec.md) | Processing Engine | Draft | Cross-platform | E02, E05 |
| [E07](E07/E07.spec.md) | Market Scanner | Draft | Cross-platform | E04, E05, E06 |
| [E08](E08/E08.spec.md) | Pattern Discovery | Draft | Cross-platform | E07 |
| [E09](E09/E09.spec.md) | Strategy Builder | Draft | Cross-platform | E05, E08 |
| [E10](E10/E10.spec.md) | Backtesting | Draft | Cross-platform | E04, E06, E09 |
| [E11](E11/E11.spec.md) | Strategy Optimization | Draft | Cross-platform | E10 |
| [E12](E12/E12.spec.md) | Paper Trading | Draft | Cross-platform | E03, E06, E09 |
| [E13](E13/E13.spec.md) | Live Trading | Draft | Cross-platform (gRPC) | E03, E09, E12 |

---

## Dependency Graph

```
                                ┌─────────────────────────────────────────┐
                                │              WAVE 1                      │
                                │                                          │
                                │            ┌───────┐                     │
                                │            │  E01  │                     │
                                │            │ Frame │                     │
                                │            └───┬───┘                     │
                                │                │                         │
                                └────────────────┼─────────────────────────┘
                                                 │
                ┌────────────────────────────────┼────────────────────────────┐
                │              WAVE 2            │                             │
                │                                │                             │
                │            ┌───────┐      ┌────┴────┐                       │
                │            │  E02  │◄─────│   E05   │                       │
                │            │Storage│      │   DSL   │                       │
                │            └───┬───┘      └────┬────┘                       │
                │                │               │                             │
                └────────────────┼───────────────┼─────────────────────────────┘
                                 │               │
        ┌────────────────────────┼───────────────┼────────────────────────────┐
        │              WAVE 3    │               │                             │
        │                        │               │                             │
        │   ┌───────┐       ┌────┴────┐     ┌────┴────┐                       │
        │   │  E03  │◄──────│   E06   │◄────│         │                       │
        │   │Broker │       │ Process │     │         │                       │
        │   └───┬───┘       └────┬────┘     │         │                       │
        │       │                │          │         │                       │
        └───────┼────────────────┼──────────┼─────────┼────────────────────────┘
                │                │          │         │
    ┌───────────┼────────────────┼──────────┼─────────┼────────────────────────┐
    │   WAVE 4  │                │          │         │                         │
    │           │                │          │         │                         │
    │      ┌────┴────┐      ┌────┴────┐     │         │                         │
    │      │   E04   │      │   E07   │◄────┘         │                         │
    │      │  Data   │      │ Scanner │               │                         │
    │      └────┬────┘      └────┬────┘               │                         │
    │           │                │                    │                         │
    └───────────┼────────────────┼────────────────────┼─────────────────────────┘
                │                │                    │
    ┌───────────┼────────────────┼────────────────────┼─────────────────────────┐
    │   WAVE 5  │                │                    │                         │
    │           │           ┌────┴────┐          ┌────┴────┐                    │
    │           │           │   E08   │          │   E09   │                    │
    │           │           │ Pattern │─────────►│Strategy │                    │
    │           │           └─────────┘          └────┬────┘                    │
    │           │                                     │                         │
    │      ┌────┴────────────────────────────────────┐│                         │
    │      │                                          ││                         │
    │      ▼                                          ▼│                         │
    │ ┌─────────┐                               ┌─────┴───┐                     │
    │ │   E10   │◄──────────────────────────────│         │                     │
    │ │Backtest │                               │         │                     │
    │ └────┬────┘                               │         │                     │
    │      │                                    │         │                     │
    └──────┼────────────────────────────────────┼─────────┼─────────────────────┘
           │                                    │         │
    ┌──────┼────────────────────────────────────┼─────────┼─────────────────────┐
    │      │         WAVE 6                     │         │                     │
    │      │                                    │         │                     │
    │ ┌────┴────┐                          ┌────┴────┐    │                     │
    │ │   E11   │                          │   E12   │◄───┘                     │
    │ │Optimize │                          │ Paper   │                          │
    │ └─────────┘                          └────┬────┘                          │
    │                                           │                               │
    └───────────────────────────────────────────┼───────────────────────────────┘
                                                │
    ┌───────────────────────────────────────────┼───────────────────────────────┐
    │                    WAVE 7                 │                               │
    │                                           │                               │
    │                                      ┌────┴────┐                          │
    │                                      │   E13   │                          │
    │                                      │  Live   │                          │
    │                                      └─────────┘                          │
    │                                                                           │
    └───────────────────────────────────────────────────────────────────────────┘
```

---

## Build Order (Waves)

### Wave 1: Foundation
| Epic | Name | Effort Estimate |
|------|------|-----------------|
| E01 | Application Framework | 2-3 weeks |

**Deliverables**: Main window, navigation, settings, DI container, logging

---

### Wave 2: Core Infrastructure
| Epic | Name | Effort Estimate |
|------|------|-----------------|
| E02 | Storage Layer | 1-2 weeks |
| E05 | DSL Parser & Evaluator | 2-3 weeks |

**Deliverables**: ClickHouse/MongoDB integration, repository pattern, formula language

**Can be developed in parallel.**

---

### Wave 3: Integration & Processing
| Epic | Name | Effort Estimate |
|------|------|-----------------|
| E03 | Broker Integration | 2-3 weeks |
| E06 | Processing Engine | 2 weeks |

**Deliverables**: gRPC broker gateway, rate limiter, batch processing

---

### Wave 4: Data & Scanning
| Epic | Name | Effort Estimate |
|------|------|-----------------|
| E04 | Historical Data Collection | 2 weeks |
| E07 | Market Scanner | 2 weeks |

**Deliverables**: Data collection UI, scanner with formula library

**Can be developed in parallel.**

---

### Wave 5: Analysis & Strategy
| Epic | Name | Effort Estimate |
|------|------|-----------------|
| E08 | Pattern Discovery | 2 weeks |
| E09 | Strategy Builder | 1-2 weeks |
| E10 | Backtesting | 2-3 weeks |

**Deliverables**: Pattern mining, strategy composition, backtest engine

---

### Wave 6: Validation
| Epic | Name | Effort Estimate |
|------|------|-----------------|
| E11 | Strategy Optimization | 2 weeks |
| E12 | Paper Trading | 2-3 weeks |

**Deliverables**: Grid search optimization, paper trading with real-time data

**Can be developed in parallel.**

---

### Wave 7: Production
| Epic | Name | Effort Estimate |
|------|------|-----------------|
| E13 | Live Trading | 3-4 weeks |

**Deliverables**: Real money trading with safety controls

---

## Technology Stack

| Component | Technology |
|-----------|------------|
| UI Framework | PySide6 (Qt 6.5+) |
| Charts | PyQtGraph |
| Data Processing | pandas, numpy |
| Technical Indicators | TA-Lib, pandas-ta |
| Time-Series Database | ClickHouse (shared with JTS) |
| Document Database | MongoDB (shared with JTS) |
| Cache | Redis (shared with JTS) |
| Broker Communication | gRPC (grpcio) |
| Testing | pytest, pytest-qt |
| Linting | ruff, mypy |
| Packaging | Poetry |

---

## Architecture Principles

### DDD (Domain-Driven Design)

```
src/
├── domain/           # Core business logic (no external dependencies)
├── application/      # Use cases and orchestration
├── infrastructure/   # External integrations (DB, gRPC)
└── presentation/     # Qt UI layer
```

### TDD (Test-Driven Development)

1. **Red**: Write failing test first
2. **Green**: Implement minimum code to pass
3. **Refactor**: Clean up while keeping tests green

### Key Patterns

- **Repository Pattern**: Abstract data access
- **Factory Pattern**: Object creation
- **Observer/Signal Pattern**: Qt signals for UI updates
- **Adapter Pattern**: Broker integration

---

## PRD Mapping

| PRD Section | Epic(s) |
|-------------|---------|
| 5.0 Broker & Account Management | E03 |
| 5.1 Historical Data Collection | E04 |
| 5.2 Market Scanner | E07 |
| 5.3 Pattern Discovery | E08 |
| 5.4 Strategy Builder | E09 |
| 5.5 Backtesting | E10 |
| 5.6 Strategy Optimization | E11 |
| 5.7 Paper Trading | E12 |
| 5.8 Live Trading | E13 |
| 6.x UI Requirements | E01, All |
| 7.x Formula Language | E05 |

---

## Cross-References

### Existing JTS Project

The existing web-based JTS at `../../project-jts/jts` provides:
- gRPC server implementation for Creon broker
- Proto definitions (`schemas/protobuf/`)
- DDD patterns (portfolio aggregate, position value objects)
- Broker registry and factory patterns

### Documentation

- PRD: `docs/prd.md`
- Platform Decision: `docs/prompt-for-local-app.md`

---

## Next Steps

1. **Review specs** with stakeholders
2. **Set up project** (Wave 1 - E01)
3. **Establish CI/CD** pipeline
4. **Begin implementation** with TDD

---

*Last Updated: 2025-12-28*

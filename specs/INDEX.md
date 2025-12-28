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
| [E02](E02/E02.spec.md) | Storage Layer | **Split** | Cross-platform | E01 |
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

## E02 Storage Layer - Detailed Breakdown

### Features

| Feature | Title | Tasks | Status |
|---------|-------|-------|--------|
| [E02-F01](E02/F01/E02-F01.spec.md) | Domain Entities | 3 | Draft |
| [E02-F02](E02/F02/E02-F02.spec.md) | Repository Interfaces | 2 | Draft |
| [E02-F03](E02/F03/E02-F03.spec.md) | Infrastructure Implementation | 6 | Draft |
| [E02-F04](E02/F04/E02-F04.spec.md) | Data Management Use Cases | 3 | Draft |

### Task Hierarchy

```
E02 Storage Layer
├── F01 Domain Entities
│   ├── T01 Create domain entities (Symbol, Candle, Formula)
│   ├── T02 Create value objects and enums [P]
│   └── T03 Implement CandleSeries aggregate
├── F02 Repository Interfaces
│   ├── T01 Define repository interfaces
│   └── T02 Implement InMemory repositories
├── F03 Infrastructure Implementation
│   ├── T01 Create ClickHouseManager [P]
│   ├── T02 Create MongoDBManager [P]
│   ├── T03 Implement ClickHouseCandleRepository
│   ├── T04 Implement ClickHouseSymbolRepository [P]
│   ├── T05 Implement MongoDBFormulaRepository [P]
│   └── T06 Implement MongoDBStrategyRepository [P]
└── F04 Data Management Use Cases
    ├── T01 Create ImportCandlesUseCase
    ├── T02 Create ExportCandlesUseCase [P]
    └── T03 Create data validation utilities [P]

[P] = Parallelizable with other [P] tasks in same feature
```

### Implementation Waves (within E02)

| Wave | Tasks | Parallel Opportunities |
|------|-------|------------------------|
| 1 | F01-T01, F01-T02, F01-T03 | T02 can run parallel with T01 |
| 2 | F02-T01, F02-T02 | Sequential |
| 3 | F03-T01 through T06 | T01/T02 parallel, T03-T06 parallel |
| 4 | F04-T01, F04-T02, F04-T03 | T02/T03 parallel, T01 last |

**Total: 4 Features, 14 Tasks**

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

# E10 Backtesting - Parallelization Report

**Generated**: 2025-12-30
**Epic**: E10 (Backtesting)
**Total Specs**: 6 Features, 18 Tasks

---

## Executive Summary

Epic E10 (Backtesting) has been split into 6 features and 18 tasks. Analysis shows:
- **4 Feature Waves** for implementation
- **Maximum parallelism**: 3 features in Wave 2
- **Critical path**: F01 → F02/F03/F04 → F05
- **Estimated time savings**: ~45% with parallel execution

---

## 1. Feature Dependency Graph

```mermaid
graph TD
    subgraph "Wave 1: Foundation"
        F01[E10-F01: Domain Models<br/>3 tasks | M complexity]
    end

    subgraph "Wave 2: Core Components"
        F02[E10-F02: Backtest Engine<br/>4 tasks | L complexity]
        F04[E10-F04: Metrics Calculator<br/>3 tasks | L complexity]
        F06[E10-F06: Export & Persistence<br/>2 tasks | M complexity]
    end

    subgraph "Wave 3: Integration"
        F03[E10-F03: Signal Evaluation<br/>2 tasks | M complexity]
    end

    subgraph "Wave 4: UI Layer"
        F05[E10-F05: UI Components<br/>4 tasks | L complexity]
    end

    F01 --> F02
    F01 --> F04
    F01 --> F06
    F01 --> F03
    F02 --> F03
    F01 --> F05
    F04 --> F05

    style F01 fill:#e1f5fe
    style F02 fill:#c8e6c9
    style F04 fill:#c8e6c9
    style F06 fill:#c8e6c9
    style F03 fill:#fff3e0
    style F05 fill:#fce4ec
```

---

## 2. Feature Wave Breakdown

### Wave 1: Foundation (1 feature)

| Feature | Title | Complexity | Tasks | Dependencies |
|---------|-------|------------|-------|--------------|
| E10-F01 | Domain Models | M | 3 | None |

**Rationale**: Foundation layer provides all data structures (BacktestConfig, BacktestResult, Trade, PerformanceMetrics) required by all other features.

### Wave 2: Core Components [P] (3 features - parallel)

| Feature | Title | Complexity | Tasks | Dependencies |
|---------|-------|------------|-------|--------------|
| E10-F02 | Backtest Engine Core | L | 4 | E10-F01 |
| E10-F04 | Metrics Calculator | L | 3 | E10-F01 |
| E10-F06 | Export and Persistence | M | 2 | E10-F01 |

**Rationale**: All three features depend only on F01 domain models. No file conflicts or data dependencies between them.

### Wave 3: Integration (1 feature)

| Feature | Title | Complexity | Tasks | Dependencies |
|---------|-------|------------|-------|--------------|
| E10-F03 | Signal Evaluation | M | 2 | E10-F01, E10-F02 |

**Rationale**: Signal evaluation requires the BacktestEngine infrastructure (from F02) to plug into.

### Wave 4: UI Layer (1 feature)

| Feature | Title | Complexity | Tasks | Dependencies |
|---------|-------|------------|-------|--------------|
| E10-F05 | UI Components | L | 4 | E10-F01, E10-F04 |

**Rationale**: UI widgets display metrics (from F04) and use domain models (from F01).

---

## 3. Task-Level Parallelization

### E10-F01: Domain Models (Wave 1)

```mermaid
graph LR
    T01[T01: Core Enums & Trade<br/>S | Foundation]
    T02[T02: Config & Supporting<br/>S | Models]
    T03[T03: Result & Metrics<br/>M | Aggregate]

    T01 --> T02
    T01 --> T03
    T02 --> T03
```

| Task | Parallel Group | Dependencies |
|------|----------------|--------------|
| T01 | wave-1-1 | None |
| T02 | wave-1-2 | T01 |
| T03 | wave-1-2 | T01, T02 |

### E10-F02: Backtest Engine Core (Wave 2)

```mermaid
graph LR
    T01[T01: Portfolio State<br/>M | Foundation]
    T02[T02: Engine Skeleton<br/>M | Structure]
    T03[T03: Simulation Loop<br/>L | Core]
    T04[T04: Equity Calculation<br/>S | Metrics]

    T01 --> T02
    T02 --> T03
    T02 --> T04
```

| Task | Parallel Group | Dependencies |
|------|----------------|--------------|
| T01 | wave-2-1 | F01 |
| T02 | wave-2-2 | T01 |
| T03 | wave-2-3-a | T02 |
| T04 | wave-2-3-a | T02 |

**[P] T03 and T04 can run in parallel** after T02.

### E10-F04: Metrics Calculator (Wave 2)

```mermaid
graph LR
    T01[T01: Return & Risk Metrics<br/>M | Calculations]
    T02[T02: Drawdown Analysis<br/>M | Analysis]
    T03[T03: Trade Stats & Orchestration<br/>M | Integration]

    T01 --> T03
    T02 --> T03
```

| Task | Parallel Group | Dependencies |
|------|----------------|--------------|
| T01 | wave-2-1-a [P] | F01 |
| T02 | wave-2-1-a [P] | F01 |
| T03 | wave-2-2 | T01, T02 |

**[P] T01 and T02 can run in parallel**.

### E10-F06: Export and Persistence (Wave 2)

```mermaid
graph LR
    T01[T01: CSV Export<br/>M | Export]
    T02[T02: Repository<br/>M | Persistence]
```

| Task | Parallel Group | Dependencies |
|------|----------------|--------------|
| T01 | wave-2-1-b [P] | F01 |
| T02 | wave-2-1-b [P] | F01, E02 |

**[P] T01 and T02 can run in parallel** (independent functionality).

### E10-F03: Signal Evaluation (Wave 3)

```mermaid
graph LR
    T01[T01: Signal Evaluation Logic<br/>M | Core]
    T02[T02: Position Management<br/>M | Lifecycle]

    T01 --> T02
```

| Task | Parallel Group | Dependencies |
|------|----------------|--------------|
| T01 | wave-3-1 | F01, F02 |
| T02 | wave-3-2 | T01 |

### E10-F05: UI Components (Wave 4)

```mermaid
graph LR
    T01[T01: TradeHistory<br/>M | Widget]
    T02[T02: MetricsSummary<br/>S | Widget]
    T03[T03: EquityCurve<br/>M | Chart]
    T04[T04: BacktestView<br/>L | Container]

    T01 --> T04
    T02 --> T04
    T03 --> T04
```

| Task | Parallel Group | Dependencies |
|------|----------------|--------------|
| T01 | wave-4-1-a [P] | F01 |
| T02 | wave-4-1-a [P] | F01, F04 |
| T03 | wave-4-1-a [P] | F01 |
| T04 | wave-4-2 | T01, T02, T03 |

**[P] T01, T02, T03 can all run in parallel**.

---

## 4. Complete Execution Timeline

```
Wave 1: Foundation
├── F01-T01: Core Enums & Trade [S]
├── F01-T02: Config & Supporting [S] (after T01)
└── F01-T03: Result & Metrics [M] (after T01, T02)

Wave 2: Core Components (3 features in parallel)
├── F02: Backtest Engine
│   ├── T01: Portfolio State [M]
│   ├── T02: Engine Skeleton [M] (after T01)
│   ├── T03: Simulation Loop [L] (after T02) [P with T04]
│   └── T04: Equity Calculation [S] (after T02) [P with T03]
├── F04: Metrics Calculator
│   ├── T01: Return & Risk Metrics [M] [P with T02]
│   ├── T02: Drawdown Analysis [M] [P with T01]
│   └── T03: Trade Stats [M] (after T01, T02)
└── F06: Export and Persistence
    ├── T01: CSV Export [M] [P with T02]
    └── T02: Repository [M] [P with T01]

Wave 3: Integration
└── F03: Signal Evaluation
    ├── T01: Signal Evaluation Logic [M]
    └── T02: Position Management [M] (after T01)

Wave 4: UI Layer
└── F05: UI Components
    ├── T01: TradeHistory [M] [P]
    ├── T02: MetricsSummary [S] [P]
    ├── T03: EquityCurve [M] [P]
    └── T04: BacktestView [L] (after T01-T03)
```

---

## 5. Time Estimates

### Per-Feature Estimates

| Feature | Tasks | Estimated Hours | With Parallelism |
|---------|-------|-----------------|------------------|
| E10-F01 | 3 | 8-12 | 8-12 (sequential) |
| E10-F02 | 4 | 16-24 | 12-18 (T03/T04 parallel) |
| E10-F04 | 3 | 12-16 | 8-12 (T01/T02 parallel) |
| E10-F06 | 2 | 8-12 | 4-6 (both parallel) |
| E10-F03 | 2 | 8-12 | 8-12 (sequential) |
| E10-F05 | 4 | 16-24 | 10-16 (T01-T03 parallel) |

### Total Time Comparison

| Execution Mode | Estimated Hours | Notes |
|----------------|-----------------|-------|
| **Serial** | 68-100 hours | All tasks sequential |
| **Feature Parallel** | 48-72 hours | Wave 2 features in parallel |
| **Full Parallel** | 38-54 hours | Tasks within features parallel |

**Savings with Full Parallelism: ~45%**

---

## 6. Recommended Implementation Order

### Phase 1: Foundation (Week 1)
1. **E10-F01**: Complete all domain models
   - T01 → T02 → T03

### Phase 2: Core Engine (Week 2-3, 3 parallel tracks)
**Track A: Backtest Engine**
1. E10-F02-T01: Portfolio State
2. E10-F02-T02: Engine Skeleton
3. E10-F02-T03 + T04 (parallel): Simulation Loop & Equity

**Track B: Metrics Calculator**
1. E10-F04-T01 + T02 (parallel): Return Metrics & Drawdown
2. E10-F04-T03: Trade Statistics

**Track C: Export/Persistence**
1. E10-F06-T01 + T02 (parallel): CSV Export & Repository

### Phase 3: Integration (Week 4)
1. **E10-F03**: Signal Evaluation
   - T01 → T02 (plugs into F02)

### Phase 4: UI (Week 5)
1. **E10-F05**: UI Components
   - T01 + T02 + T03 (parallel): Individual widgets
   - T04: Integration view

---

## 7. Worktree Strategy

For maximum parallelism, create worktrees for Wave 2:

```bash
# Create worktrees for Wave 2 features
git worktree add ../e10-f02 -b feature/e10-f02
git worktree add ../e10-f04 -b feature/e10-f04
git worktree add ../e10-f06 -b feature/e10-f06

# Each developer works on one feature
# Merge when wave complete, then proceed to Wave 3
```

---

## 8. Risk Analysis

### High Risk
- **E10-F02-T03 (Simulation Loop)**: Core logic, many edge cases
- **E10-F03-T01 (Signal Evaluation)**: Integration with FormulaService

### Medium Risk
- **E10-F04-T01/T02 (Metrics)**: Mathematical correctness critical
- **E10-F05-T03 (EquityCurve)**: PyQtGraph performance

### Low Risk
- **E10-F01 (Domain Models)**: Pure data structures
- **E10-F06 (Export)**: Straightforward CSV/DuckDB operations

---

## 9. Dependencies on External Epics

| External Epic | Used By | Interface |
|---------------|---------|-----------|
| E04 (Data) | F02 | ICandleRepository |
| E06 (Processing) | F02, F03 | FormulaService |
| E09 (Strategy) | F02, F03 | Strategy, RiskConfig |
| E02 (Storage) | F06 | DuckDB connection |
| E01 (Framework) | F05 | Base widget classes |

---

## 10. Summary

| Metric | Value |
|--------|-------|
| Total Features | 6 |
| Total Tasks | 18 |
| Waves | 4 |
| Max Parallelism | 3 features (Wave 2) |
| Parallel Task Groups | 5 groups across features |
| Serial Estimate | 68-100 hours |
| Parallel Estimate | 38-54 hours |
| **Time Savings** | **~45%** |

### Next Steps

1. Review this parallelization report
2. Begin E10-F01 implementation (foundation)
3. Create worktrees for Wave 2 when F01 complete
4. Follow recommended implementation order

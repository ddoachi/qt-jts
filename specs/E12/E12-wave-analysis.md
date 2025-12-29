# E12: Paper Trading - Wave Analysis

## Overview

This document analyzes the E12 epic's specs for parallel execution opportunities based on dependencies.

## Dependency Graph

```
External Dependencies:
├── E01: UI Framework
├── E03: Broker Integration
├── E06: Processing Engine
├── E09: Strategy Builder
└── E02: Storage Layer

E12 Internal Dependencies:
├── E12-F01: Domain Entities (no internal deps)
│   ├── E12-F01-T01: Session Entities [P]
│   ├── E12-F01-T02: Portfolio Entities [P]
│   └── E12-F01-T03: Signal Entities [P]
│
├── E12-F02: Paper Trading Engine (depends on F01, F05-T01)
│   ├── E12-F02-T01: SessionManager (depends on F01, F05-T01)
│   ├── E12-F02-T02: Runner (depends on F02-T01)
│   ├── E12-F02-T03: Signal Detection (depends on F02-T02) [P]
│   └── E12-F02-T04: Virtual Execution (depends on F02-T02) [P]
│
├── E12-F03: Real-Time Data (depends on E03)
│   ├── E12-F03-T01: Subscription (depends on E03)
│   └── E12-F03-T02: Price Updates (depends on F03-T01, F01-T02)
│
├── E12-F04: UI Components (depends on E01)
│   ├── E12-F04-T01: PaperTradingView (depends on E01)
│   ├── E12-F04-T02: PortfolioWidget (depends on F04-T01) [P]
│   ├── E12-F04-T03: SignalsWidget (depends on F04-T01) [P]
│   └── E12-F04-T04: SessionStatsWidget (depends on F04-T01) [P]
│
└── E12-F05: Persistence (depends on E02, F01)
    ├── E12-F05-T01: Session Persistence (depends on E02, F01)
    └── E12-F05-T02: Trade History (depends on F05-T01, F02)
```

## Wave Execution Plan

### Wave 1: Domain Foundation (Parallel - 3 tasks)
**Prerequisites**: E09 (Strategy entity reference - available)

| Spec ID | Title | Effort | Can Parallelize |
|---------|-------|--------|-----------------|
| E12-F01-T01 | Define Session Entities | M | ✅ Yes |
| E12-F01-T02 | Define Portfolio Entities | M | ✅ Yes |
| E12-F01-T03 | Define Signal Entities | M | ✅ Yes |

**Rationale**: All three domain entity tasks have no internal dependencies. They can be developed in parallel by different developers or in separate worktrees.

---

### Wave 2: Core Infrastructure (Parallel - 3 tasks)
**Prerequisites**: Wave 1 complete, E02 (Storage), E03 (Broker), E01 (UI)

| Spec ID | Title | Effort | Can Parallelize |
|---------|-------|--------|-----------------|
| E12-F05-T01 | Session Persistence | M | ✅ Yes |
| E12-F03-T01 | Real-Time Subscription | M | ✅ Yes |
| E12-F04-T01 | PaperTradingView | L | ✅ Yes |

**Rationale**:
- F05-T01 depends only on F01 (Wave 1) and E02 (external)
- F03-T01 depends only on E03 (external)
- F04-T01 depends only on E01 (external)

These three tasks work on different layers (persistence, data, UI) and can be parallelized.

---

### Wave 3: Engine & UI Components (Parallel - 4 tasks)
**Prerequisites**: Wave 2 complete

| Spec ID | Title | Effort | Can Parallelize |
|---------|-------|--------|-----------------|
| E12-F02-T01 | SessionManager | L | ✅ Yes |
| E12-F03-T02 | Position Price Updates | M | ✅ Yes |
| E12-F04-T02 | PortfolioWidget | M | ✅ Yes |
| E12-F04-T03 | SignalsWidget | M | ✅ Yes |
| E12-F04-T04 | SessionStatsWidget | M | ✅ Yes |

**Rationale**:
- F02-T01 depends on F01, F05-T01 (Waves 1, 2)
- F03-T02 depends on F03-T01, F01-T02 (Waves 1, 2)
- F04-T02/T03/T04 all depend on F04-T01 (Wave 2)

---

### Wave 4: Runner Implementation
**Prerequisites**: Wave 3 complete (F02-T01)

| Spec ID | Title | Effort | Can Parallelize |
|---------|-------|--------|-----------------|
| E12-F02-T02 | PaperTradingRunner | L | ❌ Sequential |

**Rationale**: Runner depends on SessionManager and is the core of the engine. Must be sequential as later tasks depend on it.

---

### Wave 5: Signal & Execution Logic (Parallel - 2 tasks)
**Prerequisites**: Wave 4 complete (F02-T02)

| Spec ID | Title | Effort | Can Parallelize |
|---------|-------|--------|-----------------|
| E12-F02-T03 | Signal Detection | M | ✅ Yes |
| E12-F02-T04 | Virtual Execution | M | ✅ Yes |

**Rationale**: Both extend the Runner with additional logic. They can be developed in parallel as they operate on different aspects (signal detection vs execution).

---

### Wave 6: Trade History
**Prerequisites**: Wave 5 complete (F02)

| Spec ID | Title | Effort | Can Parallelize |
|---------|-------|--------|-----------------|
| E12-F05-T02 | Trade History Recording | M | ❌ Final |

**Rationale**: Trade history depends on the complete engine (F02) to record executed trades.

---

## Summary

| Wave | Tasks | Parallel? | Total Effort |
|------|-------|-----------|--------------|
| 1 | 3 | ✅ Yes | M + M + M |
| 2 | 3 | ✅ Yes | M + M + L |
| 3 | 5 | ✅ Yes | L + M + M + M + M |
| 4 | 1 | ❌ No | L |
| 5 | 2 | ✅ Yes | M + M |
| 6 | 1 | ❌ No | M |

**Total**: 15 tasks across 6 waves

## Recommended Parallel Execution Strategy

### Option A: Maximum Parallelism (3 developers)
- Developer 1: F01 (entities) → F02 (engine)
- Developer 2: F03 (data) → F04-T01 (main view)
- Developer 3: F05-T01 (persistence) → F04-T02/T03/T04 (widgets)

### Option B: Feature-Focused (2 developers)
- Developer 1: F01 → F02 → F05
- Developer 2: F03 → F04

### Option C: Sequential (1 developer)
Follow waves 1 → 6 in order, parallelizing within each wave where possible.

## Worktree Commands

```bash
# Wave 1 - Parallel domain entities
/spec_work_worktree E12-F01-T01 --create
/spec_work_worktree E12-F01-T02 --create
/spec_work_worktree E12-F01-T03 --create

# Wave 2 - Parallel infrastructure
/spec_work_worktree E12-F05-T01 --create
/spec_work_worktree E12-F03-T01 --create
/spec_work_worktree E12-F04-T01 --create

# ... continue for subsequent waves
```

## Critical Path

The critical path (longest dependency chain) is:

```
E12-F01-T01 → E12-F02-T01 → E12-F02-T02 → E12-F02-T03/T04 → E12-F05-T02
```

This path determines the minimum time to complete the epic even with maximum parallelism.

---
*Generated: 2025-12-30*

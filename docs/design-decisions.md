# Design Decisions

## Why Stateless Skills + File-Based State

- Independent skill invocations (no persistent memory)
- All state in human-readable JSON/MD (version-controllable)
- Lazy loading enables token efficiency
- Easier to debug and understand system state

## Why No Database

- Simpler deployment and maintenance
- User can inspect/modify files directly
- File-based allows easy integration with version control
- Aligns with Claude Code's file-first philosophy

## Why Hybrid Roadmap Optimizer

- Simple logic (score < 70% → prioritize) handles most cases
- Deep analysis only when anomalies detected (3+ failures)
- Balances intelligence with token efficiency

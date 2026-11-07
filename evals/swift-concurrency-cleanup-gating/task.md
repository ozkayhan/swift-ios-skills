# Swift Concurrency Cleanup Gating

## Problem/Feature Description

A team on Swift 6.3 / Xcode 26.5 wants to update its cleanup code by:

- Adding async `defer` cleanup throughout the module.
- Wrapping every cancellation path in `withTaskCancellationShield`.
- Fixing a diagnostic from a closure with an explicit capture list passed to a
  `nonisolated(nonsending)` async parameter.

Review the plan and give the smallest source-grounded correction that preserves
Swift 6.3 behavior while identifying what requires Swift 6.4 / Xcode 27.

## Output Specification

Create `concurrency-cleanup-gating.md` with:

- The Swift/toolchain availability boundary for async `defer`.
- The Swift/toolchain/platform availability boundary for
  `withTaskCancellationShield`.
- How cancellation behaves inside async defer cleanup unless a cancellation
  shield is deliberately used.
- The documented Xcode 26.5 workaround options for the explicit-capture
  `nonisolated(nonsending)` closure issue.

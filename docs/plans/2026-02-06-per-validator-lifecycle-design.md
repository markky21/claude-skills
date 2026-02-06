# Per-Validator Lifecycle for Fix-Loop

## Problem

The fix-loop currently tracks termination globally: one `cleanLoopCount`, one `stallCount`, one `previousIssueCount`. All selected validators run every iteration regardless of their individual status. This wastes tokens on validators that already passed clean, and a single stuck validator stalls the entire loop.

## Solution

Each validator tracks its own lifecycle independently. Validators retire individually when they reach their clean pass threshold or stall out. The loop ends when all validators are retired (or global max iterations hit as safety net).

## Per-Validator State

Each validator gets a state object:

```
validatorState[name] = {
  status: "active" | "retired-clean" | "retired-stalled",
  cleanPasses: 0,        // consecutive clean iterations
  stallCount: 0,         // consecutive iterations with same issue count
  previousIssueCount: 0, // issue count from last run
  retiredAtIteration: null
}
```

## State Transitions

On each iteration, for each **active** validator:

**If findings == 0 (clean pass):**
- `cleanPasses++`
- `stallCount = 0`
- If `cleanPasses >= confirmationLoops` → status = `retired-clean`, record iteration

**If findings > 0:**
- `cleanPasses = 0` (reset - even if it had prior clean passes)
- If `findings == previousIssueCount` → `stallCount++`
- Else → `stallCount = 0`
- If `stallCount >= 2` → status = `retired-stalled`, record iteration
- `previousIssueCount = findings`

## Loop Termination

The loop ends when ANY of these is true:
1. All validators are retired (clean or stalled)
2. Global `maxIterations` reached (safety net)

## What Changes in fix-loop.md

### Step 3: Initialize Loop State

Replace global counters:

```
- previousIssueCount = Infinity     → REMOVE
- stallCount = 0                    → REMOVE
- cleanLoopCount = 0                → REMOVE
```

Add per-validator state map:

```
- validatorStates = {} (map of validator name → state object)
- Initialize each selected validator with:
    { status: "active", cleanPasses: 0, stallCount: 0, previousIssueCount: 0, retiredAtIteration: null }
```

Keep: `iteration`, `totalFixed`, `confirmationLoops`, `maxIterations`.

### Step 4a: Display Iteration Header

Show only **active** validators:

```
Running validators: {active validators only}
Retired: {count} ({list retired validators with status})
Scope: {file count} files changed vs main
```

### Step 4b: Run Validators in Parallel

Only launch Task agents for validators where `status == "active"`. Skip retired validators entirely.

### Step 4c-4d: Filter, Count, and Check Termination

Parse findings **per validator** (not globally). For each validator's output:

1. Count that validator's findings matching selected severity
2. Update that validator's state using the transition rules above
3. Log per-validator status:
   ```
   ddd-oop-validator: 0 issues (clean pass 1/2)
   clean-code-validator: 3 issues (stall 1/2)
   react-nextjs-validator: 0 issues → ✅ Retired clean (2/2)
   ```

After updating all states, check termination:
- If no active validators remain → go to Step 5
- If `iteration >= maxIterations` → go to Step 5
- Otherwise → continue to 4e

### Step 4e: Display Findings

Only display findings from active validators that had issues this iteration.

### Step 4f: Apply Fixes

Only pass findings from active validators (not retired ones) to the fixer agent.

### Step 5: Final Summary

Replace global summary with per-validator breakdown:

```
═══════════════════════════════════════════════════════════════
 ✅ Fix Loop Complete
═══════════════════════════════════════════════════════════════

Iterations: {used}/{max}

Validator Results:
  ✅ ddd-oop-validator:    Retired clean (2/2 passes) at iteration 3
  ✅ clean-code-validator: Retired clean (2/2 passes) at iteration 2
  ⚠️ react-nextjs-validator: Retired stalled at iteration 5
  ✅ dry-violations-detector: Retired clean (2/2 passes) at iteration 1

Issues fixed: {totalFixed}
Remaining issues: {sum of last findings from stalled validators}
```

## CLI Arguments

No changes needed. The `--confirm` flag already sets `confirmationLoops` which now applies per-validator instead of globally.

## Scope of Changes

Only one file changes: `plugins/my-personal-tools/commands/fix-loop.md`

Sections affected:
- Step 3 (initialization)
- Step 4a (iteration header)
- Step 4b (validator launch - filter to active only)
- Step 4c-4d (per-validator counting and termination)
- Step 4e (findings display)
- Step 4f (fixer input)
- Step 5 (summary)

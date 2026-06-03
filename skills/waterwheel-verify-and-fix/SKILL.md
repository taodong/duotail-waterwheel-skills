---
name: waterwheel-verify-and-fix
description: Run the full autonomous Waterwheel test-and-fix loop for a web feature. Use when the user wants a feature verified and any failures fixed automatically — implements the implement, test, debug, and re-test cycle until all tests pass. This is the main entry point for autonomous TDD with Waterwheel.
---

# Waterwheel — Verify and Fix

Orchestrates the full autonomous test-and-fix loop by coordinating the three Waterwheel component skills. Use this when the user wants a feature verified and any failing tests fixed without manual intervention.

For narrower needs, the component skills can be used directly instead:
- `waterwheel-verify-readiness` — only check the environment
- `waterwheel-run-tests` — only run the tests and report results
- `waterwheel-debug-feature` — only diagnose and fix a known failure

## Configuration

This loop respects `maxDebugAttempts` from `waterwheel.config.json` in the project root (default: `0`, meaning unlimited). It caps how many debug-and-re-test cycles may run before stopping. Treat `0` or a missing value as unlimited. Resolve it once at the start.

## Loop

1. **Verify readiness.** Use the `waterwheel-verify-readiness` skill to confirm the environment is ready. If it is not ready, stop and report what needs to be fixed. Do not continue.

2. **Run the tests.** Use the `waterwheel-run-tests` skill to execute the tests and read the results.

3. **If all executed tests passed**, report the feature as verified and stop.

4. **If any test failed**, use the `waterwheel-debug-feature` skill to diagnose and fix the cause in the application code.
   - If `waterwheel-debug-feature` reports that a test itself appears incorrect, stop the loop and surface that report to the user. Do not continue trying to fix.

5. **Return to step 2** to re-run the tests and confirm the fix.

6. **Repeat** until one of the following:
   - All tests pass → report success.
   - `maxDebugAttempts` is reached → stop and report the remaining failures and everything that was tried.
   - A test is reported as incorrect → stop and surface it to the user.
   - No meaningful progress is being made across several attempts → stop and ask the user how to proceed.

## Attempt counting

Count one attempt per debug-and-re-test cycle (steps 4–5). Track the count across the whole loop and enforce `maxDebugAttempts` here, at the orchestration level. The component skills do not track loop state themselves.

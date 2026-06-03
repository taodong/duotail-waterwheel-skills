---
name: waterwheel-debug-feature
description: Diagnose and fix a feature when its Waterwheel acceptance tests fail. Use after a failed acceptance test run to read the failure details, identify the root cause in the code, apply a fix, and prepare for a re-run.
---

# Waterwheel — Debug Feature

## Folder configuration

This skill uses these folder names by default:

- Tests folder: `tests`
- Outputs folder: `outputs`

Before running, look for a `waterwheel.config.json` file in the project root. If it exists, override the defaults with its values:

- Use `testsFolder` from the config in place of `tests`
- Use `outputsFolder` from the config in place of `outputs`

If the file does not exist, or a key is missing, keep the default for that value. Resolve the folder names once at the start and reuse them throughout.

## When to use

Use this skill when a `waterwheel-run-tests` run reported one or more failed tests. The goal is to find why the test failed and fix the underlying cause in the application code, not to change the test to make it pass.

## Diagnosis steps

1. **Read the failed result.** Open `test-results.json` in the resolved outputs folder and identify each test with status `failed`. Note the `result` detail, which describes what the agent observed.

2. **Read the per-test log.** For each failed test, open the matching `<test-name>_log.json` file in the resolved outputs folder. This records the steps the agent took and where it stopped. Use it to pinpoint the exact step that failed and what the page showed at that point.

3. **Locate the matching test spec.** Open the test's source Markdown file in the resolved tests folder to understand the intended behaviour and expected outcome.

4. **Identify the root cause in the application code.** Compare the expected behaviour against what the agent observed, then trace the cause to the relevant application code. Common causes include a missing or mislabelled element, a broken route, an unhandled state, or a regression from a recent change.

## Fixing

Apply a fix to the **application code**, not the test, unless the test itself is genuinely incorrect (in which case explain why before changing it). Make the smallest change that addresses the root cause.

After fixing, hand back to the `waterwheel-run-tests` skill to re-run the tests and confirm the fix. Repeat the diagnose-fix-verify loop until all tests pass.

## Important

Never modify a test purely to make a failing check pass. The test encodes the intended behaviour; the goal is to make the code meet it.
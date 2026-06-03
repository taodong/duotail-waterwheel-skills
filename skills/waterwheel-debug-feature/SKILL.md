---
name: waterwheel-debug-feature
description: Diagnose and fix a feature when its Waterwheel acceptance tests fail. Use after a failed acceptance test run to read the failure details, identify the root cause in the application code, and apply a fix.
---

# Waterwheel — Debug Feature

## Configuration

This skill uses these defaults:

- Tests folder: `tests`
- Outputs folder: `outputs`
- Application log path: none (optional)

Before running, look for a `waterwheel.config.json` file in the project root. If it exists, override the defaults with its values:

- Use `testsFolder` in place of `tests`
- Use `outputsFolder` in place of `outputs`
- Use `appLogPath` as the path to the web application's own log file, if provided

If the file does not exist, or a key is missing, keep the default for that value. Resolve the configuration once at the start and reuse it throughout.

## When to use

Use this skill when a test run reported one or more failed tests and you want to fix the cause. It performs a single diagnose-and-fix pass. Looping (re-running tests after a fix, and capping attempts) is handled by the `waterwheel-verify-and-fix` orchestration skill, not here.

## Diagnosis steps

1. **Read the failed result.** Open `test-results.json` in the resolved outputs folder and identify each test with status `failed`. Note the `result` detail, which describes what the agent observed.

2. **Read the per-test log.** For each failed test, open the matching `<test-name>_log.json` file in the resolved outputs folder. This records the steps the agent took and where it stopped. Use it to pinpoint the exact step that failed and what the page showed at that point.

3. **Read the application log, if available.** If `appLogPath` is set and the file exists, read it as an additional diagnostic source. Application-side errors and stack traces often reveal the root cause more directly than the browser-side observation. If `appLogPath` is not set, skip this step.

4. **Locate the matching test spec.** Open the test's source Markdown file in the resolved tests folder to understand the intended behaviour and expected outcome.

5. **Identify the root cause.** Compare the expected behaviour against what the agent observed (and any application-log errors), then trace the cause to the relevant application code. Common causes include a missing or mislabelled element, a broken route, an unhandled state, or a regression from a recent change.

## Fixing

Apply a fix to the **application code** only. Make the smallest change that addresses the root cause.

Never modify a test file. The test encodes the intended behaviour, and changing it to make a check pass defeats the purpose of the test. If your diagnosis concludes that the test itself is wrong — for example, it expects text or behaviour that contradicts the feature's actual requirements — do not edit it. Instead, **stop and report to the user** that the test appears incorrect, explaining specifically what the test expects, why you believe it is wrong, and what you would expect instead. Let the user decide how to correct the test.

After applying a fix, report what you changed. Re-running the tests to confirm the fix is the responsibility of the caller (typically the `waterwheel-verify-and-fix` skill, or the user).
---
name: waterwheel-run-tests
description: Run Waterwheel browser-based acceptance tests against a feature and read the results. Use after implementing a web feature to verify it works, by delegating test execution to the Waterwheel test agent and interpreting the pass/fail outcome.
---

# Waterwheel — Run Tests

## Configuration

This skill uses these defaults:

- Tests folder: `tests`
- Outputs folder: `outputs`
- Container name: `waterwheel-test-agent`

Before running, look for a `waterwheel.config.json` file in the project root. If it exists, override the defaults with its values:

- Use `testsFolder` in place of `tests`
- Use `outputsFolder` in place of `outputs`
- Use `containerName` in place of `waterwheel-test-agent`

If the file does not exist, or a key is missing, keep the default for that value. Resolve the configuration once at the start and reuse it throughout.

## Prerequisites

Before running the tests, confirm the QA environment is ready using the `waterwheel-verify-readiness` skill. If it is not ready, stop and report what needs to be fixed rather than attempting to run tests.

## Running the tests

Trigger the Waterwheel test agent to execute all tests in the resolved tests folder:

```shell
docker compose exec <containerName> run-qa
```

Substitute the resolved container name. Wait for the command to finish before reading results. The agent writes its output to the resolved outputs folder.

## Reading the results

After the run completes, read `test-results.json` from the resolved outputs folder. For each test it lists a `status` of `success`, `failed`, `skipped`, `abort`, or `ignored`.

Summarise the outcome for the user:

- If every executed test has status `success`, report the feature as verified.
- If any test has status `failed`, report which test(s) failed and include the `result` detail for each so the cause is clear.
- Note any `skipped` or `abort` tests, since these usually indicate an unmet dependency rather than a passing feature.

Do not report the feature as verified unless all executed tests passed.
If any test failed, what to do next depends on what the user asked for:
- If the user asked only to run the tests, or to report results without fixing, stop here and report the failures. Do not begin debugging or modifying any code.
- If the user asked to implement, fix, or verify-and-fix the feature (or gave no specific instruction about fixing), hand off to the waterwheel-debug-feature skill to investigate and fix the cause.

If you are unsure which the user wants, report the failures first and ask whether they would like you to debug and fix them.
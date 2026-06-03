---
name: waterwheel-run-tests
description: Run Waterwheel browser-based acceptance tests against a feature and read the results. Use after implementing a web feature to verify it works, by delegating test execution to the Waterwheel test agent and interpreting the pass/fail outcome.
---

# Waterwheel — Run Tests

## Folder configuration

This skill uses these folder names by default:

- Tests folder: `tests`
- Outputs folder: `outputs`

Before running, look for a `waterwheel.config.json` file in the project root. If it exists, override the defaults with its values:

- Use `testsFolder` from the config in place of `tests`
- Use `outputsFolder` from the config in place of `outputs`

If the file does not exist, or a key is missing, keep the default for that value. Resolve the folder names once at the start and reuse them throughout.

## Prerequisites

Before running the tests, confirm the QA environment is ready (the test container is running and the tests and outputs folders exist) using the `waterwheel-verify-readiness` skill. If it is not ready, stop and report what needs to be fixed rather than attempting to run tests.

## Running the tests

Trigger the Waterwheel test agent to execute all tests in the resolved tests folder:

```shell
docker compose exec waterwheel-test-agent run-qa
```

Wait for the command to finish before reading results. The agent writes its output to the resolved outputs folder.

## Reading the results

After the run completes, read `test-results.json` from the resolved outputs folder. For each test it lists a `status` of `success`, `failed`, `skipped`, `abort`, or `ignored`.

Summarise the outcome for the user:

- If every executed test has status `success`, report the feature as verified.
- If any test has status `failed`, report which test(s) failed and include the `result` detail for each so the cause is clear.
- Note any `skipped` or `abort` tests, since these usually indicate an unmet dependency rather than a passing feature.

Do not report the feature as verified unless all executed tests passed. If any test failed, hand off to the `waterwheel-debug-feature` skill to investigate and fix the cause.
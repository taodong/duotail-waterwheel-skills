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

## Running the tests

Trigger the Waterwheel test agent to execute all tests in the resolved tests folder:

```shell
docker compose exec <containerName> run-qa
```

Substitute the resolved container name. Wait for the command to finish before reading results. The agent writes its output to the resolved outputs folder.

## Reading the results

After the run completes, read `test-results.json` from the resolved outputs folder. For each test it lists a `status` of `success`, `failed`, `skipped`, `abort`, or `ignored`.

Summarise the outcome for the user:

- If every executed test has status `success` or `ignored`, report the feature as verified.
- If any test has status `failed`, report which test(s) failed and include the `result` detail for each so the cause is clear.
- Note any `skipped` or `abort` tests, since these usually indicate an unmet dependency rather than a passing feature.

Report the results and stop. Do not begin debugging or modify any code from this skill. If the user wants failures fixed automatically, that is handled by the `waterwheel-verify-and-fix` orchestration skill.
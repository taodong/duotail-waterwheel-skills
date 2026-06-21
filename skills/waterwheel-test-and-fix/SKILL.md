---
name: waterwheel-test-and-fix
description: Run the Waterwheel autonomous test-and-fix loop for a web feature. Use when the user says "run waterwheel", "run the tests", "verify the feature", "test and fix", or wants test failures diagnosed and fixed automatically.
---

# Waterwheel — Test and Fix

## Configuration

This skill uses these defaults:

- Container name: `waterwheel-agent`
- Test check interval: 30 seconds
- Application log path: none (optional)
- Max fix attempts: 0
- Deployment skill: none (optional)

Before running, look for a `waterwheel.config.json` file in the project root. If it exists, override the defaults with its values:

- Use `containerName` in place of `waterwheel-agent`
- Use `testCheckInterval` in place of `30` (seconds)
- Use `appLogPath` in place of `none` (optional)
- Use `maxFixAttempts` in place of `0`
- Use `deployInstruction` as the instructions for redeploying the application between fix attempts, in place of `none` if provided (optional)

If the file does not exist, or a key is missing, keep the default for that value. Resolve the configuration once at the start and reuse it throughout.

## 1. Ensure the agent is running

Check the container state:

```shell
docker inspect -f '{{.State.Running}}' <containerName>
```

- Prints `true` → it's running, proceed.
- Prints `false` → it exists but is stopped; start it with `docker start <containerName>`.
- Errors / "No such object" → the container doesn't exist; stop and tell the user to set up container `<containerName>` first.

## 2. Run the tests

Trigger the Waterwheel test agent to execute all user-defined tests by running this command in the background:

```shell
docker exec -d <containerName> run-qa
```

## 3. Check test results

Poll for results by running:

```shell
docker exec <containerName> check-test-result
```

- If the output indicates testing is in progress, including an orchestrator PID, the tests are still running. Wait `testCheckInterval` seconds, then run the command again. Repeat until a final result is returned.
- If the output contains `No test results found`, the test agent failed to produce or retrieve results.
   - If the command also prints `agent.log`, include that output in the report to the user.
   - Stop after reporting the failure.
- Otherwise, the output is the `exit_condition` from the results file. Evaluate it:
  - `"All tests passed"` → report that all tests passed and stop — the skill is complete.
  - `"One or more tests failed"` → report the failure and continue to step 4.
  - Any other value → report that the run did not complete, include the exit condition as the reason, and stop.

## 4. Investigate test failures

Fetch the full failure details by running:

```shell
docker exec <containerName> get-failure-detail -d
```

Use the returned details, along with the application logs at `appLogPath` (if configured and not `none`), to understand what the agent observed at the point of failure and diagnose the root cause.

## 5. Fix and retry

Attempt to fix the failures by repeating this cycle:

**Before each attempt**, check the attempt limit from `maxFixAttempts`:
- `0` or missing → fixing is not allowed; stop and report all remaining failures without making any code changes.
- Negative number → no limit; proceed.
- Positive number → track the number of fix-and-redeploy cycles completed so far. If the limit is reached, stop and report all remaining failures and a summary of everything that was tried.

**Each fix-and-retry cycle:**

1. Apply a fix to the application code based on the failure investigation from step 4.
2. Redeploy the application:
   - If `deployInstruction` is configured to a value other than `none`, follow the instructions to redeploy the application.
   - If `deployInstruction` is `none` or not configured, tell the user to manually redeploy with the fix applied, then stop and wait. The user will trigger the next run after redeployment.
3. Increment the fix attempt counter.
4. Return to **step 2** to re-run the full test suite.
5. After getting results in step 3:
   - If all tests pass, report success and stop.
   - If new failures remain, return to **step 4** to re-investigate before the next fix cycle.

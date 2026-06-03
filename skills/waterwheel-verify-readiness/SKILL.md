---
name: waterwheel-verify-readiness
description: Verify the Waterwheel QA environment is ready before running tests. Checks that the Docker test container is running and that the configured tests and outputs folders exist. Use this before running any Waterwheel acceptance tests.
---

# Waterwheel — Verify Readiness

## Configuration

This skill uses these defaults:

- Tests folder: `tests`
- Outputs folder: `outputs`
- Container name: `waterwheel-test-agent`

Before running the checks, look for a `waterwheel.config.json` file in the project root. If it exists, override the defaults with its values:

- Use `testsFolder` in place of `tests`
- Use `outputsFolder` in place of `outputs`
- Use `containerName` in place of `waterwheel-test-agent`

If the file does not exist, or a key is missing, keep the default for that value. Resolve the configuration once at the start and reuse it throughout.

## Checks

The QA environment is ready when all three conditions below are met:

1. **The test container is running:**
   ```shell
   docker ps --filter "name=<containerName>" --format "{{.Names}} {{.Status}}"
   ```
   Substitute the resolved container name. The command should return a line showing the container with an "Up" status.

2. **The tests folder exists and contains at least one Markdown file.** Check that the resolved tests folder is present and holds one or more `.md` files.

3. **The outputs folder exists.** Check that the resolved outputs folder is present.

## Reporting

Report the result of each of the three checks individually, clearly marking which passed and which failed. For any failed check, tell the user exactly what to fix (for example, start the container, add a test file, or create the outputs folder). Only report the environment as ready when all three checks pass.
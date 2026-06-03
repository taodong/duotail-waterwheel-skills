# Duotail Waterwheel Skills

Agent skills for pairing a code agent (such as Claude Code) with the [Waterwheel](https://waterwheel.duotail.com) browser test agent to run autonomous, test-driven web development.

## Skills

| Skill | Purpose |
|---|---|
| `waterwheel-verify-and-fix` | **Main entry point.** Runs the full autonomous loop: verify environment, run tests, fix failures, and re-test until all pass. |
| `waterwheel-verify-readiness` | Verify the Waterwheel test container is running and the tests/outputs folders exist. |
| `waterwheel-run-tests` | Run Waterwheel acceptance tests and report the pass/fail results. Reports only — does not fix. |
| `waterwheel-debug-feature` | Diagnose a failed test and fix the application code. Performs a single fix pass. |

`waterwheel-verify-and-fix` orchestrates the other three into the autonomous TDD loop. The three component skills can also be used on their own when you want just one step — for example, `waterwheel-run-tests` to check status without any code being changed.

## Installation

Install the main loop skill (it coordinates the others):

```shell
npx skills add taodong/duotail-waterwheel-skills@waterwheel-verify-and-fix
```

Or browse and select from the whole collection:

```shell
npx skills add taodong/duotail-waterwheel-skills
```

You will typically want all four installed so the orchestrator can call the component skills.

## Usage modes

- **Autonomous loop** — ask the agent to verify and fix a feature; it uses `waterwheel-verify-and-fix` to test, fix, and re-test until everything passes.
- **Run only** — ask the agent to run the tests or report results; it uses `waterwheel-run-tests` and stops after reporting, changing no code.
- **Debug only** — ask the agent to investigate a known failure; it uses `waterwheel-debug-feature` for a single diagnose-and-fix pass.

## Configuration

The skills work out of the box with sensible defaults. To customise, create a `waterwheel.config.json` in your project root. All keys are optional — any key you omit falls back to its default.

```json
{
  "testsFolder": "tests",
  "outputsFolder": "outputs",
  "containerName": "waterwheel-test-agent",
  "appLogPath": "",
  "maxDebugAttempts": 0
}
```

| Key | Default | Description |
|---|---|---|
| `testsFolder` | `tests` | Folder containing your Waterwheel test files. |
| `outputsFolder` | `outputs` | Folder where the agent writes test results. |
| `containerName` | `waterwheel-test-agent` | Name of the Waterwheel Docker container. |
| `appLogPath` | none | Path to your web application's own log file. When set, `waterwheel-debug-feature` reads it as an extra diagnostic source. |
| `maxDebugAttempts` | `0` (unlimited) | Caps the diagnose-fix-verify loop in `waterwheel-verify-and-fix`. Set a positive number to stop after that many attempts; `0` or omitted means no cap. |

The skills read this file each time they run, so the configuration stays consistent across all of them.
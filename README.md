# Duotail Waterwheel Skills

Agent skills for pairing a code agent (such as Claude Code) with the [Waterwheel](https://waterwheel.duotail.com) browser test agent to run autonomous, test-driven web development.

## Skills

| Skill | Purpose |
|---|---|
| `waterwheel-test-and-fix` | Run Waterwheel to perform predefined web tests, report pass/fail results, identify the root cause of failed tests, and then fix them when permissions are granted |


## Installation

```shell
npx skills add taodong/duotail-waterwheel-skills@waterwheel-test-and-fix
```

## Configuration

The skills work out of the box with sensible defaults. To customise, create a `waterwheel.config.json` in your project root. All keys are optional — any key you omit falls back to its default.

```json
{
  "containerName": "waterwheel-test-agent",
  "testCheckInterval": 30,
  "appLogPath": "none",
  "maxFixAttempts": 0,
  "deploymentSkill": "none"
}
```

| Key | Default | Description |
|---|---|---|
| `containerName` | `waterwheel-agent` | Name of the Waterwheel Docker container. |
| `testCheckInterval` | `30` | Time interval in seconds between test result checks. |
| `appLogPath` | `none` | Path to your web application's own log file. When set, `waterwheel-debug-feature` reads it as an extra diagnostic source. |
| `maxFixAttempts` | `0` (not allowed) | Caps the test and fix loop in `waterwheel-test-and-fix`. Set a positive number to stop after that many attempts; negative number means no cap. |
| `deploymentSkill` | `none` | Skill to run to redeploy the website for retesting. If not set, the fix won't be verified. |


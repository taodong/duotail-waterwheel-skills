# duotail-skills
Agent skills for duotail.com

## Duotail Skills

Agent skills from [Duotail](https://duotail.com). This collection includes skills for pairing a code agent (such as Claude Code) with the [Waterwheel](https://waterwheel.duotail.com) browser test agent to run autonomous, test-driven web development.

### Waterwheel Skills

| Skill | Purpose |
|---|---|
| `waterwheel-verify-readiness` | Verify the Waterwheel test container is running and the tests/outputs folders exist before running tests. |
| `waterwheel-run-tests` | Run Waterwheel acceptance tests against a feature and interpret the pass/fail results. |
| `waterwheel-debug-feature` | Diagnose and fix the application code when acceptance tests fail, then re-run. |

Together they form an autonomous loop: the code agent implements a feature, runs the acceptance tests, and — if any fail — debugs the cause, fixes the code, and re-runs until everything passes.

### Installation

Install an individual skill:

```shell
npx skills add taodong/duotail-skills@waterwheel-run-tests
```

Or browse and select from the whole collection:

```shell
npx skills add taodong/duotail-skills
```

## Configuration

All Waterwheel skills default to a `tests` folder for test files and an `outputs` folder for results. To use different folder names, create a `waterwheel.config.json` in your project root:

```json
{
  "testsFolder": "qa-tests",
  "outputsFolder": "qa-results"
}
```

Any key you omit falls back to its default. The skills read this file each time they run, so the configuration stays consistent across all of them. No setup step is required if you are happy with the defaults.
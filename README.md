# Duotail Waterwheel Skills

Agent skills for pairing a code agent (such as Claude Code) with the [Waterwheel](https://waterwheel.duotail.com) browser test agent to run autonomous, test-driven web development.

## Installation

Install (or update) any skill from this repo with the `skills` CLI, naming the skill after the `@`:

```shell
npx skills add taodong/duotail-waterwheel-skills@<skill-name>
```

For example, to install the full pipeline:

```shell
npx skills add taodong/duotail-waterwheel-skills@waterwheel-run
```

To install every skill in this repo at once, use `--skill '*'`:

```shell
npx skills add taodong/duotail-waterwheel-skills --skill '*'
```

## Skills

| Skill | Purpose |
|---|---|
| `waterwheel-run` | Run the full pipeline end to end: install the agent, load tests, load instructions, then run the test-and-fix loop |
| `waterwheel-test-and-fix` | Run Waterwheel to perform predefined web tests, report pass/fail results, identify the root cause of failed tests, and then fix them when permissions are granted |
| `waterwheel-load-tests` | (Re)load the markdown test tasks from the project's task folder into the Waterwheel agent container |
| `waterwheel-load-instructions` | (Re)load instruction and configuration files (instruction folder, preset context, global constants, domain permissions, local-host testing) into the Waterwheel agent container |
| `waterwheel-update-context` | Export context variables from the latest run and merge the chosen values into the project's `preset-context.json` `data` object, so future tests reuse them |
| `waterwheel-agent-install` | Install the Waterwheel test agent container into the local Docker environment, configuring its AI provider |
| `waterwheel-stop-container` | Stop the running Waterwheel test agent container without removing it, so it can be started again later |
| `waterwheel-agent-uninstall` | Uninstall the Waterwheel test agent by force-removing its Docker container |


## Configuration

The skills work out of the box with sensible defaults. To customise, create a `waterwheel.config.json` in your project root. All keys are optional — any key you omit falls back to its default.

```json
{
  "containerName": "waterwheel-agent",
  "imageName": "taojdcn/duotail-waterwheel:1.3.0",
  "allowRebuild": false,
  "aiProvider": "anthropic",
  "aiModel": "claude-sonnet-4-6",
  "aiKeyVarName": "AI_API_KEY",
  "tokenMode": "efficiency",
  "extraEnvVars": [],
  "taskFolder": "tasks",
  "instructionFolder": "none",
  "presetContextFile": "none",
  "globalConstantsFile": "none",
  "enableLocalTest": true,
  "testCheckInterval": 30,
  "appLogs": [],
  "maxFixAttempts": 0,
  "deployInstruction": "none"
}
```

| Key | Default | Description |
|---|---|---|
| `containerName` | `waterwheel-agent` | Name of the Waterwheel Docker container. |
| `imageName` | `taojdcn/duotail-waterwheel:1.3.0` | The Waterwheel agent Docker image to pull. Minimum supported version is `1.3.0`. |
| `allowRebuild` | `false` | When `false`, an existing container with the matching name is only ever started — never rebuilt. When `true`, if a matching container fails to start or its AI configuration has changed, `waterwheel-agent-uninstall` is run to remove it and a fresh installation is performed. |
| `aiProvider` | `anthropic` | The AI provider for your API key. Supported values: `anthropic`, `openai`, `gemini`, `deepseek`, or `gemma`. |
| `aiModel` | `claude-sonnet-4-6` | The AI model the Waterwheel agent should use. |
| `aiKeyVarName` | `AI_API_KEY` | The environment variable name which holds your AI API key. For safety reasons, a raw key value isn't accepted. |
| `tokenMode` | `efficiency` | Whether to apply minimum token usage features. Accepts two values: `default` (no) and `efficiency` (yes). |
| `extraEnvVars` | `[]` | A list of extra environment variables to set when installing the agent container. Each entry can be either `key=value`, or just `key` if the key is an existing environment variable on the local system. |
| `taskFolder` | `tasks` | The folder containing the markdown-format test files. |
| `instructionFolder` | `none` | If provided, all files in this folder are copied into the `/agent/instructions` folder in the container. Files are copied once, when the container is installed. |
| `presetContextFile` | `none` | A JSON file to install into the container as `/agent/instructions/preset-context.json`. If `instructionFolder` already contains a `preset-context` file, that file takes precedence. `waterwheel-update-context` also **writes** to this path — see [Security considerations](#security-considerations). |
| `globalConstantsFile` | `none` | A JSON file to install into the container as `/agent/instructions/global-context.json`. Works the same way as `presetContextFile`. |
| `enableLocalTest` | `true` | Set to `true` when the test website is hosted on the local machine (for example, `http://localhost:8080`), so the agent knows the test site is on the host machine rather than its own localhost. |
| `allowedDomains` | `["http://localhost:8080", "http://localhost:3000", "http://localhost:5173"]` | The web domains the agent is allowed to visit. See the [Playwright MCP guide](https://waterwheel.duotail.com/docs/mcp-guide#playwright-mcp) for details. |
| `testCheckInterval` | `30` | Time interval in seconds between test result checks. |
| `appLogs` | `[]` | A list of paths to your web application's own log files. When non-empty, `waterwheel-test-and-fix` reads each listed log as an extra diagnostic source. |
| `maxFixAttempts` | `0` (not allowed) | Caps the test and fix loop in `waterwheel-test-and-fix`. Set a positive number to stop after that many attempts; negative number means no cap. |
| `deployInstruction` | `none` | Instructions of redeploying the website. If the website is auto deployed, you can use something like "wait 10 seconds". If not set, the fix won't be verified. |

## Security considerations

These skills drive Docker and your local files on your behalf. They are designed for trusted, local development use. Be aware of the following before running them, especially in shared or automated environments:

- **`presetContextFile` is a write target you fully control.** `waterwheel-update-context` writes the resolved `presetContextFile` (or `<instructionFolder>/preset-context.json`) back to disk, creating the file and any missing parent directory. The path is intentionally unrestricted so you can point it anywhere — for example, swapping it per test environment — but that also means a wrong or hostile value can cause a write outside your project. Point it only at a path you trust, and review the value when you switch environments. The skill always shows you the exported values and asks which keys to merge before writing.
- **Persisted values come from inside the container.** The context values saved by `waterwheel-update-context` are generated by the agent during a run (for example, AI-created usernames or passwords) and are treated as untrusted data. They are written as plain JSON into your preset-context file — avoid committing secrets to version control.
- **`waterwheel-agent-install` reads your AI API key from the environment.** The key is referenced only by the `aiKeyVarName` environment variable and passed into the container; the skills never print, log, or accept a raw key value. Keep that variable scoped to your shell session.
- **`waterwheel-test-and-fix` can edit and redeploy your code.** When `maxFixAttempts` is non-zero it will modify application source and, if `deployInstruction` is set, redeploy. With the default `maxFixAttempts: 0` it only reports and never changes code.
- **Container lifecycle is destructive.** `waterwheel-agent-install` (on rebuild) and `waterwheel-agent-uninstall` force-remove the named container with `docker rm -f`. Make sure `containerName` refers to the Waterwheel container and not something you want to keep.

Each `SKILL.md` opens with a **Capabilities** note summarizing exactly what that skill does.


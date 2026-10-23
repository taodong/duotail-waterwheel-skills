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

### Pinning a version

The commands above track the repository's default branch, so they always fetch the latest published skills. To pin to a specific release instead, add `#<tag>` before the `@<skill-name>`:

```shell
# Install a skill from a specific release
npx skills add taodong/duotail-waterwheel-skills#v3.0.0@waterwheel-run

# Pin the whole repo to a release
npx skills add taodong/duotail-waterwheel-skills#v3.0.0 --skill '*'
```

`#<tag>` accepts any git tag or branch. Pin a version when a newer release raises the required agent image (`imageName`) or otherwise changes behavior you're not ready to adopt. Each release pairs with an agent image line:

| Skills release | Agent image (`imageName`) |
|---|---|
| `v3.0.0` | `1.5.0`+ |
| `v2.0.0` | `1.4.0` |
| `v1.0.0` | earlier than `1.4.0` |

For example, to stay on agent image `1.4.0`, pin `#v2.0.0`. Releases are listed on the repository's [tags page](https://github.com/taodong/duotail-waterwheel-skills/tags).

## Skills

| Skill | Purpose |
|---|---|
| `waterwheel-run` | Run the full pipeline end to end: install the agent, load tests, load instructions, load test skills, then run the test-and-fix loop |
| `waterwheel-test-and-fix` | Run Waterwheel to perform predefined web tests, report pass/fail results, identify the root cause of failed tests, and then fix them when permissions are granted |
| `waterwheel-rerun-tests` | Replay a subset of tests from the last full run's checkpoints — all of them, a range between two test names, or a single test, optionally with data overrides and repeated N times; reports and diagnoses failures without fixing them, and after a successful rerun offers to preserve its updated context into `preset-context.json` (requires agent image `1.5.0`+) |
| `waterwheel-load-tests` | (Re)load the markdown test tasks from the project's task folder into the Waterwheel agent container |
| `waterwheel-load-instructions` | (Re)load instruction and configuration files (instruction folder, preset context, global constants, domain permissions, local-host testing) into the Waterwheel agent container |
| `waterwheel-load-test-skills` | (Re)load the project's user-defined test skills from the `testSkills` folder into the Waterwheel agent container (requires agent image `1.4.0`+) |
| `waterwheel-update-context` | Export context variables from the latest **full run** and merge the chosen values into the project's `preset-context.json` `data` object, so future tests reuse them. Full runs only — to keep a rerun's context, use the preserve option in `waterwheel-rerun-tests` |
| `waterwheel-agent-install` | Install the Waterwheel test agent container into the local Docker environment, configuring its AI provider |
| `waterwheel-stop-container` | Stop the running Waterwheel test agent container without removing it, so it can be started again later |
| `waterwheel-agent-uninstall` | Uninstall the Waterwheel test agent by force-removing its Docker container |


## Configuration

The skills work out of the box with sensible defaults. To customise, create a `waterwheel.config.json` in your project root. All keys are optional — any key you omit falls back to its default.

```json
{
  "containerName": "waterwheel-agent",
  "imageName": "taojdcn/duotail-waterwheel:1.5.0",
  "allowRebuild": false,
  "aiProvider": "anthropic",
  "aiModel": "claude-sonnet-4-6",
  "aiBaseUrl": "none",
  "aiKeyVarName": "AI_API_KEY",
  "tokenMode": "efficiency",
  "extraEnvVars": [],
  "taskFolder": "tasks",
  "testSkills": "none",
  "instructionFolder": "none",
  "presetContextFile": "none",
  "globalConstantsFile": "none",
  "enableLocalTest": true,
  "allowedDomains": ["http://localhost:8080", "http://localhost:3000", "http://localhost:5173"],
  "testCheckInterval": 30,
  "appLogs": [],
  "maxFixAttempts": 0,
  "deployInstruction": "none"
}
```

| Key | Default | Description |
|---|---|---|
| `containerName` | `waterwheel-agent` | Name of the Waterwheel Docker container. |
| `imageName` | `taojdcn/duotail-waterwheel:1.5.0` | The Waterwheel agent Docker image to pull. Each skill states its own minimum: `waterwheel-rerun-tests` needs `1.5.0` (for `generate-rerun-config`, `rerun-tests`, and `get-test-report`) and `waterwheel-load-test-skills` needs `1.4.0` (for `load-test-skills`). Setting `aiBaseUrl` — and therefore using `aiProvider: openai-compatible` at all — also needs `1.5.0`, which is where `config-ai-provider --base-url` was added. Pinning an image older than `1.5.0` leaves those skills failing with an explicit message. |
| `allowRebuild` | `false` | When `false`, an existing container with the matching name is only ever started — never rebuilt. When `true`, if a matching container fails to start or its AI configuration has changed, `waterwheel-agent-uninstall` is run to remove it and a fresh installation is performed. |
| `aiProvider` | `anthropic` | The AI provider for your API key. Supported values: `anthropic`, `openai`, `gemini`, `deepseek`, `gemma`, or `openai-compatible`. Use `openai-compatible` for any vendor speaking the OpenAI Chat Completions format — OpenRouter, Groq, Together, Fireworks, xAI, or a self-hosted vLLM / LM Studio / LiteLLM proxy — and set `aiBaseUrl` alongside it. |
| `aiModel` | `claude-sonnet-4-6` | The AI model the Waterwheel agent should use. |
| `aiBaseUrl` | `none` | Base URL for the AI endpoint. **Required when `aiProvider` is `openai-compatible`** — give the vendor's full base path including any version segment (for example `https://openrouter.ai/api/v1`), since only `/chat/completions` is appended to it. For `gemma` it is the bare Ollama host (for example `http://host.docker.internal:11434`). Every other provider ignores it. `none` means not configured. Requires agent image `1.5.0` or newer. |
| `aiKeyVarName` | `AI_API_KEY` | The environment variable name which holds your AI API key. For safety reasons, a raw key value isn't accepted. |
| `tokenMode` | `efficiency` | Whether to apply minimum token usage features. Accepts two values: `default` (no) and `efficiency` (yes). |
| `extraEnvVars` | `[]` | A list of extra environment variables to set when installing the agent container. Each entry can be either `key=value`, or just `key` if the key is an existing environment variable on the local system. |
| `taskFolder` | `tasks` | The folder containing the markdown-format test files. |
| `testSkills` | `none` | The host folder under which each user-defined test skill lives in its own subfolder (each holding a `SKILL.md`). When set, `waterwheel-load-test-skills` loads every such skill into the agent; `none` means no test skills are loaded. Requires agent image `1.4.0` or newer. |
| `instructionFolder` | `none` | If provided, all files in this folder are copied into the `/agent/instructions` folder in the container. Files are copied once, when the container is installed. |
| `presetContextFile` | `none` | A JSON file to install into the container as `/agent/instructions/preset-context.json`. If `instructionFolder` already contains a `preset-context` file, that file takes precedence. `waterwheel-update-context` and `waterwheel-rerun-tests` (when you choose to preserve a rerun's context) also **write** to this path — see [Security considerations](#security-considerations). |
| `globalConstantsFile` | `none` | A JSON file to install into the container as `/agent/instructions/global-context.json`. Works the same way as `presetContextFile`. |
| `enableLocalTest` | `true` | Set to `true` when the test website is hosted on the local machine (for example, `http://localhost:8080`), so the agent knows the test site is on the host machine rather than its own localhost. |
| `allowedDomains` | `["http://localhost:8080", "http://localhost:3000", "http://localhost:5173"]` | The web domains the agent is allowed to visit. See the [Playwright MCP guide](https://waterwheel.duotail.com/docs/mcp-guide#playwright-mcp) for details. |
| `testCheckInterval` | `30` | Time interval in seconds between test result checks. |
| `appLogs` | `[]` | A list of paths to your web application's own log files. When non-empty, `waterwheel-test-and-fix` reads each listed log as an extra diagnostic source. |
| `maxFixAttempts` | `0` (not allowed) | Caps the test and fix loop in `waterwheel-test-and-fix`. Set a positive number to stop after that many attempts; negative number means no cap. |
| `deployInstruction` | `none` | Instructions of redeploying the website. If the website is auto deployed, you can use something like "wait 10 seconds". If not set, the fix won't be verified. |

## Security considerations

These skills drive Docker and your local files on your behalf. They are designed for trusted, local development use. Be aware of the following before running them, especially in shared or automated environments:

- **`presetContextFile` is a write target you fully control.** `waterwheel-update-context` (and `waterwheel-rerun-tests`, when you choose to preserve a successful rerun's context) writes the resolved `presetContextFile` (or `<instructionFolder>/preset-context.json`) back to disk, creating the file and any missing parent directory. The path is intentionally unrestricted so you can point it anywhere — for example, swapping it per test environment — but that also means a wrong or hostile value can cause a write outside your project. Point it only at a path you trust, and review the value when you switch environments. Both skills always show you the exported values and ask which keys to merge before writing.
- **Persisted values come from inside the container.** The context values saved by `waterwheel-update-context` and `waterwheel-rerun-tests` are generated by the agent during a run (for example, AI-created usernames or passwords) and are treated as untrusted data. They are written as plain JSON into your preset-context file — avoid committing secrets to version control.
- **`waterwheel-agent-install` reads your AI API key from the environment.** The key is referenced only by the `aiKeyVarName` environment variable and passed into the container; the skills never print, log, or accept a raw key value. Keep that variable scoped to your shell session.
- **`waterwheel-test-and-fix` can edit and redeploy your code.** When `maxFixAttempts` is non-zero it will modify application source and, if `deployInstruction` is set, redeploy. With the default `maxFixAttempts: 0` it only reports and never changes code.
- **Container lifecycle is destructive.** `waterwheel-agent-install` (on rebuild) and `waterwheel-agent-uninstall` force-remove the named container with `docker rm -f`. Make sure `containerName` refers to the Waterwheel container and not something you want to keep.

Each `SKILL.md` opens with a **Capabilities** note summarizing exactly what that skill does.


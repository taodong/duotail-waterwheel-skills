---
name: waterwheel-agent-install
description: Install the Waterwheel test agent container into the local Docker environment. Use when the user says "install waterwheel", "install waterwheel agent", "start waterwheel agent", "fix waterwheel container", or similar.
---

# Waterwheel — Agent Install

> **Capabilities:** This skill runs Docker commands (`version`, `inspect`, `start`, `run`, `rm -f`), pulls and runs the remote image `<imageName>`, reads the AI API key from a host environment variable and passes it into the container, and executes in-container configuration commands. It does **not** write to the host filesystem.

## Configuration

This skill uses these defaults:

- Container name: `waterwheel-agent`
- Image name: `taojdcn/duotail-waterwheel:1.5.0`
- Allow rebuild: `false`
- AI provider: `anthropic`
- AI model: `claude-sonnet-4-6`
- AI base URL: `none`
- AI key variable name: `AI_API_KEY`
- Token mode: `efficiency`
- Extra environment variables: none (empty list)

Before running, look for a `waterwheel.config.json` file in the project root. If it exists, override the defaults with its values:

- Use `containerName` in place of `waterwheel-agent`
- Use `imageName` in place of `taojdcn/duotail-waterwheel:1.5.0`
- Use `allowRebuild` in place of `false`
- Use `aiProvider` in place of `anthropic`
- Use `aiModel` in place of `claude-sonnet-4-6`
- Use `aiBaseUrl` in place of `none`
- Use `aiKeyVarName` in place of `AI_API_KEY`
- Use `tokenMode` in place of `efficiency`
- Use `extraEnvVars` in place of the empty list

If the file does not exist, or a key is missing, keep the default for that value. Resolve the configuration once at the start and reuse it throughout.

## 1. Verify Docker is available

Confirm Docker is installed and reachable:

```shell
docker version
```

- Succeeds → proceed.
- Errors / command not found → fail fast: report that Docker is not installed or not running, and stop.

## 2. Check for an existing container

Inspect the container named `<containerName>`:

```shell
docker inspect -f '{{.State.Running}}' <containerName>
```

- Prints `true` → the container exists and is already running; go to step 3.
- Prints `false` → the container exists but is stopped; start it with `docker start <containerName>`, then go to step 3.
  - If `docker start` fails → treat this as a container that cannot start: go to step 4 (the discrepancy path forces removal there). If `allowRebuild` is `false`, instead report that the existing container failed to start and that `allowRebuild` must be `true` to rebuild it, then stop.
- Errors / "No such object" → no matching container exists; skip to step 5.

## 3. Decide whether to keep the running container

The container is now running. Check `allowRebuild`:

- `false` → report that an existing Waterwheel container was found and started, then stop — the skill is complete.
- `true` → continue to step 4.

## 4. Compare the AI configuration

Read the running container's AI configuration:

```shell
docker exec <containerName> display-ai-config
```

Compare the returned values against the resolved configuration (`aiProvider`, `aiModel`, `tokenMode`, and `aiBaseUrl`).

Compare `aiBaseUrl` only when it is set to something other than `none`, and treat a `display-ai-config` response with no `aiBaseUrl` key as "no value". Images older than `1.5.0` do not report it, and a missing key must not be read as drift on every container. Two `openai-compatible` containers can otherwise agree on provider, model and token mode while pointing at entirely different vendors — the case this comparison exists to catch.

- All values match → report that an existing Waterwheel container was found and started, then stop — the skill is complete.
- Any discrepancy → force-remove the container so a fresh one can be installed:

```shell
docker rm -f <containerName>
```

Then continue to step 5.

## 5. Check the provider has what it needs

If `aiProvider` is `openai-compatible` and `aiBaseUrl` is `none`, **stop**: tell the user that
`openai-compatible` requires `aiBaseUrl` in `waterwheel.config.json`, set to the vendor's full base
path including any version segment (for example `https://openrouter.ai/api/v1`). Do not install.

This check runs before anything is pulled or started, because step 7 would otherwise fail on
`Error: --base-url is required for provider 'openai-compatible'.` after a container already exists —
leaving behind a container that cannot be configured.

Any other provider: continue.

## 6. Pull and install the image

Run a new container from `<imageName>`:

```shell
docker run -d --name <containerName> -e AI_API_KEY=$<aiKeyVarName> [extraEnvVars] <imageName>
```

Build the command as follows:

- `$<aiKeyVarName>` is the host environment variable named by `aiKeyVarName` (default `AI_API_KEY`). Its value is passed as the container's `AI_API_KEY`.
  - **Handle the key as a secret:** reference it only by shell expansion (`$<aiKeyVarName>`). Never resolve, print, echo, or log its value, and never include its value in any report or message to the user. If the host variable is unset or empty, stop and tell the user to set `<aiKeyVarName>` — do not substitute a literal value.
- `[extraEnvVars]` is replaced by the `extraEnvVars` list:
  - If the list is empty, omit `[extraEnvVars]` entirely.
  - Otherwise, prefix each element with `-e`. For example, `["MAX_SNAPSHOT_HISTORY=3", "ENABLE_API_LOGGING"]` becomes `-e MAX_SNAPSHOT_HISTORY=3 -e ENABLE_API_LOGGING`.
  - Ignore any `AI_API_KEY` entry in `extraEnvVars`; it is already set above.

- On success → continue to step 7.
- On error → report the error to the user and stop.

## 7. Configure the AI provider

With the container running, configure its AI provider:

```shell
docker exec <containerName> config-ai-provider --provider <aiProvider> --model <aiModel> --mode <tokenMode> [--base-url <aiBaseUrl>]
```

- `[--base-url <aiBaseUrl>]` is included only when `aiBaseUrl` is not `none`; omit it entirely otherwise.
  - Required by `openai-compatible` (already checked in step 5) and used by `gemma` as the Ollama host. Every other provider ignores it and the command warns.

- On success → continue to step 8.
- On error:
  - **The command rejects `--base-url`** (error text such as `Unknown option: --base-url`) → the image is older than `1.5.0`, which is where `--base-url` was added. **Stop**: tell the user to set `imageName` to at least `taojdcn/duotail-waterwheel:1.5.0` and reinstall the container.
  - Otherwise → report the error to the user and stop.

## 8. Report and suggest next steps

Report that the Waterwheel container `<containerName>` is installed and running. Suggest that the next step is to either load tests or load instructions for the agent.

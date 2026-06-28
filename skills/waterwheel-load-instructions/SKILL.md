---
name: waterwheel-load-instructions
description: Load the project's instruction and configuration files (instruction folder, preset context, global constants, domain permissions, local-host testing) into the Waterwheel agent container. Use when the user says "load waterwheel instructions", "reload waterwheel instructions", "load web test instructions", "reload web test instructions", "load web test configurations", or "reload web test configurations".
---

# Waterwheel — Load Instructions

> **Capabilities:** This skill runs Docker commands (`inspect`, `start`, `exec`), reads instruction/config files from the host (instruction folder, preset-context, global-constants), and pipes their contents into the container along with the domain allowlist and local-host settings. It does **not** write to the host filesystem or read any secrets.

This skill resets the Waterwheel agent's instruction files and reloads them from the project configuration: an optional instruction folder, an optional preset-context file, an optional global-constants file, the domain allowlist, and local-host testing.

Steps 1–3 fail fast: if one cannot complete, stop and report the error. Steps 4–8 are best-effort: capture each error but keep going, and report everything together in step 9.

## Configuration

This skill uses these defaults:

- Container name: `waterwheel-agent`
- Instruction folder: `none` (optional)
- Preset context file: `none` (optional)
- Global constants file: `none` (optional)
- Allowed domains: `["http://localhost:8080", "http://localhost:3000", "http://localhost:5173"]`
- Enable local test: `true`

Before running, look for a `waterwheel.config.json` file in the project root. If it exists, override the defaults with its values:

- Use `containerName` in place of `waterwheel-agent`
- Use `instructionFolder` in place of `none`
- Use `presetContextFile` in place of `none`
- Use `globalConstantsFile` in place of `none`
- Use `allowedDomains` in place of the default list
- Use `enableLocalTest` in place of `true`

If the file does not exist, or a key is missing, keep the default for that value. Resolve the configuration once at the start and reuse it throughout. Optional paths use the sentinel `none` to mean "not configured."

## 1. Verify the container exists

Confirm a container matching `<containerName>` exists:

```shell
docker inspect -f '{{.State.Running}}' <containerName>
```

- Prints `true` → the container exists and is running; skip to step 3.
- Prints `false` → the container exists but is stopped; continue to step 2.
- Errors / "No such object" / any other error → **fail fast**: stop and tell the user no container named `<containerName>` was found and that they must set it up first.

## 2. Start the container

If the container is stopped, start it:

```shell
docker start <containerName>
```

- On success, or if it was already running → continue to step 3.
- On any error → **fail fast**: stop and report the error to the user.

## 3. Determine what to load, then clear existing instructions

First decide whether any custom instructions are configured:

- If `instructionFolder` is not `none`, the folder exists on the host, and it is not empty → custom instructions are present; continue.
- Otherwise, check whether any of these are configured: `presetContextFile` not `none`, `globalConstantsFile` not `none`, `allowedDomains` present in `waterwheel.config.json`, or `enableLocalTest` present in `waterwheel.config.json`.
  - If at least one is configured → continue.
  - If none are configured → print a warning that no custom instructions were found and the agent is being reset to its default configuration, then continue.

Then clear the agent's existing instruction files (host testing is turned off — step 8 re-enables it when needed):

```shell
docker exec <containerName> reset-test-config -i
```

- On any error → **fail fast**: stop and report the error to the user.

## 4. Upload the instruction folder

Only if `instructionFolder` is not `none`, the folder exists, and it is not empty. For each **direct-child file** in the folder (subdirectories are ignored), upload it under its own basename. `upload-instruction-file` reads content from stdin:

```shell
cat "<instructionFolder>/<file>" | docker exec -i <containerName> upload-instruction-file <file>
```

Skip the following files so the dedicated config keys take precedence:

- Skip `preset-context.json` if `presetContextFile` is not `none` (it is uploaded in step 5).
- Skip `global-context.json` if `globalConstantsFile` is not `none` (it is uploaded in step 6).

Capture per-file success/failure but keep processing the remaining files.

## 5. Upload the preset context file

Only if `presetContextFile` is not `none`. Upload its content under the name `preset-context.json`:

```shell
cat "<presetContextFile>" | docker exec -i <containerName> upload-instruction-file preset-context.json
```

Capture any error and continue.

## 6. Upload the global constants file

Only if `globalConstantsFile` is not `none`. Upload its content under the name `global-context.json`:

```shell
cat "<globalConstantsFile>" | docker exec -i <containerName> upload-instruction-file global-context.json
```

Capture any error and continue.

## 7. Set the domain allowlist

Import every entry in `allowedDomains` into the container as a comma-delimited list. Quote entries containing shell-special characters:

```shell
docker exec <containerName> set-domain-permission "<domain1>,<domain2>,..."
```

If `allowedDomains` is empty, skip this step and note it. Capture any error and continue.

## 8. Enable local-host testing

Only if `enableLocalTest` is `true`. This rewrites `localhost` to `host.docker.internal` in the allowlist and global constants so the agent can reach a site on the host machine. It is idempotent:

```shell
docker exec <containerName> enable-test-on-host
```

Capture any error and continue.

## 9. Report the outcome

- If no errors occurred in steps 4–8 → report that instructions loaded successfully, with a short summary of what was applied (folder files uploaded, preset/global files, domain count, local-host testing on/off).
- If any step in 4–8 produced an error → report each error against the step/file it came from, and make clear the load did not fully succeed.

---
name: waterwheel-update-context
description: Save context variables produced by the latest Waterwheel run into the project's preset-context.json so future tests can reuse them. Use when the user says "update waterwheel context", "save context variables", "update preset context", "persist run context", or otherwise wants the latest run's generated values written back into preset context.
---

# Waterwheel — Update Context

> **Capabilities:** This skill runs Docker commands (`inspect`, `start`, `exec`), reads container-generated context values, and **writes one JSON file on the host** at the configured `presetContextFile` / `instructionFolder` path (creating it and its parent directory if needed). The path is fully user-controlled — it may point anywhere the agent can write — so the write location is only as constrained as your configuration. The values it persists originate inside the container and are treated as untrusted data; the user confirms which keys are written.

During a run, the Waterwheel agent's tests can create or update context values (for example, an AI-generated username and password for a freshly registered account). This skill exports those values from the container and merges the ones the user chooses into the project's `preset-context.json` `data` object, so they become seed values for future runs.

This skill **never** edits `waterwheel.config.json` — it only reads it and writes the resolved preset-context file.

## Configuration

This skill uses these defaults:

- Container name: `waterwheel-agent`
- Preset context file: `none` (optional)
- Instruction folder: `none` (optional)

Before running, look for a `waterwheel.config.json` file in the project root. If it exists, override the defaults with its values:

- Use `containerName` in place of `waterwheel-agent`
- Use `presetContextFile` in place of `none`
- Use `instructionFolder` in place of `none`

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

- On success → continue to step 3.
- On any error → **fail fast**: stop and report the error to the user.

## 3. Resolve the target preset-context file

Decide which file the exported values will be merged into, in this order:

1. If `presetContextFile` is not `none` → the target is `<presetContextFile>`.
2. Otherwise, if `instructionFolder` is not `none` → the target is `<instructionFolder>/preset-context.json`.
3. Otherwise → no target is configured: **stop** and tell the user to set `presetContextFile` in `waterwheel.config.json` (or configure `instructionFolder`) before running this skill.

Record the resolved path and whether the file currently exists on the host. If it does not exist, it will be **created** in step 7. Do not create it yet.

## 4. Export context variables

Export the latest run's context values from the container:

```shell
docker exec <containerName> output-context-variables
```

- Error indicating a run is in progress (`run-qa` is active) → **stop**: tell the user to wait for the current run to finish, then try again.
- Error that `test-context.json` is missing, or any other non-zero exit → **stop**: report that there are no context variables from a recent run to save.
- Prints an empty JSON object `{}` → **stop**: report there is nothing to merge.
- Prints a non-empty flat JSON object → parse it and continue. Top-level values may be scalars or nested objects; treat each top-level entry as one mergeable key.

## 5. Show the output and let the user choose

Print the exported JSON so the user can see exactly what is available. Then ask the user which keys to merge:

- **All** — merge every top-level key.
- **A subset** — the user names the specific keys to keep (pluck by key name).
- **Cancel** — make no changes and stop.

Carry forward only the chosen keys into step 6.

## 6. Merge into the target's `data` object

Build the updated file content:

- If the target file exists, read and parse it. If it does not exist, start from an empty object `{}`.
- Ensure a top-level `data` object exists (create it if absent).
- For each chosen key, set `data[<key>] = <value>` — **replacing** any existing value on a key conflict, and adding the key if it is new.
- Leave the `flow` array and every other top-level property untouched.

Track, for the report, which keys were **added** versus **replaced**.

## 7. Write the file

Write the updated content back to the resolved target path as formatted JSON (creating the file and any missing parent directory if it did not exist).

- On any write error → stop and report it to the user.

## 8. Report and prompt for reload

- Summarize the result: the target file path, which keys were added, and which were replaced.
- Then prompt the user to run the `waterwheel-load-instructions` skill so the updated `preset-context.json` is uploaded into the container and used by future runs. This skill does not trigger that reload automatically.

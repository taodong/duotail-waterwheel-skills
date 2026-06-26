---
name: waterwheel-stop-container
description: Stop the running Waterwheel test agent Docker container without removing it. Use when the user says "stop waterwheel container", "stop the waterwheel agent", "shut down waterwheel", or otherwise wants the Waterwheel container halted while keeping it installed.
---

# Waterwheel — Stop Container

## Configuration

This skill uses this default:

- Container name: `waterwheel-agent`

Before running, look for a `waterwheel.config.json` file in the project root. If it exists, override the default with its value:

- Use `containerName` in place of `waterwheel-agent`

If the file does not exist, or the key is missing, keep the default. Resolve the configuration once at the start and reuse it throughout.

## 1. Check whether the container exists and is running

Inspect the container:

```shell
docker inspect -f '{{.State.Running}}' <containerName>
```

- Prints `true` → the container exists and is running; continue to step 2.
- Prints `false` → the container exists but is already stopped; report that there is nothing to stop and stop — the skill is complete.
- Errors / "No such object" → the container does not exist; report that there is nothing to stop and stop — the skill is complete.

## 2. Stop the container

Stop the running container (this leaves it installed so it can be started again later):

```shell
docker stop <containerName>
```

- On success → report that the Waterwheel agent container `<containerName>` was stopped and stop.
- On error → report the error to the user and stop.

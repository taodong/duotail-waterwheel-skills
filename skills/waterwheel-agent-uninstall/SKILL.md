---
name: waterwheel-agent-uninstall
description: Uninstall the Waterwheel test agent by removing its Docker container. Use when the user says "uninstall waterwheel agent", "remove waterwheel container", "delete waterwheel", or otherwise wants the Waterwheel container torn down.
---

# Waterwheel — Agent Uninstall

## Configuration

This skill uses this default:

- Container name: `waterwheel-agent`

Before running, look for a `waterwheel.config.json` file in the project root. If it exists, override the default with its value:

- Use `containerName` in place of `waterwheel-agent`

If the file does not exist, or the key is missing, keep the default. Resolve the configuration once at the start and reuse it throughout.

## 1. Check whether the container exists

Inspect the container:

```shell
docker inspect -f '{{.State.Running}}' <containerName>
```

- Prints `true` or `false` → the container exists; continue to step 2.
- Errors / "No such object" → the container does not exist; report that there is nothing to uninstall and stop — the skill is complete.

## 2. Remove the container

Force-remove the container (this stops it first if it is running):

```shell
docker rm -f <containerName>
```

- On success → report that the Waterwheel agent container `<containerName>` was removed and stop.
- On error → report the error to the user and stop.

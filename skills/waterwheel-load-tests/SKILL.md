---
name: waterwheel-load-tests
description: Load the markdown test tasks from the project's task folder into the Waterwheel agent container. Use when the user says "load waterwheel tests", "reload waterwheel tests", "load web tests", "reload web tests", or otherwise asks to (re)load the web test tasks into the agent.
---

# Waterwheel — Load Tests

> **Capabilities:** This skill runs Docker commands (`inspect`, `start`, `exec`), reads markdown files from the project's task folder on the host, and pipes their contents into the container. It does **not** write to the host filesystem or read any secrets.

This skill resets the Waterwheel agent's test tasks and uploads every markdown test file from the project's task folder into the container. Each step fails fast: if a step cannot complete, stop and report the error to the user rather than continuing.

## Configuration

This skill uses these defaults:

- Container name: `waterwheel-agent`
- Task folder: `tasks`

Before running, look for a `waterwheel.config.json` file in the project root. If it exists, override the defaults with its values:

- Use `containerName` in place of `waterwheel-agent`
- Use `taskFolder` in place of `tasks`

If the file does not exist, or a key is missing, keep the default for that value. Resolve the configuration once at the start and reuse it throughout.

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

## 3. Verify the task folder has markdown files

On the host, confirm `<taskFolder>` exists and contains at least one markdown file as a **direct child** (subdirectories are ignored):

```shell
ls <taskFolder>/*.md
```

- One or more `.md` files are listed → record the full list and continue to step 4.
- The folder does not exist, or no `.md` file is found → **fail fast**: stop and tell the user that `<taskFolder>` is missing or contains no markdown test files.

## 4. Reset existing test tasks

Clear the agent's current test task files so the upload starts from a clean state:

```shell
docker exec <containerName> reset-test-config -t
```

- On any error → **fail fast**: stop and report the error to the user.

## 5. Upload each markdown file

For every `.md` file found in step 3, upload it into the container. `upload-test-task` reads the task content from stdin and keys it by the filename, so pipe the file in and pass its basename:

```shell
cat "<taskFolder>/<file>.md" | docker exec -i <containerName> upload-test-task <file>.md
```

Process the files one at a time. For each file, capture whether the upload succeeded or failed (including any error output), but do not stop the batch on a single failure — continue uploading the remaining files so the user sees the full picture.

## 6. Report the outcome

After processing every file, report a per-file status summary, for example:

- ✅ `<file>.md` — uploaded
- ❌ `<file>.md` — failed (include the error)

State how many files were uploaded successfully and how many failed. If every file failed, make clear the load did not succeed.

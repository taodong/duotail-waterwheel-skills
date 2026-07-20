---
name: waterwheel-load-test-skills
description: Load user-defined test skills from the project's test-skills folder into the Waterwheel agent container. Use when the user says "load waterwheel test skills", "reload waterwheel test skills", "load web test skills", "reload web test skills", or otherwise asks to (re)load the agent's test skills.
---

# Waterwheel — Load Test Skills

> **Capabilities:** This skill runs Docker commands (`inspect`, `start`, `exec`), reads `SKILL.md` files from the project's test-skills folder on the host, and pipes their contents into the container. It does **not** write to the host filesystem or read any secrets.

This skill uploads every user-defined test skill from the project's test-skills folder into the Waterwheel agent, one skill at a time, using the container's `load-test-skills` command. Each step fails fast: if a step cannot complete, stop and report the error to the user rather than continuing.

> **Image requirement:** The `load-test-skills` command ships in Waterwheel agent image `1.4.0` and newer. This skill does not pre-check the image version; if the command is missing it fails with an explicit message (see step 5).

## Configuration

This skill uses these defaults:

- Container name: `waterwheel-agent`
- Test skills folder: `none`

Before running, look for a `waterwheel.config.json` file in the project root. If it exists, override the defaults with its values:

- Use `containerName` in place of `waterwheel-agent`
- Use `testSkills` in place of `none`

`testSkills` is the folder on the host under which each test skill lives in its own subfolder (each containing a `SKILL.md`). The literal string `none` (the default) means no test-skills folder is configured. If the file does not exist, or a key is missing, keep the default for that value. Resolve the configuration once at the start and reuse it throughout.

## 1. Check whether test skills are configured

If `<testSkills>` is `none`, there is nothing to load. Report that no test-skills folder is configured (`testSkills` is unset) and **stop successfully** — this is a normal skip, not an error. Otherwise continue to step 2.

## 2. Verify the container exists

Confirm a container matching `<containerName>` exists:

```shell
docker inspect -f '{{.State.Running}}' <containerName>
```

- Prints `true` → the container exists and is running; skip to step 4.
- Prints `false` → the container exists but is stopped; continue to step 3.
- Errors / "No such object" / any other error → **fail fast**: stop and tell the user no container named `<containerName>` was found and that they must set it up first.

## 3. Start the container

If the container is stopped, start it:

```shell
docker start <containerName>
```

- On success → continue to step 4.
- On any error → **fail fast**: stop and report the error to the user.

## 4. Verify the test skills folder has skills

On the host, confirm `<testSkills>` exists and contains at least one subfolder with a `SKILL.md` file:

```shell
ls <testSkills>/*/SKILL.md
```

- One or more `SKILL.md` files are listed → record each skill's folder name (the directory containing the `SKILL.md`) and continue to step 5.
- The folder does not exist, or no `SKILL.md` is found → **fail fast**: stop and tell the user that `<testSkills>` is missing or contains no test-skill folders (each skill must be a subfolder holding a `SKILL.md`).

## 5. Upload each test skill

For every skill folder `<name>` found in step 4, upload it into the container. `load-test-skills` reads the skill content from stdin and writes it to `<name>/SKILL.md` inside the agent, so pipe the file in and pass its folder name. Use `--force` so an existing skill of the same name is overwritten (making reloads idempotent):

```shell
cat "<testSkills>/<name>/SKILL.md" | docker exec -i <containerName> load-test-skills -name <name> --force
```

Process the skills one at a time. Handle two failure kinds differently:

- **`load-test-skills` command not found** (error text such as `executable file not found`, `command not found`, or `no such file or directory` for the command itself) → **fail fast for the whole skill**: stop and tell the user that this requires Waterwheel agent image `1.4.0` or newer — the `load-test-skills` command was added in `1.4.0` — and they should set `imageName` to at least `taojdcn/duotail-waterwheel:1.4.0` and reinstall the container.
- **Any other per-skill error** → capture the error but do not stop the batch; continue uploading the remaining skills so the user sees the full picture.

## 6. Report the outcome

After processing every skill, report a per-skill status summary, for example:

- ✅ `<name>` — loaded
- ❌ `<name>` — failed (include the error)

State how many skills were loaded successfully and how many failed. If every skill failed, make clear the load did not succeed.

---
name: waterwheel-rerun-tests
description: Replay a chosen subset of Waterwheel tests from the last full run's checkpoints — all of them, a range between two test names, a single test, optionally with data overrides and repeated several times. Use when the user says "rerun tests", "rerun all tests", "rerun the login test", "rerun tests from X to Y", "replay test X 5 times", "rerun X with <data>", or otherwise wants part of the suite re-executed without a full run. After a successful rerun it offers to preserve the rerun's updated context into preset-context.json. Requires agent image 1.5.0+.
---

# Waterwheel — Rerun Tests

> **Capabilities:** This skill runs Docker commands (`inspect`, `start`, `exec`) to compose a rerun config, execute reruns, and read their results. It writes inside the container (`/agent/instructions/rerun-config.json`, and an optional `rerun-data.json`) and never edits application source code. It writes **one host file** only when the user explicitly chooses to preserve a successful rerun's context (step 8): the configured `presetContextFile` / `instructionFolder` preset-context path, created with its parent directory if needed. The persisted values originate inside the container and are treated as untrusted data; the user confirms which keys are written. Reruns execute your real tests against your real application, so they cause whatever side effects those tests cause — repeated once per iteration when a repeat count is given.

A rerun replays a subset of tasks instead of the whole suite, seeded from the context captured during the most recent full `run-qa` run. That is what makes a partial run possible: a test that depends on an account registered earlier in the suite can be replayed on its own.

This skill turns a natural-language request into a validated `rerun-config.json`, runs `rerun-tests`, and reports the outcome. It **diagnoses** failures but never fixes them — hand off to `waterwheel-test-and-fix` for that.

Each step fails fast: if a step cannot complete, stop and report the error to the user rather than continuing.

## Configuration

This skill uses these defaults:

- Container name: `waterwheel-agent`
- Test check interval: 30 seconds
- Preset context file: `none` (optional)
- Instruction folder: `none` (optional)

Before running, look for a `waterwheel.config.json` file in the project root. If it exists, override the defaults with its values:

- Use `containerName` in place of `waterwheel-agent`
- Use `testCheckInterval` in place of `30` (seconds)
- Use `presetContextFile` in place of `none`
- Use `instructionFolder` in place of `none`

If the file does not exist, or a key is missing, keep the default for that value. Resolve the configuration once at the start and reuse it throughout. Optional paths use the sentinel `none` to mean "not configured." The two path keys are used only by step 8.

This skill requires agent image `1.5.0` or newer — the `generate-rerun-config`, `rerun-tests`, and `get-test-report` commands do not exist in earlier images.

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

## 3. Read the test catalog

This one command establishes three things at once: that the image supports reruns, that nothing is currently running, and which tests can be replayed.

```shell
docker exec <containerName> get-test-report --list-tests
```

Evaluate the outcome:

- **"executable file not found" / "not found"** → the image is older than `1.5.0`. **Stop**: tell the user this skill needs agent image `1.5.0` or newer, and that `imageName` in `waterwheel.config.json` must be raised and the container reinstalled.
- **An "already running" / testing-in-progress message** (exit 1) → a `run-qa` or `rerun-tests` session is active. **Stop**: report it and name the stop command the message itself names — `stop-qa` for a full run, `stop-rerun` for a rerun. Both are run as `docker exec <containerName> <command>`.
- **"test report isn't available"** (exit 1) → no completed full run exists, so there are no checkpoints to seed from. **Stop**: tell the user to run a full suite first with the `waterwheel-test-and-fix` skill, then rerun.
- **A JSON array** → this is the catalog. Each entry has a `name` (the test's human-readable name) and a `file` (the task filename a rerun config needs). **The array order is the last full run's actual execution order** — every range and "all" selection below is resolved against that order, not alphabetically.

## 4. Resolve the request against the catalog

Translate what the user asked for into an ordered list of `file` values. Never invent a filename — every entry must come from the catalog.

**Selection:**

- **All tests** ("rerun all tests", "rerun everything") → every `file` in catalog order.
- **A range** ("from *Login* to *Checkout*") → find both endpoints, then take the inclusive slice of the catalog between them, in catalog order. If the second endpoint appears *before* the first in the catalog, still use the slice between them and tell the user the order was taken from the run, not from the phrasing.
- **A single test** ("rerun *Login*") → that one entry.
- **An explicit list** ("rerun *Login* and *Checkout*") → those entries, in the order the user named them.

**Matching** is case-insensitive against `name` first, then against `file`. Partial matches are acceptable only when they identify exactly one entry.

- A name that matches **nothing**, or matches **more than one** entry → **stop**: show the catalog (names and files) and ask the user which they meant. Do not guess. A task file added since the last full run is not in the catalog and **cannot be rerun at all** — the agent has no record of where it would have fallen in the run order. If that looks like the case, say so and tell the user a new full run is needed first.

**Data overrides:** collect any test data the user supplied as `KEY=value` pairs. Note which values are **not** plain strings (numbers, booleans, arrays, objects) — they are handled differently in step 6.

**Repetition:** note the repeat count `N` ("5 times", "three times"). Default to `1`.

## 5. Choose the rerun names, then confirm the plan

Every rerun this skill starts is **named**. An unnamed rerun is auto-numbered and read back by modification time, which silently reports an older folder when a rerun dies before writing results; a name makes every read unambiguous.

Build a base slug from the request: lower-case it, turn spaces into `-`, and strip everything outside `a-z`, `0-9`, `_`, and `-` (for example "Login flow" → `login-flow`, "all tests" → `all-tests`). Then:

- `N == 1` → the single name is `<slug>`.
- `N > 1` → the names are `<slug>-1`, `<slug>-2`, … `<slug>-N`, one per iteration.

**Check for collisions before running anything.** A named rerun whose output folder already exists aborts, so list the existing rerun names:

```shell
docker exec <containerName> get-test-report | jq -r '.reruns[]?.name // empty'
```

Compare each planned name against that list after normalizing both the same way the agent does (lower-case, whitespace runs → `_`, every other character stripped). On any clash, append `-r2` (then `-r3`, …) to the base slug and re-derive the names. Report the names actually used.

**Echo the plan** — the ordered task files, the rerun name(s), the data overrides, and the iteration count. When the selection covers more than one task, also state this: a rerun restores context from a **single** checkpoint — the selected task that ran earliest in the original run — and each later task continues from the previous one's state. Dependency checking is bypassed for a rerun.

**Ask the user to confirm before proceeding** when any of these is true:

- the selection is "all tests",
- `N > 1`,
- data overrides were supplied, or
- any test name was matched fuzzily rather than exactly.

A single, exactly-matched test with no overrides needs no confirmation — run it.

## 6. Run each iteration

Repeat this whole block once per iteration `i` from `1` to `N`, using `<name-i>` from step 5.

### 6a. Upload non-string data (only if needed)

`--data` values are always strings. If step 4 found numbers, booleans, arrays, or nested objects, write them as a single JSON object and upload it **once**, before the first iteration:

```shell
printf '<json>' | docker exec -i <containerName> upload-instruction-file rerun-data.json
```

Then pass `--data-file /agent/instructions/rerun-data.json` in every iteration below. The path is resolved **inside the container**, so an uploaded file is the only way to supply non-string values. Skip this entirely when all overrides are strings.

### 6b. Generate and install the config

```shell
docker exec <containerName> generate-rerun-config -f <file1> -f <file2> --name "<name-i>" \
  --data "<key>=<value>" | \
  docker exec -i <containerName> upload-instruction-file rerun-config.json
```

Pass one `-f` per selected file, in the execution order from step 4. Add `--data` only when string overrides exist, and `--data-file /agent/instructions/rerun-data.json` only when step 6a ran.

**Iteration placeholders:** if a data value contains `{n}`, replace it with the 1-based iteration number before building the command (`qa+{n}@example.com` → `qa+3@example.com` on iteration 3). This matters when repeating a test that creates something unique — the same email five times fails on a unique constraint. When no placeholder is present, every iteration gets identical data; say so when echoing the plan in step 5.

The generator writes JSON to stdout and every diagnostic to stderr. Evaluate both:

- **Non-zero exit from the generator** → **stop**: report its stderr verbatim. Nothing was installed; `upload-instruction-file` rejects empty stdin, so any existing config is untouched.
- **A `⚠️` "Rerun output folder already exists" warning on stderr** → the collision check in step 5 missed a folder, and `rerun-tests` **will** abort. **Stop**, pick a different name, and tell the user.
- **A `⚠️` duplicate-task warning** → should not occur; this skill never repeats a file within one flow. If it appears, stop and re-derive the selection.
- **An upload error** → **stop** and report it.

### 6c. Start the rerun

```shell
docker exec -d <containerName> rerun-tests
```

### 6d. Poll for the result

```shell
docker exec <containerName> check-test-result --rerun "<name-i>"
```

- **Testing-in-progress message** → wait `testCheckInterval` seconds and poll again. Repeat until a final result is returned.
- **`No test results found`** → this iteration died before writing results. The command also prints the rerun's `agent.log` and the top-level pre-rerun startup log; capture both, record the iteration as **errored**, and continue to step 6e.
- **Any other output** → it is the `exit_condition` for this iteration. Record it:
  - `"All tests passed"` → iteration passed.
  - `"One or more tests failed"` → iteration failed; continue to step 6e.
  - Any other value → iteration did not complete; record the exit condition as the reason and continue to step 6e.

### 6e. Diagnose a failed or errored iteration

```shell
docker exec <containerName> get-failure-detail -d --rerun "<name-i>"
```

Use the returned details, together with the `agent.log` output captured in step 6d, to identify the root cause. Report the diagnosis — **do not edit application code**; this skill never fixes.

### 6f. Continue

Move to the next iteration **regardless of this one's outcome**. Repeating a test exists to measure flakiness, and stopping at the first failure discards that signal. (The user can interrupt at any point.)

## 7. Report

Summarize the whole run:

- **Per iteration**: the rerun name, the outcome, and its output folder `/agent/outputs/rerun-<normalized-name>/`. For example:
  - ✅ `login-flow-1` — all tests passed
  - ❌ `login-flow-2` — one or more tests failed
  - ⚠️ `login-flow-3` — errored before producing results
- **When `N > 1`**: a pass/fail tally. If the results are **mixed**, call that out explicitly as a flakiness signal — the same tests, the same seed checkpoint, different outcomes.
- **For every failure**: the root-cause diagnosis from step 6e.
- **How to re-read any of it**: `docker exec <containerName> check-test-result --rerun "<name>"`, and `docker exec <containerName> get-test-report` for the full run plus every rerun as one JSON document.

Close with the relevant next step:

- To fix the failures and verify against the full suite → the `waterwheel-test-and-fix` skill.

Do **not** point at `waterwheel-update-context` — it is for full runs only. Preserving a rerun's context is step 8 of this skill.

## 8. Offer to preserve a successful rerun's context

A rerun starts from the last full run's stable context, but a rerun that succeeds where the full run failed may have changed that context (a newly registered account, a regenerated ID). This step lets the user keep those values for future full runs.

**Skip this step entirely** when no iteration ended with `"All tests passed"` — a failed or errored rerun's context is not persisted.

### 8a. Ask

Ask the user whether to preserve the context of a successful rerun. When more than one iteration passed, list the passing rerun names and let the user pick **one**. If the user declines, stop here — nothing is written.

### 8b. Resolve the target preset-context file

1. If `presetContextFile` is not `none` → the target is `<presetContextFile>`.
2. Otherwise, if `instructionFolder` is not `none` → the target is `<instructionFolder>/preset-context.json`.
3. Otherwise → tell the user to set `presetContextFile` in `waterwheel.config.json` (or configure `instructionFolder`) to preserve rerun context, and stop this step. The rerun report stands.

Record whether the target file currently exists on the host. Do not create it yet.

### 8c. Export the rerun's context

```shell
docker exec <containerName> output-context-variables --rerun "<chosen-name>"
```

- Non-zero exit (no `test-context.json`, or testing in progress) → report that there is nothing to preserve and stop this step.
- `{}` → report that there is nothing to preserve and stop this step.
- A non-empty flat JSON object → continue. Treat each top-level entry as one mergeable key.

This export is the rerun's **complete** context — the restored checkpoint plus everything the rerun changed — not just the changes.

### 8d. Mark what the rerun changed

```shell
docker exec <containerName> output-context-variables
```

This is the full run's context. Compare it key by key with the rerun's export and mark each rerun key as **new** (absent from the full run), **changed** (different value), or unchanged. If this command fails, continue without markers and say so.

### 8e. Let the user choose

Print the rerun's values with their markers. Then ask which keys to preserve:

- **Changed only** — every key marked new or changed.
- **All** — every top-level key.
- **A subset** — the user names the keys.
- **Skip** — make no changes and stop.

Nothing is written without an explicit choice.

### 8f. Merge and write

- If the target file exists, read and parse it. Otherwise start from `{}`.
- Ensure a top-level `data` object exists.
- For each chosen key, set `data[<key>] = <value>` — replacing an existing value on conflict, adding it otherwise.
- Leave `flow` and every other top-level property untouched.
- Write the result back as formatted JSON, creating the file and any missing parent directory. On a write error, stop and report it.

### 8g. Report and prompt for reload

- Summarize: the target path, which keys were added, and which were replaced.
- Prompt the user to run the `waterwheel-load-instructions` skill so the updated `preset-context.json` reaches the container. This skill does not reload it automatically.
- State that the preserved values take effect at the **next full run** — reruns never read `preset-context.json`, so further reruns still start from the current checkpoints.

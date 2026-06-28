---
name: waterwheel-run
description: Set up and run the full Waterwheel pipeline end to end — install the agent, load tests, load instructions, then run the test-and-fix loop. Use when the user says "set up and run waterwheel", "run the full waterwheel pipeline", "install and test with waterwheel", "set up waterwheel from scratch", or wants the complete flow in one go. (For just re-running the tests on an already-set-up agent, use waterwheel-test-and-fix instead.)
---

# Waterwheel — Run (Full Pipeline)

> **Capabilities:** This skill only orchestrates the four sub-skills below; it performs no Docker, filesystem, or secret operations of its own. Its effective capabilities are the union of those skills' — most notably installing/running a remote Docker image and reading the AI API key (`waterwheel-agent-install`) and, where permitted, editing application source code (`waterwheel-test-and-fix`). See each sub-skill's capability note.

This skill is a thin orchestrator. It runs the four Waterwheel skills below **in sequence**, each as its own step. It does not duplicate their logic — invoke each skill and let it resolve its own configuration and perform its own work.

The pipeline order is:

1. `waterwheel-agent-install`
2. `waterwheel-load-tests`
3. `waterwheel-load-instructions`
4. `waterwheel-test-and-fix`

## Sequencing rules

- Run the steps strictly in the order above. Each step depends on the previous one having succeeded.
- **Fail fast.** If any step stops with an error (for example, Docker is unavailable, the container cannot be installed, or a load step fails), stop the whole pipeline immediately. Report which step failed and why, and do not run any later steps.
- If a step stops normally to hand control back to the user (for example, `waterwheel-test-and-fix` reaches `maxFixAttempts` or needs a manual redeploy), do not start a new step. Report where the pipeline paused and what the user should do next.
- Carry the resolved configuration expectation forward: every sub-skill reads the same `waterwheel.config.json`, so the user only needs one config file for the whole run.

## 1. Install the agent

Invoke the `waterwheel-agent-install` skill. This verifies Docker, reuses or starts an existing container, optionally rebuilds on AI-config drift, and configures the AI provider.

- If it stops with an error → stop the pipeline and report the failure.
- Once the container is installed and running → continue to step 2.

## 2. Load tests

Invoke the `waterwheel-load-tests` skill to (re)load the markdown test tasks from the project's task folder into the container.

- If it stops with an error → stop the pipeline and report the failure.
- Otherwise → continue to step 3.

## 3. Load instructions

Invoke the `waterwheel-load-instructions` skill to (re)load the instruction and configuration files into the container.

- If it stops with an error → stop the pipeline and report the failure.
- Otherwise → continue to step 4.

## 4. Test and fix

Invoke the `waterwheel-test-and-fix` skill to run the full test suite and, where permitted, diagnose and fix failures.

- Let this skill run to its own completion or stopping point.
- When it finishes, report the final pipeline outcome: which steps completed, the test result, and any follow-up the user needs to take.

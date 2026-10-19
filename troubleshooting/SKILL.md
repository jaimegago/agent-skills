---
name: troubleshooting
description: Use when the user is debugging or diagnosing a problem that requires running shell commands to investigate. Triggers on any multi-step diagnostic where the next command depends on the output of the previous one — broken builds, failing tests, K8s/cluster issues, network problems, misbehaving services, unexplained errors. Do not use for one-shot lookups or when the fix is already known.
metadata:
  version: "1.0.0"
---

# Troubleshooting: one command at a time

Diagnostics is a feedback loop, not a script. Batching commands hides which one revealed the signal and burns the user's attention on output they did not need.

## Rules

1. Propose exactly ONE command per turn. Never two. Never chained with && or ;. Never a for-loop that hides multiple invocations.
2. Before the command, state in one short sentence what hypothesis it tests or what fact it establishes.
3. Present the command as a plain line — no code fence, no backticks, no $ prompt, no inline comment, no surrounding decoration. The user must be able to triple-click and paste.
4. Stop. Wait for the user to paste output. Do not speculate about what the output might be. Do not pre-write the next three steps.
5. When output arrives, interpret it explicitly (what it confirms, what it rules out), then propose the next single command using the same format.
6. If you need information that is not a shell command (a file's contents, a screenshot, a URL), ask for that one thing and stop.
7. This skill governs commands you propose to the human. It does not apply when you execute commands yourself via Bash tool — that is a separate mode.

## Format per turn

Hypothesis: <one sentence>
Command:
<the bare command on its own line>

Nothing after the command. No "then we'll...", no "this should show...", no preview of step 2.

## When to exit the loop

Stop running the loop and switch to normal explanation when: the root cause is identified, the user says stop, or three consecutive commands have produced no new information (in which case, name the dead end and ask the user how they want to proceed).

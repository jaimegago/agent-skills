---
name: human-docs-authoring
description: Create, edit, and audit human-facing docs (README.md, /docs, contributor guides). Use when asked to write, update, or review docs meant for humans. Handles both authoring and drift-hunting in one invocation.
---

# Human Docs Authoring

## Purpose

Write and maintain documentation for humans: README.md, /docs content, contributor guides, setup instructions. This is the counterpart to llm-docs-authoring — that skill handles AGENTS.md, this one handles everything a human would read.

## When to use

Invoke this skill when the developer asks any of:
- "Write a README for this repo"
- "Update the docs"
- "The README is stale, fix it"
- "Add a setup section to the docs"
- "Audit /docs for drift"

Unlike llm-docs, this skill handles both authoring and drift-hunting. Prose has natural termination (it either reads right or it doesn't), so the infinite-loop risk that forced llm-docs to split does not apply here.

## What counts as human docs

- `README.md` at the repo root
- Anything in `/docs` that is meant for humans (contributor guides, architecture overviews, setup instructions, tutorials)
- `CONTRIBUTING.md`, `INSTALL.md`, similar top-level files

What does NOT count:
- `AGENTS.md` — that is llm-docs-authoring's job
- Spec documents that ARE the product (e.g., oasis-spec's spec files) — those are source, not docs about source
- API reference generated from code — that is the build system's job

## README.md philosophy

A good README answers three questions, in this order, and nothing else:

1. **What is this?** One paragraph. What the project does, who it is for, what problem it solves.
2. **How do I use it?** Install, run, basic example. The minimum a new user needs to get something working.
3. **Where do I go next?** Links to /docs for depth, to issues for help, to contributing guide for participation.

If the README has grown beyond these three questions, propose moving the excess to /docs. A README is a landing page, not a manual.

## /docs philosophy

/docs is for the content that did not fit in the README. Organize by audience and task:

- **Getting started** — expanded tutorial, more than the README's minimum
- **Guides** — how to do specific things
- **Reference** — detailed parameter/config documentation
- **Contributing** — how to build, test, and submit changes
- **Architecture / design** — why the system is shaped the way it is (for humans who want depth)

Not every repo needs all of these. Small repos might just need README plus one or two docs. Do not create empty scaffolding.

## Authoring mode

When the ask is "write" or "update":

1. Read the current README and scan /docs.
2. Identify what the developer specifically asked for.
3. Write or edit in plain, direct prose. See "Writing style" below.
4. Show the diff.
5. Stop.

## Audit mode

When the ask is "review," "audit," or "check for drift":

1. Read README.md and walk /docs.
2. Compare against the actual repo:
   - Do documented commands still work? (Check against Makefile, package.json scripts, visible entry points.)
   - Do documented file paths exist?
   - Do internal doc links resolve?
   - Does README refer to features that no longer exist, or miss features that clearly do?
   - Has README grown beyond the three-question scope? If so, flag for splitting.
3. Produce a findings list with proposed edits.
4. Ask the developer which findings to apply.
5. Apply approved findings.
6. Stop.

Unlike llm-docs-audit, this skill CAN edit — but only after the developer approves the findings. Do not silently edit during the audit pass.

## Writing style

- **Direct.** "Run `oasisctl evaluate`" not "You can run the evaluate command by typing..."
- **Second person.** "You" not "the user" or "one."
- **Active voice.** "The CLI reads the config from `./oasis.yaml`" not "The config is read from..."
- **Copy-pasteable code examples.** If it would not work when pasted, it does not belong in the docs.
- **No motivation in user docs.** Why the architecture is shaped a certain way belongs in an architecture doc under /docs, not in README or getting-started.
- **No corporate hedging.** "This may work" is worse than "This works, with these caveats: ..."
- **Short sentences.** If a sentence has two ideas, it is probably two sentences.

## What NOT to do

- Do NOT duplicate content between README and /docs. Pick one home for each piece of information, link from the other.
- Do NOT duplicate content from AGENTS.md. If something is in AGENTS.md because it is an invariant the agent needs to know, it does not automatically need to be in human docs — only include it if a human would benefit.
- Do NOT write docs for features that do not exist yet. Docs describe what IS, not what is planned.
- Do NOT add "Why we built this" sections unless the developer specifically asks. It ages badly.
- Do NOT invent examples. If you need an example, use one from the actual codebase or ask.

## Spec repos are a special case

Some repos contain specifications where the spec documents ARE the product (not docs about the product). In spec repos:

- The spec files themselves are source, not docs. Do not edit them as part of this skill's work.
- README and /docs serve contributor guidance: how to propose changes, how the spec is versioned, how to run validation tooling.
- If unsure whether a repo is a spec repo, ask the developer before making assumptions.

## Completion

When the ask is complete:
1. Show the developer the diff (or the full file for creation).
2. If anything outside the scope might need attention, list it under "Noted but not addressed."
3. Stop.

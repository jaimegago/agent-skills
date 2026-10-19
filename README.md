# agent-skills

The maintainer's [Agent Skills](https://agentskills.io) for developing the
[joe](https://github.com/jaimegago/joe) workstream: engineering standards,
commit discipline, documentation authoring, and debugging method. Each skill
is a directory holding a `SKILL.md` with YAML frontmatter (`name`,
`description`) and a markdown body an agent reads when the task matches; a
few carry `references/` beside it.

Nothing here assumes a particular coding agent or a particular path. A skill
is a file an agent is told to read, and the repository that tells it is the
one that vendors this one.

## The contract: vendored, pinned, cited by relative path

A consuming repository carries this repository as a git submodule at
`agent-skills/`, at its root, pinned to a tag:

```bash
git submodule add https://github.com/jaimegago/agent-skills agent-skills
git -C agent-skills checkout v1.0.0
git add .gitmodules agent-skills
```

Its instruction file (`AGENTS.md`) cites each skill it applies by that
relative path, `agent-skills/<name>/SKILL.md`, together with the trigger: the
situation in which the agent should read it. Nothing auto-discovers a skill
from this path; the instruction file is the discovery.

**A pin moves only when the consuming repository moves it.** Adopting a new
tag is a commit there that advances the submodule pointer. Nothing bumps on
its own, so a repository nobody bumps runs old skills, and that is the trade
a pin makes.

**A skill is edited here and only here.** A change inside a consumer's
`agent-skills/` directory is drift, not a source.

## Versions

Tags on this repository, `vMAJOR.MINOR.PATCH`. A submodule pins a commit;
the tag is its name. Skills carry no version of their own.

- `v1.0.0` — the ten skills as they stood when the repository was created,
  2026-10-19.

## Format

A skill is portable: any Agent Skills consumer can load a directory from
here. Keep a skill short, state the why, and split it when it grows. A skill
frames how to work; it does not encode decisions a reader should be making.
Skill bodies name tools and conventions, never machine paths or credentials.

## License

MIT — see [LICENSE](LICENSE).

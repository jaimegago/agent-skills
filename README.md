# agent-skills

The maintainer's [Agent Skills](https://agentskills.io) for developing the
[joe](https://github.com/jaimegago/joe) workstream with Claude Code: engineering
standards, commit discipline, documentation authoring, and debugging method.
Each skill is a directory holding a `SKILL.md` with YAML frontmatter (`name`,
`description`) and a markdown body the agent loads when the description matches
the task; a few carry `references/` beside it.

## The checkout contract

`~/.claude/skills` is a checkout of this repository, on `main`. A skill is
edited here and only here: a local change in the checkout that is not a commit
on `main` is drift, not a source. Any machine that runs these skills clones this
repository to that path.

```bash
git clone https://github.com/jaimegago/agent-skills ~/.claude/skills
```

Keep it current the same way any other checkout is kept current:

```bash
git -C ~/.claude/skills pull --ff-only
```

## Who cites these

- [joe](https://github.com/jaimegago/joe)'s `CLAUDE.md` names `dev-standards`,
  `go-backend` and `frontend-dev` as the standards it is developed under, and
  gives a skill precedence over itself within the skill's scope.
- The joe workstream's ledger wraps `git-commit` and routes every commit through
  it.

Nothing here is specific to a machine. Skill bodies name tools and conventions,
never paths or credentials.

## Format

A skill is portable: any Agent Skills consumer can load a directory from here.
Keep a skill short, state the why, and split it when it grows. A skill frames
how to work; it does not encode decisions a reader should be making.

## License

MIT — see [LICENSE](LICENSE).

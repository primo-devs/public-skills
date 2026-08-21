# Primo skills

Agent skills we use at [Primo](https://www.primo.la) and publish so others can reuse them. Each skill follows the [Agent Skills](https://agentskills.io) layout: `skills/<name>/SKILL.md` plus optional `references/`.

## Install

```bash
npx skills add primo-devs/public-skills --skill testing
```

Or copy `skills/<name>/` into your agent's skills directory (`.claude/skills/`, `.agents/skills/`, ...).

## Skills

| Skill | What it does |
| --- | --- |
| [`testing`](skills/testing/) | Workflow for deciding *what* to test plus the guidelines for what a good test looks like. |

## License

[MIT](LICENSE)

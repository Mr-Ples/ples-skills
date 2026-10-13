# Ples Skills

Sharable agent skills. Install with the [Skills CLI](https://skills.sh/):

```bash
# Whole collection
npx skills add Mr-Ples/ples-skills -g

# One skill
npx skills add Mr-Ples/ples-skills --skill start-new-project -g -y
```

## Updating

If you installed via `npx skills add`:

```bash
# Update just this collection's skills
npx skills update -g -y

# Or update one skill by name
npx skills update start-new-project -g -y
```

This pulls the latest `SKILL.md` from GitHub into your agent directories.
Check what's installed with `npx skills list -g`.

## Skills

| Skill | What it does |
| ----- | ------------ |
| `start-new-project` | Scaffold a project in `~/Projects`, optionally set up git + GitHub via `gh`, open it in a Herdr tab with a coding agent, and run `setup-matt-pocock-skills` there. Requires Herdr + `gh`. |

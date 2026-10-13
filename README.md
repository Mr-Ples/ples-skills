# Simon Skills

Sharable agent skills. Install with the [Skills CLI](https://skills.sh/):

```bash
# Whole collection
npx skills add Mr-Ples/simon-skills -g

# One skill
npx skills add Mr-Ples/simon-skills --skill new-project -g -y
```

## Skills

| Skill | What it does |
| ----- | ------------ |
| `new-project` | Scaffold a project in `~/Projects`, optionally set up git + GitHub via `gh`, open it in a Herdr tab with a coding agent, and run `setup-matt-pocock-skills` there. Requires Herdr + `gh`. |

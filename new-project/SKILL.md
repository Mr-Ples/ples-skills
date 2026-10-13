---
name: new-project
description: Scaffold a new project in ~/Projects, optionally set up git + GitHub via gh, open it in a herdr tab with a coding agent, and run setup-matt-pocock-skills there.
disable-model-invocation: true
---

# New Project

Scaffold a new project from scratch: ask three questions, create the
folder, optionally wire up git/GitHub, open it in Herdr with a coding
agent, and configure it for the engineering skills.

## Prerequisites

- Running inside Herdr: verify with `test "${HERDR_ENV:-}" = 1`.
  If the check fails, stop and tell the user this skill requires Herdr.
- `gh` CLI for anything git/GitHub related.
- `setup-matt-pocock-skills` should be installed for step 4. If it is
  not available, tell the new agent to install it first:
  `npx skills add mattpocock/skills --skill setup-matt-pocock-skills -g -y`

## Step 1: Ask three questions

Ask one question at a time. Lead each with the recommended answer so
the user can accept it in a word.

### 1a. Project name (required, no default)

> What should the project be called?

Sanitize the answer into a safe folder name: lowercase, spaces become
hyphens, strip anything outside `[a-z0-9-_]`. Reject empty input,
`/`, and `..`. Confirm the sanitized name before proceeding.

Default location is `~/Projects/<name>` — mention it and let the user
override with a different parent directory if they want.

### 1b. Git + GitHub? (recommended: yes)

> Set up git and push to GitHub? (recommended: **yes**)

- **yes**: `git init`, first commit, `gh repo create` + push (step 2).
- **no**: plain folder, skip all git/gh work.

Before any `gh` command, check auth:

```bash
gh auth status
```

If auth fails, run `gh auth login` interactively and wait for the user
to complete it before continuing.

### 1c. Agent? (recommended: kilo)

> Which coding agent should open the project? (recommended: **kilo**)

Accept any kind listed by `herdr agent start --help`
(`kilo`, `claude`, `codex`, `opencode`, `pi`, ...).
Default to `kilo` when the user gives no preference.

## Step 2: Create the folder (and git)

```bash
mkdir -p ~/Projects/<name>
```

If the folder already exists and is non-empty, stop and ask: use the
existing folder, pick another name, or cancel. Never overwrite.

When git was requested:

```bash
cd ~/Projects/<name>
git init
git add -A
git commit -m "Initial commit"
gh repo create <name> --public --source=. --push
```

Confirm public vs private with the user before `gh repo create`
(default: **public**, since this feeds sharable skills — but ask).
If `gh auth status` fails at this point, run `gh auth login` and wait.

## Step 3: Open a Herdr tab with the agent

Parse IDs from JSON responses — never guess them.

```bash
herdr tab create --cwd ~/Projects/<name> --label <name> --no-focus
```

Read `.result.tab.tab_id` and `.result.root_pane.pane_id` from the
output. Then start the agent (derive `<agent-name>` from the project
name, lowercased, matching `[a-z][a-z0-9_-]{0,31}`):

```bash
herdr agent start <agent-name> --kind <kind> --pane <root-pane-id>
```

If the derived name is already live (`herdr agent list`), append
`-2`, `-3`, etc. Wait for idle readiness (default 30s timeout) before
continuing. Use `--focus` instead of `--no-focus` only if the user
asked to jump straight into the new project.

## Step 4: Run setup-matt-pocock-skills in the new agent

```bash
herdr agent prompt <agent-name> "Run the setup-matt-pocock-skills skill to configure this repo (issue tracker, triage labels, domain docs)." --wait --timeout 120000
```

If the prompt returns `blocked`, inspect with `herdr agent read`
and ask the user before answering. On success, report back:

- Project path: `~/Projects/<name>`
- Tab ID and agent name
- GitHub URL (if created)

---
name: start-new-project
description: Scaffold a new project in ~/Projects, optionally set up git + GitHub via gh, open it in a herdr tab with a coding agent, and run setup-matt-pocock-skills there.
disable-model-invocation: true
---

# Start New Project

Scaffold a new project from scratch: ask three questions, create the
folder, optionally wire up git/GitHub, open it in Herdr with a coding
agent, and configure it for the engineering skills.

## Prerequisites

- Running inside Herdr: verify with `test "${HERDR_ENV:-}" = 1`.
  If the check fails, stop and tell the user this skill requires Herdr.
- `gh` CLI installed. **Rule: for every git/GitHub operation, prefer
  the `gh` command whenever one exists** (`gh repo create`, `gh repo
  clone`, `gh pr *`, `gh issue *`). Only fall back to bare `git` for
  local-only operations `gh` cannot do (`git init`, `git add`,
  `git commit`) — never hand-roll what `gh` already covers (no manual
  `git remote add` + `git push` when `gh repo create --source=. --push`
  does it in one step).
- `setup-matt-pocock-skills` should be installed for step 4. If it is
  not available, tell the new agent to install it first:
  `npx skills add mattpocock/skills --skill setup-matt-pocock-skills -g -y`

## Step 1: Ask the questions

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

- **yes**: `git init` locally (no `gh` equivalent exists for this),
  first commit, then `gh repo create` + `--push` (step 2). Use `gh` for
  the remote half — never `git remote add` + `git push` by hand.
  Follow up with question 1c (visibility) before proceeding.
- **no**: plain folder, skip all git/gh work and the visibility question.

Before any `gh` command, check auth:

```bash
gh auth status
```

If auth fails, run `gh auth login` interactively and wait for the user
to complete it before continuing.

### 1c. Repo visibility? (only when git was requested; recommended: private)

> Should the GitHub repo be public or private? (recommended: **private**)

- **private**: `gh repo create <name> --private --source=. --push`
- **public**: `gh repo create <name> --public --source=. --push`

Record the choice and use the matching `gh repo create --<flag>` in
step 2. Never assume — always ask when git is requested.

### 1d. Agent? (recommended: kilo)

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

When git was requested, using the visibility chosen in question 1c.
**Use `gh` for the remote half** — `gh repo create --source=. --push`
wires up the remote and pushes in one step; do not run `git remote add`
or `git push` manually. Bare `git` is only for the local-only part
(`init`/`add`/`commit`), which `gh` cannot do:

```bash
cd ~/Projects/<name>
git init
git add -A
git commit -m "Initial commit"
gh repo create <name> --private --source=. --push
# or: gh repo create <name> --public --source=. --push
```

If `gh auth status` fails at this point, run `gh auth login` and wait.

## Step 3: Open a Herdr tab with the agent

Parse IDs from JSON responses — never guess them.

**First: check the Herdr integration for the chosen agent kind.**
Without it, `herdr agent start` never detects readiness and loops:

```bash
herdr integration status
```

If the output shows `<kind>: not installed` (e.g. `codex: not
installed`), install it before continuing:

```bash
herdr integration install <kind>
# e.g. herdr integration install kilo
```

Only proceed to `agent start` once status shows the kind as installed
(`current` or a version). If the install fails, stop and tell the user
to install the agent itself first, then re-run this skill.

**Fallback: manual agent recognition.** Some kinds (notably `cline`)
are accepted by `herdr agent start --kind` but have no Herdr
integration entry, so auto-detection never fires and `agent start`
loops. If `agent start` fails to detect readiness, or the chosen kind
is one you know has no integration, register the pane manually
instead of retrying:

```bash
herdr pane report-agent <pane-id> --source manual --agent <kind> --state idle
# e.g. herdr pane report-agent w1:p7 --source manual --agent cline --state idle
```

Then verify it appears:

```bash
herdr agent list
```

After manual registration, keep its lifecycle accurate by re-reporting
state when it changes (`--state working` while it works, `--state idle`
when done, `--state blocked` when it needs input). Do not use this
fallback when an integration exists — prefer `herdr agent start` and
real detection.

Then create the tab:

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

Send the prompt **without** `--wait` — delivery and completion are
checked separately below. (`--wait` tracks lifecycle state, not the
turn you just sent: if the agent was already working, an unrelated
turn's completion can satisfy it, or the wait can fail even though the
prompt landed fine. Fire-and-forget + verify is the reliable pattern.)

```bash
herdr agent prompt <agent-name> "Run the setup-matt-pocock-skills skill to configure this repo (issue tracker, triage labels, domain docs)."
```

**Verify delivery, then stop.** If the `prompt` command itself returns
success (exit 0, no `agent_blocked` error), the text was handed to the
pane — that is the success criterion for this skill. Optionally confirm
the agent picked it up:

```bash
herdr agent get <agent-name>
herdr agent read <agent-name> --source recent-unwrapped --lines 40
```

- State `working`, or the prompt text visible in the output → delivered.
  **Stop here and report success.** Do not wait for the new agent to
  finish its setup work — that runs on its own in its tab.
- `agent_blocked` on send → inspect with `herdr agent read` and ask
  the user before answering.
- Any other send error → report the error text, do not retry in a loop.

On success, report back and tell the user they can close this agent:

- Project path: `~/Projects/<name>`
- Tab ID and agent name
- GitHub URL (if created — get it from the `gh repo create` output,
  not by guessing the URL)
- Say: **"Success — the new agent has the setup prompt and is working
  in its own tab. You can now close this agent."**

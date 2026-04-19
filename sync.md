---
name: thedon-sync
description: Set up or sync a Thedon-style global skills ledger for the current coding agent only. Use when the agent is asked to install local Thedon sync support from this file, create the required config, compare its own global skills against the ledger, copy missing ledger skills into the current agent's global skills root, copy agent-only skills into the ledger cache, report one-agent drift, or prepare new skills for an optional GitHub push after user confirmation.
---

# Thedon Sync

## Overview

Use this skill for two jobs:

1. Setup for first-time users who point the agent at this file directly, usually with a prompt such as `Follow instruction at this file: <GitHub link to sync.md>`.
2. Sync for repeat users who already installed the local global skill and invoke `$thedon-sync`.

During setup, treat the linked `sync.md` as the source of truth and install a local global skill named `thedon-sync` by copying the full contents of this file into the current agent's global skills root.

During sync, compare one coding agent's global skills against the Thedon ledger and reconcile the difference in the direction implied by the comparison.

Favor inspection first, then clone or pull, then sync only the current agent's global skills root. Do not update Claude or Cursor roots when this skill is being used by Codex.
Treat system or internal skills as out of scope for the ledger. Exclude paths under `.system/` and do not sync them into or out of the ledger unless the user explicitly asks for that.

## Modes

Choose the mode explicitly before making changes:

- Use Setup mode when the user points the agent at this file directly, especially through a GitHub URL.
- Use Setup mode when the local global skill `thedon-sync` is not yet installed.
- Use Setup mode when config is missing and the user is clearly trying to bootstrap this workflow.
- Use Sync mode when the user invokes `$thedon-sync` after setup.
- If `$thedon-sync` is invoked but config is missing or invalid, fall back into the setup subflow needed to repair config, then continue only if the user still wants sync work in the same session.

## Setup Workflow

Setup mode exists to bootstrap local use of `$thedon-sync`.

Follow this exact order:

1. Read `~/.thedon/config.yaml` first when it exists.
2. Identify the current coding agent and its single relevant global skills root.
3. If `~/.thedon/config.yaml` does not exist, ask the user to provide the URL for an existing GitHub repo to serve as the ledger, or to create an empty repo first if they do not already have one.
4. If `~/.thedon/config.yaml` does not exist, create it from the template in this skill after the user provides the ledger repo URL.
5. Install the local global skill `thedon-sync` into the current agent's global skills root by copying the exact contents of this file into `<current-agent-global-skills-root>/thedon-sync/SKILL.md`.
6. Report the installed path clearly and tell the user that future sync runs should use `$thedon-sync`.
7. Stop setup here unless the user explicitly asks to continue with ledger inspection or sync work in the same session.

Setup mode does not automatically clone, pull, compare, sync, stage, or push ledger content unless the user explicitly asks for that after setup is complete.

## Read Config First

When `~/.thedon/config.yaml` exists, treat it as the default control plane for this workflow.

When the config file does not exist:

1. Ask the user to provide the URL for the GitHub repo that will act as the ledger.
2. If they do not already have one, ask them to create an empty repo first and then provide its URL.
3. Create `~/.thedon/config.yaml` from this template, replacing the example repo URL with the user-provided URL:

```yaml
ledger:
  repo_url: "git@github.com:user/agent-skills.git"
  local_cache: "~/.thedon/skills"

global_skills:
  enabled: true
  ledger_subdir: global_skills
  roots:
    claude: ~/.claude/skills
    cursor: ~/.cursor/skills
    codex: ~/.agents/skills
  ignore: [".DS_Store", "*.pyc", "__pycache__"]
```

If the user has not yet provided the ledger repo URL, stop after requesting that information. If they do not already have a suitable repo, tell them to create an empty repo first. Do not invent a placeholder repo URL.

Check these fields before inferring paths:

- `ledger.repo_url` for the remote ledger repository
- `ledger.local_cache` for the local checkout path
- `global_skills.enabled` to confirm the workflow is active
- `global_skills.ledger_subdir` to locate the skills subtree inside the repo
- `global_skills.roots.claude`
- `global_skills.roots.cursor`
- `global_skills.roots.codex`
- `global_skills.ignore` for files that should not be treated as skill content

If the config is missing, invalid, or still contains placeholders such as `YOUR_USER`, say that explicitly. If it is missing, ask the user for the ledger repo URL, or ask them to create an empty GitHub repo first if they do not already have one, then create the config from the template above. If it is invalid or still contains placeholders, fall back to a user-provided repo URL or path.

Do not assume the ledger lives in the current project unless the config or user request says so.

## Install Local Skill

During Setup mode, install the local global skill after config is ready or confirmed.

- Treat the source `sync.md` file the user linked or referenced as canonical for this setup run.
- If the source is a GitHub URL to this file, use that linked file as the source of truth.
- Create `<current-agent-global-skills-root>/thedon-sync/` only if needed.
- Create `<current-agent-global-skills-root>/thedon-sync/SKILL.md` by copying the exact contents of this file.
- Do not create a wrapper, placeholder, redirect, or abbreviated summary.
- Do not install the skill into roots for other agents.
- Do not stage the new `thedon-sync` skill into the ledger cache during setup by default.
- Do not offer a GitHub push for the newly installed local skill during setup by default.

If `thedon-sync` already exists locally:

- Reuse it when its purpose is clearly the same.
- If it differs from this source file, report the drift and ask before overwriting it.
- Do not silently replace an existing local skill with different content.

## Agent Scope

Work with one agent root only.

Use the configured root that matches the invoking coding agent when that mapping is available.

If the environment or config makes the current agent unclear, inspect the active toolchain and ask a concise clarification question before writing files.

Ignore roots for other agents during this workflow. Do not mirror files into every configured root.

Example:

- If the invoking agent is Codex and `global_skills.roots.codex` is configured as `~/.agents/skills`, install and compare only that root against the ledger and ignore `global_skills.roots.claude` and `global_skills.roots.cursor`.

## Sync Workflow

Sync mode assumes setup is already done and the user enters through `$thedon-sync`.

Follow this order:

1. Read `~/.thedon/config.yaml`.
2. Identify the current coding agent and its single relevant global skills root.
3. Identify the ledger source and local cache.
4. Inspect the existing local checkout before changing anything.
5. Clone or pull the ledger repo if needed.
6. Compare the ledger skills subtree against the current agent's global skills root only.
7. Exclude system or internal skills from the comparison, especially anything under `.system/`.
8. If the current agent has fewer non-system skills than the ledger, fetch missing ledger skills into the current agent's root.
9. If the current agent has more non-system skills than the ledger, sync those missing agent skills into the ledger cache.
10. If new non-system skills were synced into the ledger cache, ask the user whether to push them to the ledger GitHub remote.
11. Report what changed and what still requires manual resolution.

If config is missing during `$thedon-sync`, fall back to the setup steps needed to create or repair config, then continue only if the user still wants sync work.

## Inspect The Ledger

Prefer fast shell inspection:

- Use `rg --files` to enumerate the repo.
- Use `ls`, `find`, or `sed` to confirm whether `global_skills/` exists and how each skill is laid out.
- Treat `global_skills/<name>/SKILL.md` as canonical when present.
- Ignore `.system/` and other internal-only paths when building the ledger skill set unless the user explicitly asks to inspect them.
- If the ledger also contains flat top-level markdown files such as `foo.md`, infer that they may map to `foo/SKILL.md` for the local target, but state that inference explicitly.

Useful checks:

- Does `~/.thedon/config.yaml` exist and parse cleanly?
- What are the configured cache path and ledger subdirectory?
- What is the current agent root for this run?
- Is the repo already cloned locally?
- Is the checkout a Git repo?
- Does it have uncommitted local work that should not be overwritten?
- Which skills exist in the ledger but not in the current agent root?
- Which skills exist in the current agent root but not in the ledger?
- Which shared skills exist in both places but differ?

## Fetch Or Refresh

If the local checkout is missing:

- Clone the repo into `ledger.local_cache` when config provides it, otherwise into the agreed cache or project path.
- Do not invent a repo URL if the user or local config has not provided one.

If the local checkout exists:

- Verify it is a Git repo before pulling.
- Pull the current default branch.
- If the worktree contains user changes, avoid overwriting them. Report the state and either stop or work around it by inspecting without resetting.

## Compare And Reconcile

Compare canonical skill paths from:

- `<ledger.local_cache>/<global_skills.ledger_subdir>/`
- the current agent root only

Treat `global_skills/<name>/SKILL.md` as canonical when present.
Exclude `.system/` and other internal/system skill paths from both sides of the comparison by default.

If the current agent has fewer skills than the ledger:

- Copy missing ledger skills into the current agent root.
- Preserve the skill directory name.
- Create parent directories only as needed.
- Do not overwrite a local skill that already exists with different content unless the user asked for reconciliation.
- Do not fetch `.system/` or other internal-only skills from the ledger by default.

If the current agent has more skills than the ledger:

- Copy missing agent skills into the ledger cache under `global_skills.ledger_subdir`.
- Preserve the skill directory name.
- Do not delete local skills after syncing them into the ledger cache.
- Do not copy `.system/` or other internal-only skills into the ledger cache by default.
- After syncing new skills into the ledger cache, ask the user whether to push those changes to the ledger GitHub remote.

If the same skill exists in both places but content differs:

- Report drift clearly.
- Do not choose a winner silently.
- Only reconcile differing content when the user explicitly asks for it.

If the ledger contains a skill directory without `SKILL.md`, create the folder only if the user explicitly wants placeholders. Otherwise report it as incomplete.

## Safe Defaults

- Default to read-only inspection when the target root is ambiguous.
- Default to creating missing files only in the current agent root or the ledger cache, depending on which side is missing the skill.
- Default to excluding `.system/` and other internal/system skills from sync decisions.
- Do not run destructive Git commands such as hard reset unless the user explicitly requests it.
- State exact source and destination paths before copying files.
- Ask before pushing ledger-cache additions to the remote GitHub repo.
- Summarize setup actions, fetched skills, synced skills, and unresolved conflicts separately.

## Example Triggers

- `Follow instruction at this file: https://github.com/<owner>/<repo>/blob/<branch>/sync.md`
- `Follow instruction at this file: <link to sync.md file on GitHub>`
- `Use $thedon-sync to check whether my ledger repo has the postgres-debugger skill.`
- `Use $thedon-sync to compare this agent's global skills with the ledger and fetch any missing ones.`
- `Use $thedon-sync to inspect global_skills/ and tell me which skills for this agent are missing locally.`
- `Use $thedon-sync to sync this agent's new local skills into the ledger cache, then ask whether to push them.`

## Output

In Setup mode, return:

- the source file URL or path you used
- the current agent root you selected
- whether `~/.thedon/config.yaml` was reused or created
- the ledger repo URL or missing prerequisite you identified
- the local install path for `thedon-sync`
- whether setup is now complete
- the exact next command for future runs: `$thedon-sync`

In Sync mode, return:

- the ledger path or repo URL you inspected
- whether you cloned, pulled, or only inspected
- which current-agent skills were found
- which skills were fetched from ledger to the current agent root
- which skills were synced from the current agent root into the ledger cache
- whether a GitHub push is pending user confirmation
- any drift, ambiguity, or manual follow-up required

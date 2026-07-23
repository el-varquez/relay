---
name: init
description: Generate a relay.config.json for this project by profiling it — discovers build/test/lint commands, repo layout, and git conventions, then shows you a draft to correct. Use when the user runs /relay:init, or asks to set up or configure Relay for a project.
---

# Relay — project setup (`/relay:init`)

You profile the current project **once** and write a `relay.config.json` the rest of Relay reads
on every run. The Engineer corrects a draft rather than authoring a blank file.

Schema reference: `${CLAUDE_PLUGIN_ROOT}/skills/run/references/relay-config.md`. Read it first.

## 0. Guard

- If `./relay.config.json` already exists: **do not overwrite it.** Show what you'd change and ask
  whether to update, merge, or stop.

## 1. Profile the project

Dispatch `relay:scout` in **profile mode** — no task, just the project. Tell it explicitly:

> Profile this project for a Relay config. There is no task. Report: build / test / lint / env-setup
> commands, the repo layout (one repo, or several sibling repos with a dependency order), the
> default branch, and any git conventions (branch naming, commit style, co-author policy) stated in
> CLAUDE.md or AGENTS.md. Do not explore task-relevant code — there is no task.

Fill the gaps yourself if Scout comes back unsure:

- **Repo layout** — more than one `.git` directory in immediate subfolders means poly-repo. Infer
  `dependsOn` from what each repo imports or requires; when in doubt leave it out and say so.
- **Default branch** — `git symbolic-ref refs/remotes/origin/HEAD`, else `git branch --show-current`.
- **Branch pattern / co-author** — read the git section of `./CLAUDE.md` and the user's global
  `~/.claude/CLAUDE.md` if present.

## 2. Draft the config

Write only what you actually established. **Omit any field you are guessing at** — an absent field
falls back to discovery, which is safe; a wrong field gets executed, which is not.

Defaults to apply unless the project says otherwise:

- `ship.stopAfter`: `"pr"`
- `gates.required`: the build and lint commands you found
- `gates.advisory`: the test command **only if** the project has no test suite or documents it as
  unreliable — otherwise tests are required

## 3. ⏸ Show it and get corrections (human gate)

Show the Engineer the drafted JSON with a short line per field saying **where you got it**
(`CLAUDE.md`, lockfile, `git`, assumed default). Flag anything you guessed.

Ask them to confirm or correct — especially `ship.stopAfter`, since it decides whether Relay ever
merges, and the `gates` split, since it decides what blocks a run.

## 4. Write it

Write `./relay.config.json` at the repo root (for poly-repo: at the directory that contains the
repos, alongside whatever ties them together). Then tell them:

- what to run next — `/relay:run "<task>"`
- that every field is optional and deleting one restores discovery for it
- that `.relay/` (run state) should be gitignored; offer to add it

Keep the file minimal. A config that declares three commands and a branch pattern is a good
config; one that declares fields you inferred loosely is a liability.

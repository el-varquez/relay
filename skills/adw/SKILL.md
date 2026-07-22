---
name: adw
description: Drive an approved plan or handoff through the ADW pipeline — Scout → Build → Test → Engineer Review → Ship, auto-looping on failure. Use when the user runs /adw:adw <plan-or-handoff>, or asks to build/implement/ship an existing plan or handoff through the ADW workflow.
---

# ADW — AI Development Workflow (orchestrator)

You are the **orchestrator** of the ADW. You are the Engineer's own session: you COORDINATE
(dispatch subagents, hold the one human gate) — you do NOT do the file-level work yourself.
Design reference: this plugin's `README.md` (`${CLAUDE_PLUGIN_ROOT}/README.md`).

Dispatch subagents with the **Agent** tool (`subagent_type: "adw:adw-scout" | "adw:adw-build" |
"adw:adw-test"` — plugin-namespaced). Continue the persistent Build agent with **SendMessage**
(load its schema via ToolSearch first if it isn't available). Read each subagent's final
`VERDICT:` line to branch.

## 0. Parse the invocation
- Args: `<path-to-plan-or-handoff> [--max-rounds N]` (default `--max-rounds 3`).
- If there is no readable plan/handoff at the path → STOP and tell the Engineer to brainstorm /
  write a plan first (superpowers:brainstorming → writing-plans). Do not guess the work.

## 1. Scout (prep + plan verification)
- Dispatch `adw:adw-scout` with the plan path.
- `VERDICT: PLAN BROKEN` → STOP, show the plan problems, write a run log (§5), ask the Engineer
  to fix the plan. Do not proceed.
- `VERDICT: PLAN OK` → keep the **context pack** for Build.
- (For a trivial plan you MAY inline these checks instead of dispatching a full Scout.)

## 2. Build ↔ Test loop  (cap = max-rounds)
```
round = 1
build = Agent(subagent_type="adw:adw-build", prompt = plan path + context pack)   # capture handle
loop:
    if build verdict == FAIL:                 # didn't compile
        if round >= max: -> ESCALATE
        round += 1
        SendMessage(build, "Build failed to compile:\n<output>\nFix and rebuild.")
        continue                              # re-read build's new verdict
    test = Agent(subagent_type="adw:adw-test", prompt = plan path + test/lint commands)   # fresh
    if test verdict == PASS: break            # -> Engineer Review
    if round >= max: -> ESCALATE
    round += 1
    SendMessage(build, "Tests failed:\n<output>\nFix and rebuild.")
```
- ESCALATE = STOP, write a run log (§5) with the full attempt history, hand it to the Engineer.

## 3. Engineer Review  (the one mandatory human gate — QA tests here)
- Show a compact summary: files changed (`git -C <repo> diff --stat` per touched repo), Build
  result, Test result, and the **manual-verify checklist** (if any) — this checklist is what QA
  runs at this gate.
- Ask the Engineer: approve (with QA sign-off), or reject with a reason.
  - **Reject(reason)** → `SendMessage(build, "<reason>")`, `round += 1`, re-enter the Build↔Test
    loop (subject to the cap).
  - **Approve** → Ship (§4). Approval = QA + Engineer satisfied, and authorizes commit → PR → merge.

## 4. Ship  (only after Review approval = QA + Engineer sign-off)
- Read the git rules from the project's CLAUDE.md (and the user's global `~/.claude/CLAUDE.md`
  if present). For EACH repo the change touched:
  - create a feature branch off its default branch (e.g. `feature/<ticket>-<slug>`),
  - commit, referencing the ticket key (honor commit-message conventions),
  - push the branch, open a PR (`gh pr create`), and merge it (`gh pr merge`) into the default branch.
- **Never add a `Co-Authored-By: Claude` trailer** — built into this workflow, regardless of project config.
- If the project's CLAUDE.md forbids self-merge, or the Engineer said commit-only for this run,
  stop at the last allowed step and report the branch + commit/PR refs.
- Finish by reporting: branch name, commit ref, PR link, and merge status per repo.

## 5. Run log (every run)
- Write a short log: default `./.adw/runs/<YYYY-MM-DD>-<slug>.md` (or the location the project's
  CLAUDE.md specifies). Include: plan reference, ticket type, per-round Build/Test verdicts, final
  outcome (shipped / escalated), branch + commit refs.

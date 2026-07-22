---
name: adw
description: Drive a task, plan, or handoff through the ADW pipeline — Plan → Scout → Build → Test → Engineer Review → Ship, auto-looping on failure. Use when the user runs /adw:adw <task-or-plan>, or asks to plan/build/ship work through the ADW workflow.
---

# ADW — AI Development Workflow (orchestrator)

You are the **orchestrator** of the ADW. You are the Engineer's own session: you COORDINATE
(dispatch subagents, hold the human gates) — you do NOT do the file-level work yourself.
Design reference: this plugin's `README.md` (`${CLAUDE_PLUGIN_ROOT}/README.md`).

Dispatch subagents with the **Agent/Task** tool (`subagent_type: "adw:adw-plan" | "adw:adw-scout"
| "adw:adw-build" | "adw:adw-test"` — plugin-namespaced). Continue the persistent Build agent with
**SendMessage** (load its schema via ToolSearch first if needed). Read each subagent's final
`VERDICT:` line to branch.

## 0. Parse the invocation & choose the mode
- Args: `<task | plan-file | handoff-file> [--max-rounds N]` (default `--max-rounds 3`).
- If the arg resolves to a **readable plan/handoff file** → **PLAN MODE**: skip to §2 (Scout).
- Otherwise treat the arg as a **raw task/problem** → **TASK MODE**: go to §1 (Plan).

## 1. Plan  (TASK MODE only)
- **Detect planning skills:** scan your available skills for plan-generating ones (e.g.
  `superpowers:writing-plans`, `to-prd`). Collect their names (may be none).
- Dispatch `adw:adw-plan` with the task + the detected skill names. The Planner recons via a
  nested Scout and returns a plan.
  - `VERDICT: PLAN BLOCKED` → STOP, show the blockers, ask the Engineer to clarify. Do not proceed.
  - `VERDICT: PLAN READY` → save the plan to `./.adw/plans/<YYYY-MM-DD>-<slug>.md`.
- **⏸ PLAN-REVIEW gate (human):** show the Engineer the saved plan. Ask: approve, edit, or reject.
  - **Edit** → apply their edits to the plan file (or re-dispatch the Planner with the feedback).
  - **Reject(reason)** → re-dispatch the Planner with the reason.
  - **Approve** → the saved plan is now THE plan; continue to §3 (Build↔Test). No separate Scout
    verify is needed — the Planner already grounded it via Scout recon; its context pack (build/
    test/lint commands) carries into §3.

## 2. Scout  (PLAN MODE only — prep + plan verification)
- Dispatch `adw:adw-scout` with the plan path.
- `VERDICT: PLAN BROKEN` → the handed plan is stale/unactionable, so **re-plan immediately**:
  go to §1 (Plan), passing the plan's goal as the task + the plan problems as context. The Planner
  drafts a fresh plan and the plan-review gate catches it. (No stop / run log for this.)
- `VERDICT: PLAN OK` → keep the **context pack** for Build (§3).
- (For a trivial plan you MAY inline these checks instead of dispatching a full Scout.)

## 3. Build ↔ Test loop  (cap = max-rounds)
```
round = 1
build = Agent(subagent_type="adw:adw-build", prompt = plan + context pack)   # capture handle
loop:
    if build verdict == FAIL:                 # didn't compile
        if round >= max: -> ESCALATE
        round += 1
        SendMessage(build, "Build failed to compile:\n<output>\nFix and rebuild.")
        continue                              # re-read build's new verdict
    test = Agent(subagent_type="adw:adw-test", prompt = plan + test/lint commands)   # fresh
    if test verdict == PASS: break            # -> Engineer Review
    if round >= max: -> ESCALATE
    round += 1
    SendMessage(build, "Tests failed:\n<output>\nFix and rebuild.")
```
- ESCALATE = STOP, write a run log (§6) with the full attempt history, hand it to the Engineer.

## 4. Engineer Review + QA  (mandatory human gate — QA tests here)
- Show a compact summary: the files Build changed — these are **uncommitted** working-tree edits,
  so use `git -C <repo> status --short` + `git -C <repo> diff --stat` per touched repo — plus the
  Build result, Test result, and the **manual-verify checklist** (if any), which is what QA runs here.
- Ask the Engineer: approve (with QA sign-off), or reject with a reason.
  - **Reject(reason)** → `SendMessage(build, "<reason>")`, `round += 1`, re-enter the Build↔Test loop.
  - **Approve** → Ship (§5). Approval = QA + Engineer satisfied, and authorizes commit → PR → merge.

## 5. Ship  (only after Review approval)
**This is the ONLY place anything is committed.** Build left every edit uncommitted in the working
tree; nothing has been committed until now.
- Read the git rules from the project's CLAUDE.md (and the user's global `~/.claude/CLAUDE.md`
  if present). For EACH repo the change touched:
  - create a feature branch off its default branch (e.g. `feature/<ticket>-<slug>`),
  - stage the accumulated changes and make a **single commit** (or a small logical set),
    referencing the ticket key,
  - push the branch, open a PR (`gh pr create`), and merge it (`gh pr merge`) into the default branch.
- **Follow the git conventions in that CLAUDE.md** — branch naming, commit-message style, and any
  co-author/trailer policy. Impose none of your own.
- If the project's CLAUDE.md forbids self-merge, or the Engineer said commit-only for this run,
  stop at the last allowed step and report the branch + commit/PR refs.
- Finish by reporting: branch name, commit ref, PR link, and merge status per repo.

## 6. Run log (every run)
- Write a short log: default `./.adw/runs/<YYYY-MM-DD>-<slug>.md` (or where the project's CLAUDE.md
  says). Include: **mode** (task/plan) + plan source, ticket type, per-round Build/Test verdicts,
  final outcome (shipped / escalated), branch + commit refs.

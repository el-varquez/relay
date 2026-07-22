# ADW — AI Development Workflow

A Claude Code plugin that takes an **approved plan or handoff** and drives it through
**Scout → Build → Test → Engineer Review → Ship**, auto-looping on failure. One
project-agnostic workflow that reads each project's own `CLAUDE.md` to learn how *that*
project builds and tests.

## Install

```
/plugin marketplace add el-varquez/adw
/plugin install adw@adw
```

Then, from inside any project:

```
/adw:adw "<a task or problem>"            # NEW: ADW plans it for you first
/adw:adw <path-to-plan-or-handoff>        # or hand it an existing plan
/adw:adw <task-or-plan> --max-rounds 5
```

No plan yet? Just pass the task — the Planner drafts one and you approve it before any building.

## The flow

```mermaid
flowchart LR
    E(["Engineer: /adw:adw (task | plan | handoff)"]) --> M{"task or plan?"}

    M -->|task| P["🧭 Planner (adw-plan)"]
    P -->|"sub-agent"| PS["🔍 Scout (adw-scout)<br/>recon"]
    PS -->|"context pack"| P
    P -->|"PLAN READY"| PR{{"⏸ Plan Review (you)"}}
    P -->|"PLAN BLOCKED"| X(["⛔ clarify"])
    PR -->|"reject"| P
    PR -->|"approve"| B

    M -->|plan| S["🔍 Scout (adw-scout)<br/>verify plan"]
    S -->|"PLAN OK"| B
    S -->|"PLAN BROKEN"| P

    B["🔨 Build (adw-build)<br/>implement + compile"] -->|"PASS"| T["✅ Test (adw-test)<br/>tests + lint"]
    T -->|"FAIL"| B
    T -->|"PASS"| R{{"⏸ Engineer Review + QA (you)"}}
    R -->|"reject"| B
    R -->|"approve"| SH["🚢 Ship (orchestrator)<br/>commit → push → PR → merge"]
    SH --> DONE(["🎉 Merged"])
```

*Renders as a diagram in Obsidian, GitHub, and most Markdown viewers. `adw-scout` appears twice — it's one agent in two roles: verifying a handed plan, or reconning for the Planner. A broken handed-plan is re-planned by the Planner. Build↔Test loops at most 3 rounds (`--max-rounds N`), then escalates to you.*

<details>
<summary>Plain-text version</summary>

- **Engineer** runs `/adw:adw` with a task, plan, or handoff.
- **task** → **Planner** (`adw-plan`) spawns the **Scout** (`adw-scout`) sub-agent to recon, then drafts a plan → you approve/edit it at **Plan Review** (or the Planner reports it's blocked → you clarify).
- **plan / handoff** → **Scout** (`adw-scout`) verifies it → *PLAN OK* continues; *PLAN BROKEN* hands it to the **Planner** to re-plan.
- **Build** (`adw-build`) implements + compiles → **Test** (`adw-test`) runs tests + lint.
- Test fail → back to Build. Test pass → **Engineer Review + QA** (you). Reject → back to Build.
- Approve → **Ship** (orchestrator): commit → push → PR → merge.
- Build↔Test loops at most 3 rounds (`--max-rounds N`), then escalates to you.

</details>

## How it works

- **Task or plan.** Give ADW a finished plan, or just a task — the **Planner** agent recons the
  code (via a nested Scout) and drafts a grounded plan for you to approve before any building
  starts. It uses `superpowers:writing-plans` if installed, otherwise plans plainly.
- **Project-agnostic.** Nothing is hardcoded. The orchestrator (your Claude Code session)
  reads *this* project's `CLAUDE.md` for build/test/lint commands and git conventions, then
  delegates to subagents.
- **Two read-only verifiers bracket one writer.** `adw:adw-scout` verifies the *plan* before
  Build; `adw:adw-test` verifies the *result* after. Only `adw:adw-build` edits code — enforced
  by tool scope.
- **QA is part of Review.** The Engineer Review gate is where QA runs the manual-verify
  checklist. Approval means QA + engineer signed off — that authorizes Ship.
- **Persistent Build.** The Build agent is continued across retries, so it remembers prior
  attempts. Both fail loop-backs feed it. Cap: 3 rounds, then it escalates to you.
- **Nothing commits until Ship.** Build only edits the working tree — it never commits, branches,
  or pushes. All changes are committed once, at Ship, after Engineer + QA approval.

## Stages

| Stage | Writes? | Job | Passes when |
|-------|---------|-----|-------------|
| Plan *(task mode)* | no | Planner turns a task into a grounded plan, reconning via Scout | you approve the plan |
| Scout | no | discover build/test/lint cmds; verify plan vs code; emit context pack | plan actionable → `PLAN OK` |
| Build | yes | implement the plan, then compile | clean compile → `PASS` |
| Test  | no | run the plan's Verify (tests, else compile + lint) | every gate green → `PASS` |
| Review| — | the mandatory human gate; QA tests here | you approve with QA sign-off |
| Ship  | git | commit → push → PR → merge | merged |

## Notes

- **Run logs** land in `./.adw/runs/<date>-<slug>.md` per project.
- **Git conventions** (branch naming, commit-message style, co-author trailers) follow *your*
  project or global `CLAUDE.md` — the workflow imposes none of its own.
- Requires Claude Code with plugin support.

## Credits

- Inspired by [IndyDevDan](https://www.youtube.com/@indydevdan)'s **Agentic Developer Workflow (ADW)** concept.

## License

MIT © 2026 el-varquez

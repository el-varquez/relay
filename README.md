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
/adw:adw <path-to-plan-or-handoff>            # e.g. /adw:adw path/to/your-plan.md
/adw:adw <path-to-plan-or-handoff> --max-rounds 5
```

No plan yet? Brainstorm and write one first, then feed it in.

## The flow

```
Engineer: /adw:adw <plan|handoff>
    │
    ▼
🔍 SCOUT   read-only · discover build/test/lint cmds + verify plan vs code
    │  PLAN BROKEN → stop + run log        PLAN OK (+ context pack)
    ▼
🔨 BUILD   the only writer · implement plan + compile     ◀─┐  fail: loop back
    │  PASS                                                  │  (same agent, keeps memory)
    ▼                                                        │
✅ TEST    read-only · run plan's Verify (tests, else lint) ─┘
    │  PASS (+ manual-verify checklist)
    ▼
⏸  ENGINEER REVIEW + QA   QA runs the checklist here · approve = QA + Eng sign-off
    │  approve                          reject(reason) → loop back to Build
    ▼
🚢 SHIP    commit → push → PR → merge (per the project's git rules)

Build↔Test auto-loops at most 3 rounds (--max-rounds N), then escalates to you.
```

## How it works

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

## Stages

| Stage | Writes? | Job | Passes when |
|-------|---------|-----|-------------|
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
- **Contributors:** el-varquez · [Claude Code](https://claude.com/claude-code)

## License

MIT © 2026 el-varquez

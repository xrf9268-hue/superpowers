# Feature Proposal: Task-based Worktree Workflow with Persistent State Tracking

> **Issue Title:** Feature: Task-based worktree workflow with persistent state tracking

## Summary

Propose a new workflow pattern for **large requirements** that enables:
1. **Per-task worktree isolation** - Each task gets its own git worktree and branch
2. **Persistent state tracking** - Task status stored in `plan.md` (survives session interruptions)
3. **Parallel execution markers** - Indicate which tasks can run in parallel worktrees
4. **Spec/PRD references** - Link each task to its source requirement

This addresses limitations in the current workflow which assumes single-session completion with in-memory state (TodoWrite).

---

## Problem

Current Superpowers workflow has limitations for large requirements:

| Issue | Current Behavior | Impact |
|-------|-----------------|--------|
| Session interruption | TodoWrite state lost | Must restart/re-understand progress |
| Cross-day development | No persistent progress | Can't easily hand off or resume |
| Multi-person collaboration | Single worktree per feature | Can't truly parallelize |
| Requirement traceability | Tasks disconnected from specs | Hard to verify completeness |

---

## Proposed Workflow

```
feature-A/ (requirement branch)
    │
    ├── docs/plans/plan.md   ← [ ] Task 3 (pending)
    │
    └── git worktree add task-3/ -b task-3
                │
                ├── ... develop code ...
                │
                ├── Update plan.md: [ ] → [x]
                │
                └── commit (code + status update together)
                          │
                          ↓ merge

feature-A/
    └── docs/plans/plan.md   ← [x] Task 3 (completed via merge)
```

---

## Proposed plan.md Template Format

```markdown
# Feature Name - Implementation Plan

**Status:** In Progress
**Created:** 2025-01-16
**Branch:** feature-A
**PRD:** docs/specs/feature-a-prd.md

---

## Tasks

### Task 1: Project Setup
- **Status:** [x] Completed
- **Parallel:** false
- **Spec:** docs/specs/feature-a-prd.md#setup-requirements
- **Description:**
  - Initialize go.mod
  - Create directory structure
- **Completed:** 2025-01-16, commit `abc123`

---

### Task 2: CLI Framework
- **Status:** [x] Completed
- **Parallel:** false (depends on Task 1)
- **Spec:** docs/specs/feature-a-prd.md#cli-interface
- **Description:**
  - Add cobra dependency
  - Create root command
- **Completed:** 2025-01-16, commit `def456`

---

### Task 3: Sierpinski Algorithm
- **Status:** [ ] Pending
- **Parallel:** true ← Can run in parallel with Task 4
- **Spec:** docs/specs/feature-a-prd.md#sierpinski-requirements
- **Description:**
  - Create internal/sierpinski/
  - Implement Generate() function
  - Add unit tests

---

### Task 4: Mandelbrot Algorithm
- **Status:** [ ] Pending
- **Parallel:** true ← Can run in parallel with Task 3
- **Spec:** docs/specs/feature-a-prd.md#mandelbrot-requirements
- **Description:**
  - Create internal/mandelbrot/
  - Implement Render() function
  - Add unit tests

---

### Task 5: Integration
- **Status:** [ ] Pending
- **Parallel:** false (depends on Task 3, 4)
- **Spec:** docs/specs/feature-a-prd.md#integration
- **Description:**
  - Wire algorithms to CLI
  - Add E2E tests
```

---

## Key Features

### 1. Persistent State Tracking

```markdown
- **Status:** [ ] Pending    ← Checkbox in markdown
- **Status:** [x] Completed  ← Updated when task done
```

- State survives session interruptions
- Easy to parse programmatically
- Clear visual indicator in any markdown viewer

### 2. Parallel Execution Markers

```markdown
- **Parallel:** true   ← Can create separate worktree
- **Parallel:** false  ← Must wait for dependencies
```

Benefits:
- AI/human can identify parallelizable work
- Multiple worktrees can be created for `Parallel: true` tasks
- Prevents conflicts by marking dependent tasks

### 3. Spec/PRD References (inspired by AWS Kiro)

```markdown
- **Spec:** docs/specs/prd.md#section-name
```

Benefits:
- Traceability: Each task linked to source requirement
- Verification: Can check if all spec items are covered
- Context: Implementer can read exact requirements
- Completeness: Easy to audit "did we implement everything?"

---

## Proposed Skills Changes

### New Skill: `task-based-worktrees`

```markdown
# Task-Based Worktrees

Create isolated worktree per task for large requirements.

## When to Use
- Large requirement (10+ tasks)
- Multi-day development
- Multiple people collaborating
- Need persistent progress tracking

## Process
1. Read plan.md, find first `[ ]` task with `Parallel: true` or next sequential task
2. Create worktree: `git worktree add .worktrees/task-N/ -b task-N`
3. Execute task (TDD, implement, test)
4. Update plan.md: `[ ]` → `[x]`, add completion info
5. Commit (code + status update atomically)
6. Merge back to feature branch
7. Remove worktree
8. Repeat for next task
```

### New Skill: `cross-session-resume`

```markdown
# Cross-Session Resume

Resume work from where you left off.

## Process
1. Read plan.md
2. Parse task statuses
3. Find first `[ ]` (pending) task
4. Check if worktree already exists for it
5. Either continue in existing worktree or create new one
6. Resume execution
```

### Modify: `writing-plans`

Add template fields:
- `Parallel:` marker for each task
- `Spec:` reference for each task
- `Status:` checkbox format

### Modify: `dispatching-parallel-agents`

Enhance to:
- Read `Parallel: true` markers from plan.md
- Create multiple worktrees for parallel tasks
- Dispatch agents to different worktrees
- Coordinate merge back

---

## Comparison with Current Workflow

| Aspect | Current | Proposed |
|--------|---------|----------|
| State storage | TodoWrite (memory) | plan.md (Git) |
| Session survival | ❌ Lost | ✅ Persistent |
| Multi-person | ❌ Single worktree | ✅ Per-task worktree |
| Parallelization | Subagent (same worktree) | True Git parallel |
| Requirement tracing | ❌ None | ✅ Spec references |
| Resume capability | ❌ Manual | ✅ Automatic |

---

## Use Cases

### Use Case 1: Large Feature (30 tasks, 1 week)

```
Day 1: Complete Task 1-5, plan.md shows [x] [x] [x] [x] [x] [ ] [ ] ...
Day 2: New session, read plan.md, resume from Task 6
Day 3: Hand off to colleague, they see exactly where to continue
```

### Use Case 2: Parallel Algorithm Implementation

```
Task 3 (Sierpinski): Parallel=true
Task 4 (Mandelbrot): Parallel=true

→ Create two worktrees simultaneously
→ Two agents/people work in parallel
→ Both merge back when done
→ No conflicts (different files)
```

### Use Case 3: Requirement Audit

```
PM: "Did we implement all the PRD requirements?"

→ Scan plan.md for all Spec references
→ Cross-reference with PRD sections
→ Identify any unlinked requirements
```

---

## Implementation Suggestion

### Phase 1: Template Enhancement
- Update `writing-plans` skill with new format
- Add `Status`, `Parallel`, `Spec` fields

### Phase 2: New Skills
- Create `task-based-worktrees` skill
- Create `cross-session-resume` skill

### Phase 3: Integration
- Modify `dispatching-parallel-agents` to use markers
- Add plan.md parsing utilities to `lib/skills-core.js`

---

## References

- AWS Kiro: Task-spec linking pattern
- Current `using-git-worktrees` skill: Worktree creation
- Current `writing-plans` skill: Plan template
- Current `dispatching-parallel-agents` skill: Parallel execution

---

## Discussion

Questions for maintainers:
1. Should this be an alternative workflow or replace the current one?
2. Should `Parallel` use explicit dependency graph (e.g., `depends: [task-1, task-2]`) instead of boolean?
3. How to handle merge conflicts when multiple parallel tasks modify plan.md status?
4. Should there be a CLI tool to parse/update plan.md status programmatically?

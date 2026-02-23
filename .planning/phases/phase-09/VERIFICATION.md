# Phase 9: Approval Workflow - Verification Checklist

**Phase:** 9 - Approval Workflow (FINAL)
**Verification Date:** TBD (post-execution)

---

## Pre-Execution Checklist

- [x] Research completed (RESEARCH.md)
- [x] Execution plan created (PLAN.md)
- [x] Requirements mapped to tasks
- [x] Dependencies verified (Phase 7 ✅, Phase 8 ✅)
- [ ] Phase ready for execution

---

## Requirements Coverage

| Requirement | Plan Reference | Status |
|-------------|----------------|--------|
| APPROVE-01: Generate migration preview | Wave 1-2, Tasks 1.1-2.1 | ⏳ Planned |
| APPROVE-02: Display preview to user | Wave 2, Task 2.1 | ⏳ Planned |
| APPROVE-03: Require explicit approval | Wave 3, Task 3.2 | ⏳ Planned |
| APPROVE-04: Allow modification | Wave 3, Task 3.2 (edit mode) | ⏳ Planned |

---

## Test Scenarios

### Scenario 1: Happy Path Approval
**Steps:**
1. Run `/wgsd migrate` on GSD project
2. Observe preview display
3. Type `yes`
4. Observe migration execution

**Expected Results:**
- [ ] Preview shows focus groups with numbers
- [ ] Preview shows concepts with mappings
- [ ] Preview shows planned channels
- [ ] Migration proceeds after `yes`
- [ ] All items created as shown in preview

---

### Scenario 2: Cancel Migration
**Steps:**
1. Run `/wgsd migrate`
2. Type `no`

**Expected Results:**
- [ ] Clean exit message displayed
- [ ] No files created or modified
- [ ] Backup removed (if any was created)

---

### Scenario 3: Edit Focus Groups
**Steps:**
1. Run `/wgsd migrate`
2. Type `edit`
3. `rename-fg 1 newname`
4. `add-fg custom`
5. `remove-fg 2`
6. `done`
7. Review updated preview
8. Type `yes`

**Expected Results:**
- [ ] Renamed focus group shown in preview
- [ ] New focus group added to preview
- [ ] Removed focus group not in preview
- [ ] Concepts reassigned from removed FG
- [ ] Only modified list created

---

### Scenario 4: Edit Concepts
**Steps:**
1. Run `/wgsd migrate`
2. Type `edit`
3. `move-concept 1 different-fg`
4. `exclude-concept 2`
5. `include-concept 2`
6. `done`
7. Type `yes`

**Expected Results:**
- [ ] Concept 1 shows new assignment
- [ ] Concept 2 marked excluded, then re-included
- [ ] Final preview reflects all changes
- [ ] Correct concept files created

---

### Scenario 5: Empty Project
**Steps:**
1. Run `/wgsd migrate` on project with no ROADMAP.md

**Expected Results:**
- [ ] Preview shows "core" as default focus group
- [ ] No concepts listed (or minimal default)
- [ ] Migration still works cleanly

---

### Scenario 6: Invalid Edit Commands
**Steps:**
1. Run `/wgsd migrate`
2. Type `edit`
3. `rename-fg 99 test` (invalid number)
4. `move-concept abc xyz` (invalid args)
5. `unknown-command`

**Expected Results:**
- [ ] Error message for each invalid command
- [ ] No crash or unexpected behavior
- [ ] Can continue editing after errors
- [ ] `done` still works

---

## Integration Verification

### With Phase 7 (Logic Fix)
- [ ] Concepts correctly derived from Phases (not Focus Groups)
- [ ] Preview shows Phase → Concept mapping clearly

### With Phase 8 (Slack Automation)
- [ ] Slack connectivity checked before preview
- [ ] Channels section shows correct status (connected/not configured)
- [ ] Only approved channels created
- [ ] Channel creation happens after approval

---

## Code Quality Checklist

- [ ] All bash arrays correctly indexed
- [ ] String parsing handles edge cases (spaces, special chars)
- [ ] Edit commands validated before execution
- [ ] Exit codes appropriate
- [ ] Rollback/cleanup works on cancel

---

## Performance Verification

- [ ] Preview generation is fast (< 1 second)
- [ ] Edit mode is responsive
- [ ] No excessive API calls before approval

---

## Documentation Updates

After successful execution:
- [ ] SKILL.md updated with approval workflow note
- [ ] README.md mentions preview/approval feature
- [ ] Inline comments in migrate-planning.md

---

## Sign-off

| Role | Name | Date | Signature |
|------|------|------|-----------|
| Developer | — | — | ⏳ |
| Reviewer | — | — | ⏳ |

---

*Verification checklist for Phase 9 — complete after execution*

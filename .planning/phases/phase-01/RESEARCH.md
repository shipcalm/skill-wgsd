# Phase 1 Research: Foundation & Core Infrastructure

**Date:** 2026-02-22
**Phase:** 1 - Foundation & Core Infrastructure
**Requirements:** INTEGRATE-01, INTEGRATE-02, INTEGRATE-06, CHANNEL-01, CHANNEL-02, INTEGRATE-03

---

## Research Summary

### Workspace Management Best Practices

**Industry Pattern: Monorepo with Worktrees**

From research on tools like Nx, Turborepo, Graphite, and GitKraken:

1. **Directory Organization**: A unified structure helps in easy navigation and discovery within the repository
2. **Sparse Checkouts**: Enable developers to work with a large repository without cloning the entire codebase
3. **Worktrees for Parallel Work**: Git worktrees allow multiple checkouts of the same repository simultaneously
4. **Clean History**: Use trunk-based development with clean branches

**WGSD Application:**
- `wgsd/{project}/` workspace structure isolates WGSD projects from other work
- Worktrees enable focus group and implementation parallel development
- `develop` as the clean base branch maintains integration point integrity

### Git Branch Strategy Patterns

**Trunk-Based Development (Recommended for WGSD)**

| Branch Type | Purpose | Lifecycle |
|-------------|---------|-----------|
| `develop/main` | Integration point | Permanent |
| `focus-groups/{name}` | Planning work | Long-lived |
| `implementations/{name}` | Code execution | Short-lived (1-3 days) |

**Key Insights:**
- Keep branches small to prevent performance degradation
- Use rebase for linear history in implementations
- Focus groups can have longer-lived branches since they contain planning, not code

### Channel Naming Conventions

**Observed Patterns in Enterprise Slack:**

| Pattern | Example | Use Case |
|---------|---------|----------|
| `{project}-{type}` | `mvn-dev` | Project + category |
| `{project}-{type}-{name}` | `mvn-fg-security` | Project + category + specifics |
| Lowercase with dashes | `auth-v2-impl` | Readability |
| 3-5 char stubs | `mvn`, `oc`, `wgsd` | Quick recognition |

**WGSD Naming Scheme:**
```
{stub}-dev           → Core development channel
{stub}-fg-{name}     → Focus group channels  
{stub}-cpt-{name}    → Concept channels
{stub}-impl-{name}   → Implementation channels
{stub}-community     → Public community channel
```

### Planning Structure Migration

**GSD to WGSD Structure Mapping:**

| GSD Structure | WGSD Structure | Migration |
|---------------|----------------|-----------|
| `.planning/ROADMAP.md` | `.planning/MASTER-ROADMAP.md` | Rename + enhance |
| `.planning/STATE.md` | `.planning/STATE.md` | Keep |
| `.planning/REQUIREMENTS.md` | `.planning/focus-groups/{fg}/concepts/` | Split by domain |
| `.planning/phases/` | `.planning/focus-groups/` | Transform phases → focus groups |

---

## Technical Decisions

### Decision 1: Workspace Root Location

**Options:**
1. `~/wgsd/{project}/` - User home based
2. `~/.openclaw/workspace/wgsd/{project}/` - OpenClaw workspace based
3. `./wgsd/{project}/` - Current directory relative

**Decision:** Option 2 - OpenClaw workspace based
**Rationale:** 
- Consistent with OpenClaw conventions
- Agent always has access to workspace
- Git operations work reliably

### Decision 2: Branch Base Strategy

**Options:**
1. Base off `main` always
2. Base off `develop` always  
3. Detect existing primary branch

**Decision:** Option 3 - Detect and use existing
**Rationale:**
- Some repos use `main`, others `develop`
- GSD projects may already have `develop`
- More flexible for migration

### Decision 3: Stub Collection Method

**Options:**
1. Auto-derive from repo name
2. Always ask user
3. Suggest from repo name, allow override

**Decision:** Option 3 - Suggest with override
**Rationale:**
- Reduces friction for simple cases (mvn → marvin)
- Allows custom stubs when needed (oc → openclaw)
- Interactive confirmation prevents mistakes

### Decision 4: Naming Convention Enforcement

**Options:**
1. Soft validation (warn but allow)
2. Hard validation (reject non-conforming)
3. Auto-correct with confirmation

**Decision:** Option 3 - Auto-correct with confirmation
**Rationale:**
- Reduces user frustration
- Maintains consistency
- Educational (shows correct format)

---

## Implementation Approach

### Phased Delivery

Phase 1 will be delivered in 3 sub-tasks:

1. **Task 1.1: Workspace Management** (INTEGRATE-01, INTEGRATE-06)
   - Workspace initialization workflow
   - Git operations library
   - Workspace status command

2. **Task 1.2: Naming Conventions** (CHANNEL-01, CHANNEL-02)
   - Stub collection and validation
   - Channel naming validator
   - Auto-correction utilities

3. **Task 1.3: Planning Migration** (INTEGRATE-02, INTEGRATE-03)
   - Branch strategy implementation
   - GSD → WGSD structure migration
   - Planning file transformation

### Reusable Components

From existing codebase analysis:

| Component | Location | Reuse Strategy |
|-----------|----------|----------------|
| `setup-repo.md` | workflows/ | Enhance for workspace management |
| `create-channel.md` | workflows/ | Add naming validation |
| `create-focus-group.md` | workflows/ | Extract git operations |

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Workspace init time | < 30 seconds |
| Naming validation accuracy | 100% (reject invalid) |
| Migration preservation | Zero data loss |
| User prompts | ≤ 3 per workflow |

---

## Risk Analysis

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Git worktree conflicts | Medium | High | Validate clean state before operations |
| Naming collision | Low | Medium | Check existing channels before create |
| Migration data loss | Low | High | Create backup before transformation |
| User confusion | Medium | Low | Clear documentation and error messages |

---

*Research completed: 2026-02-22*
*Ready for execution planning*

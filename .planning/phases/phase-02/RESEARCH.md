# Phase 2: Migration Wizard - Domain Research

**Phase:** 2 - Migration Wizard
**Requirements:** 9 (MIGRATE-01 through MIGRATE-08 + INTEGRATE-08)
**Research Date:** 2026-02-22

---

## Domain Analysis

### GSD State Detection (MIGRATE-01)

**Current GSD State Signals:**
1. **Branch State**
   - Current branch name (may indicate phase: `phase-01-foundation`)
   - Uncommitted changes (files modified but not staged/committed)
   - Ahead/behind remote status
   
2. **Planning Phase State**
   - `.planning/STATE.md` - Current phase indicator
   - `.planning/phases/` - Which phases exist, which is active
   - `.planning/ROADMAP.md` - Overall project progress
   
3. **Work-in-Progress Indicators**
   - Files in `phases/phase-XX/` being edited
   - Draft documents without verification markers
   - TODO/FIXME comments in planning docs
   
**Detection Strategy:**
```
1. Check git status → branch, dirty files, sync state
2. Parse STATE.md → current phase, completion %
3. Analyze phases/ → identify active phase by recency/content
4. Scan for WIP markers → incomplete docs, draft status
```

---

### Focus Group Suggestions (MIGRATE-02, MIGRATE-03)

**Roadmap Analysis Strategy:**
1. Parse `ROADMAP.md` phase headers
2. Extract phase names and map to focus group domains
3. Look for cross-cutting concerns (security, performance)
4. Identify natural groupings by dependency

**Codebase Structure Analysis:**
| Directory Pattern | Suggested Focus Group |
|-------------------|----------------------|
| `/api`, `/routes`, `/endpoints` | api |
| `/components`, `/ui`, `/frontend` | frontend |
| `/lib`, `/core`, `/shared` | core |
| `/auth`, `/security` | security |
| `/db`, `/models`, `/migrations` | data |
| `/tests`, `/__tests__` | testing |
| `/docs`, `/wiki` | documentation |
| `/scripts`, `/tools`, `/infra` | infrastructure |

**Intelligent Mapping Logic:**
- Phase-01 "Foundation" → core focus group
- Phase-02 "Authentication" → security focus group
- Phases with multiple concerns → split into multiple FGs

---

### WIP Preservation (MIGRATE-04, MIGRATE-05)

**Work-in-Progress Detection:**
1. Check if current phase has uncommitted work
2. Check if current branch has unpushed commits
3. Check if current phase is marked incomplete in STATE.md

**Implementation Generation:**
- Extract phase name from STATE.md or branch
- Convert to implementation format: `{stub}-impl-{phase-name}`
- Example: "Phase 2: Authentication" → `mvn-impl-auth-phase2`

**Auto-Naming Strategy:**
1. Parse phase name: "Phase 2: Security Implementation"
2. Extract key terms: ["security", "implementation"]
3. Generate slug: "security-v2" or "security-phase2"
4. Validate uniqueness against existing implementations

---

### Migration Safety (INTEGRATE-08)

**Transaction Model:**
```
1. BACKUP: Copy .planning/ to .planning-backup-{timestamp}
2. DETECT: Gather all GSD state information
3. PLAN: Generate migration plan with all changes
4. CONFIRM: Show user what will change
5. EXECUTE: Apply changes atomically
6. VERIFY: Validate migration success
7. COMMIT: Git commit all changes
8. CLEANUP: Remove backup (optional)

On FAILURE at any step:
- ROLLBACK: Restore from backup
- REPORT: Show what failed and why
```

**Rollback Points:**
- Pre-migration backup (always created)
- Git stash of uncommitted changes
- Git reflog for commit recovery

---

### Team Communication (MIGRATE-08)

**Announcement Template Elements:**
1. **What Changed** - GSD → WGSD migration complete
2. **New Structure** - Focus groups, concepts, implementations
3. **New Channels** - List of created Slack channels
4. **New Workflow** - How to contribute
5. **Action Items** - What team members need to do

**Personalization Tokens:**
- `{project_name}` - Project display name
- `{stub}` - Slack channel stub
- `{focus_groups}` - List of focus groups
- `{dev_channel}` - Main dev channel name
- `{migration_date}` - When migration occurred

---

## Phase 2 Deliverables

| Deliverable | Type | Requirements |
|-------------|------|--------------|
| `workflows/migrate.md` | Workflow | All MIGRATE-*, INTEGRATE-08 |
| `agents/migration-analyzer.md` | Agent | MIGRATE-01, MIGRATE-02, MIGRATE-03 |
| `workflows/lib/state-detection.md` | Library | MIGRATE-01 |
| `workflows/lib/wip-preservation.md` | Library | MIGRATE-04, MIGRATE-05 |
| `workflows/lib/rollback.md` | Library | INTEGRATE-08 |
| `templates/announcement.md` | Template | MIGRATE-08 |

---

## Integration with Phase 1

**Reusing Phase 1 Libraries:**
- `lib/git-ops.md` - For state detection, backup, rollback
- `lib/branch-ops.md` - For WIP preservation branches
- `lib/naming.md` - For implementation naming validation
- `migrate-planning.md` - Base for enhanced migration
- `planning-migrator.md` - Base for analyzer agent

**Extending Phase 1 Workflows:**
- `init.md` - Call migrate workflow after workspace setup
- `workspace-status.md` - Include migration state

---

## Success Criteria Verification

1. **MIGRATE-01**: `wgsd analyze` shows accurate GSD state report
2. **MIGRATE-02**: System suggests 2-5 focus groups from roadmap
3. **MIGRATE-03**: System suggests additional FGs from directory structure
4. **MIGRATE-04**: WIP converts to active implementation without data loss
5. **MIGRATE-05**: Implementation names follow `{stub}-impl-{name}` pattern
6. **MIGRATE-06**: Workspace created (delegated to Phase 1 `init.md`)
7. **MIGRATE-07**: Planning artifacts transformed correctly
8. **MIGRATE-08**: Team announcement ready for posting
9. **INTEGRATE-08**: Failed migration leaves original untouched

---

*Research complete - ready for planning*

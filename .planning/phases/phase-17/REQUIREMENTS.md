# Phase 17: Migration Auto-Channel Creation

**Status:** Planned  
**Created:** 2026-02-23  
**Dependencies:** None (bug fix)  
**Priority:** High  

---

## Problem Statement

WGSD v2.2 migration workflow (`workflows/migrate.md`) currently provides manual instructions for Slack channel creation after migration completes:

```
📝 Next Steps:
   1. Create Slack channel: #${STUB}-dev
   2. Create focus group channels: #${STUB}-fg-*
```

However, migration should automatically create these channels as part of the process, not require manual post-migration steps.

**Evidence:** Marvin migration (2026-02-23) required manual creation of 7 channels after migration.

---

## Requirements

### REQ-17-01: Automatic Dev Channel Creation
**Priority:** Must Have  
Migration workflow must automatically create the main development channel (`#{stub}-dev`) during migration process.

### REQ-17-02: Automatic Focus Group Channel Creation  
**Priority:** Must Have  
Migration workflow must automatically create focus group channels (`#{stub}-fg-{name}`) for each focus group discovered during migration.

### REQ-17-03: Automatic Implementation Channel Creation
**Priority:** Must Have  
Migration workflow must automatically create implementation channels (`#{stub}-impl-{name}`) for any work-in-progress preserved during migration.

### REQ-17-04: Automatic Team Invitation
**Priority:** Should Have  
Migration workflow should automatically invite relevant team members to created channels based on WGSD configuration.

### REQ-17-05: Graceful Error Handling
**Priority:** Must Have  
Channel creation failures must be handled gracefully without breaking the entire migration process.

---

## Success Criteria

- [ ] Migration creates `#{stub}-dev` channel automatically
- [ ] Migration creates all required `#{stub}-fg-*` channels  
- [ ] Migration creates implementation channels for active work
- [ ] Team members are invited to relevant channels
- [ ] Channel creation failures are handled gracefully
- [ ] Migration success report shows created channels with IDs
- [ ] No manual post-migration channel creation steps required

---

## Technical Notes

- Integration point: After Step 10 (Commit Migration) in `workflows/migrate.md`
- Use existing `create-channel.md` workflow for consistency  
- Channel information available: `$STUB`, `$UNIQUE_FGS`, `$IMPL_NAME`
- Error handling: Warn on failure but don't abort migration

---

*Phase created to address migration workflow bug identified in Marvin deployment*

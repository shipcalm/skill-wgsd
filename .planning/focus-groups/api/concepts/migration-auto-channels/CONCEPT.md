# Concept: Migration Auto-Channel Creation

**Status:** Draft  
**Focus Group:** api  
**Created:** 2026-02-23  
**Priority:** High  

---

## Problem

The WGSD v2.2 migration workflow currently only provides manual instructions to create Slack channels after migration:

```
📝 Next Steps:
   1. Create Slack channel: #${STUB}-dev
   2. Create focus group channels: #${STUB}-fg-*
```

But migration should automatically create these channels as part of the process, not require manual steps.

---

## Solution

Modify `workflows/migrate.md` to include automatic channel creation:

1. **Step 11: Create Slack Channels**
   - Call `create-channel` workflow for dev channel  
   - Call `create-channel` workflow for each focus group
   - Call `create-channel` workflow for active implementations
   - Invite team members automatically
   - Configure mention requirements

2. **Integration Points**
   - After Step 10 (Commit Migration)
   - Before Step 12 (Success Report)
   - Use existing `create-channel.md` workflow

---

## Acceptance Criteria

- [ ] Migration automatically creates `#${stub}-dev` channel
- [ ] Migration automatically creates `#${stub}-fg-*` channels for each focus group
- [ ] Migration automatically creates `#${stub}-impl-*` channels for active implementations  
- [ ] Team members are automatically invited to relevant channels
- [ ] Mention requirements are configured correctly (dev/impl = no mention, fg = mention required)
- [ ] Channel creation failures are handled gracefully with rollback
- [ ] Success report shows created channels with IDs

---

## Technical Notes

The migration workflow has all necessary information:
- `$STUB` for channel naming
- `$UNIQUE_FGS` for focus group list  
- `$IMPL_NAME` for active implementation
- User IDs are available in WGSD config

---

## Impact Assessment

- **User Experience:** ✅ Eliminates manual post-migration steps
- **Developer Experience:** ✅ Complete one-step migration
- **Risk:** 🟡 Medium (channel creation can fail, needs error handling)

---

*Bug identified during Marvin WGSD v2.2 migration - 2026-02-23*

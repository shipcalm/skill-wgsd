# Concept: Migration Auto-Canvas Creation

**Status:** Draft  
**Focus Group:** api  
**Created:** 2026-02-23  
**Priority:** High  

---

## Problem

The WGSD v2.2 migration workflow does not create Slack canvases automatically. Users must manually run canvas creation commands after migration.

However, canvases are essential for the WGSD workflow:
- **Master dashboard** in dev channel (aggregated roadmap)
- **Focus group canvases** in fg channels (roadmaps per focus group)  
- **Implementation canvases** in impl channels (active work tracking)

Without these canvases, the collaborative visual aspect of WGSD is missing.

---

## Solution

Modify `workflows/migrate.md` to include automatic canvas creation:

1. **Step 12: Create Slack Canvases** (after channel creation)
   - Call `canvas-create.md` workflow with `setup-all-canvases`
   - Create master dashboard in dev channel
   - Create focus group canvases for each fg channel
   - Create implementation canvas for active impl
   - Handle canvas creation failures gracefully

2. **Integration Points**  
   - After Step 11 (Create Slack Channels)
   - Before final Success Report
   - Use existing `canvas-create.md` workflows

---

## Acceptance Criteria

- [ ] Migration automatically creates master dashboard canvas in dev channel
- [ ] Migration automatically creates focus group canvases in each fg channel  
- [ ] Migration automatically creates implementation canvas for active impl
- [ ] Canvases are populated with current roadmap data from `.planning/`
- [ ] Canvas creation failures are handled gracefully (warn but don't fail migration)
- [ ] Canvas registry is created and populated automatically
- [ ] Success report shows created canvases with IDs and links

---

## Technical Implementation

1. **Add canvas creation step to migration:**
   ```bash
   # Step 12: Create Slack Canvases
   if command -v wgsd &> /dev/null; then
     echo "📊 Creating WGSD canvases..."
     wgsd setup-canvases "$PROJECT_NAME" || echo "⚠️ Canvas creation failed (non-fatal)"
   fi
   ```

2. **Dependencies:**
   - Channels must exist first (Step 11)
   - WGSD config must be created (Step 5)
   - Focus groups must be established (Step 4)

---

## Impact Assessment

- **User Experience:** ✅ Complete visual workflow immediately available
- **Developer Experience:** ✅ No manual post-migration canvas setup
- **Team Collaboration:** ✅ Roadmaps visible in Slack from day one
- **Risk:** 🟡 Medium (canvas creation can fail, should be non-fatal)

---

## Alternative Approaches

1. **Lazy Creation:** Create canvases on first use (more complex)
2. **Separate Command:** Keep manual `wgsd setup-canvases` (current state)
3. **Auto Creation:** Include in migration (recommended)

---

*Bug identified during Marvin WGSD v2.2 migration - 2026-02-23*

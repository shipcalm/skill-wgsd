# Phase 8: Slack Channel Automation - Research

**Phase:** 8 - Slack Channel Automation
**Research Date:** 2026-02-23
**Focus:** Integrating automated Slack channel creation into migration workflow

---

## Current State Analysis

### Existing Workflows

| Workflow | Purpose | Called By | Creates Channels? |
|----------|---------|-----------|-------------------|
| `migrate-planning.md` | GSD → WGSD migration | User | ❌ No |
| `setup-core-channels.md` | Create dev + community | `init.md` | ✅ Yes |
| `create-focus-group.md` | Create focus group channel | User | ✅ Yes |

### Gap Identified

The migration workflow (`migrate-planning.md`) completes file transformations but leaves channel creation as a manual "Next Steps" task:

```bash
# Current Step 12 output:
echo "Next Steps:"
echo "  1. Review WGSD-CONFIG.md and update stub if needed"
echo "  2. Create Slack channel: #<stub>-dev"  # ← Manual!
echo "  3. Create first focus group: wgsd create-focus-group <name>"
```

Users expect migration to be a **complete operation** — all channels should exist when it finishes.

---

## Existing Slack API Capabilities

### lib/slack-api.md Provides

| Function | Description | Ready? |
|----------|-------------|--------|
| `slack_get_token()` | Get bot token from OpenClaw config | ✅ |
| `slack_create_channel()` | Create private/public channel | ✅ |
| `slack_set_topic()` | Set channel topic | ✅ |
| `slack_set_purpose()` | Set channel description | ✅ |
| `slack_create_canvas()` | Create canvas in channel | ✅ |
| `slack_find_channel()` | Find existing channel by name | ✅ |
| `openclaw_register_channel()` | Register with OpenClaw | ✅ |
| `get_channel_privacy()` | Determine privacy by type | ✅ |
| `get_channel_topic()` | Generate standard topic | ✅ |
| `get_channel_purpose()` | Generate standard purpose | ✅ |

All required Slack API functions already exist!

### setup-core-channels.md Provides

- Private dev channel creation with topic/purpose
- Public community channel creation with topic/purpose
- Channel registry updates
- OpenClaw registration patch generation
- Idempotent handling (name_taken → find existing)

### create-focus-group.md Provides

- Focus group channel creation
- Canvas with roadmap
- Git branch/worktree setup
- Welcome message

---

## Integration Strategy

### Option A: Call Existing Workflows (Recommended)
**Pros:** Reuse tested code, DRY principle, consistent behavior
**Cons:** Multiple subprocess calls

```bash
# In migrate-planning.md:
/wgsd setup-core-channels "$STUB"
for fg in "${FOCUS_GROUPS[@]}"; do
  /wgsd create-focus-group "$fg" --minimal
done
```

### Option B: Inline Channel Creation
**Pros:** Single workflow, no dependencies
**Cons:** Code duplication, inconsistent if originals change

### Option C: Hybrid - Inline Core, Call Focus Groups
**Pros:** Core channels are simple, focus groups are complex
**Cons:** Mixed approaches

**Recommendation:** Option A with graceful error handling

---

## Channel Creation Order

1. **Core Dev Channel** (`{stub}-dev`)
   - Must exist first — it's the master coordination channel
   - Private, contains master dashboard canvas
   
2. **Community Channel** (`{stub}-community`)
   - Public channel for external feedback
   - Can exist independently

3. **Focus Group Channels** (`{stub}-fg-{name}`)
   - One per suggested focus group from migration analysis
   - Each gets roadmap canvas synced with git

---

## Error Handling Strategy

### Channel Already Exists
- **Current behavior:** `name_taken` error returned
- **Desired behavior:** Find existing ID, continue silently
- **Implementation:** Use `slack_find_channel()` fallback

### API Rate Limits
- Slack limit: ~50 requests/minute for channel creation
- Migration typically creates 3-6 channels
- **Mitigation:** Not needed for typical use, but add 1s delay between calls

### Partial Failure
- **Scenario:** Dev created, community fails
- **Strategy:** Log failures, continue with remaining, report at end
- **Rollback:** Archive created channels on total failure (optional)

### Token Missing/Invalid
- **Detection:** Check token before any channel operations
- **Response:** Fail fast with clear instructions

---

## Focus Group Detection from Migration

Migration analyzer (updated in Phase 7) now outputs:

```yaml
suggested_focus_groups:
  - name: core
    concepts: [concept-1, concept-2]
  - name: migration  
    concepts: [concept-3]
```

These should drive automatic focus group channel creation during migration.

---

## Integration Points in migrate-planning.md

| Step | Current | Phase 8 Addition |
|------|---------|------------------|
| 7.5 (NEW) | — | Validate Slack token |
| 8 | Create WGSD-CONFIG.md | Add channel IDs to config |
| 10 (NEW) | — | Create dev + community channels |
| 11 (NEW) | — | Create focus group channels |
| 12 | Validate migration | Validate channels exist |
| 14 | Success report | Include channel URLs |

---

## Best Practices from Industry Research

### Slack Workspace Setup Automation

1. **Preflight Check** — Verify bot permissions before starting
2. **Idempotent Operations** — Handle existing resources gracefully
3. **Logging** — Track all API calls and responses
4. **Summary Report** — Show what was created vs. what existed

### Channel Naming Consistency

Already enforced by WGSD naming convention:
- `{stub}-dev` (private)
- `{stub}-community` (public)
- `{stub}-fg-{name}` (private)
- `{stub}-cpt-{name}` (private)
- `{stub}-impl-{name}` (private)

### Canvas Initialization

Each channel type gets appropriate canvas:
- Dev → Master Dashboard
- Focus Group → Focus Group Roadmap
- Community → Public Roadmap (subset)

---

## Technical Considerations

### Stub Detection

Migration already detects stub in Step 8:
```bash
PROJECT_NAME=$(basename "$(pwd)")
SUGGESTED_STUB=$(echo "$PROJECT_NAME" | tr '[:upper:]' '[:lower:]' | tr -cd 'a-z' | cut -c1-4)
```

Should allow user override before channel creation.

### Focus Group Names from Analysis

Phase 7 added concept clustering. Extract focus group names:
```bash
FOCUS_GROUPS=$(grep -oP '(?<=name: )[a-z-]+' analysis_output.yaml)
```

### Canvas Content from Git

Focus group canvases should pull from `.planning/focus-groups/{fg}/ROADMAP.md`
(created earlier in migration).

---

## Conclusion

Phase 8 is well-positioned:
- All Slack API functions exist
- Channel creation workflows exist
- Migration just needs to **call them at the right time**

The main work is integration:
1. Add Slack token validation step
2. Call `setup-core-channels` after config creation
3. Loop through focus groups and create channels
4. Update success report with channel links

Estimated complexity: **Medium** (integration work, not new functionality)

---

*Research complete — ready for execution planning*

# Phase 9: Approval Workflow - Research

**Phase:** 9 - Approval Workflow (FINAL)
**Requirements:** APPROVE-01 through APPROVE-04
**Research Date:** 2026-02-23

---

## Objective

Research approval UX patterns to design an intuitive preview and approval workflow for WGSD migration.

---

## Current Migration Flow Analysis

### Current Steps (migrate-planning.md)

| Step | Action | Type |
|------|--------|------|
| 1 | Detect GSD Structure | Analysis |
| 2 | Create Backup | Prep |
| 3 | Validate Slack Connectivity | Analysis |
| 4 | Create WGSD Directory Structure | **Execution** |
| 5 | Transform PROJECT.md | **Execution** |
| 6 | Transform ROADMAP.md | **Execution** |
| 7 | Transform Phases | **Execution** |
| 8 | Analyze REQUIREMENTS.md | Analysis |
| 9 | Generate WGSD-CONFIG.md | **Execution** |
| 10 | Create Core Slack Channels | **Execution** |
| 11 | Create Focus Group Channels | **Execution** |
| 12 | Update STATE.md | **Execution** |
| 13 | Validate Migration | Verification |
| 14 | Commit Migration | **Execution** |
| 15 | Report Success | Reporting |

### Problem

Execution steps (4-12, 14) run immediately after analysis. User has no opportunity to:
- Review what will be created
- Modify suggestions
- Cancel if something looks wrong

---

## Approval UX Patterns Research

### Pattern 1: Preview → Confirm (Git/GitHub)

**Example:** `git commit --dry-run`, GitHub PR merge preview

**Flow:**
1. Show diff/changes that would happen
2. Single confirm button
3. Execute on confirm

**Pros:** Simple, familiar
**Cons:** No modification option

---

### Pattern 2: Wizard with Review Step (Installers)

**Example:** Software installers, setup wizards

**Flow:**
1. Gather choices through multiple steps
2. Show summary before final step
3. "Back" button to modify
4. "Install" to proceed

**Pros:** Familiar, allows modification
**Cons:** More complex to implement

---

### Pattern 3: Interactive Confirmation (Terraform)

**Example:** `terraform plan` → `terraform apply`

**Flow:**
1. Generate detailed plan with exact changes
2. Show plan to user
3. User types "yes" to confirm or modifies .tf files
4. Re-run plan if modified

**Pros:** Very explicit, promotes review
**Cons:** Requires separate commands

---

### Pattern 4: Inline Editing Preview (Forms)

**Example:** Form previews with edit buttons

**Flow:**
1. Show preview of all data
2. Each section has "Edit" button
3. Click to modify in-place
4. "Submit" when satisfied

**Pros:** Easy modification, single flow
**Cons:** Complex UI for CLI

---

### Pattern 5: YAML/Config Preview (Kubernetes/Helm)

**Example:** `helm install --dry-run --debug`

**Flow:**
1. Generate full config that would be applied
2. Show config to user
3. User can save config, edit, and re-apply
4. Execute with final config

**Pros:** Full transparency
**Cons:** Technical, not beginner-friendly

---

## Recommended Pattern: Hybrid Terraform + Wizard

Combine the best of patterns 2 and 3:

1. **Analysis Phase** - Run all detection and analysis
2. **Preview Display** - Show comprehensive plan (like Terraform)
3. **Modification Prompt** - Offer to modify specific items
4. **Explicit Approval** - Require "yes" to proceed
5. **Execution Phase** - Run actual migration

### Preview Format Design

```
═══════════════════════════════════════════════════════════════
📋 MIGRATION PREVIEW
═══════════════════════════════════════════════════════════════

📁 Project: skill-wgsd
📍 Source: GSD (.planning/)

─────────────────────────────────────────────────────────────────
🎯 FOCUS GROUPS TO CREATE (3)
─────────────────────────────────────────────────────────────────
  [1] security      ← Requirements analysis (auth, permission keywords)
  [2] infrastructure ← Requirements analysis (devops, deploy keywords)
  [3] api           ← Requirements analysis (webhook, endpoint keywords)

─────────────────────────────────────────────────────────────────
💡 CONCEPTS TO CREATE (5)
─────────────────────────────────────────────────────────────────
  [1] auth-v2          → security         ← Phase 3: Auth Improvements
  [2] webhook-system   → api              ← Phase 4: Webhook Integration
  [3] ci-pipeline      → infrastructure   ← Phase 5: CI/CD Setup
  [4] rate-limiting    → security         ← Phase 6: Security Hardening
  [5] cache-layer      → infrastructure   ← Phase 7: Performance

─────────────────────────────────────────────────────────────────
📱 SLACK CHANNELS TO CREATE
─────────────────────────────────────────────────────────────────
  ✅ Slack connected (bot: @jarvis)
  
  Core:
    • #wgsd-dev          (private) - Main development
    • #wgsd-community    (public)  - Community feedback
  
  Focus Groups:
    • #wgsd-fg-security       (private)
    • #wgsd-fg-infrastructure (private)
    • #wgsd-fg-api            (private)

─────────────────────────────────────────────────────────────────
📄 FILES TO TRANSFORM
─────────────────────────────────────────────────────────────────
  • PROJECT.md      → Add WGSD workflow section
  • ROADMAP.md      → Create MASTER-ROADMAP.md
  • STATE.md        → Add WGSD status section
  • (new) WGSD-CONFIG.md created
  • phases/         → focus-groups/ (preserve content)

═══════════════════════════════════════════════════════════════
```

### Modification Options

After preview, offer:

```
─────────────────────────────────────────────────────────────────
⚙️  OPTIONS
─────────────────────────────────────────────────────────────────

  [Y] Approve and proceed with migration
  [N] Cancel migration
  [E] Edit suggestions:
      • rename-fg <num> <new-name>  - Rename focus group
      • remove-fg <num>             - Remove focus group
      • move-concept <num> <fg>     - Move concept to different FG
      • exclude-phase <num>         - Exclude phase from migration
      • add-fg <name>               - Add custom focus group
      
Enter choice (Y/N/E):
```

### Modification Loop

If user chooses `E`:
1. Accept modification command
2. Update preview data structure
3. Re-display preview with changes highlighted
4. Return to options prompt

---

## Integration Strategy

### Where to Insert Approval

Insert between analysis and execution:

```
CURRENT:                          PROPOSED:
─────────────────                 ─────────────────
Step 1: Detect                    Step 1: Detect
Step 2: Backup                    Step 2: Backup  
Step 3: Slack Check               Step 3: Slack Check
                                  Step 4: Full Analysis
Step 4: Create dirs       ───►    Step 5: Generate Preview
Step 5: Transform files           Step 6: Display Preview
...                               Step 7: Approval Loop
Step 8: Analyze reqs              Step 8: (if approved) Execute
...                               ...
```

### Data Flow

```
Analysis Phase:
  └─► Collect: focus_groups[], concepts[], channels[], files_to_transform[]
  └─► Store in bash arrays/variables

Preview Phase:
  └─► Read collected data
  └─► Format and display

Modification Phase:
  └─► Parse user commands
  └─► Update data arrays
  └─► Re-render preview

Execution Phase:
  └─► Read final approved data
  └─► Create only approved focus groups
  └─► Create only approved concepts
  └─► Create only approved channels
```

---

## Key Design Decisions

### D1: Preview Before Backup?

**Decision:** No - backup should happen before analysis in case analysis fails.

### D2: How to Handle Modification State?

**Decision:** Use bash arrays and temp files:
```bash
# Store suggestions in arrays
FOCUS_GROUPS=("security" "api" "infrastructure")
CONCEPTS_DATA=("auth-v2:security:Phase 3" "webhook:api:Phase 4")

# Modification updates arrays in place
# Preview re-renders from arrays
```

### D3: What if User Modifies and Slack Fails?

**Decision:** Slack creation happens after approval. If it fails, migration still succeeds (channels can be created manually later). Preview shows Slack status but doesn't block approval.

### D4: How Explicit Should Approval Be?

**Decision:** Require exact `yes` input (not just `y`), matching Terraform pattern:
```
Type 'yes' to approve and proceed with migration: 
```

---

## Implementation Considerations

### Bash Limitations

- No native JSON handling → use arrays and string parsing
- No rich TUI → use box-drawing characters for formatting
- Loop handling → use while loops with read

### AskUser Integration

The workflow uses `AskUser` tool which maps to agent prompting user. We can:
1. Display preview as formatted text output
2. Use AskUser to get approval choice
3. Loop back if modification requested

---

## Success Criteria for Phase 9

- [ ] Preview shows all focus groups, concepts, channels clearly
- [ ] Preview shows mapping (Phase X → Concept Y → Focus Group Z)
- [ ] User must type 'yes' to proceed
- [ ] User can cancel cleanly with 'no'
- [ ] User can modify suggestions before approval
- [ ] Modifications update preview in real-time
- [ ] Only approved items are created
- [ ] Seamless integration with existing workflow

---

## References

- Terraform plan/apply pattern: https://developer.hashicorp.com/terraform/cli/commands/apply
- GitHub PR merge preview UX
- Software installer wizard patterns
- WGSD Phase 7-8 artifacts for context

---

*Research complete - ready for execution plan*

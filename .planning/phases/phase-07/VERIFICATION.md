# Phase 7: Migration Logic Fix - Verification Plan

**Phase:** 7 - Migration Logic Fix
**Verification Date:** TBD
**Verifier:** TBD

---

## Verification Strategy

Phase 7 fixes semantic mapping issues. Verification focuses on:
1. Correct output terminology (Concepts, not Focus Groups from phases)
2. Proper domain clustering
3. Correct file structure generation

---

## Test Scenarios

### Scenario 1: Fresh GSD Project Analysis

**Setup:**
```bash
# Create test GSD project with 4 phases
mkdir -p /tmp/test-gsd/.planning
cat > /tmp/test-gsd/.planning/ROADMAP.md << 'EOF'
# Project Roadmap

## Phase 1: Foundation Setup
Core infrastructure and project scaffolding.

## Phase 2: Authentication System
User auth with OAuth and session management.

## Phase 3: API Development
REST API endpoints with versioning.

## Phase 4: Dashboard UI
Admin dashboard with React components.
EOF
```

**Test:**
```bash
wgsd analyze /tmp/test-gsd
```

**Expected Output:**
```
📊 Extracting Concepts from Phases...

📄 Found 4 phases:
   • foundation-setup ← Phase 1: Foundation Setup
   • authentication-system ← Phase 2: Authentication System
   • api-development ← Phase 3: API Development
   • dashboard-ui ← Phase 4: Dashboard UI

📂 Suggested Focus Groups (3):
   • core (1 concept: foundation-setup)
   • security (1 concept: authentication-system)
   • api (1 concept: api-development)
   • frontend (1 concept: dashboard-ui)

ANALYSIS_RESULT:
  suggested_concepts: [foundation-setup, authentication-system, api-development, dashboard-ui]
  suggested_focus_groups: [core, security, api, frontend]
```

**Verification Checklist:**
- [ ] Output shows "Concepts" (not "Focus Groups from phases")
- [ ] 4 concepts created from 4 phases
- [ ] Focus groups are domain-based (core, security, api, frontend)
- [ ] NOT 4 separate focus groups matching phase names

---

### Scenario 2: Related Phases Clustering

**Setup:**
```bash
# Create project with related phases
cat > /tmp/test-gsd-cluster/.planning/ROADMAP.md << 'EOF'
# Project Roadmap

## Phase 1: User Registration
Sign up flow with email verification.

## Phase 2: Login System
Login with password and social auth.

## Phase 3: Password Reset
Forgot password flow with token expiry.

## Phase 4: Session Management
Session tokens, refresh, and logout.
EOF
```

**Test:**
```bash
wgsd analyze /tmp/test-gsd-cluster
```

**Expected Output:**
```
📂 Suggested Focus Groups (1):
   • security (4 concepts)
     - user-registration
     - login-system
     - password-reset
     - session-management
```

**Verification Checklist:**
- [ ] All 4 phases cluster into ONE focus group ("security")
- [ ] NOT 4 separate focus groups
- [ ] Reasoning mentions "auth/security domain"

---

### Scenario 3: Full Migration

**Setup:**
Use same test project from Scenario 1.

**Test:**
```bash
wgsd migrate /tmp/test-gsd --stub test
```

**Expected File Structure:**
```
/tmp/test-gsd/.planning/
├── WGSD-CONFIG.md
├── MASTER-ROADMAP.md
├── focus-groups/
│   ├── core/
│   │   ├── STATE.md
│   │   ├── ROADMAP.md
│   │   └── concepts/
│   │       └── foundation-setup.md
│   ├── security/
│   │   └── concepts/
│   │       └── authentication-system.md
│   ├── api/
│   │   └── concepts/
│   │       └── api-development.md
│   └── frontend/
│       └── concepts/
│           └── dashboard-ui.md
└── active-implementations/
```

**Verification Checklist:**
- [ ] Concepts created as `.md` files in `concepts/` subdirectory
- [ ] Each concept file references source phase
- [ ] Focus groups contain concepts (not phases)
- [ ] WGSD-CONFIG.md lists both concepts AND focus groups

---

### Scenario 4: Concept File Format

**Test:** Read generated concept file

**Expected Content:**
```markdown
# Concept: Foundation Setup

**Status:** Migrated
**Focus Group:** core
**Source:** GSD Phase 1
**Migrated:** 2026-02-23

---

## Description

Core infrastructure and project scaffolding.

---

## Acceptance Criteria

*Imported from Phase requirements, if available*

---

## Implementation Notes

*To be populated during concept development*

---

*Concept migrated from GSD Phase 1*
```

**Verification Checklist:**
- [ ] File named `{slug}.md` (not phase number)
- [ ] Contains source phase reference
- [ ] Status shows "Migrated" (not Draft)
- [ ] Assigned to correct focus group

---

### Scenario 5: WGSD-CONFIG.md Format

**Test:** Read generated config file

**Expected Content:**
```markdown
## Concepts (Migrated from Phases)

| Concept | Source | Focus Group |
|---------|--------|-------------|
| foundation-setup | Phase 1 | core |
| authentication-system | Phase 2 | security |
| api-development | Phase 3 | api |
| dashboard-ui | Phase 4 | frontend |

## Focus Groups

| Name | Status | Concepts | Channel |
|------|--------|----------|---------|
| core | Active | 1 | #test-fg-core |
| security | Active | 1 | #test-fg-security |
| api | Active | 1 | #test-fg-api |
| frontend | Active | 1 | #test-fg-frontend |
```

**Verification Checklist:**
- [ ] Concepts section exists
- [ ] Each concept shows source Phase
- [ ] Focus Groups section shows concept counts
- [ ] No "phases" section (phases are now concepts)

---

## Regression Tests

### Regression 1: Empty Roadmap

**Test:** Project with no ROADMAP.md

**Expected:**
- No concepts suggested
- No focus groups suggested
- Migration still succeeds with default structure

---

### Regression 2: Single Phase

**Test:** Project with only 1 phase

**Expected:**
- 1 concept created
- 1 focus group suggested (or fallback to "core")
- No errors about missing focus groups

---

### Regression 3: Existing WGSD Project

**Test:** Run analyze on already-migrated WGSD project

**Expected:**
- Detects existing WGSD-CONFIG.md
- Warns user about re-migration
- Does not corrupt existing structure

---

## Acceptance Criteria Matrix

| Requirement | Criterion | Verification Method |
|-------------|-----------|---------------------|
| MIG-FIX-01 | migrate.md maps Phase → Concept | Scenario 3: Check structure |
| MIG-FIX-01 | Focus Groups are thematic groupings | Scenario 2: Cluster test |
| MIG-FIX-02 | Analyzer outputs `suggested_concepts[]` | Scenario 1: Check output |
| MIG-FIX-02 | Analyzer suggests Focus Groups separately | Scenario 1: Check format |
| MIG-FIX-03 | Phase description → Concept description | Scenario 4: Check content |
| MIG-FIX-03 | Phase requirements → Concept criteria | Scenario 4: Check criteria |
| MIG-FIX-03 | Concepts assigned to Focus Groups | Scenario 5: Check config |

---

## Manual Verification Script

```bash
#!/bin/bash
# Phase 7 verification script

echo "=== Phase 7 Verification ==="
echo ""

# Setup
rm -rf /tmp/test-gsd
mkdir -p /tmp/test-gsd/.planning
cat > /tmp/test-gsd/.planning/ROADMAP.md << 'EOF'
# Roadmap
## Phase 1: Foundation
Core setup.
## Phase 2: Authentication  
User auth.
## Phase 3: API
REST endpoints.
## Phase 4: UI
Dashboard.
EOF

# Test 1: Analysis
echo "Test 1: Analysis Output"
wgsd analyze /tmp/test-gsd 2>&1 | tee /tmp/analyze-output.txt

# Check for correct terminology
if grep -q "Suggested Concepts" /tmp/analyze-output.txt; then
  echo "✅ Uses 'Concepts' terminology"
else
  echo "❌ FAIL: Still uses 'Focus Groups' for phases"
fi

# Test 2: Migration  
echo ""
echo "Test 2: Migration Structure"
wgsd migrate /tmp/test-gsd --stub test --yes

# Check structure
if [ -d "/tmp/test-gsd/.planning/focus-groups/core/concepts" ]; then
  echo "✅ Concept directories created"
else
  echo "❌ FAIL: Missing concept directories"
fi

# Check concept files
if [ -f "/tmp/test-gsd/.planning/focus-groups/*/concepts/*.md" ]; then
  echo "✅ Concept files created"
else
  echo "❌ FAIL: Missing concept files"
fi

echo ""
echo "=== Verification Complete ==="
```

---

## Post-Verification

After all tests pass:

1. [ ] Update STATE.md → Phase 7 status to "✅ Complete"
2. [ ] Commit changes with message: `fix: correct Phase→Concept mapping in migration (MIG-FIX-01/02/03)`
3. [ ] Proceed to Phase 8 or 9

---

*Verification plan complete — ready for implementation*

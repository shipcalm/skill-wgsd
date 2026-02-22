# Plan 1.3: Branch Strategy & Planning Migration

**Requirements:** INTEGRATE-02, INTEGRATE-03
**Estimated Duration:** 0.5 day
**Dependencies:** Plan 1.1 (workspace management)

---

## Objective

Implement branch strategy enforcement for WGSD workspaces and create the GSD → WGSD planning structure migration tools.

---

## Requirements Addressed

### INTEGRATE-02: Branch strategy enforcement (develop base, clean checkouts)
- Ensure clean develop branch as integration point
- Enforce worktree-based branching for focus groups
- Validate clean state before operations
- Support both main and develop as primary branch

### INTEGRATE-03: .planning/ structure migration from GSD to WGSD format
- Detect existing GSD planning structure
- Transform phases → focus groups
- Preserve all planning content
- Create WGSD-specific configuration files

---

## Branch Strategy Specification

### Branch Hierarchy

```
main/develop (primary)
├── focus-groups/
│   ├── security        ← Planning work (long-lived)
│   ├── onboarding      ← Planning work (long-lived)
│   └── billing         ← Planning work (long-lived)
└── implementations/
    ├── auth-v2         ← Code execution (short-lived)
    └── payment-api     ← Code execution (short-lived)
```

### Branch Rules

| Branch Type | Base | Merge Target | Lifetime | Clean Checkout Required |
|-------------|------|--------------|----------|------------------------|
| Primary (main/develop) | - | - | Permanent | Yes |
| focus-groups/{name} | Primary | None (planning only) | Long-lived | No |
| implementations/{name} | Primary | Primary | 1-3 days | Yes |

### Worktree Strategy

| Path | Branch | Purpose |
|------|--------|---------|
| `{workspace}/` | Primary | Main checkout |
| `{workspace}/concepts/{fg}/` | focus-groups/{fg} | Focus group planning |
| `{workspace}/implementations/{name}/` | implementations/{name} | Code execution |

---

## Planning Structure Migration

### GSD Structure Detection

```
.planning/
├── PROJECT.md            ✓ Exists in GSD
├── ROADMAP.md            ✓ Exists in GSD  
├── REQUIREMENTS.md       ✓ Exists in GSD
├── STATE.md              ✓ Exists in GSD
├── phases/               ✓ Exists in GSD (if phased)
│   ├── phase-01/
│   └── phase-02/
└── codebase/             ✓ Optional in GSD
```

### WGSD Target Structure

```
.planning/
├── PROJECT.md            ← Enhanced with WGSD context
├── MASTER-ROADMAP.md     ← Aggregated roadmap (new)
├── STATE.md              ← Enhanced with focus group status
├── WGSD-CONFIG.md        ← WGSD configuration (new)
├── WORKSPACES.md         ← Workspace registry (new)
├── focus-groups/         ← Converted from phases/requirements
│   ├── {fg-1}/
│   │   ├── ROADMAP.md
│   │   ├── STATE.md
│   │   ├── CHANNELS.md
│   │   └── concepts/
│   └── {fg-2}/
├── active-implementations/ ← Empty initially
└── codebase/             ← Preserved if exists
```

### Migration Mapping

| GSD Source | WGSD Target | Transformation |
|------------|-------------|----------------|
| PROJECT.md | PROJECT.md | Add WGSD workflow section |
| ROADMAP.md | MASTER-ROADMAP.md | Rename, add focus group refs |
| REQUIREMENTS.md | focus-groups/*/concepts/ | Split by domain |
| STATE.md | STATE.md | Add focus group status section |
| phases/* | focus-groups/* | Rename, restructure |
| codebase/* | codebase/* | Preserve unchanged |

---

## Deliverables

### 1. Library: `lib/branch-ops.md`

**Functions:**

```
detect_primary_branch()     → main or develop
ensure_clean_checkout()     → OK or error with dirty files
create_focus_branch(name)   → Create focus-groups/{name} branch
create_impl_branch(name)    → Create implementations/{name} branch
setup_worktree(path, branch) → Create worktree with validation
validate_branch_state()     → Check all branches are clean
```

### 2. Workflow: `workflows/migrate-planning.md`

**Purpose:** Migrate GSD planning structure to WGSD format

**Process:**
```
1. Detect existing GSD structure
2. Backup current .planning/
3. Create WGSD directory structure
4. Transform and copy content
5. Generate WGSD configuration files
6. Validate migration completeness
7. Commit changes
```

### 3. Workflow Update: `workflows/init.md`

Add branch setup step:
```
1. Detect or select primary branch
2. Ensure clean checkout
3. Create base branches (focus-groups/, implementations/)
4. Set up initial worktree structure
```

### 4. Agent: `agents/planning-migrator.md`

**Purpose:** Intelligent content transformation during migration

**Capabilities:**
- Analyze REQUIREMENTS.md to suggest focus groups
- Split requirements by domain
- Generate ROADMAP.md summaries
- Preserve all original content

---

## Implementation Steps

### Step 1: Create branch-ops library (30 min)

Location: `workflows/lib/branch-ops.md`

```markdown
---
name: wgsd:lib:branch-ops
description: Branch strategy enforcement library
---

## detect_primary_branch

<operation>
<process>
# Check for develop first (preferred for WGSD)
if git rev-parse --verify origin/develop >/dev/null 2>&1; then
  echo "develop"
elif git rev-parse --verify origin/main >/dev/null 2>&1; then
  echo "main"
elif git rev-parse --verify develop >/dev/null 2>&1; then
  echo "develop"
elif git rev-parse --verify main >/dev/null 2>&1; then
  echo "main"
else
  echo "ERROR: No primary branch (main or develop) found"
  exit 1
fi
</process>
</operation>

## ensure_clean_checkout

<operation>
<input>branch (optional, default: current)</input>
<process>
# Check for uncommitted changes
if [ -n "$(git status --porcelain)" ]; then
  echo "ERROR: Uncommitted changes detected"
  echo "DIRTY_FILES:"
  git status --porcelain
  echo ""
  echo "ACTION_REQUIRED: Commit or stash changes before proceeding"
  exit 1
fi

# Check if we need to switch branches
if [ -n "$branch" ]; then
  CURRENT=$(git branch --show-current)
  if [ "$CURRENT" != "$branch" ]; then
    git checkout "$branch" 2>&1
    if [ $? -ne 0 ]; then
      echo "ERROR: Failed to checkout $branch"
      exit 1
    fi
  fi
fi

echo "OK: Clean checkout on $(git branch --show-current)"
</process>
</operation>

## create_focus_branch

<operation>
<input>name</input>
<process>
PRIMARY=$(detect_primary_branch)
BRANCH="focus-groups/$name"

# Check if branch already exists
if git rev-parse --verify "$BRANCH" >/dev/null 2>&1; then
  echo "EXISTS: Branch $BRANCH already exists"
  exit 0
fi

# Create branch from primary
git checkout "$PRIMARY"
git checkout -b "$BRANCH"

if [ $? -eq 0 ]; then
  echo "OK: Created branch $BRANCH from $PRIMARY"
else
  echo "ERROR: Failed to create branch $BRANCH"
  exit 1
fi
</process>
</operation>

## create_impl_branch

<operation>
<input>name</input>
<process>
PRIMARY=$(detect_primary_branch)
BRANCH="implementations/$name"

# Ensure clean checkout first
ensure_clean_checkout "$PRIMARY"

# Check if branch already exists
if git rev-parse --verify "$BRANCH" >/dev/null 2>&1; then
  echo "ERROR: Implementation branch $BRANCH already exists"
  exit 1
fi

# Create branch from primary
git checkout -b "$BRANCH" "$PRIMARY"

if [ $? -eq 0 ]; then
  echo "OK: Created implementation branch $BRANCH from $PRIMARY"
else
  echo "ERROR: Failed to create branch $BRANCH"
  exit 1
fi
</process>
</operation>

## setup_worktree

<operation>
<input>path, branch</input>
<process>
# Validate branch exists
if ! git rev-parse --verify "$branch" >/dev/null 2>&1; then
  echo "ERROR: Branch $branch does not exist"
  exit 1
fi

# Check if worktree already exists
if [ -d "$path" ]; then
  echo "EXISTS: Worktree at $path already exists"
  exit 0
fi

# Create worktree
git worktree add "$path" "$branch" 2>&1

if [ $? -eq 0 ]; then
  echo "OK: Created worktree at $path for $branch"
else
  echo "ERROR: Failed to create worktree"
  exit 1
fi
</process>
</operation>
```

### Step 2: Create planning migration workflow (45 min)

Location: `workflows/migrate-planning.md`

```markdown
---
name: wgsd:migrate-planning
description: Migrate GSD planning structure to WGSD format
allowed-tools:
  - Read
  - Write
  - Bash
---

<objective>
Transform existing GSD .planning/ structure to WGSD format while preserving all content.
</objective>

<process>

## Step 1: Detect GSD Structure

```bash
echo "🔍 Detecting GSD planning structure..."

GSD_FILES=""
[ -f .planning/PROJECT.md ] && GSD_FILES="$GSD_FILES PROJECT.md"
[ -f .planning/ROADMAP.md ] && GSD_FILES="$GSD_FILES ROADMAP.md"
[ -f .planning/REQUIREMENTS.md ] && GSD_FILES="$GSD_FILES REQUIREMENTS.md"
[ -f .planning/STATE.md ] && GSD_FILES="$GSD_FILES STATE.md"
[ -d .planning/phases ] && GSD_FILES="$GSD_FILES phases/"
[ -d .planning/codebase ] && GSD_FILES="$GSD_FILES codebase/"

if [ -z "$GSD_FILES" ]; then
  echo "⚠️  No GSD structure detected. Creating fresh WGSD structure."
else
  echo "✅ Found GSD files: $GSD_FILES"
fi
```

## Step 2: Create Backup

```bash
BACKUP_DIR=".planning-backup-$(date +%Y%m%d-%H%M%S)"
cp -r .planning "$BACKUP_DIR"
echo "✅ Backup created: $BACKUP_DIR"
```

## Step 3: Create WGSD Directory Structure

```bash
mkdir -p .planning/focus-groups
mkdir -p .planning/active-implementations
```

## Step 4: Transform Content

### PROJECT.md Enhancement

Read existing PROJECT.md and append WGSD workflow section:

```markdown
## WGSD Workflow

This project uses **WGSD (We Get Shit Done)** collaborative development:

1. **Focus Groups**: Long-lived topic discussions
2. **Concepts**: Feature ideas developed socially  
3. **Implementations**: Short-lived code execution

### Active Focus Groups

{List of focus groups - populated during migration}

### Current Implementations

{List of implementations - initially empty}
```

### ROADMAP.md → MASTER-ROADMAP.md

```bash
if [ -f .planning/ROADMAP.md ]; then
  # Rename to MASTER-ROADMAP.md
  mv .planning/ROADMAP.md .planning/MASTER-ROADMAP.md
  
  # Add WGSD header
  sed -i '1i # Master Roadmap\n\n**WGSD Project Roadmap - Aggregated from all focus groups**\n' .planning/MASTER-ROADMAP.md
fi
```

### REQUIREMENTS.md → Focus Groups

Invoke planning-migrator agent to:
1. Analyze REQUIREMENTS.md
2. Identify natural domain groupings
3. Create focus-groups/{domain}/ directories
4. Split requirements into concept files

## Step 5: Create WGSD Configuration Files

Create `.planning/WGSD-CONFIG.md`:

```markdown
# WGSD Configuration

**Migrated From:** GSD
**Migration Date:** {date}
**Slack Stub:** {stub}

## Channel Mappings

### Main Development
- **Channel:** #{stub}-dev

### Focus Groups
{Populated from migration analysis}

## Git Configuration

**Primary Branch:** {detected}
**Focus Groups Base:** focus-groups/
**Implementations Base:** implementations/

## Migration Notes

- Original planning backed up to: {backup_dir}
- Requirements split into {n} focus groups
- {n} concepts created from requirements
```

## Step 6: Validate Migration

```bash
echo "🔍 Validating migration..."

ERRORS=""

# Check required files exist
[ ! -f .planning/PROJECT.md ] && ERRORS="$ERRORS\n- Missing PROJECT.md"
[ ! -f .planning/WGSD-CONFIG.md ] && ERRORS="$ERRORS\n- Missing WGSD-CONFIG.md"
[ ! -d .planning/focus-groups ] && ERRORS="$ERRORS\n- Missing focus-groups/"

if [ -n "$ERRORS" ]; then
  echo "❌ Migration validation failed:"
  echo -e "$ERRORS"
  exit 1
fi

echo "✅ Migration validated successfully"
```

## Step 7: Commit Migration

```bash
git add .planning/
git commit -m "feat: migrate GSD planning to WGSD structure"
echo "✅ Migration committed"
```

</process>

<success_criteria>
- [ ] GSD structure detected and analyzed
- [ ] Backup created before any modifications
- [ ] WGSD directory structure created
- [ ] Content transformed and preserved
- [ ] Configuration files generated
- [ ] Migration validated
- [ ] Changes committed to git
</success_criteria>
```

### Step 3: Create planning migrator agent (30 min)

Location: `agents/planning-migrator.md`

```markdown
---
name: wgsd:planning-migrator
description: Agent for intelligent planning structure migration
---

<objective>
Analyze GSD planning files and intelligently transform them to WGSD structure.
</objective>

<capabilities>
- Analyze REQUIREMENTS.md to suggest focus group domains
- Split requirements by functional area
- Generate concept files from requirements
- Preserve all content and context
- Create meaningful focus group names
</capabilities>

<domain_detection>
Common domains to detect:
- Security/Auth → security focus group
- Onboarding/Setup → onboarding focus group
- Billing/Payments → billing focus group
- API/Integration → api focus group
- UI/Frontend → frontend focus group
- Infrastructure/DevOps → infrastructure focus group
</domain_detection>

<process>
1. Read REQUIREMENTS.md content
2. Identify domain keywords in each requirement
3. Group requirements by detected domain
4. For ungrouped requirements, create "general" focus group
5. Generate concept files for each requirement
6. Create focus group ROADMAP.md with concept list
</process>
```

### Step 4: Update init workflow with branch setup (20 min)

Add to `workflows/init.md`:

```markdown
## Step: Branch Strategy Setup

<process>
1. Detect primary branch (main/develop)
2. Ensure clean checkout on primary
3. Create focus-groups/base branch
4. Create implementations/base branch  
5. Push base branches to origin
6. Return to primary branch
</process>
```

---

## Verification Checklist

### Branch Operations Tests

- [ ] `detect_primary_branch` finds develop when present
- [ ] `detect_primary_branch` falls back to main
- [ ] `ensure_clean_checkout` blocks on dirty state
- [ ] `create_focus_branch` creates from primary
- [ ] `create_impl_branch` requires clean state
- [ ] `setup_worktree` creates correct directory

### Migration Tests

- [ ] Empty .planning/ gets fresh WGSD structure
- [ ] GSD with ROADMAP.md migrates correctly
- [ ] REQUIREMENTS.md splits into focus groups
- [ ] Backup created before modification
- [ ] All original content preserved
- [ ] WGSD-CONFIG.md generated correctly

### Integration Tests

- [ ] Full init → migrate workflow completes
- [ ] Resulting structure matches WGSD spec
- [ ] Git history shows clean migration commit

---

## Success Criteria

| Criterion | Measurement |
|-----------|-------------|
| Primary branch detected | Correct branch identified |
| Clean state enforced | Dirty state blocks operations |
| Branches created | focus-groups/ and implementations/ exist |
| Migration complete | All GSD content transformed |
| Content preserved | Zero data loss from original |
| Config generated | WGSD-CONFIG.md accurate |

---

## Rollback Plan

If migration fails:

1. Restore from backup directory
2. Remove partially created WGSD structure
3. Reset git to pre-migration state
4. Report error with specific failure point

```bash
# Rollback command
BACKUP_DIR=$(ls -d .planning-backup-* | tail -1)
rm -rf .planning
mv "$BACKUP_DIR" .planning
git checkout -- .planning/
echo "Rollback complete"
```

---

*Plan created: 2026-02-22*
*Ready for execution after Plan 1.1*

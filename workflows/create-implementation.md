---
name: wgsd:create-implementation
description: Create implementation from a mature concept (max 2-4 concurrent)
argument-hint: "[concept-name]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Message
  - Canvas
---

<objective>
Promote a mature concept to active implementation with:
- Dedicated implementation channel for focused execution
- Git branch and worktree for code development  
- Traditional GSD workflow (requirements → roadmap → phases)
- Implementation tracking and status reporting
- Enforce 2-4 concurrent implementation limit
</objective>

<process>

## Step 1: Validate Implementation Capacity

```bash
REPO_NAME=$(basename $(pwd))
CONCEPT_NAME="{concept-name}"

# Check current implementation count
CURRENT_IMPLEMENTATIONS=0
if [ -d .planning/active-implementations ]; then
    CURRENT_IMPLEMENTATIONS=$(find .planning/active-implementations -maxdepth 1 -type d | grep -v "^\\.planning/active-implementations$" | wc -l)
fi

MAX_IMPLEMENTATIONS=4

if [ $CURRENT_IMPLEMENTATIONS -ge $MAX_IMPLEMENTATIONS ]; then
    echo "❌ Implementation capacity reached (${CURRENT_IMPLEMENTATIONS}/${MAX_IMPLEMENTATIONS})"
    echo "💡 Complete existing implementations before starting new ones:"
    find .planning/active-implementations -maxdepth 1 -type d | grep -v "^\\.planning/active-implementations$" | sed 's|.planning/active-implementations/||'
    exit 1
fi

echo "✅ Implementation slots available: $((MAX_IMPLEMENTATIONS - CURRENT_IMPLEMENTATIONS)) remaining"
```

## Step 2: Find and Validate Concept

```bash
# Search for concept across all focus groups
CONCEPT_FILE=""
FOCUS_GROUP=""

for fg_dir in .planning/focus-groups/*/; do
    if [ -f "${fg_dir}concepts/${CONCEPT_NAME}.md" ]; then
        CONCEPT_FILE="${fg_dir}concepts/${CONCEPT_NAME}.md"
        FOCUS_GROUP=$(basename "$fg_dir")
        echo "📍 Found concept: ${CONCEPT_NAME} in ${FOCUS_GROUP} focus group"
        break
    fi
done

if [ -z "$CONCEPT_FILE" ]; then
    echo "❌ Concept '${CONCEPT_NAME}' not found in any focus group"
    echo "💡 Available concepts:"
    find .planning/focus-groups -name "*.md" -path "*/concepts/*" | sed 's|.planning/focus-groups/||;s|/concepts/| → |;s|\.md||'
    exit 1
fi

# Check if concept is ready for implementation
if ! grep -q "🚀 Ready for Implementation" "$CONCEPT_FILE"; then
    echo "⚠️  Concept '${CONCEPT_NAME}' is not marked as ready for implementation"
    echo "📄 Current status in file: $(grep "Status:" "$CONCEPT_FILE")"
    echo "💡 Update concept status to '🚀 Ready for Implementation' first"
    exit 1
fi

echo "✅ Concept validated and ready for implementation"
```

## Step 3: Create Implementation Structure

```bash
IMPL_NAME="${CONCEPT_NAME}-impl"
mkdir -p .planning/active-implementations/${IMPL_NAME}

echo "🏗️  Creating implementation: ${IMPL_NAME}"
echo "📱 Channel: #${REPO_NAME}-impl-${CONCEPT_NAME}"
```

## Step 4: Create Implementation Planning Files

**.planning/active-implementations/{impl-name}/concept-source.md:**
```markdown
# Implementation Source

**Implementation:** {Implementation Name}
**Source Concept:** {concept-name}
**Source Focus Group:** {focus-group}
**Promoted:** {date}

## Original Concept

**File:** `.planning/focus-groups/{focus-group}/concepts/{concept-name}.md`
**Problem Statement:** {extracted from concept}
**Priority:** {extracted from concept}

## Concept Context

{Copy relevant sections from original concept file}

## Implementation Decision

**Reason for Implementation:** {why this concept was selected}
**Target Timeline:** 1-3 days maximum
**Assigned Developers:** {to be filled}

## Links

- **Source Concept:** `.planning/focus-groups/{focus-group}/concepts/{concept-name}.md`
- **Focus Group Channel:** #{repo-name}-{focus-group}
- **Implementation Channel:** #{repo-name}-impl-{concept-name}

---

*This implementation tracks back to the {focus-group} focus group discussion*
```

**.planning/active-implementations/{impl-name}/STATE.md:**
```markdown
# {Implementation Name} - State

**Created:** {date}
**Source Concept:** {concept-name} from {focus-group}
**Status:** Initializing
**Target Version:** TBD (determined at merge)
**Days Active:** 0

## Current Phase

**Phase:** 0 - Requirements Definition
**Progress:** Starting
**Next:** Define implementation requirements from concept

## Team

**Channel:** #{repo-name}-impl-{concept-name}
**Developers:** {to be assigned}

## Timeline

- **Day 0** ({date}): Implementation created from concept
- **Target:** 1-3 day implementation cycle
- **Merge Target:** develop branch

## Blockers & Dependencies

*None currently*

## Recent Activity

- {timestamp}: Implementation channel created
- {timestamp}: Requirements definition starting

---

**Daily Updates:** This file should be updated daily during implementation
```

## Step 5: Create Git Branch and Worktree

```bash
# Create implementation branch off develop
git checkout develop
git pull origin develop  # Get latest changes

# Create implementation branch
git checkout -b implementations/${CONCEPT_NAME}
echo "# Implementation: ${CONCEPT_NAME}" > .implementation-readme.md
git add .implementation-readme.md
git commit -m "feat: start ${CONCEPT_NAME} implementation branch"
git push -u origin implementations/${CONCEPT_NAME}

# Create worktree for implementation work
git worktree add implementations/${CONCEPT_NAME} implementations/${CONCEPT_NAME}

echo "✅ Git structure created:"
echo "   Branch: implementations/${CONCEPT_NAME}"
echo "   Worktree: implementations/${CONCEPT_NAME}/"
```

## Step 6: Create Implementation Channel

Create Slack channel:
- **Name:** {repo-name}-impl-{concept-name}
- **Topic:** "Implementation of {concept-name} concept from #{repo-name}-{focus-group}. Target: 1-3 days execution."
- **Type:** Public channel
- **Add Jarvis with no-mention mode**

## Step 7: Create Implementation Canvas

**Canvas Title:** "{Concept Name} Implementation Status"
**Canvas Content:**
```
# {Concept Name} Implementation

**Source:** {concept-name} from {focus-group} focus group
**Status:** Initializing
**Timeline:** 1-3 days
**Target Version:** TBD

## Current Phase
Phase 0: Requirements Definition

## Team
*To be assigned*

## Daily Progress
*Will be updated daily*

## Blockers
*None currently*
```

## Step 8: Start GSD Workflow

Switch to implementation worktree and start traditional GSD:

```bash
cd implementations/${CONCEPT_NAME}

# Initialize GSD in implementation directory with concept context
echo "🎯 Starting GSD workflow for implementation"
echo "📋 Using concept context for requirements gathering"
```

Use traditional `/gsd new-project` workflow but with concept as starting context:
- Requirements based on concept problem statement
- Research if needed (may be lighter since concept already explored)
- Roadmap with phases
- Execution with task-level commits

## Step 9: Update Master Configuration

Update .planning/WGSD-CONFIG.md:
```markdown
### Active Implementations
- **{concept-name}-impl**: #{repo-name}-impl-{concept-name} (started {date})
```

Update .planning/MASTER-ROADMAP.md with new implementation.

## Step 10: Update Source Concept Status

Mark original concept as implemented:

```bash
# Update concept status
sed -i 's/Status: 🚀 Ready for Implementation/Status: 🔧 In Implementation/' \
.planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_NAME}.md

# Add implementation reference
echo "## Implementation" >> .planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_NAME}.md
echo "**Implementation Started:** {date}" >> .planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_NAME}.md
echo "**Implementation Channel:** #{repo-name}-impl-{concept-name}" >> .planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_NAME}.md

# Commit changes
git add .planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_NAME}.md
git commit -m "docs: mark ${CONCEPT_NAME} concept as in implementation"
```

## Step 11: Team Notifications

Send announcement to source focus group:

```markdown
🚀 **Concept Promoted to Implementation!**

**Concept:** {Concept Name}
**Implementation Channel:** #{repo-name}-impl-{concept-name}

The {concept-name} concept has been promoted to active implementation!

**What happens now:**
- Dedicated implementation channel for focused 1-3 day execution
- Traditional GSD workflow (requirements → phases → execution)
- Daily progress updates and blocking issue resolution
- Target merge to develop branch within 3 days

**Implementation tracking:**
- **Status:** `.planning/active-implementations/{impl-name}/STATE.md`
- **Source concept:** Updated with implementation reference

*Great work on maturing this concept through social discussion!*
```

Send welcome to implementation channel:

```markdown
⚡ **Implementation Started: {Concept Name}**

**Source Concept:** {concept-name} from #{repo-name}-{focus-group}
**Timeline:** 1-3 days maximum
**Git Branch:** implementations/{concept-name}
**Worktree:** implementations/{concept-name}/

**This is a focused implementation channel** - different from concept discussion:
- Fast-paced execution using traditional GSD workflow
- Requirements → Roadmap → Phases → Task execution
- Daily progress updates required
- Blocking issues resolved immediately

**Canvas above syncs with:** `.planning/active-implementations/{impl-name}/STATE.md`

**Next Steps:**
1. Define implementation requirements from concept
2. Create GSD roadmap and phases  
3. Begin implementation with task-level commits
4. Daily status updates in this channel

*Jarvis will guide the GSD workflow - no @ mentions needed*
```

## Step 12: Return Status

```
---

## ✅ Implementation Created: {Concept Name}

**Source:** {concept-name} from {focus-group} focus group
**Channel:** #{repo-name}-impl-{concept-name}
**Git Branch:** implementations/{concept-name}
**Worktree:** implementations/{concept-name}/
**Timeline:** 1-3 days maximum

**Implementation slots:** {current+1}/{max-implementations} used

**Ready for:**
- Requirements definition from concept context
- GSD roadmap and phase planning
- Focused code execution and daily updates
- Merge to develop within 3 days

**Next:** Switch to implementation channel and start GSD requirements gathering

---
```

</process>

<success_criteria>
- [ ] Implementation capacity validated (under 4 concurrent)  
- [ ] Source concept found and validated as ready
- [ ] Implementation planning structure created
- [ ] Git branch and worktree set up off develop
- [ ] Implementation channel created with proper topic
- [ ] Canvas created for implementation status tracking
- [ ] Source concept updated with implementation reference
- [ ] Master configuration updated with new implementation
- [ ] Team notifications sent to both channels
- [ ] Ready to start traditional GSD workflow in implementation context
</success_criteria>
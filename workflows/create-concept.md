---
name: wgsd:create-concept
description: Create a new concept within the current focus group
argument-hint: "[concept-name]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Canvas
---

<objective>
Create a new concept within the current focus group based on social discussion.
Captures idea with proper structure, updates roadmap, and syncs with canvas.
Should be run from within a focus group channel context.
</objective>

<process>

## Step 1: Determine Context

```bash
# Determine which focus group this concept belongs to
# This should be run from a focus group channel context
REPO_NAME=$(basename $(pwd))
CONCEPT_NAME="{concept-name}"

# Try to detect focus group from current channel or working directory
if [ -d concepts ]; then
    # We're likely in a worktree, detect which one
    PWD_PATH=$(pwd)
    if [[ "$PWD_PATH" == *"/concepts/"* ]]; then
        FOCUS_GROUP=$(basename "$PWD_PATH")
        echo "📍 Detected focus group from worktree: ${FOCUS_GROUP}"
    fi
fi

# If not detected, ask user
if [ -z "$FOCUS_GROUP" ]; then
    echo "🤔 Which focus group should this concept belong to?"
    find .planning/focus-groups -maxdepth 1 -type d | grep -v "^\\.planning/focus-groups$" | sed 's|.planning/focus-groups/||'
    exit 1
fi

echo "💡 Creating concept: ${CONCEPT_NAME}"
echo "📁 Focus group: ${FOCUS_GROUP}"
echo "📱 Channel: #${REPO_NAME}-${FOCUS_GROUP}"
```

## Step 2: Validate Focus Group Exists

```bash
if [ ! -d ".planning/focus-groups/${FOCUS_GROUP}" ]; then
    echo "❌ Focus group '${FOCUS_GROUP}' doesn't exist"
    echo "💡 Create it first: /wgsd create-focus-group ${FOCUS_GROUP}"
    exit 1
fi

if [ -f ".planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_NAME}.md" ]; then
    echo "⚠️  Concept '${CONCEPT_NAME}' already exists in ${FOCUS_GROUP}"
    echo "📄 Edit existing concept or choose different name"
    exit 1
fi

echo "✅ Focus group validated"
```

## Step 3: Gather Concept Context

Ask user for concept details:

**Required Information:**
- **Brief Description:** What is this concept about? (1-2 sentences)
- **Problem Statement:** What problem does this solve?
- **Initial Ideas:** Any early thoughts on approach or implementation?
- **Priority Level:** Low/Medium/High
- **Dependencies:** Does this relate to other concepts or focus groups?

**Optional Information:**
- **Target Users:** Who would benefit from this?
- **Success Metrics:** How would we measure success?
- **Rough Scope:** Small/Medium/Large effort estimate

## Step 4: Create Concept File

**.planning/focus-groups/{focus-group}/concepts/{concept-name}.md:**
```markdown
# {Concept Name}

**Focus Group:** {Focus Group}
**Created:** {date}
**Status:** 📝 Draft
**Priority:** {priority}
**Creator:** {creator-name}

## Problem Statement

{problem-statement}

## Concept Overview

{brief-description}

## Initial Ideas

{initial-ideas}

## Dependencies & References

{dependencies}
{Cross-references to other focus groups or concepts}

## Discussion History

### {date} - Concept Created
- Initial idea captured from channel discussion
- Status: Draft - needs social discussion and refinement

*Add discussion updates here as the concept evolves*

## Implementation Considerations

**Rough Scope:** {scope-estimate}
**Target Users:** {target-users}
**Success Metrics:** {success-metrics}

*These will be refined as concept matures*

## Next Steps

- [ ] Social discussion in #{repo-name}-{focus-group}
- [ ] Gather team input and refine approach
- [ ] Research similar solutions or approaches
- [ ] Define success criteria more clearly
- [ ] Move to "Exploring" status when discussion is active

---

**Status Progression:** 📝 Draft → 🔍 Exploring → ✅ Mature → 🚀 Ready for Implementation

*Edit this file as discussions evolve, or use channel discussion and sync later*
```

## Step 5: Update Focus Group Roadmap

Update .planning/focus-groups/{focus-group}/ROADMAP.md:

```bash
# Add concept to the Draft section
sed -i "/### 📝 Draft/a\\
- **${CONCEPT_NAME}** ({priority}) - {brief-description} *[created {date}]*" \
.planning/focus-groups/${FOCUS_GROUP}/ROADMAP.md

# Update concept count in State
CONCEPT_COUNT=$(find .planning/focus-groups/${FOCUS_GROUP}/concepts -name "*.md" | wc -l)
sed -i "s/Active Concepts: [0-9]*/Active Concepts: $CONCEPT_COUNT/" \
.planning/focus-groups/${FOCUS_GROUP}/STATE.md
```

## Step 6: Commit to Focus Group Branch

```bash
# Switch to focus group worktree if we're not already there
if [ ! -d concepts/${FOCUS_GROUP} ]; then
    echo "📁 Switching to focus group worktree"
    cd concepts/${FOCUS_GROUP}
fi

# Add and commit the new concept
git add .planning/focus-groups/${FOCUS_GROUP}/concepts/${CONCEPT_NAME}.md
git add .planning/focus-groups/${FOCUS_GROUP}/ROADMAP.md
git add .planning/focus-groups/${FOCUS_GROUP}/STATE.md
git commit -m "feat(${FOCUS_GROUP}): add ${CONCEPT_NAME} concept [Draft]"
git push origin focus-groups/${FOCUS_GROUP}

echo "✅ Concept committed to focus group branch"
```

## Step 7: Update Canvas

Update focus group channel canvas with new concept:
- Pull current ROADMAP.md content
- Sync to canvas in #{repo-name}-{focus-group} channel
- Ensure bidirectional sync is maintained

## Step 8: Announce in Channel

Send message to focus group channel:

```markdown
💡 **New Concept Created: {Concept Name}**

**Status:** 📝 Draft  
**Priority:** {priority}
**Problem:** {brief-problem-statement}

**File:** `.planning/focus-groups/{focus-group}/concepts/{concept-name}.md`

**Next Steps:**
- Team discussion and feedback
- Refine problem statement and approach  
- Gather requirements and constraints
- Move to 🔍 Exploring status when ready

*The concept file and roadmap canvas have been updated. Feel free to edit either location - they sync automatically.*

**What do you think about this concept? Any initial reactions or ideas?**
```

## Step 9: Update Master Roadmap

```bash
# Update master roadmap concept count
cd ../.. # Back to repo root
TOTAL_CONCEPTS=$(find .planning/focus-groups -name "*.md" -path "*/concepts/*" | wc -l)

# Trigger roadmap update
echo "📊 Updating master roadmap with new concept"
```

## Step 10: Return Status

```
---

## ✅ Concept Created: {Concept Name}

**Focus Group:** {Focus Group}
**Status:** 📝 Draft
**File:** `.planning/focus-groups/{focus-group}/concepts/{concept-name}.md`
**Channel:** #{repo-name}-{focus-group}

**The concept is now:**
- 📄 Documented with initial context  
- 🔄 Synced to focus group canvas
- 💬 Ready for team discussion
- 📈 Tracked in focus group roadmap

**Next Actions:**
- Gather team input in #{repo-name}-{focus-group}
- Refine problem statement and approach
- Research similar solutions
- Move to "Exploring" status when discussion is active

**Progression:** 📝 Draft → 🔍 Exploring → ✅ Mature → 🚀 Ready for Implementation

---
```

</process>

<success_criteria>
- [ ] Concept file created with proper structure and context
- [ ] Focus group roadmap updated with new concept
- [ ] Changes committed to focus group git branch
- [ ] Canvas synced with updated roadmap
- [ ] Team notified in focus group channel
- [ ] Master roadmap concept count updated
- [ ] Clear next steps provided for concept development
</success_criteria>
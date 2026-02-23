---
name: wgsd:merge-to-roadmap
description: Merge fully approved concept to roadmap branch
argument-hint: "<concept-slug> [--force]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Message
---

# Merge to Roadmap

Merge a fully approved concept to the roadmap branch, making it available for implementation.

**This is typically triggered automatically by approval completion, but can be run manually.**

---

## Objective

When a concept receives full matrix approval (all impacted focus groups approve):
1. Merge the concept branch to the `roadmap` branch
2. Update the roadmap manifest with the new entry
3. Archive the concept branch
4. Mark the concept as ready for implementation
5. Notify the team

---

## Process

### Step 1: Validate Concept Status

```bash
CONCEPT_SLUG="$1"
FORCE_FLAG="${2:-}"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "📋 Merge to Roadmap: ${CONCEPT_SLUG}"
echo "   Time: ${TIMESTAMP}"
echo ""

# Find concept location (supports directory and legacy file format)
CONCEPT_PATH=""
FOCUS_GROUP=""
CONCEPT_TYPE=""

for fg_dir in .planning/focus-groups/*/; do
    [ -d "$fg_dir" ] || continue
    
    if [ -d "${fg_dir}concepts/${CONCEPT_SLUG}" ]; then
        CONCEPT_PATH="${fg_dir}concepts/${CONCEPT_SLUG}"
        FOCUS_GROUP=$(basename "$fg_dir")
        CONCEPT_TYPE="directory"
        break
    elif [ -f "${fg_dir}concepts/${CONCEPT_SLUG}.md" ]; then
        CONCEPT_PATH="${fg_dir}concepts/${CONCEPT_SLUG}.md"
        FOCUS_GROUP=$(basename "$fg_dir")
        CONCEPT_TYPE="file"
        break
    fi
done

if [ -z "$CONCEPT_PATH" ]; then
    echo "❌ Concept not found: ${CONCEPT_SLUG}"
    echo ""
    echo "Available concepts:"
    find .planning/focus-groups -type d -name "concepts" -exec ls {} \; 2>/dev/null
    exit 1
fi

echo "📍 Found: ${CONCEPT_SLUG} in ${FOCUS_GROUP} (${CONCEPT_TYPE} format)"
```

### Step 2: Verify Full Approval

```bash
echo ""
echo "🔍 Verifying approval status..."

# Check for approval marker (created by Phase 12 completion trigger)
if [ "$CONCEPT_TYPE" = "directory" ] && [ -f "${CONCEPT_PATH}/.approved" ]; then
    echo "✅ Approval marker found"
    APPROVAL_DATE=$(grep "approved_at:" "${CONCEPT_PATH}/.approved" | sed 's/approved_at: //')
else
    # Manual verification via impact matrix
    IMPACT_MATRIX="${CONCEPT_PATH}/impact-matrix.md"
    
    if [ -n "$IMPACT_MATRIX" ] && [ -f "$IMPACT_MATRIX" ]; then
        # Parse approval status from impact matrix
        PENDING=$(grep -c "status: pending" "$IMPACT_MATRIX" 2>/dev/null || echo "0")
        BLOCKED=$(grep -c "status: blocked" "$IMPACT_MATRIX" 2>/dev/null || echo "0")
        REJECTED=$(grep -c "status: rejected" "$IMPACT_MATRIX" 2>/dev/null || echo "0")
        APPROVED=$(grep -c "status: approved" "$IMPACT_MATRIX" 2>/dev/null || echo "0")
        
        echo "   Approved: ${APPROVED}, Pending: ${PENDING}, Blocked: ${BLOCKED}, Rejected: ${REJECTED}"
        
        if [ "$REJECTED" -gt 0 ]; then
            echo "❌ Cannot merge: Concept has rejections"
            echo "   Review feedback and address concerns before proceeding"
            exit 1
        fi
        
        if [ "$BLOCKED" -gt 0 ]; then
            echo "❌ Cannot merge: Concept has blocked approvals"
            echo "   Resolve blocking dependencies first"
            exit 1
        fi
        
        if [ "$PENDING" -gt 0 ] && [ "$FORCE_FLAG" != "--force" ]; then
            echo "❌ Cannot merge: Concept has ${PENDING} pending approval(s)"
            echo ""
            echo "Options:"
            echo "  1. Wait for all focus groups to approve"
            echo "  2. Use --force to override (admin only)"
            exit 1
        fi
        
        if [ "$PENDING" -gt 0 ] && [ "$FORCE_FLAG" = "--force" ]; then
            echo "⚠️ FORCE FLAG: Overriding ${PENDING} pending approval(s)"
        fi
    else
        echo "⚠️ No impact matrix found - proceeding with manual merge"
    fi
fi

echo "✅ Approval verification passed"
```

### Step 3: Check for Existing Roadmap Entry

```bash
echo ""
echo "🔍 Checking roadmap status..."

# Ensure roadmap branch exists
source workflows/lib/branch-ops.md 2>/dev/null || true
ROADMAP_BRANCH=$(ensure_roadmap_branch 2>/dev/null || echo "roadmap")

# Check if already on roadmap
if git show "roadmap:.planning/ROADMAP-MANIFEST.md" 2>/dev/null | grep -q "| ${CONCEPT_SLUG} |"; then
    if [ "$FORCE_FLAG" != "--force" ]; then
        echo "⚠️ Concept already on roadmap: ${CONCEPT_SLUG}"
        echo "   To force re-merge, use --force flag"
        exit 0
    else
        echo "⚠️ FORCE FLAG: Re-merging existing roadmap entry"
    fi
fi

echo "✅ Ready to merge to roadmap"
```

### Step 4: Prepare Merge

```bash
echo ""
echo "🔀 Preparing merge to roadmap..."

# Store current branch
ORIGINAL_BRANCH=$(git branch --show-current)

# Fetch latest
git fetch origin roadmap 2>/dev/null || true
git fetch origin "concepts/${CONCEPT_SLUG}" 2>/dev/null || true

# Stash any local changes
git stash push -m "WGSD: roadmap merge stash" 2>/dev/null || true

# Checkout roadmap
git checkout roadmap
git pull origin roadmap 2>/dev/null || true

echo "   On branch: roadmap"
```

### Step 5: Merge Concept Branch

```bash
CONCEPT_BRANCH="concepts/${CONCEPT_SLUG}"
MERGE_TYPE="branch"

# Check if concept branch exists
if git rev-parse --verify "$CONCEPT_BRANCH" >/dev/null 2>&1; then
    echo "   Found local branch: ${CONCEPT_BRANCH}"
elif git rev-parse --verify "origin/${CONCEPT_BRANCH}" >/dev/null 2>&1; then
    CONCEPT_BRANCH="origin/${CONCEPT_BRANCH}"
    echo "   Found remote branch: ${CONCEPT_BRANCH}"
else
    echo "⚠️ No concept branch found, using cherry-pick strategy"
    MERGE_TYPE="copy"
fi

if [ "$MERGE_TYPE" = "branch" ]; then
    echo ""
    echo "🔀 Merging ${CONCEPT_BRANCH} to roadmap..."
    
    git merge "$CONCEPT_BRANCH" --no-ff -m "feat(roadmap): add ${CONCEPT_SLUG} [fully-approved]

Concept: ${CONCEPT_SLUG}
Focus Group: ${FOCUS_GROUP}
Approved: ${TIMESTAMP}

This concept has received full matrix approval and is now
ready for implementation.

Triggered by: /wgsd merge-to-roadmap
Co-authored-by: WGSD Automation <wgsd@openclaw.dev>"
    
    MERGE_RESULT=$?
    
    if [ $MERGE_RESULT -ne 0 ]; then
        echo ""
        echo "❌ Merge conflict detected"
        echo ""
        echo "Conflicting files:"
        git status --short | grep "^UU"
        echo ""
        echo "Options:"
        echo "  1. Resolve conflicts manually, then run:"
        echo "     git add . && git commit"
        echo "     git push origin roadmap"
        echo ""
        echo "  2. Abort and return to original state:"
        echo "     git merge --abort"
        echo "     git checkout ${ORIGINAL_BRANCH}"
        echo ""
        git merge --abort 2>/dev/null
        git checkout "$ORIGINAL_BRANCH" 2>/dev/null
        git stash pop 2>/dev/null || true
        exit 1
    fi
    
    echo "✅ Merge successful"
else
    # Copy concept from current state when no branch exists
    echo ""
    echo "📋 Copying concept to roadmap (no branch)..."
    
    git checkout "$ORIGINAL_BRANCH" -- "$CONCEPT_PATH" 2>/dev/null || true
    git add "$CONCEPT_PATH"
    git commit -m "feat(roadmap): add ${CONCEPT_SLUG} [fully-approved]

Concept: ${CONCEPT_SLUG}
Focus Group: ${FOCUS_GROUP}
Approved: ${TIMESTAMP}

Copied from ${ORIGINAL_BRANCH} (no concept branch available)

Triggered by: /wgsd merge-to-roadmap"
    
    echo "✅ Copy successful"
fi
```

### Step 6: Update Roadmap Manifest

```bash
echo ""
echo "📝 Updating roadmap manifest..."

MANIFEST=".planning/ROADMAP-MANIFEST.md"

# Get approval info
IMPACTS="1"
PRIORITY="P1"

if [ -f "${CONCEPT_PATH}/impact-matrix.md" ]; then
    IMPACTS=$(grep -c "### Focus Group:" "${CONCEPT_PATH}/impact-matrix.md" 2>/dev/null || echo "1")
    PRIORITY=$(grep -m1 "priority:" "${CONCEPT_PATH}/impact-matrix.md" | sed 's/.*priority: *//' | xargs || echo "P1")
elif [ -f "${CONCEPT_PATH}/CONCEPT.md" ]; then
    PRIORITY=$(grep -m1 "\*\*Priority:\*\*" "${CONCEPT_PATH}/CONCEPT.md" | sed 's/.*\*\*Priority:\*\* *//' | xargs || echo "P1")
fi

# Remove existing entry if present (for re-merge)
if grep -q "| ${CONCEPT_SLUG} |" "$MANIFEST" 2>/dev/null; then
    grep -v "| ${CONCEPT_SLUG} |" "$MANIFEST" > "${MANIFEST}.tmp"
    mv "${MANIFEST}.tmp" "$MANIFEST"
fi

# Add new entry to manifest (after the header row)
# Find the line with the header separator and insert after it
if grep -q "^|---------|" "$MANIFEST"; then
    sed -i "/^|---------|/a | ${CONCEPT_SLUG} | ${TIMESTAMP%%T*} | ${IMPACTS} FG(s) | ${PRIORITY} |" "$MANIFEST"
fi

# Add sync history entry
echo "| ${TIMESTAMP%%T*} | Concept Merged | ${CONCEPT_SLUG} from ${FOCUS_GROUP} |" >> "$MANIFEST"

git add "$MANIFEST"
git commit --amend --no-edit

echo "✅ Manifest updated"
```

### Step 7: Push Roadmap

```bash
echo ""
echo "📤 Pushing roadmap branch..."

git push origin roadmap

if [ $? -ne 0 ]; then
    echo "⚠️ Push failed - may need manual push"
    echo "   Run: git push origin roadmap"
fi

echo "✅ Roadmap branch pushed"
```

### Step 8: Archive Concept Branch

```bash
echo ""
echo "📦 Archiving concept branch..."

# Archive concept branch (rename, don't delete)
if git rev-parse --verify "concepts/${CONCEPT_SLUG}" >/dev/null 2>&1; then
    ARCHIVE_BRANCH="archive/concepts/${CONCEPT_SLUG}-$(date +%Y%m%d)"
    
    # Rename local branch
    git branch -m "concepts/${CONCEPT_SLUG}" "$ARCHIVE_BRANCH" 2>/dev/null || true
    
    # Push archive and delete original on remote
    git push origin "$ARCHIVE_BRANCH" 2>/dev/null || true
    git push origin --delete "concepts/${CONCEPT_SLUG}" 2>/dev/null || true
    
    echo "✅ Archived to ${ARCHIVE_BRANCH}"
else
    echo "   No concept branch to archive"
fi
```

### Step 9: Return to Original Branch

```bash
# Return to original branch
git checkout "$ORIGINAL_BRANCH" 2>/dev/null || git checkout "$(detect_primary_branch)"
git stash pop 2>/dev/null || true
```

### Step 10: Mark Concept as Roadmapped

```bash
echo ""
echo "📝 Marking concept as roadmapped..."

# Update concept status in the original branch
if [ "$CONCEPT_TYPE" = "directory" ]; then
    CONCEPT_FILE="${CONCEPT_PATH}/CONCEPT.md"
else
    CONCEPT_FILE="${CONCEPT_PATH}"
fi

if [ -f "$CONCEPT_FILE" ]; then
    # Update status
    sed -i "s/\*\*Status:\*\* .*/\*\*Status:\*\* 📋 On Roadmap/" "$CONCEPT_FILE"
    
    # Add roadmap marker section if not present
    if ! grep -q "## 📋 Roadmap Status" "$CONCEPT_FILE"; then
        cat >> "$CONCEPT_FILE" << EOF

---

## 📋 Roadmap Status

**Merged to Roadmap:** ${TIMESTAMP}
**Status:** Ready for Implementation
**Implementation:** \`implementations/${CONCEPT_SLUG}\`

To create implementation:
\`\`\`
/wgsd create-implementation ${CONCEPT_SLUG}
\`\`\`
EOF
    fi
    
    git add "$CONCEPT_FILE"
    git commit -m "docs(${FOCUS_GROUP}): mark ${CONCEPT_SLUG} as roadmapped"
    git push origin "$(git branch --show-current)" 2>/dev/null || true
    
    echo "✅ Concept marked as roadmapped"
fi
```

### Step 11: Generate Notification

```bash
echo ""
echo "═══════════════════════════════════════════════════════════"
echo "✅ Merged to Roadmap: ${CONCEPT_SLUG}"
echo "═══════════════════════════════════════════════════════════"
echo ""
echo "📊 Summary:"
echo "   Concept: ${CONCEPT_SLUG}"
echo "   Focus Group: ${FOCUS_GROUP}"
echo "   Priority: ${PRIORITY}"
echo "   Impacts: ${IMPACTS} focus group(s)"
echo ""
echo "🎯 Next Steps:"
echo ""
echo "   Create implementation:"
echo "   \`\`\`"
echo "   /wgsd create-implementation ${CONCEPT_SLUG}"
echo "   \`\`\`"
echo ""
echo "   View on roadmap:"
echo "   \`\`\`"
echo "   git checkout roadmap"
echo "   cat .planning/ROADMAP-MANIFEST.md"
echo "   \`\`\`"
echo ""
echo "📢 Notification Message:"
echo ""
cat << EOF
📋 **Concept Added to Roadmap: ${CONCEPT_SLUG}**

The concept has been fully approved and merged to the roadmap branch.

**Focus Group:** ${FOCUS_GROUP}
**Priority:** ${PRIORITY}
**Impacts:** ${IMPACTS} focus group(s)

**Ready for Implementation!**

\`\`\`
/wgsd create-implementation ${CONCEPT_SLUG}
\`\`\`

View roadmap: \`git checkout roadmap\`
EOF
```

---

## Integration with Phase 12

This workflow is triggered automatically by `approval_matrix_trigger_on_complete()` when a concept receives full approval from all impacted focus groups.

**Trigger output from Phase 12:**
```
TRIGGER:roadmap_merge:<concept_name>
```

**Manual invocation:**
```bash
/wgsd merge-to-roadmap <concept-slug>
/wgsd merge-to-roadmap <concept-slug> --force  # Admin override
```

---

## Success Criteria

- [ ] Concept fully approved before merge (unless --force)
- [ ] Roadmap branch updated with concept content
- [ ] ROADMAP-MANIFEST.md updated with entry
- [ ] Concept branch archived (not deleted)
- [ ] Concept status updated to "On Roadmap"
- [ ] Team notified of roadmap addition
- [ ] No merge conflicts (or properly resolved)

---

## Error Recovery

| Error | Recovery |
|-------|----------|
| Merge conflict | Manual resolution or abort |
| Push failed | Manual push after resolution |
| Concept not found | Check focus group paths |
| Already on roadmap | Use --force to re-merge |

```bash
# If merge was aborted mid-process
git checkout <original-branch>
git stash pop  # Recover any stashed changes
```

---

*Workflow created for WGSD Phase 13 - Roadmap Branch Architecture*

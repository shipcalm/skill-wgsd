# Phase 10: Concept Directory Architecture

**Status:** 🟡 Planned  
**Priority:** P0 - Critical (Foundation for v2.2)  
**Target:** Transform concepts from single .md files to comprehensive directories  
**Requirements:** 5

---

## Phase Overview

This is the foundational phase for WGSD v2.2. All subsequent phases (11-16) depend on concepts being proper directories with independent branches, enabling:

1. **Multi-artifact concepts** - specs, wireframes, research all in one place
2. **Independent development** - parallel concept work without conflicts
3. **Cross-cutting impact** - Phase 11 requires directories for impact-matrix.md
4. **Matrix approval** - Phase 12 needs approval-matrix.json in concept dirs

### Architecture Transformation

```
BEFORE (v2.0-2.1):
.planning/focus-groups/{fg}/concepts/{name}.md  ← Single file

AFTER (v2.2):
concepts/{name}/                    ← Independent branch & directory
├── CONCEPT.md                      ← Primary description (required)
├── impact-matrix.md                ← Cross-cutting impacts (Phase 11)
├── approval-matrix.json            ← Approval tracking (Phase 12)
├── API-SPEC.md                     ← Optional: API specification
├── acceptance-criteria.md          ← Optional: Detailed AC
└── wireframes/                     ← Optional: Design artifacts
```

---

## Requirements Breakdown

### CONCEPT-DIR-01: Create Concept as Directory ✅ Planned

**Goal:** Modify `create-concept` workflow to create directories instead of files

**Execution Plan:**

1. **Analyze current workflow** (10 min)
   - Review `workflows/create-concept.md` structure
   - Identify all file creation points
   - Note cross-references to update

2. **Update create-concept.md workflow** (30 min)
   - Change Step 4: Create `concepts/{name}/CONCEPT.md` not `concepts/{name}.md`
   - Add directory creation before file creation
   - Update git add commands to handle directory
   - Update commit messages

3. **Add backward compatibility detection** (15 min)
   - Check if concept exists as file or directory
   - Support reading old single-file concepts
   - Add migration hint for legacy concepts

4. **Update related workflows** (20 min)
   - `concept-development.md` - update concept file location detection
   - `promote-concept.md` - handle directory promotion
   - `canvas-sync.md` - update concept scanning

5. **Test** (15 min)
   - Create new concept, verify directory structure
   - Read old concept, verify backward compat
   - Verify git operations work with directories

**Files Modified:**
- `workflows/create-concept.md`
- `workflows/concept-development.md`
- `workflows/promote-concept.md`
- `workflows/canvas-sync.md`

**Dependencies:** None

---

### CONCEPT-DIR-02: Concept Directory Structure Template ✅ Planned

**Goal:** Create standard directory template for concepts

**Execution Plan:**

1. **Create template directory** (10 min)
   ```
   templates/concept-directory/
   ├── CONCEPT.md.tmpl
   ├── impact-matrix.md.tmpl
   └── README.md              ← Template usage guide
   ```

2. **Design CONCEPT.md template** (20 min)
   - Migrate content from current single-file template
   - Add frontmatter for structured metadata
   - Include status progression guide
   - Add links to related artifacts

3. **Design impact-matrix.md template** (15 min)
   - YAML frontmatter for structured impact data
   - Per-focus-group impact sections
   - Priority and impact description fields
   - Approval placeholder (for Phase 11)

4. **Create optional artifact templates** (15 min)
   - `API-SPEC.md.tmpl` - OpenAPI-style structure
   - `acceptance-criteria.md.tmpl` - Given/When/Then format
   - Document when each should be used

5. **Add template config** (10 min)
   - Config option to customize default artifacts
   - Option to skip optional artifacts on creation
   - Config in WGSD-CONFIG.md

**Files Created:**
- `templates/concept-directory/CONCEPT.md.tmpl`
- `templates/concept-directory/impact-matrix.md.tmpl`
- `templates/concept-directory/API-SPEC.md.tmpl`
- `templates/concept-directory/acceptance-criteria.md.tmpl`
- `templates/concept-directory/README.md`

**Dependencies:** None

---

### CONCEPT-DIR-03: Independent Concept Branches ✅ Planned

**Goal:** Create `concepts/{name}` branches for independent concept development

**Execution Plan:**

1. **Update branch-ops.md library** (20 min)
   - Add `create-concept-branch` function
   - Branch from focus group branch (not develop)
   - Naming: `concepts/{concept-name}`
   - Support branch listing and cleanup

2. **Update create-concept.md workflow** (25 min)
   - After creating directory, create dedicated branch
   - Push initial commit to branch
   - Set up branch tracking
   - Update announcement to include branch name

3. **Update concept-development.md workflow** (20 min)
   - Switch to concept branch for edits
   - PR workflow: `concepts/{name}` → `focus-groups/{fg}`
   - Update branch detection logic

4. **Add branch lifecycle management** (15 min)
   - Archive branch on concept completion
   - Delete branch after successful implementation
   - Track active concept branches

5. **Document branching model** (10 min)
   - Add section to SKILL.md architecture
   - Document branch flow in concept workflows

**Branch Flow:**
```
concepts/{name}          ← Concept development happens here
    │
    ▼ (PR on approval)
focus-groups/{fg}        ← Focus group review
    │
    ▼ (full approval)
roadmap                  ← Ready for implementation (Phase 13)
```

**Files Modified:**
- `workflows/lib/branch-ops.md`
- `workflows/create-concept.md`
- `workflows/concept-development.md`
- `workflows/promote-concept.md`
- `SKILL.md` (architecture section)

**Dependencies:** CONCEPT-DIR-01 (directory must exist before branch)

---

### CONCEPT-DIR-04: Concept Worktree Support ✅ Planned

**Goal:** Optional git worktree setup for parallel concept development

**Execution Plan:**

1. **Update git-ops.md library** (20 min)
   - Add `create-concept-worktree` function
   - Worktree path: `worktrees/{concept-name}/`
   - Handle worktree existence checks
   - Add cleanup function

2. **Add worktree option to create-concept.md** (15 min)
   - `--worktree` flag for worktree creation
   - Default: no worktree (branch only)
   - Output worktree path if created

3. **Integrate worktree path into Canvas** (15 min)
   - Add worktree path field to canvas templates
   - Show "No worktree" when not configured
   - Make path copyable

4. **Add worktree management commands** (20 min)
   - `wgsd concept worktree {name}` - create worktree for existing concept
   - `wgsd concept cleanup {name}` - remove worktree on completion
   - Update completion workflows

5. **Document worktree usage** (10 min)
   - When to use worktrees vs branch-only
   - Performance considerations
   - Cleanup procedures

**Files Modified:**
- `workflows/lib/git-ops.md`
- `workflows/create-concept.md`
- `workflows/lib/canvas-templates.md`
- `workflows/canvas-sync.md`

**Dependencies:** CONCEPT-DIR-03 (branch must exist for worktree)

---

### CONCEPT-DIR-05: Multi-Artifact Concept Sync ✅ Planned

**Goal:** Canvas displays all concept artifacts with proper formatting

**Execution Plan:**

1. **Update canvas-templates.md** (25 min)
   - Add concept directory scanner
   - Template for artifact listing
   - Primary artifacts (CONCEPT.md, impact-matrix.md) inline
   - Secondary artifacts as links

2. **Update canvas-sync.md workflow** (20 min)
   - Detect concept directories vs single files
   - Read all .md files in concept directory
   - Handle wireframes/ directory as link
   - Track artifact changes for sync

3. **Design artifact display hierarchy** (15 min)
   ```
   ## 💡 {Concept Name}
   
   {CONCEPT.md content inline}
   
   ### 📊 Impact Matrix
   {impact-matrix.md content inline}
   
   ### 📎 Related Artifacts
   - [API Specification](link) 
   - [Acceptance Criteria](link)
   - [Wireframes (3 files)](link)
   ```

4. **Add artifact summary to listings** (15 min)
   - Roadmap view shows artifact count
   - Quick indicator: "📄 3 artifacts" 
   - Click expands to full view

5. **Test multi-artifact sync** (15 min)
   - Create concept with multiple artifacts
   - Verify Canvas displays correctly
   - Test incremental sync on artifact add

**Files Modified:**
- `workflows/lib/canvas-templates.md`
- `workflows/canvas-sync.md`
- `workflows/lib/canvas-api.md`

**Dependencies:** CONCEPT-DIR-01, CONCEPT-DIR-02

---

## Execution Sequence

```
CONCEPT-DIR-01 (directories) ─────┐
                                  │
CONCEPT-DIR-02 (templates) ───────┤
                                  ├──► CONCEPT-DIR-03 (branches)
                                  │          │
                                  │          ▼
                                  │    CONCEPT-DIR-04 (worktrees)
                                  │
                                  └──► CONCEPT-DIR-05 (canvas sync)
```

**Recommended Order:**
1. CONCEPT-DIR-02 (templates) - No dependencies, foundational
2. CONCEPT-DIR-01 (directories) - Uses templates, enables everything else
3. CONCEPT-DIR-03 (branches) - Requires directories
4. CONCEPT-DIR-04 (worktrees) - Requires branches
5. CONCEPT-DIR-05 (canvas sync) - Can run in parallel with 03/04

**Estimated Total Time:** ~5 hours

---

## Verification Checklist

After Phase 10 implementation:

- [ ] `wgsd create-concept test-concept` creates `concepts/test-concept/` directory
- [ ] Directory contains CONCEPT.md with proper template
- [ ] Old single-file concepts still readable (backward compat)
- [ ] Concept creates `concepts/test-concept` branch
- [ ] Branch contains concept directory
- [ ] Optional worktree at `worktrees/test-concept/` works
- [ ] Canvas shows concept with all artifacts
- [ ] Primary artifacts (CONCEPT.md) displayed inline
- [ ] Secondary artifacts linked
- [ ] Concept development PR workflow works with new structure
- [ ] No breaking changes to existing focus groups

---

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Breaking existing concepts | High | Backward compat detection + migration guide |
| Branch naming conflicts | Medium | Prefix with `concepts/` + uniqueness check |
| Worktree complexity | Low | Make worktrees optional, document clearly |
| Canvas size growth | Medium | Collapsible sections, artifact summaries |

---

## Phase Dependencies

```
Phase 10 (THIS) ─────────────┐
                             ├──► Phase 11 (Cross-Cutting Impact)
                             │    Uses: impact-matrix.md template
                             │
                             ├──► Phase 12 (Matrix Approval)
                             │    Uses: approval-matrix.json in dirs
                             │
                             └──► Phase 16 (Emergency Hotfix)
                                  Independent but needs concepts stable
```

---

**Last Updated:** 2026-02-23  
**Phase Status:** ✅ Executed

---

## Execution Summary

**Executed:** 2026-02-23  
**Duration:** ~30 minutes

### Completed Requirements

#### CONCEPT-DIR-02: Template Structure ✅
- Created `templates/concept-directory/API-SPEC.md.tmpl`
- Created `templates/concept-directory/acceptance-criteria.md.tmpl`
- Updated `templates/concept-directory/README.md`
- Templates support all required variables

#### CONCEPT-DIR-01: Directory-Based Concepts ✅
- Completely rewrote `workflows/create-concept.md`
- Creates concept directories with CONCEPT.md + impact-matrix.md
- Supports --api-spec and --ac flags for optional artifacts
- Backward compatibility detection for legacy single-file concepts

#### CONCEPT-DIR-03: Independent Branches ✅
- Added `create_concept_branch()` to `workflows/lib/branch-ops.md`
- Added `delete_concept_branch()` for cleanup
- Added `list_concept_branches()` for discovery
- Updated `validate_branch_state()` for concept tracking
- Branch naming: `concepts/{slug}` (not `concept/`)

#### CONCEPT-DIR-04: Worktree Support ✅
- Added `git_concept_worktree_add()` to `workflows/lib/git-ops.md`
- Added `git_concept_worktree_remove()` for cleanup
- Added `git_list_concept_worktrees()` for discovery
- Worktree path: `worktrees/{concept-slug}/`
- Auto-creates worktrees/.gitignore

#### CONCEPT-DIR-05: Multi-Artifact Canvas Sync ✅
- Updated `format_concepts_list()` in canvas-templates.md
- Added `format_concept_detail()` for detailed display
- Updated `build_live_roadmap()` in canvas-sync.md
- Supports both v2.2 directories and legacy files

### Supporting Changes

- Updated `workflows/concept-development.md` for directory detection
- Updated `workflows/promote-concept.md` for directory paths
- Updated `SKILL.md` architecture documentation
- All changes maintain backward compatibility with legacy concepts

### Verification

- [x] Template structure created and documented
- [x] Create-concept workflow generates directories
- [x] Branch operations support concept branches
- [x] Worktree operations support concept development
- [x] Canvas templates handle multi-artifact display
- [x] Backward compatibility maintained for legacy concepts
- [x] SKILL.md documents v2.2 architecture

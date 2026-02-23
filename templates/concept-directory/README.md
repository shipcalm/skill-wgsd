# Concept Directory Template

**WGSD v2.2 - Concept Directory Architecture**

This directory contains templates for WGSD concept directories. In v2.2, concepts are directories containing multiple artifacts rather than single files.

## Template Files

| File | Required | Purpose |
|------|----------|---------|
| `CONCEPT.md.tmpl` | ✅ Yes | Core concept description |
| `impact-matrix.md.tmpl` | ✅ Yes | Cross-cutting impact declarations (Phase 11) |
| `API-SPEC.md.tmpl` | ⏳ Optional | API specification for technical concepts |
| `acceptance-criteria.md.tmpl` | ⏳ Optional | Detailed acceptance criteria with user stories |

## Template Variables

All templates use `{{VARIABLE_NAME}}` syntax for substitution:

| Variable | Description | Example |
|----------|-------------|---------|
| `{{CONCEPT_NAME}}` | Human-readable concept name | "OAuth Integration" |
| `{{CONCEPT_SLUG}}` | URL/path-safe concept name | "oauth-integration" |
| `{{FOCUS_GROUP}}` | Primary focus group name | "security" |
| `{{DATE}}` | Creation date | "2026-02-23" |
| `{{PRIORITY}}` | Initial priority | "P1" |
| `{{CREATOR}}` | Creator name/handle | "@alice" |
| `{{REPO_NAME}}` | Repository name | "mvn" |
| `{{WORKTREE_PATH}}` | Worktree path or "None" | "worktrees/oauth-integration/" |
| `{{PROBLEM_STATEMENT}}` | Problem being solved | "Users can't authenticate..." |
| `{{BRIEF_DESCRIPTION}}` | Concept overview | "Implement OAuth 2.0..." |
| `{{INITIAL_IDEAS}}` | Early thoughts | "We could use Auth0..." |
| `{{SCOPE_ESTIMATE}}` | Rough scope | "Medium" |
| `{{TARGET_USERS}}` | Who benefits | "All API consumers" |
| `{{SUCCESS_METRICS}}` | Measurement criteria | "95% auth success rate" |

## Directory Structure

When a concept is created, it generates:

```
concepts/{concept-slug}/
├── CONCEPT.md              ← From CONCEPT.md.tmpl
├── impact-matrix.md        ← From impact-matrix.md.tmpl
├── API-SPEC.md            ← Optional, if --api-spec flag
├── acceptance-criteria.md  ← Optional, if --ac flag
└── wireframes/             ← Optional, if --wireframes flag
```

## Usage

Templates are applied by the `create-concept` workflow:

```bash
# Basic concept (CONCEPT.md + impact-matrix.md)
/wgsd create-concept oauth-integration

# With optional artifacts
/wgsd create-concept oauth-integration --api-spec --ac

# With worktree
/wgsd create-concept oauth-integration --worktree
```

## Customization

To customize templates for your project:

1. Copy template to `.planning/templates/concept-directory/`
2. Modify as needed
3. Project-level templates override skill defaults

---

*WGSD v2.2 - Concept Directory Architecture*

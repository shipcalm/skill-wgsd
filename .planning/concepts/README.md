# Independent Concepts Directory - WGSD v2.3

**Architecture:** Independent Concept Management  
**Version:** v2.3.0  
**Created:** 2026-02-23

---

## Overview

This directory contains **independent concepts** that are no longer owned by specific focus groups. Concepts can impact multiple focus groups and are managed through UUID-based identification.

**Key Benefits:**
- ✅ **No Duplication** - Concepts can't be accidentally copied across focus groups
- ✅ **Cross-Cutting Impact** - Natural many-to-many concept ↔ focus group relationships
- ✅ **Merge/Split Operations** - Dynamic concept lifecycle management
- ✅ **UUID-Based Tracking** - Reliable concept identification and references

---

## Architecture Change

### Before (v2.2)
```
.planning/focus-groups/
├── security/concepts/oauth-integration/
├── api/concepts/rate-limiting/
└── frontend/concepts/component-lib/
```

### After (v2.3)
```
.planning/concepts/
├── oauth-integration-a1b2c3d4/     # UUID-based naming
├── rate-limiting-e5f6g7h8/         # Independent of focus groups
└── component-library-i9j0k1l2/     # Cross-cutting by design
```

---

## Concept Structure

Each concept is a directory with:

```
{concept-uuid}/
├── CONCEPT.md              # Main concept description
├── impact-matrix.md        # Cross-focus-group impacts
├── API-SPEC.md            # Optional API specification
├── acceptance-criteria.md  # Optional acceptance criteria
└── metadata.yml           # UUID, relationships, version tracking
```

---

## Key Operations

### Concept Management
- `/wgsd create-concept [name]` - Create new independent concept
- `/wgsd merge-concepts source target` - Merge related concepts  
- `/wgsd split-concept parent child1 child2` - Split complex concepts
- `/wgsd list-concepts` - Show all independent concepts

### Focus Group Operations
- `/wgsd merge-focus-groups source target` - Consolidate focus groups
- `/wgsd split-focus-group parent child1 child2` - Split focus groups
- `/wgsd assign-concept concept focus-group` - Add concept impact

### Migration
- `/wgsd migrate-to-v23` - Automated v2.2 → v2.3 migration
- `/wgsd validate-v23` - Verify v2.3 installation integrity

---

## UUID System

Concepts use UUID-based identification:
- **Format:** `{human-name}-{uuid-suffix}`
- **Example:** `oauth-integration-a1b2c3d4`
- **Registry:** `.concept-ids.json` tracks all UUIDs
- **Benefits:** Unique identification, merge tracking, relationship management

---

## Focus Group Integration

Focus groups reference independent concepts via:

```markdown
## Active Concepts

- [OAuth Integration](../concepts/oauth-integration-a1b2c3d4/) - P0, Security + API impact
- [Rate Limiting](../concepts/rate-limiting-e5f6g7h8/) - P1, API impact
```

---

## Validation

The system includes comprehensive validation:
- Concept UUID uniqueness
- Impact matrix consistency  
- Focus group relationship integrity
- Metadata format validation

Run `/wgsd validate-v23` to check system health.

---

*WGSD v2.3 Independent Concept Architecture - Transforming collaborative development*
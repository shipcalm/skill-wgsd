```yaml
---
concept: test-concept
status: draft
created: 2026-02-23
updated: 2026-02-23
impacts:
  - focus_group: security
    priority: P1
    type: security
    description: "Test concept introduces new authentication mechanism"
    status: pending
    approver: null
    approved_date: null
    notified: false
    notified_date: null
  - focus_group: api
    priority: P2
    type: api-change
    description: "New endpoints will be added to support test concept features"
    status: pending
    approver: null
    approved_date: null
    notified: false
    notified_date: null
---
```

# Impact Matrix: test-concept

This file tracks which focus groups are impacted by the `test-concept` concept and their approval status.

## Cross-Cutting Impacts

This concept impacts the following focus groups:

### 🔐 Security Focus Group (P1)
**Type:** security  
**Description:** Test concept introduces new authentication mechanism  
**Status:** ⏳ Pending review  
**SLA:** 24 hours  

### 🔌 API Focus Group (P2)
**Type:** api-change  
**Description:** New endpoints will be added to support test concept features  
**Status:** ⏳ Pending review  
**SLA:** 72 hours  

## Approval Matrix

| Focus Group | Priority | Type | Status | Approver | Date |
|-------------|----------|------|---------|----------|------|
| security    | P1       | security | pending |          |      |
| api         | P2       | api-change | pending |        |      |

## Notes

This concept has been declared to impact security and api focus groups. Each focus group must review and approve before this concept can proceed to implementation.

**Next Steps:**
1. Security focus group should review the authentication mechanism impact
2. API focus group should review the endpoint changes
3. Use `/wgsd approve test-concept` in each focus group channel when ready
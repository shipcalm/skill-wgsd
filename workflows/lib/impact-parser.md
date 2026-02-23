---
name: wgsd:lib:impact-parser
description: Parse and validate impact-matrix.md files for cross-cutting concern tracking
---

# Impact Parser Library

Parse, validate, and manipulate impact-matrix.md files for WGSD concepts.

---

## Schema Definition

### Impact Object Schema

```yaml
impact:
  focus_group: string        # Required: Focus group slug (e.g., "security", "api")
  priority: enum             # Required: P0 | P1 | P2 | P3
  type: enum                 # Required: See impact types below
  description: string        # Required: Human-readable impact description
  status: enum               # Required: pending | approved | rejected | blocked
  approver: string | null    # Optional: User who approved (null if pending)
  approved_date: string | null  # Optional: ISO date of approval
  notified: boolean          # Optional: Whether notification was sent (default: false)
  notified_date: string | null  # Optional: When notification was sent
```

### Valid Values

**Priority:**
- `P0` - Critical (4h SLA)
- `P1` - High (24h SLA)
- `P2` - Medium (72h SLA)
- `P3` - Low (7d SLA)

**Impact Types:**
- `primary-owner` - Focus group owns this concept
- `breaking-change` - API/behavior breaking changes
- `api-change` - Non-breaking API modifications
- `documentation` - Documentation updates required
- `integration` - Integration touchpoints affected
- `testing` - Test suite changes needed
- `behavior` - Behavior changes (non-breaking)
- `security` - Security implications
- `performance` - Performance considerations

**Status:**
- `pending` - Awaiting review
- `approved` - Focus group approved
- `rejected` - Focus group rejected
- `blocked` - Waiting on dependency

---

## impact_parse_file

Extract impacts array from impact-matrix.md frontmatter.

```bash
# Usage: impact_parse_file <impact_matrix_path>
# Returns: JSON array of impacts or error
impact_parse_file() {
  local file_path="$1"
  
  if [ ! -f "$file_path" ]; then
    echo "ERROR: Impact matrix file not found: $file_path"
    return 1
  fi
  
  # Extract YAML frontmatter between ```yaml and ```
  local yaml_content=$(sed -n '/^```yaml$/,/^```$/p' "$file_path" | sed '1d;$d')
  
  if [ -z "$yaml_content" ]; then
    echo "ERROR: No YAML frontmatter found in $file_path"
    return 1
  fi
  
  # Parse impacts array using yq or fallback to grep/awk
  if command -v yq &> /dev/null; then
    echo "$yaml_content" | yq -o=json '.impacts // []'
  else
    # Fallback: Use Python for YAML parsing
    python3 -c "
import sys
import yaml
import json

content = '''$yaml_content'''
data = yaml.safe_load(content)
impacts = data.get('impacts', [])
print(json.dumps(impacts, indent=2))
" 2>/dev/null || echo "[]"
  fi
  
  return 0
}
```

---

## impact_get_metadata

Extract metadata (concept, status, etc.) from impact-matrix.md.

```bash
# Usage: impact_get_metadata <impact_matrix_path>
# Returns: JSON object with metadata
impact_get_metadata() {
  local file_path="$1"
  
  if [ ! -f "$file_path" ]; then
    echo "ERROR: Impact matrix file not found: $file_path"
    return 1
  fi
  
  local yaml_content=$(sed -n '/^```yaml$/,/^```$/p' "$file_path" | sed '1d;$d')
  
  if [ -z "$yaml_content" ]; then
    echo "ERROR: No YAML frontmatter found"
    return 1
  fi
  
  # Extract top-level fields (excluding impacts array)
  python3 -c "
import yaml
import json

content = '''$yaml_content'''
data = yaml.safe_load(content)
# Remove impacts array for metadata-only output
data.pop('impacts', None)
print(json.dumps(data, indent=2))
" 2>/dev/null || echo "{}"
  
  return 0
}
```

---

## impact_validate

Validate an impacts array against the schema.

```bash
# Usage: impact_validate <json_impacts>
# Returns: Validation result (success or errors)
impact_validate() {
  local impacts_json="$1"
  
  local valid_priorities='["P0", "P1", "P2", "P3"]'
  local valid_types='["primary-owner", "breaking-change", "api-change", "documentation", "integration", "testing", "behavior", "security", "performance"]'
  local valid_statuses='["pending", "approved", "rejected", "blocked"]'
  
  python3 -c "
import json
import sys

impacts = json.loads('''$impacts_json''')
valid_priorities = $valid_priorities
valid_types = $valid_types
valid_statuses = $valid_statuses

errors = []
for i, impact in enumerate(impacts):
    prefix = f'Impact[{i}]'
    
    # Required fields
    if 'focus_group' not in impact or not impact['focus_group']:
        errors.append(f'{prefix}: missing required field \"focus_group\"')
    
    if 'priority' not in impact:
        errors.append(f'{prefix}: missing required field \"priority\"')
    elif impact['priority'] not in valid_priorities:
        errors.append(f'{prefix}: invalid priority \"{impact[\"priority\"]}\" (must be {valid_priorities})')
    
    if 'type' not in impact:
        errors.append(f'{prefix}: missing required field \"type\"')
    elif impact['type'] not in valid_types:
        errors.append(f'{prefix}: invalid type \"{impact[\"type\"]}\" (must be {valid_types})')
    
    if 'description' not in impact or not impact['description']:
        errors.append(f'{prefix}: missing required field \"description\"')
    
    if 'status' not in impact:
        errors.append(f'{prefix}: missing required field \"status\"')
    elif impact['status'] not in valid_statuses:
        errors.append(f'{prefix}: invalid status \"{impact[\"status\"]}\" (must be {valid_statuses})')

if errors:
    print('INVALID')
    for err in errors:
        print(f'  - {err}')
    sys.exit(1)
else:
    print('VALID')
    print(f'  {len(impacts)} impact(s) validated')
    sys.exit(0)
"
  return $?
}
```

---

## impact_add

Add a new impact to an impact-matrix.md file.

```bash
# Usage: impact_add <impact_matrix_path> <focus_group> <priority> <type> <description>
# Returns: Success message or error
impact_add() {
  local file_path="$1"
  local focus_group="$2"
  local priority="$3"
  local type="$4"
  local description="$5"
  
  if [ ! -f "$file_path" ]; then
    echo "ERROR: Impact matrix file not found: $file_path"
    return 1
  fi
  
  # Validate inputs
  case "$priority" in
    P0|P1|P2|P3) ;;
    *) echo "ERROR: Invalid priority '$priority' (must be P0, P1, P2, or P3)"; return 1 ;;
  esac
  
  local valid_types="primary-owner breaking-change api-change documentation integration testing behavior security performance"
  if ! echo "$valid_types" | grep -qw "$type"; then
    echo "ERROR: Invalid type '$type'"
    echo "Valid types: $valid_types"
    return 1
  fi
  
  # Check if focus group already has an impact
  local existing=$(impact_parse_file "$file_path")
  if echo "$existing" | grep -q "\"focus_group\": \"$focus_group\""; then
    echo "ERROR: Impact for focus group '$focus_group' already exists"
    echo "Use impact_update to modify existing impacts"
    return 1
  fi
  
  # Create new impact entry
  local today=$(date +%Y-%m-%d)
  local new_impact="  - focus_group: $focus_group
    priority: $priority
    type: $type
    description: \"$description\"
    status: pending
    approver: null
    approved_date: null
    notified: false
    notified_date: null"
  
  # Find the line number after the last impact entry or after "impacts:" line
  # Insert the new impact
  python3 -c "
import re

with open('$file_path', 'r') as f:
    content = f.read()

# Find the YAML block
yaml_match = re.search(r'(\`\`\`yaml\n---\n)(.*?)(---\n\`\`\`)', content, re.DOTALL)
if not yaml_match:
    print('ERROR: Could not find YAML frontmatter')
    exit(1)

yaml_content = yaml_match.group(2)

# Check if impacts: exists
if 'impacts:' not in yaml_content:
    # Add impacts section
    yaml_content = yaml_content.rstrip() + '\nimpacts:\n'

# Find where to insert (after last impact entry or after impacts:)
lines = yaml_content.split('\n')
insert_idx = None
in_impacts = False

for i, line in enumerate(lines):
    if line.strip() == 'impacts:':
        in_impacts = True
        insert_idx = i + 1
    elif in_impacts:
        if line.strip().startswith('- focus_group:'):
            # Track last impact
            pass
        elif line.strip().startswith('#') or (line.strip() and not line.startswith(' ')):
            # End of impacts section
            insert_idx = i
            break
        elif not line.strip():
            # Empty line might be end
            continue
        insert_idx = i + 1

# Insert new impact
new_impact = '''$new_impact'''
lines.insert(insert_idx, new_impact)
new_yaml = '\n'.join(lines)

# Reconstruct file
new_content = yaml_match.group(1) + new_yaml + yaml_match.group(3)
new_content = content[:yaml_match.start()] + new_content + content[yaml_match.end():]

with open('$file_path', 'w') as f:
    f.write(new_content)

print('SUCCESS')
"
  
  if [ $? -eq 0 ]; then
    # Update the 'updated' timestamp
    local today=$(date +%Y-%m-%d)
    sed -i "s/^updated: .*/updated: $today/" "$file_path" 2>/dev/null || true
    
    echo "✅ Added impact: $focus_group ($priority, $type)"
    echo "IMPACT_ADDED:$focus_group:$priority:$type"
    return 0
  else
    return 1
  fi
}
```

---

## impact_update

Update an existing impact in impact-matrix.md.

```bash
# Usage: impact_update <impact_matrix_path> <focus_group> <field> <value>
# Fields: priority, type, description, status, approver, approved_date
# Returns: Success message or error
impact_update() {
  local file_path="$1"
  local focus_group="$2"
  local field="$3"
  local value="$4"
  
  if [ ! -f "$file_path" ]; then
    echo "ERROR: Impact matrix file not found: $file_path"
    return 1
  fi
  
  # Validate field
  local valid_fields="priority type description status approver approved_date notified notified_date"
  if ! echo "$valid_fields" | grep -qw "$field"; then
    echo "ERROR: Invalid field '$field'"
    echo "Valid fields: $valid_fields"
    return 1
  fi
  
  python3 -c "
import re
import yaml

with open('$file_path', 'r') as f:
    content = f.read()

# Extract YAML
yaml_match = re.search(r'\`\`\`yaml\n---\n(.*?)---\n\`\`\`', content, re.DOTALL)
if not yaml_match:
    print('ERROR: Could not find YAML frontmatter')
    exit(1)

yaml_content = yaml_match.group(1)
data = yaml.safe_load(yaml_content)

# Find and update the impact
found = False
for impact in data.get('impacts', []):
    if impact.get('focus_group') == '$focus_group':
        old_value = impact.get('$field')
        # Handle value type conversion
        value = '$value'
        if value == 'null':
            value = None
        elif value == 'true':
            value = True
        elif value == 'false':
            value = False
        
        impact['$field'] = value
        found = True
        print(f'OLD_VALUE:{old_value}')
        print(f'NEW_VALUE:{value}')
        break

if not found:
    print(f'ERROR: No impact found for focus group \"$focus_group\"')
    exit(1)

# Update the updated timestamp
from datetime import date
data['updated'] = date.today().isoformat()

# Rebuild YAML content
new_yaml = yaml.dump(data, default_flow_style=False, allow_unicode=True, sort_keys=False)

# Replace in content
new_content = content[:yaml_match.start()] + '\`\`\`yaml\n---\n' + new_yaml + '---\n\`\`\`' + content[yaml_match.end():]

with open('$file_path', 'w') as f:
    f.write(new_content)

print('SUCCESS')
"
  
  if [ $? -eq 0 ]; then
    echo "✅ Updated $focus_group.$field = $value"
    echo "IMPACT_UPDATED:$focus_group:$field:$value"
    return 0
  else
    return 1
  fi
}
```

---

## impact_remove

Remove an impact from impact-matrix.md.

```bash
# Usage: impact_remove <impact_matrix_path> <focus_group>
# Returns: Success message or error
impact_remove() {
  local file_path="$1"
  local focus_group="$2"
  
  if [ ! -f "$file_path" ]; then
    echo "ERROR: Impact matrix file not found: $file_path"
    return 1
  fi
  
  python3 -c "
import re
import yaml

with open('$file_path', 'r') as f:
    content = f.read()

yaml_match = re.search(r'\`\`\`yaml\n---\n(.*?)---\n\`\`\`', content, re.DOTALL)
if not yaml_match:
    print('ERROR: Could not find YAML frontmatter')
    exit(1)

yaml_content = yaml_match.group(1)
data = yaml.safe_load(yaml_content)

# Find and remove the impact
original_count = len(data.get('impacts', []))
data['impacts'] = [i for i in data.get('impacts', []) if i.get('focus_group') != '$focus_group']
new_count = len(data['impacts'])

if original_count == new_count:
    print(f'ERROR: No impact found for focus group \"$focus_group\"')
    exit(1)

# Update timestamp
from datetime import date
data['updated'] = date.today().isoformat()

new_yaml = yaml.dump(data, default_flow_style=False, allow_unicode=True, sort_keys=False)
new_content = content[:yaml_match.start()] + '\`\`\`yaml\n---\n' + new_yaml + '---\n\`\`\`' + content[yaml_match.end():]

with open('$file_path', 'w') as f:
    f.write(new_content)

print('SUCCESS')
"
  
  if [ $? -eq 0 ]; then
    echo "✅ Removed impact: $focus_group"
    echo "IMPACT_REMOVED:$focus_group"
    return 0
  else
    return 1
  fi
}
```

---

## impact_list_focus_groups

Get list of impacted focus groups from a concept.

```bash
# Usage: impact_list_focus_groups <impact_matrix_path>
# Returns: List of focus group names
impact_list_focus_groups() {
  local file_path="$1"
  
  local impacts=$(impact_parse_file "$file_path")
  if [ $? -ne 0 ]; then
    echo "$impacts"
    return 1
  fi
  
  echo "$impacts" | python3 -c "
import json
import sys
impacts = json.load(sys.stdin)
for impact in impacts:
    print(impact.get('focus_group', ''))
"
  return 0
}
```

---

## impact_get_pending

Get all pending impacts (awaiting approval).

```bash
# Usage: impact_get_pending <impact_matrix_path>
# Returns: JSON array of pending impacts
impact_get_pending() {
  local file_path="$1"
  
  local impacts=$(impact_parse_file "$file_path")
  if [ $? -ne 0 ]; then
    echo "$impacts"
    return 1
  fi
  
  echo "$impacts" | python3 -c "
import json
import sys
impacts = json.load(sys.stdin)
pending = [i for i in impacts if i.get('status') == 'pending']
print(json.dumps(pending, indent=2))
"
  return 0
}
```

---

## impact_get_unnotified

Get impacts that haven't been notified yet.

```bash
# Usage: impact_get_unnotified <impact_matrix_path>
# Returns: JSON array of unnotified impacts
impact_get_unnotified() {
  local file_path="$1"
  
  local impacts=$(impact_parse_file "$file_path")
  if [ $? -ne 0 ]; then
    echo "$impacts"
    return 1
  fi
  
  echo "$impacts" | python3 -c "
import json
import sys
impacts = json.load(sys.stdin)
unnotified = [i for i in impacts if not i.get('notified', False)]
print(json.dumps(unnotified, indent=2))
"
  return 0
}
```

---

## impact_diff

Compare two impact states and identify changes.

```bash
# Usage: impact_diff <old_impacts_json> <new_impacts_json>
# Returns: JSON object with added, removed, modified impacts
impact_diff() {
  local old_json="$1"
  local new_json="$2"
  
  python3 -c "
import json

old = json.loads('''$old_json''')
new = json.loads('''$new_json''')

old_by_fg = {i['focus_group']: i for i in old}
new_by_fg = {i['focus_group']: i for i in new}

old_fgs = set(old_by_fg.keys())
new_fgs = set(new_by_fg.keys())

added = list(new_fgs - old_fgs)
removed = list(old_fgs - new_fgs)
modified = []

for fg in old_fgs & new_fgs:
    old_impact = old_by_fg[fg]
    new_impact = new_by_fg[fg]
    
    changes = []
    for key in ['priority', 'type', 'description', 'status']:
        if old_impact.get(key) != new_impact.get(key):
            changes.append({
                'field': key,
                'old': old_impact.get(key),
                'new': new_impact.get(key)
            })
    
    if changes:
        modified.append({
            'focus_group': fg,
            'changes': changes
        })

result = {
    'added': added,
    'removed': removed,
    'modified': modified,
    'has_changes': bool(added or removed or modified)
}

print(json.dumps(result, indent=2))
"
  return 0
}
```

---

## impact_summary

Generate a human-readable summary of impacts.

```bash
# Usage: impact_summary <impact_matrix_path>
# Returns: Formatted summary
impact_summary() {
  local file_path="$1"
  
  local impacts=$(impact_parse_file "$file_path")
  local metadata=$(impact_get_metadata "$file_path")
  
  if [ $? -ne 0 ]; then
    echo "ERROR: Could not parse impact matrix"
    return 1
  fi
  
  python3 -c "
import json

impacts = json.loads('''$impacts''')
metadata = json.loads('''$metadata''')

concept = metadata.get('concept', 'Unknown')
status = metadata.get('status', 'draft')

print(f'📋 Impact Summary: {concept}')
print(f'   Status: {status}')
print(f'   Total Impacts: {len(impacts)}')
print()

status_emoji = {
    'pending': '⏳',
    'approved': '✅',
    'rejected': '🚫',
    'blocked': '⏸️'
}

priority_emoji = {
    'P0': '🔴',
    'P1': '🟠',
    'P2': '🟡',
    'P3': '🟢'
}

for impact in impacts:
    fg = impact.get('focus_group', '?')
    priority = impact.get('priority', '?')
    itype = impact.get('type', '?')
    istatus = impact.get('status', 'pending')
    desc = impact.get('description', '')
    
    p_icon = priority_emoji.get(priority, '⚪')
    s_icon = status_emoji.get(istatus, '❓')
    
    print(f'{s_icon} {fg}: {p_icon} {priority} ({itype})')
    if desc:
        print(f'   └─ {desc}')
"
  return 0
}
```

---

## Usage Examples

```bash
# Parse impacts from a file
impacts=$(impact_parse_file "concepts/oauth-integration/impact-matrix.md")

# Validate impacts
impact_validate "$impacts"

# Add a new impact
impact_add "concepts/oauth-integration/impact-matrix.md" \
  "security" "P1" "integration" "Token validation flow changes"

# Update an impact
impact_update "concepts/oauth-integration/impact-matrix.md" \
  "security" "status" "approved"

# Remove an impact
impact_remove "concepts/oauth-integration/impact-matrix.md" "frontend"

# Get unnotified impacts (for notification system)
impact_get_unnotified "concepts/oauth-integration/impact-matrix.md"

# Compare old vs new impacts (for change tracking)
impact_diff "$old_impacts" "$new_impacts"

# Print summary
impact_summary "concepts/oauth-integration/impact-matrix.md"
```

---

*Library created for WGSD Phase 11 - Cross-Cutting Impact System*

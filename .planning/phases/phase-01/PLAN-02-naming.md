# Plan 1.2: Naming Conventions

**Requirements:** CHANNEL-01, CHANNEL-02
**Estimated Duration:** 0.5 day
**Dependencies:** None (can run parallel to Plan 1.1)

---

## Objective

Implement standardized naming conventions for Slack channels with stub collection and validation.

---

## Requirements Addressed

### CHANNEL-01: Ask for repo slack stub during setup
- Prompt user for short Slack channel prefix
- Suggest stub from repository name
- Validate stub format (2-5 lowercase characters)
- Store stub in workspace configuration

### CHANNEL-02: Create standardized channel naming convention implementation
- Define naming patterns for all channel types
- Implement validator that rejects non-conforming names
- Auto-correct common mistakes with confirmation
- Provide clear error messages for invalid names

---

## Naming Convention Specification

### Channel Types

| Type | Pattern | Example | Description |
|------|---------|---------|-------------|
| Core Dev | `{stub}-dev` | `mvn-dev` | Main development community |
| Focus Group | `{stub}-fg-{name}` | `mvn-fg-security` | Long-lived topic channels |
| Concept | `{stub}-cpt-{name}` | `mvn-cpt-byof` | Concept discussion channels |
| Implementation | `{stub}-impl-{name}` | `mvn-impl-auth-v2` | Short-lived execution channels |
| Community | `{stub}-community` | `mvn-community` | Public feedback channel |

### Naming Rules

1. **Stub Format**
   - 2-5 lowercase letters only
   - No numbers, no special characters
   - Examples: `mvn`, `oc`, `wgsd`, `mvnos`

2. **Name Format** (for fg, cpt, impl)
   - Lowercase letters, numbers, and hyphens
   - No underscores, no special characters
   - 1-30 characters
   - No leading/trailing hyphens
   - Examples: `security`, `auth-v2`, `byof-filesystem`

3. **Full Channel Format**
   - Maximum 80 characters (Slack limit)
   - Must match pattern: `{stub}-{type}-{name}` or `{stub}-{special}`

---

## Deliverables

### 1. Library: `lib/naming.md`

**Functions:**

```
validate_stub(stub)         → OK or error with suggestion
validate_name(name)         → OK or error with corrected version
generate_channel_name(stub, type, name) → Full channel name
parse_channel_name(channel) → {stub, type, name} or error
suggest_stub(repo_name)     → Suggested stub from repo name
auto_correct_name(name)     → Corrected version of name
```

### 2. Workflow Update: `workflows/init.md`

Add stub collection step:
```
1. Suggest stub from repository name
2. Prompt user to confirm or override
3. Validate stub format
4. Store in .planning/WGSD-CONFIG.md
```

### 3. Workflow Update: `workflows/create-channel.md`

Add naming validation:
```
1. Parse requested channel name
2. Validate against naming rules
3. Auto-correct if needed (with confirmation)
4. Reject if uncorrectable
5. Proceed with creation
```

### 4. Configuration: `.planning/WGSD-CONFIG.md` update

Add stub storage:
```markdown
## Project Configuration

**Slack Stub:** mvn
**Repository:** marvin
**Full Channel Prefix:** mvn-
```

---

## Implementation Steps

### Step 1: Create naming library (45 min)

Location: `workflows/lib/naming.md`

```markdown
---
name: wgsd:lib:naming
description: Channel naming conventions library
---

## validate_stub

<operation>
<input>stub</input>
<process>
# Check length
if [ ${#stub} -lt 2 ] || [ ${#stub} -gt 5 ]; then
  echo "ERROR: Stub must be 2-5 characters (got ${#stub})"
  exit 1
fi

# Check format (lowercase letters only)
if ! echo "$stub" | grep -qE '^[a-z]+$'; then
  echo "ERROR: Stub must be lowercase letters only"
  echo "SUGGESTION: $(echo "$stub" | tr '[:upper:]' '[:lower:]' | tr -cd 'a-z' | cut -c1-5)"
  exit 1
fi

echo "OK: Stub '$stub' is valid"
</process>
</operation>

## suggest_stub

<operation>
<input>repo_name</input>
<process>
# Common repo-to-stub mappings
case "$repo_name" in
  marvin|marvin-*) echo "mvn" ;;
  openclaw|openclaw-*) echo "oc" ;;
  *)
    # Take first 3-4 consonants or first 3 chars
    SUGGESTION=$(echo "$repo_name" | tr '[:upper:]' '[:lower:]' | tr -cd 'a-z' | cut -c1-4)
    if [ ${#SUGGESTION} -lt 2 ]; then
      SUGGESTION=$(echo "$repo_name" | tr '[:upper:]' '[:lower:]' | cut -c1-3)
    fi
    echo "$SUGGESTION"
    ;;
esac
</process>
</operation>

## validate_name

<operation>
<input>name</input>
<process>
# Check length
if [ ${#name} -lt 1 ] || [ ${#name} -gt 30 ]; then
  echo "ERROR: Name must be 1-30 characters"
  exit 1
fi

# Check format
if ! echo "$name" | grep -qE '^[a-z0-9]+(-[a-z0-9]+)*$'; then
  CORRECTED=$(echo "$name" | tr '[:upper:]' '[:lower:]' | tr '_' '-' | tr -cd 'a-z0-9-' | sed 's/--*/-/g' | sed 's/^-//' | sed 's/-$//')
  echo "ERROR: Invalid name format"
  echo "CORRECTED: $CORRECTED"
  exit 1
fi

echo "OK: Name '$name' is valid"
</process>
</operation>

## generate_channel_name

<operation>
<input>stub, type, name</input>
<process>
case "$type" in
  dev) CHANNEL="${stub}-dev" ;;
  community) CHANNEL="${stub}-community" ;;
  fg|focus-group) CHANNEL="${stub}-fg-${name}" ;;
  cpt|concept) CHANNEL="${stub}-cpt-${name}" ;;
  impl|implementation) CHANNEL="${stub}-impl-${name}" ;;
  *)
    echo "ERROR: Unknown channel type: $type"
    exit 1
    ;;
esac

# Check total length
if [ ${#CHANNEL} -gt 80 ]; then
  echo "ERROR: Channel name too long (${#CHANNEL} chars, max 80)"
  exit 1
fi

echo "$CHANNEL"
</process>
</operation>

## parse_channel_name

<operation>
<input>channel_name</input>
<process>
# Remove leading # if present
CHANNEL=$(echo "$channel_name" | sed 's/^#//')

# Parse the channel name
if echo "$CHANNEL" | grep -qE '^[a-z]+-dev$'; then
  STUB=$(echo "$CHANNEL" | sed 's/-dev$//')
  echo "stub=$STUB type=dev name="
elif echo "$CHANNEL" | grep -qE '^[a-z]+-community$'; then
  STUB=$(echo "$CHANNEL" | sed 's/-community$//')
  echo "stub=$STUB type=community name="
elif echo "$CHANNEL" | grep -qE '^[a-z]+-fg-'; then
  STUB=$(echo "$CHANNEL" | sed 's/-fg-.*//')
  NAME=$(echo "$CHANNEL" | sed 's/^[a-z]*-fg-//')
  echo "stub=$STUB type=fg name=$NAME"
elif echo "$CHANNEL" | grep -qE '^[a-z]+-cpt-'; then
  STUB=$(echo "$CHANNEL" | sed 's/-cpt-.*//')
  NAME=$(echo "$CHANNEL" | sed 's/^[a-z]*-cpt-//')
  echo "stub=$STUB type=cpt name=$NAME"
elif echo "$CHANNEL" | grep -qE '^[a-z]+-impl-'; then
  STUB=$(echo "$CHANNEL" | sed 's/-impl-.*//')
  NAME=$(echo "$CHANNEL" | sed 's/^[a-z]*-impl-//')
  echo "stub=$STUB type=impl name=$NAME"
else
  echo "ERROR: Unrecognized channel format"
  exit 1
fi
</process>
</operation>
```

### Step 2: Update init workflow for stub collection (20 min)

Add to `workflows/init.md`:

```markdown
## Step: Collect Slack Stub

<process>
1. Call suggest_stub(repo_name) to get suggestion
2. Prompt user: "Slack channel stub? (suggested: {suggestion})"
3. Validate with validate_stub()
4. If invalid, show error and re-prompt
5. Store in WGSD-CONFIG.md
</process>

<interaction>
📱 **Slack Channel Stub**

Your project will use channels like `#{stub}-dev`, `#{stub}-fg-security`, etc.

Suggested stub for "{repo_name}": **{suggestion}**

Enter stub (or press Enter for suggested): _
</interaction>
```

### Step 3: Update create-channel with validation (20 min)

Add validation step to `workflows/create-channel.md`:

```markdown
## Step: Validate Channel Name

<process>
1. Parse requested name with parse_channel_name()
2. If parsing fails, check if it's a partial name
3. For partial names, construct full name with stub
4. Validate all components
5. If invalid, offer auto-corrected version
6. Proceed or abort based on user confirmation
</process>
```

### Step 4: Create user-facing documentation (15 min)

Add to SKILL.md intake section:

```markdown
**Naming Rules:**
- Stubs: 2-5 lowercase letters (e.g., `mvn`, `oc`)
- Names: lowercase letters, numbers, hyphens (e.g., `auth-v2`)
- Invalid characters are auto-corrected with confirmation
```

---

## Verification Checklist

### Stub Validation Tests

- [ ] `mvn` → Valid
- [ ] `oc` → Valid
- [ ] `wgsd` → Valid
- [ ] `m` → Invalid (too short)
- [ ] `marvin` → Invalid (too long)
- [ ] `MVN` → Invalid (uppercase) → Suggests `mvn`
- [ ] `mvn2` → Invalid (number) → Suggests `mvn`

### Name Validation Tests

- [ ] `security` → Valid
- [ ] `auth-v2` → Valid
- [ ] `byof-filesystem` → Valid
- [ ] `Auth_V2` → Invalid → Corrects to `auth-v2`
- [ ] `security!!` → Invalid → Corrects to `security`
- [ ] `--security--` → Invalid → Corrects to `security`

### Channel Generation Tests

- [ ] `generate_channel_name(mvn, dev, "")` → `mvn-dev`
- [ ] `generate_channel_name(mvn, fg, security)` → `mvn-fg-security`
- [ ] `generate_channel_name(mvn, impl, auth-v2)` → `mvn-impl-auth-v2`

### Parsing Tests

- [ ] `parse_channel_name(mvn-dev)` → `stub=mvn type=dev`
- [ ] `parse_channel_name(#mvn-fg-security)` → `stub=mvn type=fg name=security`
- [ ] `parse_channel_name(random-name)` → Error

---

## Success Criteria

| Criterion | Measurement |
|-----------|-------------|
| Stub collection works | User prompted and value stored |
| Validation catches errors | All invalid inputs rejected |
| Auto-correction works | Reasonable corrections suggested |
| Channel names conform | All created channels match pattern |
| Error messages clear | User understands what's wrong |

---

## User Experience

### Happy Path
```
🏠 Initializing WGSD workspace...

📱 Slack Channel Stub
   Suggested: mvn (from "marvin")
   Enter stub or press Enter: [Enter]
   
✅ Using stub: mvn
   Channels will be: #mvn-dev, #mvn-fg-*, #mvn-impl-*
```

### Correction Path
```
🏠 Creating focus group: Security_Team

⚠️  Name "Security_Team" needs correction:
   • Contains uppercase letters
   • Contains underscore
   
   Corrected: "security-team"
   
   Use corrected name? (Y/n): y
   
✅ Creating focus group: security-team
```

---

*Plan created: 2026-02-22*
*Ready for execution*

---
name: wgsd:lib:naming
description: Channel naming conventions library for WGSD Slack integration
---

# Naming Conventions Library

Standardized naming validation and generation for WGSD Slack channels.

---

## Channel Naming Convention

### Pattern
```
{stub}-{type}[-{name}]
```

### Types

| Type | Pattern | Example | Description |
|------|---------|---------|-------------|
| dev | `{stub}-dev` | `mvn-dev` | Main development channel |
| community | `{stub}-community` | `mvn-community` | Public feedback channel |
| fg | `{stub}-fg-{name}` | `mvn-fg-security` | Focus group channel |
| cpt | `{stub}-cpt-{name}` | `mvn-cpt-byof` | Concept discussion |
| impl | `{stub}-impl-{name}` | `mvn-impl-auth-v2` | Implementation channel |

### Rules

**Stub:**
- 2-5 lowercase letters only
- No numbers, hyphens, or special characters
- Examples: `mvn`, `oc`, `wgsd`, `acme`

**Name:**
- 1-30 lowercase characters
- Letters, numbers, and hyphens only
- No leading/trailing hyphens
- No consecutive hyphens
- Examples: `security`, `auth-v2`, `byof-filesystem`

**Full Channel:**
- Maximum 80 characters (Slack limit)
- Must match pattern exactly

---

## validate_stub

Validate a Slack channel stub.

```bash
# Usage: validate_stub <stub>
# Returns: 0 if valid, 1 if invalid (with error message)
validate_stub() {
  local stub="$1"
  
  # Check if empty
  if [ -z "$stub" ]; then
    echo "ERROR: Stub cannot be empty"
    return 1
  fi
  
  # Check length
  local len=${#stub}
  if [ "$len" -lt 2 ]; then
    echo "ERROR: Stub too short (minimum 2 characters, got $len)"
    echo "SUGGESTION: Add more letters"
    return 1
  fi
  if [ "$len" -gt 5 ]; then
    echo "ERROR: Stub too long (maximum 5 characters, got $len)"
    echo "SUGGESTION: $(echo "$stub" | cut -c1-4)"
    return 1
  fi
  
  # Check format (lowercase letters only)
  if ! echo "$stub" | grep -qE '^[a-z]+$'; then
    local corrected=$(echo "$stub" | tr '[:upper:]' '[:lower:]' | tr -cd 'a-z')
    echo "ERROR: Stub must be lowercase letters only"
    echo "SUGGESTION: $corrected"
    return 1
  fi
  
  echo "OK: Stub '$stub' is valid"
  return 0
}
```

---

## suggest_stub

Generate a suggested stub from repository or project name.

```bash
# Usage: suggest_stub <repo_name>
# Returns: Suggested stub (stdout)
suggest_stub() {
  local name="$1"
  
  # Known mappings
  case "$name" in
    marvin|marvin-*) echo "mvn"; return 0 ;;
    openclaw|openclaw-*) echo "oc"; return 0 ;;
    skill-wgsd|wgsd-*) echo "wgsd"; return 0 ;;
  esac
  
  # Clean the name
  local clean=$(echo "$name" | tr '[:upper:]' '[:lower:]' | tr -cd 'a-z')
  
  # Try to extract meaningful abbreviation
  if [ ${#clean} -le 5 ]; then
    # Short enough to use directly
    echo "$clean"
  elif [ ${#clean} -le 8 ]; then
    # Take first 4 characters
    echo "${clean:0:4}"
  else
    # Take first 3 characters
    echo "${clean:0:3}"
  fi
}
```

---

## validate_name

Validate a channel name component (the part after the type prefix).

```bash
# Usage: validate_name <name>
# Returns: 0 if valid, 1 if invalid (with error message and corrected version)
validate_name() {
  local name="$1"
  
  # Check if empty
  if [ -z "$name" ]; then
    echo "ERROR: Name cannot be empty"
    return 1
  fi
  
  # Check length
  local len=${#name}
  if [ "$len" -gt 30 ]; then
    echo "ERROR: Name too long (maximum 30 characters, got $len)"
    echo "CORRECTED: $(echo "$name" | cut -c1-30)"
    return 1
  fi
  
  # Check format
  if ! echo "$name" | grep -qE '^[a-z0-9]+(-[a-z0-9]+)*$'; then
    # Auto-correct the name
    local corrected=$(echo "$name" | \
      tr '[:upper:]' '[:lower:]' | \
      tr '_' '-' | \
      tr -cd 'a-z0-9-' | \
      sed 's/--*/-/g' | \
      sed 's/^-//' | \
      sed 's/-$//')
    
    if [ -z "$corrected" ]; then
      echo "ERROR: Name contains no valid characters"
      return 1
    fi
    
    echo "ERROR: Invalid name format"
    echo "CORRECTED: $corrected"
    return 1
  fi
  
  echo "OK: Name '$name' is valid"
  return 0
}
```

---

## auto_correct_name

Automatically correct an invalid name to valid format.

```bash
# Usage: auto_correct_name <name>
# Returns: Corrected name (stdout)
auto_correct_name() {
  local name="$1"
  
  echo "$name" | \
    tr '[:upper:]' '[:lower:]' | \
    tr '_' '-' | \
    tr ' ' '-' | \
    tr -cd 'a-z0-9-' | \
    sed 's/--*/-/g' | \
    sed 's/^-//' | \
    sed 's/-$//' | \
    cut -c1-30
}
```

---

## generate_channel_name

Generate a full channel name from components.

```bash
# Usage: generate_channel_name <stub> <type> [name]
# Returns: Full channel name (stdout) or error
generate_channel_name() {
  local stub="$1"
  local type="$2"
  local name="$3"
  
  local channel=""
  
  case "$type" in
    dev)
      channel="${stub}-dev"
      ;;
    community)
      channel="${stub}-community"
      ;;
    fg|focus-group)
      if [ -z "$name" ]; then
        echo "ERROR: Focus group requires a name"
        return 1
      fi
      channel="${stub}-fg-${name}"
      ;;
    cpt|concept)
      if [ -z "$name" ]; then
        echo "ERROR: Concept requires a name"
        return 1
      fi
      channel="${stub}-cpt-${name}"
      ;;
    impl|implementation)
      if [ -z "$name" ]; then
        echo "ERROR: Implementation requires a name"
        return 1
      fi
      channel="${stub}-impl-${name}"
      ;;
    *)
      echo "ERROR: Unknown channel type: $type"
      echo "VALID_TYPES: dev, community, fg, cpt, impl"
      return 1
      ;;
  esac
  
  # Check total length
  local len=${#channel}
  if [ "$len" -gt 80 ]; then
    echo "ERROR: Channel name too long ($len chars, max 80)"
    return 1
  fi
  
  echo "$channel"
  return 0
}
```

---

## parse_channel_name

Parse a channel name into its components.

```bash
# Usage: parse_channel_name <channel_name>
# Returns: stub=X type=X name=X (stdout) or error
parse_channel_name() {
  local channel="$1"
  
  # Remove leading # if present
  channel=$(echo "$channel" | sed 's/^#//')
  
  # Match patterns in order of specificity
  
  # Focus group: {stub}-fg-{name}
  if echo "$channel" | grep -qE '^[a-z]+-fg-[a-z0-9-]+$'; then
    local stub=$(echo "$channel" | sed 's/-fg-.*//')
    local name=$(echo "$channel" | sed 's/^[a-z]*-fg-//')
    echo "stub=$stub type=fg name=$name"
    return 0
  fi
  
  # Concept: {stub}-cpt-{name}
  if echo "$channel" | grep -qE '^[a-z]+-cpt-[a-z0-9-]+$'; then
    local stub=$(echo "$channel" | sed 's/-cpt-.*//')
    local name=$(echo "$channel" | sed 's/^[a-z]*-cpt-//')
    echo "stub=$stub type=cpt name=$name"
    return 0
  fi
  
  # Implementation: {stub}-impl-{name}
  if echo "$channel" | grep -qE '^[a-z]+-impl-[a-z0-9-]+$'; then
    local stub=$(echo "$channel" | sed 's/-impl-.*//')
    local name=$(echo "$channel" | sed 's/^[a-z]*-impl-//')
    echo "stub=$stub type=impl name=$name"
    return 0
  fi
  
  # Dev: {stub}-dev
  if echo "$channel" | grep -qE '^[a-z]+-dev$'; then
    local stub=$(echo "$channel" | sed 's/-dev$//')
    echo "stub=$stub type=dev name="
    return 0
  fi
  
  # Community: {stub}-community
  if echo "$channel" | grep -qE '^[a-z]+-community$'; then
    local stub=$(echo "$channel" | sed 's/-community$//')
    echo "stub=$stub type=community name="
    return 0
  fi
  
  echo "ERROR: Unrecognized channel format: $channel"
  echo "EXPECTED: {stub}-{type}[-{name}]"
  echo "EXAMPLES: mvn-dev, mvn-fg-security, mvn-impl-auth-v2"
  return 1
}
```

---

## validate_channel_name

Full validation of a channel name against WGSD conventions.

```bash
# Usage: validate_channel_name <channel_name>
# Returns: 0 if valid, 1 if invalid (with detailed error)
validate_channel_name() {
  local channel="$1"
  
  # Remove leading # if present
  channel=$(echo "$channel" | sed 's/^#//')
  
  # Check total length
  if [ ${#channel} -gt 80 ]; then
    echo "ERROR: Channel name too long (${#channel} chars, max 80)"
    return 1
  fi
  
  # Try to parse
  local parsed=$(parse_channel_name "$channel")
  if [ $? -ne 0 ]; then
    echo "$parsed"
    return 1
  fi
  
  # Extract components
  local stub=$(echo "$parsed" | sed 's/.*stub=\([^ ]*\).*/\1/')
  local type=$(echo "$parsed" | sed 's/.*type=\([^ ]*\).*/\1/')
  local name=$(echo "$parsed" | sed 's/.*name=\([^ ]*\).*/\1/')
  
  # Validate stub
  local stub_result=$(validate_stub "$stub")
  if [ $? -ne 0 ]; then
    echo "$stub_result"
    return 1
  fi
  
  # Validate name if present
  if [ -n "$name" ]; then
    local name_result=$(validate_name "$name")
    if [ $? -ne 0 ]; then
      echo "$name_result"
      return 1
    fi
  fi
  
  echo "OK: Channel '$channel' is valid"
  echo "PARSED: $parsed"
  return 0
}
```

---

## Usage Examples

```bash
# Validate a stub
validate_stub "mvn"      # OK
validate_stub "MARVIN"   # ERROR: suggests "marvin" (too long), then "marv"

# Suggest a stub
suggest_stub "my-awesome-project"  # Returns: "mya" or similar

# Generate channel names
generate_channel_name "mvn" "dev"                    # mvn-dev
generate_channel_name "mvn" "fg" "security"          # mvn-fg-security
generate_channel_name "mvn" "impl" "auth-v2"         # mvn-impl-auth-v2

# Parse channel names
parse_channel_name "mvn-fg-security"   # stub=mvn type=fg name=security
parse_channel_name "#mvn-dev"          # stub=mvn type=dev name=

# Auto-correct invalid names
auto_correct_name "Security_Team"      # security-team
auto_correct_name "Auth V2!!!"         # auth-v2
```

---

*Library created for WGSD Phase 1*

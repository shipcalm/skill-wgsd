---
name: wgsd:apply-channel-mapping
description: Apply channel-to-directory mapping for the current session
allowed-tools:
  - Read
  - Write
  - Bash
---

# Apply Channel Mapping

Automatically sets the working directory based on the current Slack channel using the WGSD channel mappings configuration.

---

## Objective

When invoked in a Slack channel:
1. Read the channel mapping from `wgsd-config.json`
2. Change to the configured working directory for that channel
3. Set up git context if enabled
4. Confirm the mapping is active

---

## Process

### Step 1: Read Channel Mapping Configuration

```bash
# Read the WGSD config
CONFIG_FILE="github/skill-wgsd/wgsd-config.json"
if [ ! -f "$CONFIG_FILE" ]; then
  echo "❌ WGSD config file not found: $CONFIG_FILE"
  exit 1
fi

CHANNEL_ID="$1"
if [ -z "$CHANNEL_ID" ]; then
  echo "❌ Channel ID required"
  exit 1
fi
```

### Step 2: Extract Channel Mapping

```bash
# Extract the working directory for this channel
WORKING_DIR=$(jq -r ".channel_mappings[\"$CHANNEL_ID\"].working_directory // empty" "$CONFIG_FILE")
CHANNEL_NAME=$(jq -r ".channel_mappings[\"$CHANNEL_ID\"].channel_name // empty" "$CONFIG_FILE")
AUTO_CD=$(jq -r ".channel_mappings[\"$CHANNEL_ID\"].auto_cd // false" "$CONFIG_FILE")
GIT_INTEGRATION=$(jq -r ".channel_mappings[\"$CHANNEL_ID\"].git_integration // false" "$CONFIG_FILE")

if [ -z "$WORKING_DIR" ]; then
  echo "ℹ️  No mapping configured for channel $CHANNEL_ID"
  exit 0
fi
```

### Step 3: Apply Directory Mapping

```bash
# Get base directory from config
BASE_DIR=$(jq -r ".workspace_config.base_directory // \"/home/jarvis/.openclaw/workspace\"" "$CONFIG_FILE")
FULL_PATH="$BASE_DIR/$WORKING_DIR"

if [ ! -d "$FULL_PATH" ]; then
  echo "❌ Mapped directory does not exist: $FULL_PATH"
  exit 1
fi

# Change to the mapped directory
cd "$FULL_PATH" || exit 1
echo "✅ Mapped channel $CHANNEL_NAME to directory: $WORKING_DIR"
echo "📍 Current working directory: $(pwd)"
```

### Step 4: Git Integration (Optional)

```bash
if [ "$GIT_INTEGRATION" = "true" ] && [ -d ".git" ]; then
  # Show git status
  echo ""
  echo "🔄 Git Status:"
  git status --short
  
  # Show current branch
  CURRENT_BRANCH=$(git branch --show-current)
  echo "🌿 Current branch: $CURRENT_BRANCH"
fi
```

---

## Success Criteria

- [ ] Channel mapping configuration is read successfully
- [ ] Working directory exists and is accessible  
- [ ] Current session switches to mapped directory
- [ ] Git context is displayed if enabled
- [ ] Clear confirmation message shown

---

## Error Handling

- Missing config file → Create default config
- Unknown channel → No action taken (silent)
- Missing directory → Error with clear message
- Git errors → Continue without git integration
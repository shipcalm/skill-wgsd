---
name: wgsd:init
description: Initialize a new WGSD workspace from a git repository
allowed-tools:
  - Read
  - Write
  - Bash
  - AskUser
---

# WGSD Workspace Initialization

Initialize a new WGSD workspace with proper directory structure, git integration, and configuration.

---

## Objective

Set up a WGSD-enabled development workspace that:
- Creates standardized workspace directory structure
- Clones or links the target repository
- Detects and checks out the primary branch (develop/main)
- Collects project configuration (Slack stub)
- Registers workspace in the WGSD registry

---

## Inputs

| Input | Required | Description | Example |
|-------|----------|-------------|---------|
| repository | Yes | Git URL or local path | `https://github.com/user/repo` or `/path/to/repo` |
| project_name | No | Override derived project name | `marvin` |
| slack_stub | No | Override suggested Slack stub | `mvn` |

---

## Process

### Step 1: Validate Inputs

```bash
# Determine repository type and extract project name
if [[ "$REPOSITORY" == http* ]] || [[ "$REPOSITORY" == git@* ]]; then
  REPO_TYPE="remote"
  # Extract project name from URL
  PROJECT_NAME=${PROJECT_NAME:-$(basename "$REPOSITORY" .git)}
elif [ -d "$REPOSITORY" ]; then
  REPO_TYPE="local"
  PROJECT_NAME=${PROJECT_NAME:-$(basename "$REPOSITORY")}
else
  echo "❌ Invalid repository: $REPOSITORY"
  echo "   Provide a git URL or valid local path"
  exit 1
fi

echo "📦 Project: $PROJECT_NAME"
echo "🔗 Repository: $REPOSITORY ($REPO_TYPE)"
```

### Step 2: Create Workspace Directory

```bash
WGSD_ROOT="$HOME/.openclaw/workspace/wgsd"
WORKSPACE_PATH="$WGSD_ROOT/$PROJECT_NAME"

# Check if workspace already exists
if [ -d "$WORKSPACE_PATH" ]; then
  echo "⚠️  Workspace already exists: $WORKSPACE_PATH"
  echo ""
  read -p "Reinitialize? (y/N): " REINIT
  if [[ ! "$REINIT" =~ ^[Yy] ]]; then
    echo "Aborted. Use 'wgsd status' to check existing workspace."
    exit 0
  fi
  echo "🗑️  Removing existing workspace..."
  rm -rf "$WORKSPACE_PATH"
fi

# Create workspace structure
mkdir -p "$WORKSPACE_PATH"
mkdir -p "$WORKSPACE_PATH/concepts"
mkdir -p "$WORKSPACE_PATH/implementations"
echo "✅ Created workspace: $WORKSPACE_PATH"
```

### Step 3: Clone or Link Repository

```bash
REPO_PATH="$WORKSPACE_PATH/repo"

if [ "$REPO_TYPE" == "remote" ]; then
  echo "📥 Cloning repository..."
  git clone "$REPOSITORY" "$REPO_PATH" 2>&1
  if [ $? -ne 0 ]; then
    echo "❌ Clone failed"
    rm -rf "$WORKSPACE_PATH"
    exit 1
  fi
else
  echo "🔗 Linking local repository..."
  ln -s "$(realpath "$REPOSITORY")" "$REPO_PATH"
fi

cd "$REPO_PATH"
echo "✅ Repository ready: $REPO_PATH"
```

### Step 4: Detect Primary Branch

```bash
# Detect primary branch (develop preferred)
if git rev-parse --verify origin/develop >/dev/null 2>&1; then
  PRIMARY_BRANCH="develop"
elif git rev-parse --verify origin/main >/dev/null 2>&1; then
  PRIMARY_BRANCH="main"
elif git rev-parse --verify develop >/dev/null 2>&1; then
  PRIMARY_BRANCH="develop"
elif git rev-parse --verify main >/dev/null 2>&1; then
  PRIMARY_BRANCH="main"
else
  echo "❌ No primary branch (main or develop) found"
  exit 1
fi

echo "🌿 Primary branch: $PRIMARY_BRANCH"
```

### Step 5: Ensure Clean Checkout

```bash
# Checkout primary branch
git checkout "$PRIMARY_BRANCH" 2>&1

# Verify clean state
DIRTY=$(git status --porcelain | wc -l)
if [ "$DIRTY" -gt 0 ]; then
  echo "⚠️  Repository has uncommitted changes:"
  git status --short
  echo ""
  echo "Commit or stash changes before continuing."
fi

echo "✅ Checked out $PRIMARY_BRANCH (clean)"
```

### Step 6: Collect Slack Stub

<interaction>
📱 **Slack Channel Stub**

WGSD channels follow the pattern: `#{stub}-dev`, `#{stub}-fg-*`, `#{stub}-impl-*`

**Rules:**
- 2-5 lowercase letters only
- No numbers or special characters
- Examples: `mvn`, `oc`, `wgsd`

</interaction>

```bash
# Suggest stub from project name
suggest_stub() {
  local name="$1"
  case "$name" in
    marvin|marvin-*) echo "mvn" ;;
    openclaw|openclaw-*) echo "oc" ;;
    *)
      # Take first 3-4 characters, lowercase, letters only
      echo "$name" | tr '[:upper:]' '[:lower:]' | tr -cd 'a-z' | cut -c1-4
      ;;
  esac
}

SUGGESTED_STUB=$(suggest_stub "$PROJECT_NAME")
echo "Suggested stub for '$PROJECT_NAME': $SUGGESTED_STUB"

# Validate stub format
validate_stub() {
  local stub="$1"
  if [ ${#stub} -lt 2 ] || [ ${#stub} -gt 5 ]; then
    return 1
  fi
  if ! echo "$stub" | grep -qE '^[a-z]+$'; then
    return 1
  fi
  return 0
}

# Use provided stub or prompt
if [ -n "$SLACK_STUB" ]; then
  if validate_stub "$SLACK_STUB"; then
    FINAL_STUB="$SLACK_STUB"
  else
    echo "❌ Invalid stub: $SLACK_STUB"
    exit 1
  fi
else
  FINAL_STUB="$SUGGESTED_STUB"
fi

echo "✅ Using Slack stub: $FINAL_STUB"
echo "   Channels: #${FINAL_STUB}-dev, #${FINAL_STUB}-fg-*, #${FINAL_STUB}-impl-*"
```

### Step 7: Create WGSD Configuration

```bash
# Ensure .planning directory exists
mkdir -p "$REPO_PATH/.planning"

# Create or update WGSD-CONFIG.md
cat > "$REPO_PATH/.planning/WGSD-CONFIG.md" << EOF
# WGSD Configuration

**Project:** $PROJECT_NAME
**Workspace:** $WORKSPACE_PATH
**Repository:** $REPOSITORY

---

## Slack Configuration

**Stub:** $FINAL_STUB
**Channel Prefix:** #${FINAL_STUB}-

### Channel Patterns
| Type | Pattern | Example |
|------|---------|---------|
| Main Dev | \`${FINAL_STUB}-dev\` | #${FINAL_STUB}-dev |
| Focus Group | \`${FINAL_STUB}-fg-{name}\` | #${FINAL_STUB}-fg-security |
| Concept | \`${FINAL_STUB}-cpt-{name}\` | #${FINAL_STUB}-cpt-byof |
| Implementation | \`${FINAL_STUB}-impl-{name}\` | #${FINAL_STUB}-impl-auth-v2 |
| Community | \`${FINAL_STUB}-community\` | #${FINAL_STUB}-community |

---

## Git Configuration

**Primary Branch:** $PRIMARY_BRANCH
**Focus Groups Base:** focus-groups/
**Implementations Base:** implementations/

---

## Workspace Structure

\`\`\`
$WORKSPACE_PATH/
├── repo/              # Main repository checkout
├── concepts/          # Focus group worktrees
└── implementations/   # Implementation worktrees
\`\`\`

---

*Initialized: $(date -Iseconds)*
EOF

echo "✅ Created WGSD-CONFIG.md"
```

### Step 8: Register Workspace

```bash
REGISTRY_FILE="$WGSD_ROOT/WORKSPACES.md"

# Create registry if it doesn't exist
if [ ! -f "$REGISTRY_FILE" ]; then
  cat > "$REGISTRY_FILE" << 'EOF'
# WGSD Workspaces

Active WGSD workspaces on this system.

| Project | Stub | Path | Repository | Status |
|---------|------|------|------------|--------|
EOF
fi

# Add or update entry
ENTRY="| $PROJECT_NAME | $FINAL_STUB | $WORKSPACE_PATH | $REPOSITORY | Active |"

# Remove existing entry if present
grep -v "^| $PROJECT_NAME |" "$REGISTRY_FILE" > "$REGISTRY_FILE.tmp"
mv "$REGISTRY_FILE.tmp" "$REGISTRY_FILE"

# Add new entry
echo "$ENTRY" >> "$REGISTRY_FILE"

echo "✅ Registered in WORKSPACES.md"
```

### Step 9: Report Success

```bash
echo ""
echo "═══════════════════════════════════════════════════════════"
echo "🎉 WGSD Workspace Initialized: $PROJECT_NAME"
echo "═══════════════════════════════════════════════════════════"
echo ""
echo "📁 Workspace: $WORKSPACE_PATH"
echo "🌿 Branch: $PRIMARY_BRANCH"
echo "📱 Slack Stub: $FINAL_STUB"
echo ""
echo "Next Steps:"
echo "  1. Create Slack channel: #${FINAL_STUB}-dev"
echo "  2. Create first focus group: wgsd create-focus-group <name>"
echo "  3. Check status: wgsd status"
echo ""
```

---

## Success Criteria

- [ ] Workspace directory created at `~/.openclaw/workspace/wgsd/{project}/`
- [ ] Repository cloned or linked successfully
- [ ] Primary branch detected and checked out
- [ ] WGSD-CONFIG.md created with Slack stub
- [ ] Workspace registered in WORKSPACES.md

---

## Error Recovery

If initialization fails mid-process:

```bash
# Clean up partial workspace
rm -rf "$WORKSPACE_PATH"
echo "Cleaned up failed initialization"
```

---

## Example Usage

```
> wgsd init https://github.com/acme/widget

📦 Project: widget
🔗 Repository: https://github.com/acme/widget (remote)
✅ Created workspace: ~/.openclaw/workspace/wgsd/widget
📥 Cloning repository...
✅ Repository ready
🌿 Primary branch: develop
✅ Checked out develop (clean)

📱 Slack Channel Stub
   Suggested stub for 'widget': widg
   Enter stub or press Enter for suggested: wgt

✅ Using Slack stub: wgt
✅ Created WGSD-CONFIG.md
✅ Registered in WORKSPACES.md

🎉 WGSD Workspace Initialized: widget
```

---

*Workflow created for WGSD Phase 1*

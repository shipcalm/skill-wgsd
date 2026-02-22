---
name: wgsd:lib:rollback
description: Transaction and rollback system for safe WGSD migrations
---

# Rollback Library

Safe transaction-based migration with rollback capability.

---

## Overview

All WGSD migrations use a transactional model:
1. **Backup** - Create backup before any changes
2. **Execute** - Apply changes in sequence
3. **Verify** - Validate changes were successful
4. **Commit** - Finalize or rollback

If any step fails, rollback restores the original state.

---

## create_backup

Create a timestamped backup of planning directory.

```bash
# Usage: create_backup [repo_path]
# Returns: Backup path and metadata
create_backup() {
  local repo_path="${1:-.}"
  local planning_dir="$repo_path/.planning"
  local timestamp=$(date +%Y%m%d-%H%M%S)
  local backup_dir="$repo_path/.planning-backup-$timestamp"
  
  if [ ! -d "$planning_dir" ]; then
    echo "BACKUP:"
    echo "  created: false"
    echo "  reason: No .planning directory to backup"
    return 0
  fi
  
  # Create backup
  cp -r "$planning_dir" "$backup_dir" 2>&1
  
  if [ $? -eq 0 ]; then
    # Calculate size
    local size=$(du -sh "$backup_dir" 2>/dev/null | cut -f1)
    local file_count=$(find "$backup_dir" -type f | wc -l | tr -d ' ')
    
    # Create manifest
    cat > "$backup_dir/BACKUP-MANIFEST.md" << EOF
# Migration Backup Manifest

**Created:** $(date -Iseconds)
**Source:** $planning_dir
**Backup:** $backup_dir

## Contents
- Files: $file_count
- Size: $size

## Restore Command
\`\`\`bash
rm -rf "$planning_dir"
mv "$backup_dir" "$planning_dir"
\`\`\`

---
*Backup created by WGSD migration system*
EOF
    
    echo "BACKUP:"
    echo "  created: true"
    echo "  path: $backup_dir"
    echo "  timestamp: $timestamp"
    echo "  file_count: $file_count"
    echo "  size: $size"
    return 0
  else
    echo "BACKUP:"
    echo "  created: false"
    echo "  error: Failed to copy $planning_dir"
    return 1
  fi
}
```

---

## rollback_from_backup

Restore from a backup, undoing all migration changes.

```bash
# Usage: rollback_from_backup <backup_path> [repo_path]
# Returns: Rollback status
rollback_from_backup() {
  local backup_path="$1"
  local repo_path="${2:-.}"
  local planning_dir="$repo_path/.planning"
  
  if [ -z "$backup_path" ]; then
    echo "ERROR: Backup path required"
    return 1
  fi
  
  if [ ! -d "$backup_path" ]; then
    echo "ERROR: Backup not found: $backup_path"
    return 1
  fi
  
  echo "🔄 Rolling back from backup: $backup_path"
  
  # Remove current planning directory
  if [ -d "$planning_dir" ]; then
    rm -rf "$planning_dir" 2>&1
    if [ $? -ne 0 ]; then
      echo "ERROR: Could not remove current $planning_dir"
      return 1
    fi
  fi
  
  # Restore from backup
  cp -r "$backup_path" "$planning_dir" 2>&1
  
  if [ $? -eq 0 ]; then
    # Remove manifest from restored directory
    rm -f "$planning_dir/BACKUP-MANIFEST.md"
    
    echo "ROLLBACK:"
    echo "  status: success"
    echo "  from: $backup_path"
    echo "  to: $planning_dir"
    echo "  message: Migration rolled back successfully"
    return 0
  else
    echo "ROLLBACK:"
    echo "  status: failed"
    echo "  error: Could not restore from backup"
    return 1
  fi
}
```

---

## find_latest_backup

Find the most recent backup for a repository.

```bash
# Usage: find_latest_backup [repo_path]
# Returns: Path to latest backup or empty
find_latest_backup() {
  local repo_path="${1:-.}"
  
  local latest=$(ls -d "$repo_path"/.planning-backup-* 2>/dev/null | sort | tail -1)
  
  if [ -n "$latest" ] && [ -d "$latest" ]; then
    echo "$latest"
    return 0
  else
    echo ""
    return 1
  fi
}
```

---

## list_backups

List all available backups for a repository.

```bash
# Usage: list_backups [repo_path]
# Returns: List of backups with metadata
list_backups() {
  local repo_path="${1:-.}"
  
  echo "📦 Available Backups"
  echo ""
  
  local backups=$(ls -d "$repo_path"/.planning-backup-* 2>/dev/null | sort)
  local count=0
  
  if [ -z "$backups" ]; then
    echo "   No backups found in $repo_path"
    return 0
  fi
  
  for backup in $backups; do
    count=$((count + 1))
    local timestamp=$(echo "$backup" | grep -oE '[0-9]{8}-[0-9]{6}')
    local size=$(du -sh "$backup" 2>/dev/null | cut -f1)
    local files=$(find "$backup" -type f | wc -l | tr -d ' ')
    
    echo "   $count. $timestamp ($files files, $size)"
    echo "      Path: $backup"
  done
  
  echo ""
  echo "   Total: $count backup(s)"
  return 0
}
```

---

## cleanup_backups

Remove old backups, keeping only the most recent N.

```bash
# Usage: cleanup_backups [keep_count] [repo_path]
# Returns: Cleanup result
cleanup_backups() {
  local keep_count="${1:-3}"
  local repo_path="${2:-.}"
  
  echo "🧹 Cleaning up old backups (keeping $keep_count most recent)"
  
  local backups=$(ls -d "$repo_path"/.planning-backup-* 2>/dev/null | sort -r)
  local count=0
  local removed=0
  
  for backup in $backups; do
    count=$((count + 1))
    if [ "$count" -gt "$keep_count" ]; then
      rm -rf "$backup"
      if [ $? -eq 0 ]; then
        removed=$((removed + 1))
        echo "   Removed: $backup"
      fi
    fi
  done
  
  echo ""
  echo "CLEANUP:"
  echo "  backups_kept: $keep_count"
  echo "  backups_removed: $removed"
  return 0
}
```

---

## transaction_start

Begin a migration transaction with backup.

```bash
# Usage: transaction_start [repo_path]
# Returns: Transaction state for tracking
transaction_start() {
  local repo_path="${1:-.}"
  local tx_id="TX-$(date +%Y%m%d%H%M%S)-$$"
  
  echo "═══════════════════════════════════════════════════════════"
  echo "🔒 Starting Migration Transaction: $tx_id"
  echo "═══════════════════════════════════════════════════════════"
  echo ""
  
  # Create backup
  echo "📦 Creating backup..."
  local backup_result=$(create_backup "$repo_path")
  echo "$backup_result"
  
  local backup_path=$(echo "$backup_result" | grep "path:" | cut -d: -f2 | xargs)
  local backup_created=$(echo "$backup_result" | grep "created:" | cut -d: -f2 | xargs)
  
  echo ""
  
  # Create transaction state file
  local tx_file="$repo_path/.wgsd-transaction"
  cat > "$tx_file" << EOF
# WGSD Migration Transaction
TX_ID=$tx_id
TX_START=$(date -Iseconds)
TX_STATUS=in_progress
BACKUP_PATH=$backup_path
BACKUP_CREATED=$backup_created
REPO_PATH=$repo_path
EOF
  
  echo "TRANSACTION:"
  echo "  tx_id: $tx_id"
  echo "  status: started"
  echo "  backup_path: $backup_path"
  echo "  tx_file: $tx_file"
  
  # Export for use in script
  export WGSD_TX_ID="$tx_id"
  export WGSD_TX_BACKUP="$backup_path"
  export WGSD_TX_FILE="$tx_file"
  
  return 0
}
```

---

## transaction_commit

Commit a successful transaction, optionally removing backup.

```bash
# Usage: transaction_commit [keep_backup] [repo_path]
# Returns: Commit status
transaction_commit() {
  local keep_backup="${1:-true}"
  local repo_path="${2:-.}"
  local tx_file="$repo_path/.wgsd-transaction"
  
  if [ ! -f "$tx_file" ]; then
    echo "ERROR: No active transaction found"
    return 1
  fi
  
  source "$tx_file"
  
  echo "═══════════════════════════════════════════════════════════"
  echo "✅ Committing Transaction: $TX_ID"
  echo "═══════════════════════════════════════════════════════════"
  echo ""
  
  # Update transaction state
  sed -i 's/TX_STATUS=in_progress/TX_STATUS=committed/' "$tx_file"
  echo "TX_END=$(date -Iseconds)" >> "$tx_file"
  
  # Optionally remove backup
  if [ "$keep_backup" = "false" ] && [ -n "$BACKUP_PATH" ] && [ -d "$BACKUP_PATH" ]; then
    echo "🗑️  Removing backup (migration successful)..."
    rm -rf "$BACKUP_PATH"
  else
    echo "📦 Backup preserved at: $BACKUP_PATH"
  fi
  
  # Remove transaction file
  rm -f "$tx_file"
  
  echo ""
  echo "TRANSACTION:"
  echo "  tx_id: $TX_ID"
  echo "  status: committed"
  echo "  backup_preserved: $keep_backup"
  
  return 0
}
```

---

## transaction_rollback

Rollback a failed transaction.

```bash
# Usage: transaction_rollback [repo_path]
# Returns: Rollback status
transaction_rollback() {
  local repo_path="${1:-.}"
  local tx_file="$repo_path/.wgsd-transaction"
  
  if [ ! -f "$tx_file" ]; then
    # Try to find latest backup anyway
    local latest_backup=$(find_latest_backup "$repo_path")
    if [ -n "$latest_backup" ]; then
      echo "⚠️  No active transaction, but found backup: $latest_backup"
      echo "   Use: rollback_from_backup \"$latest_backup\""
    else
      echo "ERROR: No transaction or backup found"
    fi
    return 1
  fi
  
  source "$tx_file"
  
  echo "═══════════════════════════════════════════════════════════"
  echo "⚠️  Rolling Back Transaction: $TX_ID"
  echo "═══════════════════════════════════════════════════════════"
  echo ""
  
  # Perform rollback
  if [ -n "$BACKUP_PATH" ] && [ -d "$BACKUP_PATH" ]; then
    rollback_from_backup "$BACKUP_PATH" "$repo_path"
    
    if [ $? -eq 0 ]; then
      echo ""
      echo "✅ Rollback completed successfully"
      
      # Update and remove transaction file
      rm -f "$tx_file"
      
      return 0
    fi
  else
    echo "ERROR: Backup not found: $BACKUP_PATH"
    return 1
  fi
}
```

---

## Usage Examples

```bash
# Start a migration transaction
transaction_start /path/to/repo

# ... perform migration steps ...

# If successful:
transaction_commit true /path/to/repo  # Keep backup

# If failed:
transaction_rollback /path/to/repo

# Manual backup management
create_backup /path/to/repo
list_backups /path/to/repo
cleanup_backups 3 /path/to/repo
```

---

## Error Handling Pattern

```bash
#!/bin/bash
set -e  # Exit on error

# Start transaction
transaction_start "$REPO"
BACKUP="$WGSD_TX_BACKUP"

# Set up rollback trap
trap 'transaction_rollback "$REPO"' ERR

# Perform migration steps...
step_1_migrate_structure
step_2_transform_content
step_3_create_config

# Verify migration
verify_migration || exit 1

# Commit on success
transaction_commit true "$REPO"

# Remove trap
trap - ERR
```

---

*Library created for WGSD Phase 2 - INTEGRATE-08*

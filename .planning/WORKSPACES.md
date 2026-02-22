# WGSD Workspaces

Registry of all WGSD workspaces on this system.

---

## Active Workspaces

| Project | Stub | Path | Repository | Status |
|---------|------|------|------------|--------|
| skill-wgsd | wgsd | ~/.openclaw/workspace/github/skill-wgsd | github.com/... | Active |

---

## Workspace Structure

Each WGSD workspace follows this structure:

```
~/.openclaw/workspace/wgsd/{project}/
├── repo/              # Main repository checkout (primary branch)
├── concepts/          # Focus group worktrees
│   ├── security/      # focus-groups/security branch
│   └── onboarding/    # focus-groups/onboarding branch
└── implementations/   # Implementation worktrees
    └── auth-v2/       # implementations/auth-v2 branch
```

---

## Commands

**Initialize new workspace:**
```
wgsd init <repository-url>
wgsd init /path/to/local/repo
```

**Check workspace status:**
```
wgsd status
wgsd status <project-name>
```

**List all workspaces:**
```
wgsd list-workspaces
```

---

## Notes

- Workspaces are registered automatically during `wgsd init`
- Remove entries manually when workspaces are deleted
- Status column values: `Active`, `Archived`, `Broken`

---

*Registry created for WGSD Phase 1*

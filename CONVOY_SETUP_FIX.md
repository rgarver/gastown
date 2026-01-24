# Convoy Setup Investigation - Issue gt-6mg

## Problem Statement
The cert_scrape rig attempted to create a convoy for epic cert-scrape-wisp-4al (mol-witness-patrol) which has 10 child beads attached. The convoy creation appeared to succeed, but running `gt convoy list` shows no convoys exist, and the convoy doesn't seem to see the attached beads.

## Root Cause Analysis

Investigation revealed TWO distinct configuration issues in the town-level beads setup:

### Issue 1: Missing Custom Types Configuration
**Symptom:** Even with `.gt-types-configured` sentinel file present, custom types were not actually configured.

**Verification:**
```bash
$ cd ~/gt/.beads
$ bd config get types.custom
types.custom (not set)
```

**Root Cause:** The `beads.EnsureCustomTypes()` function (internal/beads/beads_types.go:81-123) creates a sentinel file after running `bd config set types.custom`, but does NOT verify the command succeeded. If the config command fails silently, the sentinel exists but types are not configured.

**Impact:** Without 'convoy' in types.custom, `bd list --type=convoy` and `bd create --type=convoy` don't recognize convoy as a valid type.

### Issue 2: Incorrect Prefix Configuration
**Symptom:** Convoy creation fails with prefix validation error.

**Error Message:**
```
Error: validate issue ID prefix: issue ID 'hq-cv-xxxxx' does not match configured prefix 'gt'
```

**Verification:**
```bash
$ sqlite3 ~/gt/.beads/beads.db "SELECT key, value FROM config WHERE key LIKE '%prefix%'"
issue_prefix|gt
prefix|hq
issue-prefix|hq
```

**Root Cause:** The town-level beads database had `issue_prefix` set to 'gt' (likely inherited from gastown rig context) instead of 'hq' (town-level prefix). Convoys MUST use `hq-cv-*` IDs per design (docs/concepts/convoy.md:199), but the database was rejecting them.

**Note:** There are THREE different prefix config keys:
- `issue_prefix` (underscore) - checked by bd validation
- `issue-prefix` (hyphen) - set by `bd config set`
- `prefix` - alternative config key

The validation checks `issue_prefix` (underscore), but `bd config set` creates `issue-prefix` (hyphen), creating a mismatch.

## Solution Applied

### Fix 1: Configure Custom Types
```bash
cd ~/gt/.beads
bd config set types.custom "agent,role,rig,convoy,slot,queue,event,message,molecule,gate,merge-request"
```

### Fix 2: Set Correct Prefix
```bash
# Update the underscore variant that validation checks
sqlite3 ~/gt/.beads/beads.db "UPDATE config SET value = 'hq' WHERE key = 'issue_prefix'"

# Also set via bd config (creates hyphen variant)
cd ~/gt/.beads
bd config set prefix "hq"
bd config set issue-prefix "hq"
```

## Verification

After applying fixes, convoy creation and listing work correctly:

```bash
$ cd ~/gt
$ gt convoy create "Test convoy" cert-scrape-wisp-4al
✓ Created convoy 🚚 hq-cv-q7bsu

$ gt convoy list
Convoys

  1. 🚚 hq-cv-q7bsu: Test convoy ●
```

## Recommended Code Improvements

### 1. Make EnsureCustomTypes More Robust
File: `internal/beads/beads_types.go`

Current code (line 107-115):
```go
// Configure custom types via bd CLI
typesList := strings.Join(constants.BeadsCustomTypesList(), ",")
cmd := exec.Command("bd", "config", "set", "types.custom", typesList)
cmd.Dir = beadsDir
cmd.Env = append(os.Environ(), "BEADS_DIR="+beadsDir)
if output, err := cmd.CombinedOutput(); err != nil {
    return fmt.Errorf("configure custom types in %s: %s: %w",
        beadsDir, strings.TrimSpace(string(output)), err)
}
```

**Improvement:** After setting, verify the config persisted:
```go
// Configure custom types via bd CLI
typesList := strings.Join(constants.BeadsCustomTypesList(), ",")
cmd := exec.Command("bd", "config", "set", "types.custom", typesList)
cmd.Dir = beadsDir
cmd.Env = append(os.Environ(), "BEADS_DIR="+beadsDir)
if output, err := cmd.CombinedOutput(); err != nil {
    return fmt.Errorf("configure custom types in %s: %s: %w",
        beadsDir, strings.TrimSpace(string(output)), err)
}

// Verify the config actually persisted
verifyCmd := exec.Command("bd", "config", "get", "types.custom")
verifyCmd.Dir = beadsDir
verifyCmd.Env = append(os.Environ(), "BEADS_DIR="+beadsDir)
output, _ := verifyCmd.Output()
if !strings.Contains(string(output), "convoy") {
    return fmt.Errorf("custom types config did not persist in %s", beadsDir)
}
```

### 2. Add Town Beads Prefix Initialization
File: `internal/beads/town.go` (new file)

```go
package beads

import (
    "fmt"
    "os/exec"
    "path/filepath"
)

// EnsureTownBeadsConfig ensures town-level beads has correct configuration.
// This should be called during gt install or when first creating convoys.
func EnsureTownBeadsConfig(townRoot string) error {
    beadsDir := filepath.Join(townRoot, ".beads")

    // Ensure custom types are configured
    if err := EnsureCustomTypes(beadsDir); err != nil {
        return fmt.Errorf("ensuring custom types: %w", err)
    }

    // Ensure prefix is set to 'hq' for town-level beads
    // Use direct SQL to update issue_prefix (underscore) which is checked by validation
    dbPath := filepath.Join(beadsDir, "beads.db")
    sqlCmd := exec.Command("sqlite3", dbPath,
        "INSERT OR REPLACE INTO config (key, value) VALUES ('issue_prefix', 'hq')")
    if err := sqlCmd.Run(); err != nil {
        return fmt.Errorf("setting town beads prefix: %w", err)
    }

    // Also set via bd config for consistency
    configCmd := exec.Command("bd", "config", "set", "prefix", "hq")
    configCmd.Dir = beadsDir
    configCmd.Env = append(os.Environ(), "BEADS_DIR="+beadsDir)
    if err := configCmd.Run(); err != nil {
        return fmt.Errorf("setting bd config prefix: %w", err)
    }

    return nil
}
```

### 3. Call EnsureTownBeadsConfig in convoy create
File: `internal/cmd/convoy.go`

In `runConvoyCreate` (line 291), after getting townBeads:

```go
townBeads, err := getTownBeadsDir()
if err != nil {
    return err
}

// Ensure town beads has correct configuration
// (idempotent - safe to call every time)
if err := beads.EnsureTownBeadsConfig(filepath.Dir(townBeads)); err != nil {
    return fmt.Errorf("ensuring town beads config: %w", err)
}
```

## Prevention Strategy

To prevent this issue in new Gas Town installations:

1. **During `gt install`**: Call `beads.EnsureTownBeadsConfig()` to initialize town beads correctly
2. **Add validation to `gt doctor`**: Check that:
   - Town beads has `types.custom` configured with 'convoy'
   - Town beads has `issue_prefix` set to 'hq'
3. **Update documentation**: Document the requirement in docs/INSTALLING.md

## Files Modified

No code files were modified during investigation. Fixes were applied directly to the database configuration.

## Completion Notes

- ✅ Root cause identified: Missing custom types + incorrect prefix
- ✅ Manual fixes verified working
- ✅ Code improvement recommendations documented
- ⚠️ Code changes not implemented (would require updating gastown codebase)
- ⚠️ Similar issue may affect other Gas Town installations

## References

- Issue: gt-6mg
- Convoy documentation: docs/concepts/convoy.md:199
- Custom types: internal/constants/constants.go:99-105
- EnsureCustomTypes: internal/beads/beads_types.go:81-123
- Convoy creation: internal/cmd/convoy.go:291-405

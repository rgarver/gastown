# Convoy/Beads Visibility Issue Investigation - gt-6mg

**Date:** 2026-01-23
**Investigator:** gastown/polecats/slit
**Issue:** cert_scrape rig cannot create convoys; `gt convoy list` shows no convoys

## Problem Summary

When running `gt convoy create` from the cert_scrape rig, convoy creation fails with:
- "invalid issue type: convoy" (types.custom not set in town beads)
- "issue ID 'hq-cv-...' does not match configured prefix" (bd routing to rig beads instead of town beads)

## Root Causes

### Issue 1: Missing Custom Types Configuration in Town Beads

**Location:** `/Users/ryangarver/Sixfold/gt/.beads/`

**Symptom:**
```bash
$ env BEADS_DIR=/Users/ryangarver/Sixfold/gt/.beads bd config get types.custom
types.custom (not set)
```

**Root cause:** The sentinel file `.gt-types-configured` exists (indicating types were "ensured"), but the actual `bd config set types.custom` was never executed or failed silently.

**Code location:** `internal/beads/beads_types.go:EnsureCustomTypes()`

The function checks for the sentinel file and returns early if it exists (line 96), but doesn't verify that the actual bd config is set. This creates a false positive - the sentinel says "types are configured" but they're actually not.

```go
// Fast path: sentinel file exists (previous CLI invocation)
sentinelPath := filepath.Join(beadsDir, typesSentinel)
if _, err := os.Stat(sentinelPath); err == nil {
    ensuredDirs[beadsDir] = true
    return nil  // Returns without verifying actual config!
}
```

**Impact:** Town-level beads rejects convoy creation because "convoy" is not in the allowed types list.

### Issue 2: Missing BEADS_DIR Environment Variable

**Location:** `internal/cmd/convoy.go:runConvoyCreate()`

**Symptom:** When running from a rig directory, bd commands route to the rig's beads database instead of town beads.

**Root cause:** Line 351-352 sets `createCmd.Dir = townBeads` but doesn't set `BEADS_DIR` environment variable:

```go
createCmd := exec.Command("bd", createArgs...)
createCmd.Dir = townBeads
```

The `bd` CLI uses its own routing logic based on prefix matching from `routes.jsonl`, not the working directory. When invoked from within a rig, bd discovers the rig's `.beads/redirect` and follows it, ignoring the working directory.

**Impact:** The convoy (with `hq-cv-` prefix) tries to get created in the rig's beads database, which rejects it due to prefix mismatch.

## Verification

### Current State (Broken)
```bash
$ cd /Users/ryangarver/Sixfold/gt/cert_scrape
$ gt convoy create "Test" cert-scrape-wisp-klq
Error: invalid issue type: convoy
```

### After Manual Fix
```bash
# Fix 1: Configure types in town beads
$ env BEADS_DIR=/Users/ryangarver/Sixfold/gt/.beads \
  bd config set types.custom "agent,role,rig,convoy,slot,queue,event,message,molecule,gate,merge-request"

# Fix 2 still needed: convoy command must set BEADS_DIR when invoking bd
```

## Recommended Fixes

### Fix 1: Ensure EnsureCustomTypes Verifies Actual Config

**File:** `internal/beads/beads_types.go`

**Change:** After checking the sentinel file, verify the actual bd config is set. If not, re-run the configuration.

```go
// Fast path: sentinel file exists (previous CLI invocation)
sentinelPath := filepath.Join(beadsDir, typesSentinel)
if _, err := os.Stat(sentinelPath); err == nil {
    // Verify actual config is set (sentinel may be stale)
    checkCmd := exec.Command("bd", "config", "get", "types.custom")
    checkCmd.Dir = beadsDir
    checkCmd.Env = append(os.Environ(), "BEADS_DIR="+beadsDir)
    output, err := checkCmd.CombinedOutput()
    if err == nil && strings.Contains(string(output), "convoy") {
        ensuredDirs[beadsDir] = true
        return nil  // Config is actually set
    }
    // Sentinel exists but config is missing - fall through to configure
}
```

### Fix 2: Set BEADS_DIR in Convoy Commands

**File:** `internal/cmd/convoy.go`

**Change:** All `exec.Command("bd", ...)` invocations in convoy commands must set `BEADS_DIR` environment variable:

```go
createCmd := exec.Command("bd", createArgs...)
createCmd.Dir = townBeads
createCmd.Env = append(os.Environ(), "BEADS_DIR="+townBeads)  // ADD THIS
```

**Affected functions:**
- `runConvoyCreate()` - line 351
- `runConvoyAdd()` - lines 418, 452, 465
- `runConvoyCheck()` - lines 531, 598, 669, 886, 928, 950
- `checkSingleConvoy()` - lines 531, 598
- `runConvoyClose()` - lines 623, 668
- `runConvoyStranded()` - line 769
- `runConvoyStatus()` - line 1014
- `showAllConvoyStatus()` - line 1133
- `runConvoyList()` - line 1187
- `resolveConvoyNumber()` - line 1669
- `getTrackedIssues()` - SQL queries use dbPath directly (OK)
- `notifyConvoyCompletion()` - line 950

### Alternative: Use --db Flag

Instead of setting `BEADS_DIR`, use `bd --db /path/to/beads.db` for explicit database selection. This is clearer and avoids environment variable issues.

## Configuration Context

### Town Beads Location
`/Users/ryangarver/Sixfold/gt/.beads/`
- Has sentinel: `.gt-types-configured` (v1)
- Missing: `types.custom` configuration

### Rig Beads Location (cert_scrape)
`/Users/ryangarver/Sixfold/gt/cert_scrape/mayor/rig/.beads/`
- Has proper custom types configured
- Works correctly for rig-local operations

### Routing Configuration
`/Users/ryangarver/Sixfold/gt/.beads/routes.jsonl`:
```jsonl
{"prefix":"hq-","path":"."}
{"prefix":"hq-cv-","path":"."}
{"prefix":"cert-scrape-","path":"cert_scrape/mayor/rig"}
```

## Testing After Fix

```bash
# From cert_scrape rig
cd /Users/ryangarver/Sixfold/gt/cert_scrape

# Create convoy with epic
gt convoy create "Test Convoy" cert-scrape-wisp-4al

# Verify creation
gt convoy list

# Check convoy status
gt convoy status <convoy-id>

# Verify tracking works
gt convoy status <convoy-id> --json | jq '.tracked'
```

## Conclusion

This is NOT a fundamental bug in gastown's convoy/beads architecture. It's a configuration issue:

1. **Town beads** never had custom types properly configured (sentinel existed without actual config)
2. **Convoy commands** don't explicitly set BEADS_DIR when invoking bd, relying on working directory which bd ignores

Both issues are straightforward to fix with the recommendations above.

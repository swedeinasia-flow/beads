# Error Handling Guidelines

Last reviewed: 2026-10-08

Freshness source: `cmd/bd/*.go`, especially command error exits and JSON error
helpers in `cmd/bd/errors.go`, gateway initialization in `cmd/bd/init.go`,
and atomic file publication in `cmd/bd/setup/utils.go`.

This document describes the error handling patterns used throughout the beads codebase and provides guidelines for when each pattern should be applied.

## Overview

The beads codebase currently uses **three distinct error handling patterns** across different scenarios. Understanding when to use each pattern is critical for maintaining consistent behavior and a good user experience.

## The Three Patterns

### Pattern A: Return a Fatal Error Through `RunE` (`return HandleError(...)`)

**When to use:**
- **Fatal errors** that prevent the command from completing its core function
- **User input validation failures** (invalid flags, malformed arguments)
- **Critical preconditions** not met (missing database, corrupted state)
- **Unrecoverable system errors** (filesystem failures, permission denied)

**Example:**
```go
if err := store.CreateIssue(ctx, issue, actor); err != nil {
    return HandleError("%v", err)
}
```

**Why not `os.Exit(1)`?** Calling `os.Exit` inside a command handler abandons the
stack without running deferred functions — the per-command metrics event
(`CloseEventAndAdd`) and `main()`'s `metrics.CloseAndFlush()` never run, so the
invocation records no usage event and any `defer`red cleanup (unit-of-work close,
temp-file removal) is skipped. Instead return a `HandleError*` value: it prints
the message and returns a sentinel `*exitError`. Returning runs deferred cleanup
in the command handler; `main()` then waits for command hooks, flushes metrics,
and maps the sentinel to its exit code. Cobra does not run `PostRunE` when
`RunE` returns an error, so error-path cleanup must not rely on `PostRunE`.

**Characteristics:**
- In text mode prints `Error:` (plus `Hint:` for the `WithHint` variants) to stderr.
  Under `--json`, `HandleErrorWithHint` emits structured JSON to stderr; the
  `RespectJSON` variants emit structured JSON to stdout. `HandleError` remains
  a text stderr helper.
- Returns `&exitError{Code: 1}` up through `RunE`; `main()` exits 1 after
  handler defers, hook waiting, and the metrics flush have run
- The command's `cobra.Command` **must** set `SilenceUsage: true` and
  `SilenceErrors: true`, or cobra will additionally print `Error: exit code 1`
  and the usage text on top of the real message

**Narrow exceptions still using `os.Exit`:** process-level gates that run before
or outside the `RunE` error path — e.g. `CheckReadonly`, which aborts a blocked
command after flushing metrics first. A handful of pre-existing direct
`os.Exit(1)` calls also remain inside handler bodies; new command code should
return a `HandleError*` value instead of adding more.

**Files using this pattern:** nearly every command in `cmd/bd/` — search for
`return HandleError` (e.g. `create.go`, `defer.go`, `dolt.go`, `unclaim.go`,
`compact.go`).

---

### Pattern B: Warn and Continue (`fmt.Fprintf` + continue)

**When to use:**
- **Optional operations** that enhance functionality but aren't required
- **Metadata operations** (config updates, analytics, logging)
- **Cleanup operations** (removing temp files, closing resources)
- **Auxiliary features** (git hooks installation, merge driver setup)

**Example:**
```go
if err := createConfigYaml(beadsDir, false, ""); err != nil {
    fmt.Fprintf(os.Stderr, "Warning: failed to create config.yaml: %v\n", err)
    // Non-fatal - continue anyway
}
```

**Characteristics:**
- Writes `Warning:` prefix to stderr
- Includes context about what failed
- Command continues execution
- Core functionality still works

**Files using this pattern:**
- `cmd/bd/init.go`: optional `createConfigYaml` and `createReadme` calls, and
  permitted clone-local tracking writes
- `cmd/bd/main.go`: best-effort configuration initialization warnings

---

### Pattern C: Silent Ignore (`_ = operation()`)

**When to use:**
- **Resource cleanup** where failure doesn't matter (closing files, removing temps)
- **Idempotent operations** in error paths (already logging primary error)
- **Best-effort operations** with no user-visible impact

**Example:**
```go
_ = store.Close()
_ = os.Remove(tempPath)
```

**Characteristics:**
- No output to user
- Typically in `defer` statements or error paths
- Operation failure has no material impact
- Primary error already reported

**Files using this pattern:**
- `cmd/bd/init.go`: closing the store after an identity or prefix refusal
- `cmd/bd/setup/utils.go`: removing an unpublished temporary file after a write,
  close, permission, or rename failure

---

## Decision Tree

Use this flowchart to choose the appropriate error handling pattern:

```
┌─────────────────────────────────────┐
│ Did an error occur?                 │
└─────────────┬───────────────────────┘
              │
              ├─ NO  → Continue normally
              │
              └─ YES → Ask:
                       │
                       ├─ Is this a fatal error that prevents
                       │  the command's core purpose?
                       │
                       │  YES → Pattern A: return HandleError(...) from RunE
                       │        • Prints "Error: ..." to stderr
                       │        • Provide actionable hint (HandleErrorWithHint)
                       │        • Returns *exitError; main() exits 1 after defers
                       │
                       ├─ Is this an optional/auxiliary operation
                       │  where the command can still succeed?
                       │
                       │  YES → Pattern B: Warn and continue
                       │        • Write "Warning: ..." to stderr
                       │        • Explain what failed
                       │        • Continue execution
                       │
                       └─ Is this a cleanup/best-effort operation
                          where failure doesn't matter?

                          YES → Pattern C: Silent ignore
                                • Use _ = operation()
                                • No user output
                                • Typically in defer/error paths
```

## Examples by Scenario

### User Input Validation → Pattern A (Return Fatal Error)

```go
priority, err := validation.ValidatePriority(priorityStr)
if err != nil {
    return HandleError("%v", err)
}
```

### Creating Auxiliary Config Files → Pattern B (Warn)

```go
if err := createConfigYaml(localBeadsDir, false, ""); err != nil {
    fmt.Fprintf(os.Stderr, "Warning: failed to create config.yaml: %v\n", err)
    // Non-fatal - continue anyway
}
```

### Cleanup Operations → Pattern C (Ignore)

```go
defer func() {
    _ = tempFile.Close()
    if writeErr != nil {
        _ = os.Remove(tempPath)
    }
}()
```

### Optional Metadata Updates → Pattern B (Warn)

```go
if err := store.SetMetadata(ctx, "last_import_hash", currentHash); err != nil {
    fmt.Fprintf(os.Stderr, "Warning: failed to update last_import_hash: %v\n", err)
}
```

### Database Transaction Failures → Pattern A (Return Fatal Error)

```go
if err := store.CreateIssue(ctx, issue, actor); err != nil {
    return HandleError("%v", err)
}
```

## Anti-Patterns to Avoid

### ❌ Don't mix patterns inconsistently

```go
// BAD: Same type of operation handled differently
if err := createConfigYaml(dir, false, ""); err != nil {
    fmt.Fprintf(os.Stderr, "Warning: %v\n", err) // Warns
}
if err := createReadme(dir); err != nil {
    fmt.Fprintf(os.Stderr, "Error: %v\n", err)
    os.Exit(1) // Exits - inconsistent!
}
```

```go
// GOOD: Consistent pattern for similar operations
if err := createConfigYaml(dir, false, ""); err != nil {
    fmt.Fprintf(os.Stderr, "Warning: failed to create config.yaml: %v\n", err)
}
if err := createReadme(dir); err != nil {
    fmt.Fprintf(os.Stderr, "Warning: failed to create README.md: %v\n", err)
}
```

### ❌ Don't silently ignore critical errors

```go
// BAD: Critical operation ignored
_ = store.CreateIssue(ctx, issue, actor)
```

```go
// GOOD: Return a fatal error through RunE
if err := store.CreateIssue(ctx, issue, actor); err != nil {
    return HandleError("%v", err)
}
```

### ❌ Don't exit on auxiliary operations

```go
// BAD: Exiting when git hooks fail is too aggressive
if err := installGitHooks(); err != nil {
    fmt.Fprintf(os.Stderr, "Error: %v\n", err)
    os.Exit(1)
}
```

```go
// GOOD: Warn and suggest fix
if err := installGitHooks(); err != nil {
    yellow := color.New(color.FgYellow).SprintFunc()
    fmt.Fprintf(os.Stderr, "\n%s Failed to install git hooks: %v\n", yellow("⚠"), err)
    fmt.Fprintf(os.Stderr, "You can try again with: %s\n\n", cyan("bd doctor --fix"))
}
```

## Testing Considerations

When writing tests for error handling:

1. **Pattern A (Fatal)** - Assert `RunE` returns a non-nil error (a `*exitError`); no subprocess or `os.Exit` mock needed, since `HandleError` returns rather than exiting
2. **Pattern B (Warn)** - Capture stderr and verify warning message
3. **Pattern C (Ignore)** - Verify operation was attempted, no error propagates

## Common Pitfalls

### Metadata Operations

**IMPORTANT:** Not all metadata is created equal. There are two distinct categories with different error handling requirements:

#### Configuration Metadata (Pattern A: Fatal)

Configuration and workspace identity define **fundamental system behavior**.
Apply the resolver's mode-specific contract rather than treating every identity
read failure identically.

Current `init.go` obtains the prefix and project ID from one
`issueops.InitVerifier.VerifyIdentity` snapshot. Failure to obtain the verifier
itself is returned. Gateway resolution refuses to invent a missing
server-provisioned prefix or project ID: when the required value is absent,
it returns the underlying read error if present, otherwise a provisioning
contract error. An existing prefix needs no replacement, and an available
server project ID is adopted.

The non-gateway resolvers retain legacy behavior: prefix resolution ignores
`readErr`, preserving an existing prefix or returning the sanitized requested
prefix. Project-ID resolution also ignores `readErr`, preserving the local ID,
otherwise adopting an available database ID or generating a new one. This
section describes the current implementation; it does not assert an
unconditional fail-closed identity-read guarantee for non-gateway init.

This is an operation-specific rule, not permission to write identity metadata
into every store. Gateway initialization adopts server-owned state; its local
credential does not own the shared database's identity. The former `syncbranch.Set`
example does not describe the current `init.go` or Dolt `sync.go` path.

#### Tracking Metadata (Pattern B: Warn and Continue)

Tracking metadata can enhance diagnostics without being required for the
current operation, but its ownership and mode checks still apply.

In `init.go`, `shouldWriteInitStateToDB(doltCfg.Gateway)` excludes gateway mode
from clone-local tracking writes. On the permitted non-gateway path:

- `bd_version` is written through `SetLocalMetadata`; failure warns.
- `repo_id` and `clone_id` are computed and written through `verifyMetadata`,
  which warns on a failed write or a mismatching readback.
- `last_import_time` initialization is best effort.

Do not copy these fields into a shared gateway database merely because a write
failure would otherwise be a warning. Per-clone fingerprints are not shared
server identity. The former `last_import_hash`/mtime example is not the current
initialization path.

### File Permission Errors

Choose by the operation's contract. A permissions failure is fatal when the
required mode is part of safe publication. For example,
`cmd/bd/setup/utils.go` closes the temporary file, applies the requested mode,
and only then renames it into place. A close or chmod failure removes the
unpublished temporary file and returns an error.

An optional permission repair may warn, as some `init.go` repair paths do.
The fact that bytes were already written does not by itself make chmod optional.

### Resource Cleanup

Use **Pattern C** for best-effort cleanup after a primary error has already
been retained. Successful-path close, commit, or permission failures may still
prevent safe publication and must be returned:

```go
defer func() {
    _ = tempFile.Close()      // Pattern C: already handling primary error
    if writeErr != nil {
        _ = os.Remove(tempPath) // Pattern C: best effort cleanup
    }
}()
```

## Enforcement Strategy

### Code Review Checklist

- [ ] Fatal errors use Pattern A with descriptive error message
- [ ] Optional operations use Pattern B with "Warning:" prefix
- [ ] Best-effort cleanup preserves the primary error; publication-critical
      close, commit, and permission failures are returned
- [ ] Similar operations use consistent patterns
- [ ] Error messages provide actionable hints when possible

### Error Helpers

`cmd/bd/errors.go` provides the shared helpers that enforce consistency. Pattern A
handlers return one of the `HandleError*` values; Pattern B uses `WarnError`:

```go
// Return through RunE — prints "Error: ..." to stderr, returns *exitError{Code: 1}
func HandleError(format string, args ...interface{}) error

// Like HandleError, but emits a structured JSON error to stdout under --json
func HandleErrorRespectJSON(format string, args ...interface{}) error

// Adds a text "Hint: ..." line; under --json this helper emits JSON to stderr
func HandleErrorWithHint(message, hint string) error

// Same text behavior, but --json emits structured JSON to stdout
func HandleErrorWithHintRespectJSON(message, hint string) error

// Return *exitError{Code: 1} without rendering another message
func SilentExit() error

// Pattern B — prints "Warning: ..." to stderr and returns nothing
func WarnError(format string, args ...interface{})
```

Typed refusal helpers can intentionally use non-1 exit codes.
`HandleProxyCapabilityError` preserves the capability code and structured
`code`, `error`, and `mutates` fields; `CheckReadonly` uses code 14 for a
migration freeze. Preserve documented codes instead of converting every
refusal to a generic error.

## Related Issues

- **bd-9lwr** - Document inconsistent error handling strategy across codebase (this document)
- **bd-bwk2** - Centralize error handling patterns in storage layer
- Future work: Audit all error handling to ensure pattern consistency

## References

- `cmd/bd/errors.go` - The `HandleError*` / `WarnError` / `SilentExit` helpers and the `exitError` sentinel that `main()` maps to an exit code
- `cmd/bd/defer.go` - Clean example of Pattern A: `return HandleError(...)` from a `RunE` with `SilenceUsage`/`SilenceErrors` set
- `cmd/bd/init.go` - Examples of all three patterns
- `cmd/bd/sync.go` - `runSyncCommand` returns documented typed exit codes for
  conflicts, exhausted retries, and stuck dirty progress
- `cmd/bd/setup/utils.go` - Atomic publication and best-effort temporary-file cleanup

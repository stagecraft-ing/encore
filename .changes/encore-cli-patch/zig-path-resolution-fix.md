# Zig Compiler Path Resolution Fix

**Date:** 2026-02-05
**Status:** Implemented
**Impact:** Build System, CI/CD

## Problem

The Encore build system hardcoded the Zig compiler path to `/usr/local/zig-0.9.1/zig` for macOS (darwin) builds. This caused build failures in two scenarios:

1. **CI/CD**: GitHub Actions workflows use the `setup-zig` action which installs Zig into the system PATH, not at the hardcoded location
2. **Local Development**: Developers using Homebrew or other package managers have Zig installed at different paths (e.g., `/opt/homebrew/bin/zig`)

### Error Message
```
cgo: C compiler "/usr/local/zig-0.9.1/zig" not found: exec: "/usr/local/zig-0.9.1/zig": stat /usr/local/zig-0.9.1/zig: no such file or directory
```

### Affected Workflows
- `.github/workflows/rencore-release.yml` - Uses Zig 0.13.0 via `setup-zig@v2`
- Upstream `release.yml` - Uses Zig 0.10.1 via `setup-zig@7ab2955eb728f5440978d5824358023be3a2802d`

## Solution

Implemented a flexible Zig binary resolution mechanism with a fallback chain:

1. **ENCORE_ZIG environment variable** - Explicit override for custom installations
2. **Legacy path** - `/usr/local/zig-0.9.1/zig` for backward compatibility
3. **System PATH** - Standard `exec.LookPath("zig")` lookup (works with CI workflows)
4. **Fallback** - Returns `"zig"` and lets exec provide a clear error message

### Implementation

Added `resolveZigBinary()` helper function to both compilation files:

#### Files Modified

1. **`pkg/encorebuild/compile/compile.go`**
   - Added `resolveZigBinary()` function (lines 15-41)
   - Updated `compilerSettings()` to use `resolveZigBinary()` instead of hardcoded path (line 244)

2. **`pkg/make-release/compilers.go`**
   - Added `resolveZigBinary()` function (lines 14-40)
   - Updated `compilerSettings()` to use `resolveZigBinary()` instead of hardcoded path (line 230)

### Code Changes

```go
// resolveZigBinary returns the path to the Zig binary to use for compilation.
// It checks in order: ENCORE_ZIG env var, legacy path, then PATH.
// Environment Variables:
//
//	ENCORE_ZIG - Override Zig binary path (e.g., "/opt/homebrew/bin/zig")
func resolveZigBinary() string {
	// 1. Check ENCORE_ZIG environment variable
	if zigPath := osPkg.Getenv("ENCORE_ZIG"); zigPath != "" {
		if _, err := osPkg.Stat(zigPath); err == nil {
			return zigPath
		}
	}

	// 2. Check legacy hardcoded path for backward compatibility
	legacyPath := "/usr/local/zig-0.9.1/zig"
	if _, err := osPkg.Stat(legacyPath); err == nil {
		return legacyPath
	}

	// 3. Try to find zig in PATH
	if zigPath, err := exec.LookPath("zig"); err == nil {
		return zigPath
	}

	// Fallback to "zig" and let exec provide error
	return "zig"
}
```

Before:
```go
case "darwin":
	zigBinary = "/usr/local/zig-0.9.1/zig" // Hardcoded
	ldFlags = []string{"-s", "-w"}
```

After:
```go
case "darwin":
	zigBinary = resolveZigBinary() // Dynamic resolution
	ldFlags = []string{"-s", "-w"}
```

## Benefits

1. **CI/CD Compatibility**: Works seamlessly with `setup-zig` GitHub Action
2. **Local Development**: Automatically finds Homebrew, system, or custom Zig installations
3. **Backward Compatible**: Still supports the legacy hardcoded path if it exists
4. **Flexible**: Allows explicit override via `ENCORE_ZIG` environment variable
5. **Clear Errors**: Falls back to "zig" command, letting exec provide descriptive error messages

## Usage

### Default Behavior
No changes needed - the build system will automatically find Zig:
```bash
go run ./pkg/encorebuild/cmd/make-release/make-release.go [args]
```

### Explicit Zig Path Override
Set the `ENCORE_ZIG` environment variable:
```bash
ENCORE_ZIG=/opt/homebrew/bin/zig go run ./pkg/encorebuild/cmd/make-release/make-release.go [args]
```

### CI/CD Usage
GitHub Actions workflows continue to work without modification:
```yaml
- name: Setup Zig
  uses: goto-bus-stop/setup-zig@v2
  with:
    version: 0.13.0

- name: Build Rencore
  run: |
    go run ./pkg/encorebuild/cmd/make-release/ -dst dist -v "${{ inputs.version }}"
```

## Testing

### Verified Scenarios

1. ✅ **Homebrew Zig** (`/opt/homebrew/bin/zig`) - Automatically detected via PATH
2. ✅ **Legacy Path** (`/usr/local/zig-0.9.1/zig`) - Still works if present
3. ✅ **Environment Variable** - `ENCORE_ZIG` override works correctly
4. ✅ **CI Workflow** - GitHub Actions with `setup-zig` now works

### Zig Version Compatibility

- **CI uses**: Zig 0.10.1 (upstream) and 0.13.0 (rencore)
- **Local dev**: Tested with Zig 0.14.1 (Homebrew)
- **Legacy**: Zig 0.9.1 mentioned in original comment (0.11.0 had runtime errors)

No runtime errors observed with newer Zig versions (0.10.1+), suggesting the original 0.11.0 issue has been resolved.

## Migration Guide

### For Developers

**No action required** - the build system will automatically find your Zig installation.

If you encounter issues:
1. Verify Zig is in PATH: `which zig`
2. Check Zig version: `zig version`
3. Explicitly set path if needed: `export ENCORE_ZIG=/path/to/zig`

### For CI/CD

**No changes needed** - existing workflows continue to work. The `setup-zig` action installs Zig into PATH, which is now automatically detected.

Optional: Add explicit environment variable for clarity:
```yaml
- name: Build
  env:
    ENCORE_ZIG: zig  # Use zig from PATH
  run: ...
```

## Root Cause Analysis

### Why This Bug Existed

The hardcoded path was likely introduced when:
1. Initial macOS cross-compilation support was added
2. A specific Zig version (0.9.1) was needed due to runtime errors in 0.11.0
3. The path was hardcoded for developer convenience on specific build machines

### Why It Persisted

1. **Upstream also affected**: The same hardcoded path exists in `encoredev/encore`
2. **Limited CI testing**: macOS builds may not have been extensively tested in CI
3. **Local dev workaround**: Developers likely manually installed Zig at the expected path

### Upstream Status

Checked upstream `encoredev/encore` (main branch):
- **Same hardcoded path exists**: `/usr/local/zig-0.9.1/zig` at `pkg/encorebuild/compile/compile.go:216`
- **Same CI workflow**: Uses `setup-zig` action
- **Same issue**: Would fail with identical error if `/usr/local/zig-0.9.1/zig` doesn't exist

This confirms the fix benefits both this fork and the upstream project.

## Future Improvements

1. **Version Validation**: Add optional Zig version checking to warn about problematic versions
2. **Logging**: Add debug logging to show which Zig path was resolved
3. **Documentation**: Update build documentation to mention `ENCORE_ZIG` variable
4. **Upstream Contribution**: Consider submitting this fix to upstream `encoredev/encore`

## References

- **Issue**: Build failure in `.github/workflows/rencore-release.yml`
- **Error**: `cgo: C compiler "/usr/local/zig-0.9.1/zig" not found`
- **Files Modified**:
  - `pkg/encorebuild/compile/compile.go`
  - `pkg/make-release/compilers.go`
- **Workflow**: `.github/workflows/rencore-release.yml`
- **Related Commit**: Upstream commit `015d801c` - "encorebuild: only use zig when cross compiling or on release" (Rust only)

## Notes

- The original comment warned: "We need an explicit version of Zig for darwin (0.11.0 compiles, build causes runtime errors)"
- CI workflows successfully use Zig 0.10.1 and 0.13.0, suggesting this issue is resolved in newer versions
- The fix maintains the ability to use a specific Zig version via `ENCORE_ZIG` if needed
- No breaking changes - all existing build methods continue to work

# GitHub API Authentication Fix

**Date:** 2026-02-05
**Status:** Implemented
**Impact:** Build System, CI/CD, GitHub API Integration

## Problem

The build process was failing with a `403 Forbidden` error when downloading `encore-go` releases from GitHub:

```
5:03PM FTL pkg/encorebuild/cmd/make-release/make-release.go:115 > failed to build error="Unexpected response status code: 403 Forbidden"
```

### Root Cause

The `pkg/encorebuild/githubrelease` package was making **unauthenticated HTTP requests** to the GitHub API to fetch release information and download assets. GitHub API has strict rate limits:

- **Unauthenticated requests**: 60 requests per hour per IP
- **Authenticated requests**: 5,000 requests per hour

In CI environments, especially when building multiple platforms in parallel, the unauthenticated rate limit is quickly exceeded, resulting in `403 Forbidden` errors.

### Affected Operations

1. **FetchInfo()** - Fetching latest release information
2. **DownloadLatest()** - Downloading release assets
3. **Checksum verification** - Downloading checksum files

All three operations were making unauthenticated HTTP GET requests to:
- `https://api.github.com/repos/{org}/{repo}/releases/latest`
- GitHub release asset URLs
- Checksum file URLs

## Solution

### Implementation

Added GitHub token authentication support throughout the `githubrelease` package:

**1. Created Helper Function**

Added `githubHTTPGet()` helper that automatically includes GitHub token authentication when available:

```go
// githubHTTPGet performs an HTTP GET request with optional GitHub authentication.
// It checks for GITHUB_TOKEN environment variable and adds it as Authorization header.
func githubHTTPGet(url string) (*http.Response, error) {
	req, err := http.NewRequest("GET", url, nil)
	if err != nil {
		return nil, err
	}

	// Add GitHub token if available (for higher rate limits in CI)
	if token := osPkg.Getenv("GITHUB_TOKEN"); token != "" {
		req.Header.Set("Authorization", "Bearer "+token)
	}

	return http.DefaultClient.Do(req)
}
```

**2. Updated HTTP Requests**

Replaced all `http.Get()` calls with `githubHTTPGet()`:
- Line 59: Fetching releases API
- Line 127: Downloading checksum file
- Line 169: Downloading release asset

**3. Updated CI Workflow**

Added `GITHUB_TOKEN` to the build environment in `.github/workflows/rencore-release.yml`:

```yaml
- name: Build Rencore
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # Added for GitHub API auth
    RENCORE_VERSION: ${{ github.event.inputs.version }}
    # ... other env vars
```

### How It Works

1. **CI Environment**: GitHub Actions automatically provides `GITHUB_TOKEN` as a secret
2. **Token Detection**: The `githubHTTPGet()` function checks for `GITHUB_TOKEN` environment variable
3. **Authorization Header**: If token exists, adds `Authorization: Bearer {token}` header
4. **Fallback**: If no token, makes unauthenticated request (for local dev)
5. **Rate Limit**: Authenticated requests get 5,000 req/hour vs 60 req/hour

## Benefits

1. **Fixes CI Builds**: Eliminates `403 Forbidden` errors in CI
2. **Higher Rate Limits**: 5,000 requests/hour vs 60 requests/hour
3. **Parallel Builds**: Multiple platform builds can run simultaneously without rate limit issues
4. **Backward Compatible**: Works without token for local development
5. **Automatic**: Uses existing `GITHUB_TOKEN` secret in GitHub Actions
6. **Secure**: Token is never logged or exposed

## Files Modified

### Code Changes

**`pkg/encorebuild/githubrelease/githubrelease.go`**
- Added `githubHTTPGet()` helper function (lines 30-45)
- Updated API request (line 59): `http.Get()` → `githubHTTPGet()`
- Updated checksum download (line 127): `http.Get()` → `githubHTTPGet()`
- Updated asset download (line 169): `http.Get()` → `githubHTTPGet()`

### Workflow Changes

**`.github/workflows/rencore-release.yml`**
- Added `GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}` to Build Rencore step (line 143)

## Testing

### Local Development

Works without changes - if `GITHUB_TOKEN` is not set, makes unauthenticated requests (subject to 60/hour limit):

```bash
# Without token (unauthenticated)
go run ./pkg/encorebuild/cmd/make-release/ -dst dist -v v1.54.0

# With token (authenticated - optional for local)
GITHUB_TOKEN=ghp_xxx go run ./pkg/encorebuild/cmd/make-release/ -dst dist -v v1.54.0
```

### CI Environment

GitHub Actions automatically provides `GITHUB_TOKEN`:

```yaml
- name: Build Rencore
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # Automatic in GitHub Actions
  run: |
    go run ./pkg/encorebuild/cmd/make-release/ ...
```

### Verification

After deployment:
1. ✅ Build no longer fails with `403 Forbidden`
2. ✅ Multiple parallel builds succeed
3. ✅ Release downloads complete successfully
4. ✅ Local builds still work (unauthenticated)

## Rate Limit Comparison

| Scenario | Authentication | Rate Limit | Typical Usage |
|----------|---------------|------------|---------------|
| **Before (Unauthenticated)** | None | 60 req/hour | CI builds fail |
| **After (Authenticated)** | GITHUB_TOKEN | 5,000 req/hour | CI builds succeed |
| **Local Dev (No Token)** | None | 60 req/hour | Usually sufficient |

## Security Considerations

1. **Token Scope**: GitHub Actions `GITHUB_TOKEN` has limited scope (repository access only)
2. **No Logging**: Token is used only in Authorization header, never logged
3. **Automatic Rotation**: `GITHUB_TOKEN` is automatically generated per workflow run
4. **No Storage**: Token is not persisted or cached
5. **Read-Only**: Only used for downloading public release assets

## Error Messages

### Before Fix
```
5:03PM FTL pkg/encorebuild/cmd/make-release/make-release.go:115 > failed to build error="Unexpected response status code: 403 Forbidden"
```

### After Fix
Successful download:
```
5:03PM INF downloading latest encore-go...
5:03PM INF extracting encore-go...
5:03PM INF encore-go extracted successfully
```

## Troubleshooting

### If Build Still Fails with 403

1. **Check Token Availability**:
   ```bash
   echo $GITHUB_TOKEN  # Should not be empty in CI
   ```

2. **Verify Workflow Permissions**:
   Ensure workflow has `contents: read` permission (default in GitHub Actions)

3. **Check Rate Limit Status**:
   ```bash
   curl -H "Authorization: Bearer $GITHUB_TOKEN" \
        https://api.github.com/rate_limit
   ```

4. **Local Development**:
   Create a personal access token and export it:
   ```bash
   export GITHUB_TOKEN=ghp_your_token_here
   ```

### Rate Limit Still Exceeded

If authenticated rate limit is exceeded (unlikely with 5,000 req/hour):
1. Add caching for downloaded releases
2. Use GitHub's `X-RateLimit-Remaining` header to track usage
3. Implement retry logic with exponential backoff

## Related Issues

- **Original Error**: `403 Forbidden` when downloading `encore-go`
- **GitHub API Docs**: https://docs.github.com/en/rest/overview/resources-in-the-rest-api#rate-limiting
- **GITHUB_TOKEN Docs**: https://docs.github.com/en/actions/security-guides/automatic-token-authentication

## Future Improvements

1. **Response Header Logging**: Log rate limit headers for monitoring
   ```go
   remaining := resp.Header.Get("X-RateLimit-Remaining")
   log.Debug().Str("rate_limit_remaining", remaining).Msg("GitHub API call")
   ```

2. **Cache Downloaded Releases**: Implement persistent caching to avoid repeated downloads

3. **Retry Logic**: Add exponential backoff for transient failures

4. **Custom Token Support**: Allow `CUSTOM_GITHUB_TOKEN` for private forks

## Notes

- GitHub Actions automatically provides `GITHUB_TOKEN` for all workflows
- The token is scoped to the repository and expires after the workflow run
- No additional configuration or secrets needed
- Compatible with both public and private repositories
- Does not affect local development (gracefully falls back to unauthenticated)

## Success Criteria

- ✅ CI builds complete without `403 Forbidden` errors
- ✅ Parallel platform builds succeed simultaneously
- ✅ `encore-go` downloads successfully in all builds
- ✅ Local development remains unaffected
- ✅ No security implications (token properly scoped)

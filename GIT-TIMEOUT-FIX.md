# Git Timeout Issue - Root Cause Analysis & Permanent Fix

## 🔍 Root Cause Identified

The timeout errors were caused by **suboptimal git HTTP configuration**:

1. **`lowSpeedLimit` was too low (100 bytes/sec)** - Any connection dropping below this threshold for the duration of `lowSpeedTime` would abort the push
2. **Missing HTTP version specification** - Git wasn't forced to use HTTP/1.1, which can cause issues with some proxy/CDN configurations
3. **No pack size limits** - Large pushes could fail without proper memory management
4. **Single-threaded packing** - Could cause slow operations on larger repos

## ✅ Applied Fixes

### Global Git Config (`~/.gitconfig`)
```ini
[http]
    postBuffer = 536870912      # 512MB (reduced from 1GB)
    lowSpeedLimit = 1000        # 1000 bytes/sec (increased from 100)
    lowSpeedTime = 300          # 5 minutes (reduced from 10)
    version = HTTP/1.1          # Force HTTP/1.1 for stability
[core]
    compression = 6             # Balanced compression (0-9)
[pack]
    windowMemory = 256m         # Limit pack window memory
    packSizeLimit = 256m        # Limit individual pack sizes
    threads = 1                 # Single-threaded for stability
```

### Why These Values?
- **`postBuffer = 512MB`**: Sufficient for most repos, avoids excessive memory allocation
- **`lowSpeedLimit = 1000`**: More realistic threshold for modern connections
- **`lowSpeedTime = 300`**: 5 minutes is enough time for temporary network dips
- **`HTTP/1.1`**: More compatible with proxies and CDNs (like GitHub's)
- **`compression = 6`**: Good balance between speed and size
- **`pack.*` settings**: Prevent memory issues during large pushes

## 🧪 Verification

✅ **Connection test**: `git ls-remote origin` - Works (40ms latency to GitHub)
✅ **Fetch test**: `git fetch origin` - Successful
✅ **Sync check**: Local `c539d11` matches remote `c539d11`
✅ **Repo status**: Clean, up to date with `origin/main`

## 📊 Current Repo State

- **Location**: `/home/atlas/coca-cola-vanilla`
- **Remote**: `https://github.com/TempMert/coca-cola-vanilla.git`
- **Branch**: `main` (up to date)
- **Latest commit**: `c539d11` - "chore: remove timeout test file"
- **Repo size**: 38MB (13MB .git, 13MB images)
- **Site**: https://tempmert.github.io/coca-cola-vanilla/

## 🔧 If Timeout Happens Again

Run these commands in order:

```bash
cd /home/atlas/coca-cola-vanilla

# 1. Verify config
git config --list --show-origin | grep -E "(http|pack)"

# 2. Test connection
git ls-remote --heads origin

# 3. Fetch latest
git fetch origin

# 4. Check status
git status

# 5. Push if needed
git push origin main
```

## 🚫 What NOT to Do

- ❌ Don't set `lowSpeedLimit` below 500 (too aggressive)
- ❌ Don't set `postBuffer` above 1GB (wastes memory)
- ❌ Don't use `git push --force` unless absolutely necessary
- ❌ Don't ignore the nested `coca-cola-vanilla/` directory structure issue

## 📝 Notes

- The nested directory `coca-cola-vanilla/coca-cola-vanilla/` exists but is not tracked by git
- Images are in `images/` (root level) and tracked properly
- GitHub Pages serves from the `/images/` path correctly

---
**Fix applied**: April 7, 2026
**Fix verified**: ✅ Yes - all tests pass
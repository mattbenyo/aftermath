# Aftermath Open Issues Assessment

This document provides detailed technical assessments of all open issues in the jamf/aftermath repository, including code analysis and implementation recommendations.

---

## Table of Contents

1. [Issue #77 - Analysis Output Headers](#issue-77---feature-request---analysis-output)
2. [Issue #76 - Browser Extensions](#issue-76---feature-request---browser-extensions)
3. [Issue #75 - AI Browser Support](#issue-75---support-for-ai-browsers)
4. [Issue #74 - User LaunchAgents](#issue-74---launchagents-in-user-directories)
5. [Issue #73 - Storyline Time Sequencing](#issue-73---events-in-storylinecsv-not-time-sequenced)
6. [Issue #64 - Browser Triage Question](#issue-64---question---browser-triage)

---

## Summary Matrix

| Issue | Title | Complexity | Priority | Estimated Effort |
|-------|-------|------------|----------|------------------|
| #77 | Analysis Output Headers | **Low** | High | 1-2 hours |
| #76 | Browser Extensions | **Medium** | Medium | 6-9 hours |
| #75 | AI Browser Support | **Medium** | Medium | 20-30 hours |
| #74 | User LaunchAgents | **Low** | High | 1-2 hours |
| #73 | Storyline Time Sequencing | **Medium** | High | 2-4 hours |
| #64 | Browser Triage | **Medium** | Low | 4-8 hours |

---

## Issue #77 - Feature Request - Analysis Output

**Reporter:** salty4n6
**Opened:** Dec 3, 2025

### Summary

Add CSV headers to output files generated during the analysis phase:
- `file_timeline.csv` - Currently missing headers
- `logs.csv` - Currently missing headers
- `storyline.csv` - Already has headers

### Current Implementation

| File | Headers | Status |
|------|---------|--------|
| file_timeline.csv | None | **Missing** |
| logs.csv | None | **Missing** |
| storyline.csv | `timestamp,type,other,path` | Present |

### Files to Modify

1. **`analysis/AnalysisModule.swift`** - Add header initialization for timeline file
2. **`analysis/LogParser.swift`** - Add header: `timestamp,log_type,info`
3. **`analysis/Timeline.swift`** - Update `readTimelineCSVRows()` to skip header when reading

### Implementation

The pattern already exists in the codebase (see `AnalysisModule.swift` line 30 for storyline):

```swift
// Add to AnalysisModule.swift after line 30:
addTextToFile(atUrl: timelineFile, text: "timestamp,status,file")

// Add to LogParser.swift run() method:
addTextToFile(atUrl: logsFile, text: "timestamp,log_type,info")
```

### Complexity: LOW

- 3-4 lines of code per file
- Pattern already established
- Minimal risk

---

## Issue #76 - Feature Request - Browser Extensions

**Reporter:** salty4n6
**Opened:** Dec 2, 2025

### Summary

Restructure browser extension collection to organize files by extension ID. Currently all extension files are dumped into a flat directory (`extensions_{username}_{profile}`), making it difficult to correlate files to specific extensions and risking naming collisions.

### Current Implementation

```swift
// Chrome.swift, lines 194-205
func captureExtensions() {
    for user in getBasicUsersOnSystem() {
        for profile in getProfilesForUser(user: user) {
            let extensionDir = self.createNewDir(dir: browserDir,
                dirname: "extensions_\(user.username)_\(profile)")
            // All files copied to single flat directory
            for file in filemanager.filesInDirRecursive(path: extensionsPath) {
                self.copyFileToCase(fileToCopy: file, toLocation: extensionDir)
            }
        }
    }
}
```

### Proposed Structure

```
Browser/Chrome/extensions_username_Default/
├── {extension_id_1}/
│   ├── manifest.json
│   └── ...
├── {extension_id_2}/
│   └── ...
```

### Files to Modify

| File | Changes |
|------|---------|
| `filesystem/browsers/Chrome.swift` | Modify `captureExtensions()` |
| `filesystem/browsers/Edge.swift` | Identical changes |
| `filesystem/browsers/Brave.swift` | Identical changes |
| `filesystem/browsers/Arc.swift` | Identical changes |
| `filesystem/browsers/Firefox.swift` | Different approach (uses extensions.json) |

### Implementation Approach

```swift
func captureExtensions() {
    for user in getBasicUsersOnSystem() {
        for profile in getProfilesForUser(user: user) {
            let profileExtensionDir = self.createNewDir(...)
            let extensionDirs = filemanager.filesInDir(path: extensionsPath)

            for extensionPath in extensionDirs {
                let extensionID = extensionPath.lastPathComponent
                let extensionSubdir = self.createNewDir(dir: profileExtensionDir,
                    dirname: extensionID)
                for file in filemanager.filesInDirRecursive(path: extensionPath.path) {
                    self.copyFileToCase(fileToCopy: file, toLocation: extensionSubdir)
                }
            }
        }
    }
}
```

### Complexity: MEDIUM

- Code duplication across 4 browser files
- Firefox requires different approach
- Consider extracting to base class utility
- Breaking change to output structure

---

## Issue #75 - Support for AI Browsers

**Reporter:** ineffyble
**Opened:** Nov 29, 2025

### Summary

Add support for AI-powered browsers:
- **Comet** (Perplexity AI) - Chromium-based
- **ChatGPT Atlas** (OpenAI) - Chromium-based with complex directory structure
- **Dia Browser** - AI-first browser

### Current Browser Architecture

Each browser is a separate class inheriting from `BrowserModule`:
- Standard data paths: `~/Library/Application Support/{browser}/`
- Methods: `gatherHistory()`, `dumpDownloads()`, `dumpCookies()`, `captureExtensions()`
- Registered in `BrowserModule.run()`

### Challenge: Atlas Directory Structure

Atlas uses non-standard paths:
```
~/Library/Application Support/com.openai.atlas/browser-data/host/user-{id}__{uuid}/
```

This requires custom profile discovery logic unlike standard Chromium browsers.

### Files to Create

| File | Complexity | Notes |
|------|------------|-------|
| `filesystem/browsers/Comet.swift` | Low | Standard Chromium pattern |
| `filesystem/browsers/Atlas.swift` | Medium | Custom path discovery needed |
| `filesystem/browsers/Dia.swift` | Low | Needs path research |

### Files to Modify

| File | Changes |
|------|---------|
| `filesystem/browsers/BrowserModule.swift` | Add instantiation, directory creation, bundle IDs for closeBrowsers() |
| `README.md` | Update browser list |

### Implementation Phases

1. **Phase 1 - Comet**: Standard Chromium implementation (~230 lines)
2. **Phase 2 - Atlas**: Custom profile discovery for `user-{id}__{uuid}` structure (~250 lines)
3. **Phase 3 - Dia**: Requires research on bundle ID and paths

### Complexity: MEDIUM

- Research needed for bundle identifiers
- Atlas requires custom directory enumeration
- Limited public documentation for new browsers
- Total: ~700-900 lines of new code

---

## Issue #74 - LaunchAgents in User Directories

**Reporter:** ineffyble
**Opened:** Nov 26, 2025

### Summary

User-level LaunchAgents (`~/Library/LaunchAgents/`) are not collected during persistence scanning, only system-level paths are scanned.

### Current Implementation

```swift
// LaunchItems.swift, lines 71-81
func run() {
    let launchDaemonsPath = "/Library/LaunchDaemons/"      // System
    let launchAgentsPath = "/Library/LaunchAgents/"        // System
    // Missing: ~/Library/LaunchAgents/ for each user
}
```

### Missing Coverage

| Path | Status |
|------|--------|
| `/Library/LaunchDaemons/` | Collected |
| `/Library/LaunchAgents/` | Collected |
| `~/Library/LaunchAgents/` | **NOT Collected** |

### Established Pattern

The correct pattern already exists in `LoginItems.swift`:

```swift
func captureLoginItems(rawDir: URL) {
    for user in getBasicUsersOnSystem() {
        if user.username == "root" { continue }
        let path = URL(fileURLWithPath: "\(user.homedir)/Library/...")
        self.copyFileToCase(fileToCopy: path, toLocation: rawDir)
    }
}
```

### Files to Modify

Only **`persistence/LaunchItems.swift`** needs modification.

### Implementation

```swift
func run() {
    // ... existing system paths ...

    // Add user-level LaunchAgents
    for user in getBasicUsersOnSystem() {
        let userLaunchAgentsPath = "\(user.homedir)/Library/LaunchAgents/"
        let userLaunchAgents = filemanager.filesInDirRecursive(path: userLaunchAgentsPath)
        captureLaunchData(urlLocations: userLaunchAgents, capturedLaunchFile: capturedLaunchFile)
    }
}
```

### Complexity: LOW

- 8-12 lines of code
- Pattern already established
- Helper methods available
- Additive change (backward compatible)

---

## Issue #73 - Events in storyline.csv Not Time-Sequenced

**Reporter:** n-sangsasitorn
**Opened:** Oct 3, 2025

### Summary

Events in `storyline.csv` are grouped by source (Safari, Firefox, Chrome, etc.) rather than being sorted chronologically across all sources.

### Root Cause: Critical Bug in sortCSV()

**Location:** `aftermath/Aftermath.swift`, lines 93-108

```swift
static func sortCSV(unsortedArr: [[String]]) throws -> [[String]] {
    var arr = unsortedArr
    let rejectedStrings = ["birth", "accessed"]
    try arr.sort { lhs, rhs in
        guard let lhsStr = lhs.first, let rhsStr = rhs.first,
              rejectedStrings.contains(lhsStr), rejectedStrings.contains(rhsStr) else {
            return false  // <-- BUG: ALWAYS returns false
        }
        // Date comparison code is NEVER reached
    }
    return arr
}
```

### Why It Fails

1. `lhs.first` extracts the timestamp (e.g., "2025-01-20T15:30:45Z")
2. Guard checks if timestamp is in `["birth", "accessed"]`
3. Timestamps are ISO8601 strings, **never** match "birth" or "accessed"
4. Guard **always fails**, closure **always returns false**
5. Sort has no ordering guidance, maintains insertion order
6. Result: Events stay grouped by source, not sorted by time

### Files to Modify

| File | Changes |
|------|---------|
| `aftermath/Aftermath.swift` | Fix `sortCSV()` logic |
| `tests/aftermath/AftermathTests.swift` | Add/update tests |

### Fix Options

**Option A: Remove rejectedStrings check (Recommended)**
```swift
static func sortCSV(unsortedArr: [[String]]) throws -> [[String]] {
    var arr = unsortedArr
    try arr.sort { lhs, rhs in
        guard let lhsStr = lhs.first, let rhsStr = rhs.first else {
            return false
        }
        let lhsDate = try Date("\(lhsStr)Z", strategy: .iso8601)
        let rhsDate = try Date("\(rhsStr)Z", strategy: .iso8601)
        return lhsDate > rhsDate
    }
    return arr
}
```

**Option B: Check TYPE column instead of timestamp**
- If "birth"/"accessed" should be in the type column, check `lhs[1]` not `lhs.first`

### Complexity: MEDIUM

- Fix itself is simple
- Original intent of rejectedStrings unclear
- Requires careful testing
- Bug introduced in commit `94caefa` (Nov 22, 2024)

---

## Issue #64 - Question - Browser Triage

**Reporter:** createchange
**Opened:** Oct 25, 2024

### Summary

Two questions about browser data collection:

1. Does `--disable browser-killswitch` mean no browser data is collected?
2. Why not copy database files instead of force-killing browsers?

### Current Behavior

```swift
// BrowserModule.swift, lines 31-35
if Command.disableFeatures["browser-killswitch"] == false {
    closeBrowsers()  // Force terminate all browsers (DEFAULT)
} else {
    self.log("Not force closing browsers")  // Just logs, still attempts collection
}
```

**Answer to Q1:** No, disabling browser-killswitch does NOT disable collection. It only prevents force-terminating browsers. Collection still attempts to proceed, but SQLite queries may silently fail if databases are locked.

**Answer to Q2:** The user's suggestion is valid - copying locked SQLite files works and would be less destructive.

### Current Issues

- No fallback when database is locked
- Silent failures with no user feedback
- Force-killing destroys forensic context (user session state)

### Suggested Improvements

| Task | Complexity | Effort |
|------|------------|--------|
| Document current behavior | Low | 30-60 min |
| Implement database copying | Medium | 4-8 hours |
| Add error handling/logging | Medium | 2-4 hours |

### Database Copying Approach

```swift
func gatherHistory() {
    // Instead of direct access:
    let tempCopy = createTempCopy(of: historyDB)
    defer { removeTempFile(tempCopy) }

    if sqlite3_open(tempCopy.path, &db) == SQLITE_OK {
        // Query the copy
    }
}
```

### Complexity: MEDIUM

- Would improve tool significantly
- Affects 6 browser files
- Need to handle WAL files
- Makes tool better for live incident response

---

## Recommended Priority Order

Based on complexity, impact, and effort:

1. **#74 - User LaunchAgents** - Low effort, high impact (security gap)
2. **#77 - Analysis Output Headers** - Low effort, improves usability
3. **#73 - Storyline Sorting** - Bug fix, critical for analysis functionality
4. **#76 - Browser Extensions** - Medium effort, improves forensic clarity
5. **#75 - AI Browsers** - Medium effort, adds new capability
6. **#64 - Browser Triage** - Documentation first, then optional enhancement

---

*Generated: January 2026*

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Aftermath is a Swift-based macOS incident response framework by JAMF. It runs as a command-line tool with two main modes:
- **Collection mode** (default): Gathers forensic artifacts from a potentially compromised Mac
- **Analysis mode** (`--analyze`): Parses collected data to generate timelines and storylines

Requires macOS 12.0+ and must be run as root with Full Disk Access.

## Build Commands

```bash
# Build the project
xcodebuild -scheme "aftermath"

# Build for release
xcodebuild -scheme "aftermath" -configuration Release

# Run tests
xcodebuild test -scheme "tests"

# Run a single test
xcodebuild test -scheme "tests" -only-testing:tests/AftermathTests/testGetPlistAsDict

# After building, binary is at build/Release/aftermath
sudo ./build/Release/aftermath
```

## Architecture

### Entry Point and Command Processing
- `Command.swift`: Main entry point (`@main`), handles CLI argument parsing and orchestrates module execution
- `main.swift`: Legacy entry point (superseded by Command.swift's `@main`)

### Module System
All collection/analysis modules inherit from `AftermathModule` and implement `AMProto`:
```swift
protocol AMProto {
    var name: String { get }
    var dirName: String { get }
    var description: String { get }
    var moduleDirRoot: URL { get }
}
```

Modules have a `run()` method that executes their logic. Each module creates its own subdirectory within the case directory.

### Collection Modules (in execution order)
1. `ESModule` - Endpoint Security logs via eslogger (macOS 13+)
2. `NetworkModule` - tcpdump packet capture + network connections
3. `SystemReconModule` - System information, installed apps, security status
4. `ProcessModule` - Process tree using TrueTree implementation
5. `PersistenceModule` - ASEPs (Launch Items, Cron, Login Items, BTM, etc.)
6. `FileSystemModule` - Browser data, Slack data, file metadata walking
7. `ArtifactsModule` - TCC database, configuration profiles, shell history, logs
8. `UnifiedLogModule` - macOS unified log parsing with customizable predicates

### Analysis Module
`AnalysisModule` coordinates post-collection analysis:
- `DatabaseParser` - Parses SQLite databases (TCC, LSQuarantine, browsers)
- `LogParser` - Parses unified logs
- `ProcessParser` - Parses process data
- `Timeline` - Creates chronological file timeline from metadata.csv
- `Storyline` - Correlates events across data sources

### Key Support Classes
- `CaseFiles`: Manages case directory structure and ZIP archive creation (uses ZIPFoundation)
- `Aftermath`: Static utilities for shell commands, plist parsing, date formatting
- `CHelpers`: C-level file attribute helpers (birth time, permissions, xattrs)

### Directory Structure Pattern
```
/aftermath          - Core framework classes
/analysis           - Analysis mode modules
/artifacts          - Artifact collection (TCC, profiles, logs)
/endpointSecurity   - ESModule for eslogger
/extensions         - Swift extensions (Data, FileManager, URL, String)
/filesystem         - File walking, browsers, Slack
/helpers            - C helpers
/network            - Network capture and connections
/persistence        - ASEP collection modules
/processes          - TrueTree-based process collection
/systemRecon        - System reconnaissance
/unifiedlogs        - Unified log collection
/tests              - XCTest unit tests
```

### Command-Line Options
Key options parsed in `Command.setup()`:
- `--analyze <path>`: Run analysis on collected ZIP
- `--deep` / `-d`: Deep filesystem scan (time-intensive)
- `--disable <features>`: Disable specific collection features
- `--es-logs <events>`: Customize Endpoint Security events
- `--logs <file>`: External unified log predicates file
- `-o <path>`: Output location (default: /tmp)

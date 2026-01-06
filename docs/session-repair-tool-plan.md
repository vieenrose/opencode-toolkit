# OpenCode Session Repair Tool - Implementation Plan

## Problem Statement

**OpenCode Issue #6418**: Error when changing models mid-session (GLM 4.7/MinMax 2.1 → Opus 4.5)
- Error: `Invalid 'signature' in 'thinking' block`
- Session becomes unrecoverable

**Root Cause**: Similar to Claude Code Issue #10199, thinking blocks (reasoning parts in OpenCode) can become corrupted during model switching, causing API errors that persist across all subsequent messages.

## OpenCode Storage Architecture

| Component | Location | Format |
|-----------|----------|--------|
| Data directory | `~/.local/share/opencode/` | N/A |
| Storage directory | `{data}/storage/` | N/A |
| Sessions | `{storage}/session/{projectID}/{sessionID}.json` | JSON |
| Messages | `{storage}/message/{sessionID}/{messageID}.json` | JSON |
| Parts | `{storage}/part/{messageID}/{partID}.json` | JSON |
| Backups | `{data}/repair-backups/{timestamp}/` | JSON |

### Session/Message Schema

**Session.Info** (packages/opencode/src/session/index.ts:39-79):
```typescript
{
  id: string,
  projectID: string,
  directory: string,
  title: string,
  time: { created: number, updated: number },
  // ... other fields
}
```

**MessageV2.Part** (packages/opencode/src/session/message-v2.ts):
```typescript
{
  type: "reasoning" | "text" | "tool" | ...,
  id: string,
  sessionID: string,
  messageID: string,
  // ... type-specific fields
}
```

**ReasoningPart** (corruption target):
```typescript
{
  type: "reasoning",
  text: string,
  time: { start: number, end?: number },
  // ... metadata may contain invalid signatures
}
```

## Solution Design

### 1. Core Module (`src/repair-core.ts`)

**Responsibilities**:
- Scan storage for all sessions
- Read session metadata and message/part relationships
- Detect corrupted reasoning parts
- Remove or repair corrupted parts

**Key Functions**:
```typescript
async function scanAllSessions(): Promise<SessionInfo[]>
async function loadSessionMessages(sessionID: string): Promise<SessionData>
async function detectCorruptedReasoningParts(sessionData: SessionData): Promise<CorruptedPart[]>
async function removeReasoningParts(sessionID: string, partIDs: string[]): Promise<void>
async function repairSession(sessionID: string): Promise<RepairResult>
```

**Detection Logic**:
- Check if reasoning part metadata contains "signature" field with invalid format
- Look for malformed JSON in reasoning text
- Flag parts from specific model transitions (e.g., GLM → Opus)

### 2. Session Scanner (`src/session-scanner.ts`)

**Responsibilities**:
- Discover all project directories
- List sessions per project
- Extract session metadata for TUI display
- Calculate statistics (message count, reasoning parts count)

**Key Functions**:
```typescript
async function findProjects(): Promise<ProjectInfo[]>
async function listSessionsForProject(projectID: string): Promise<SessionInfo[]>
async function getSessionStats(sessionID: string): Promise<SessionStats>
```

### 3. Backup Manager (`src/backup-manager.ts`)

**Responsibilities**:
- Create timestamped backups before repair
- Store backup metadata for rollback
- Support incremental backups (only changed files)
- Prune old backups

**Key Functions**:
```typescript
async function createBackup(sessionID: string): Promise<BackupID>
async function restoreBackup(backupID: string): Promise<void>
async function listBackups(sessionID: string): Promise<BackupInfo[]>
async function pruneOldBackups(maxAge: number): Promise<void>
```

**Backup Structure**:
```
~/.local/share/opencode/repair-backups/
├── 2025-01-06T10-30-00Z_session_<id>/
│   ├── session.json
│   ├── messages/
│   │   ├── msg1.json
│   │   └── msg2.json
│   └── parts/
│       ├── part1.json
│       └── part2.json
└── backup-manifest.json
```

### 4. TUI Module (`src/repair-tui.ts`)

**Library**: `blessed` (terminal UI framework)

**Screens**:

1. **Session List Screen**
   - Display sessions with:
     - Title
     - Project path
     - Message count
     - Last updated date
     - Corruption status (✓/✗)
   - Navigation: Arrow keys, Enter to select
   - Actions: `r` = repair, `d` = details, `q` = quit

2. **Session Detail Screen**
   - Show full session info
   - List all messages with reasoning parts
   - Highlight corrupted parts (red)
   - Navigation: `q` = back, `r` = repair, `f` = force repair

3. **Confirmation Dialog**
   - "Repair session: {title}?"
   - Show affected parts count
   - Options: `y` = yes, `n` = no, `a` = repair all

4. **Progress Screen**
   - Progress bar for repair operation
   - Status messages (backup → repair → verify)
   - Success/error summary

**Key Functions**:
```typescript
async function runTUI(): Promise<void>
function renderSessionList(sessions: SessionInfo[]): void
function renderSessionDetails(session: SessionData): void
function renderConfirmation(sessionID: string, corruptParts: number): Promise<boolean>
function renderProgress(stage: RepairStage, progress: number): void
```

### 5. CLI Entry Point (`src/index.ts`)

**Commands**:
```bash
opencode-repair              # Launch TUI (default)
opencode-repair --list       # List all sessions
opencode-repair --check <id> # Check specific session
opencode-repair --repair <id> # Repair specific session (non-interactive)
opencode-repair --backup <id> # Create backup of session
opencode-repair --restore <backup-id> # Restore from backup
opencode-repair --help       # Show help
```

**Exit Codes**:
- `0` = Success
- `1` = Error
- `2` = User cancelled
- `3` = No corruption found

## Implementation Steps

1. **Phase 1: Core Infrastructure** (2-3 hours)
   - Initialize project structure
   - Implement storage scanner
   - Create backup manager
   - Write tests for core functions

2. **Phase 2: Repair Logic** (2-3 hours)
   - Implement corruption detection
   - Write part removal logic
   - Add validation after repair
   - Test with sample corrupted sessions

3. **Phase 3: TUI Interface** (3-4 hours)
   - Set up blessed framework
   - Build session list screen
   - Add detail and confirmation screens
   - Implement progress display
   - Handle keyboard navigation

4. **Phase 4: CLI & Integration** (1-2 hours)
   - Implement CLI argument parsing
   - Add non-interactive repair mode
   - Write comprehensive help text
   - Create usage documentation

5. **Phase 5: Testing & Polish** (2 hours)
   - End-to-end testing
   - Error handling improvements
   - Performance optimization
   - Documentation review

**Total Estimated Time**: 10-14 hours

## Dependencies

```json
{
  "dependencies": {
    "blessed": "^0.1.81",
    "xdg-basedir": "^4.0.0"
  },
  "devDependencies": {
    "@types/bun": "latest",
    "typescript": "^5.0.0"
  }
}
```

## File Structure

```
opencode-repair/
├── package.json
├── tsconfig.json
├── README.md
├── src/
│   ├── index.ts              # CLI entry point
│   ├── repair-core.ts        # Core repair logic
│   ├── session-scanner.ts    # Session discovery
│   ├── backup-manager.ts     # Backup/restore
│   ├── repair-tui.ts         # TUI interface
│   ├── models/
│   │   ├── session-info.ts
│   │   ├── repair-result.ts
│   │   └── backup-info.ts
│   └── utils/
│       ├── logger.ts
│       └── validation.ts
└── test/
    ├── core.test.ts
    ├── backup.test.ts
    └── integration.test.ts
```

## Safety Considerations

1. **Always backup before repair**
   - Mandatory backup creation
   - Verify backup integrity before proceeding
   - Store backup ID in session metadata

2. **Validation after repair**
   - Verify session is readable
   - Check message/part relationships
   - Test if OpenCode can load the session

3. **Rollback support**
   - Easy restore from backup
   - Keep multiple backup versions
   - Automatic cleanup of old backups (configurable)

4. **Logging**
   - Detailed logs of all operations
   - Store logs in `~/.local/share/opencode/repair-logs/`
   - Include timestamps and session IDs

## Testing Strategy

1. **Unit Tests**
   - Test each function in isolation
   - Mock filesystem operations
   - Verify corruption detection logic

2. **Integration Tests**
   - Test full repair flow
   - Create test sessions with known corruption
   - Verify backup/restore cycle

3. **Manual Testing**
   - Test TUI on different terminals
   - Test with real OpenCode sessions
   - Verify OpenCode can load repaired sessions

## Success Criteria

- ✅ TUI successfully lists all sessions with titles
- ✅ Corrupted reasoning parts are detected accurately
- ✅ Repair removes corrupted parts without breaking sessions
- ✅ Backups are created before every repair
- ✅ Repaired sessions load successfully in OpenCode
- ✅ CLI provides both interactive and non-interactive modes
- ✅ Documentation is clear and complete

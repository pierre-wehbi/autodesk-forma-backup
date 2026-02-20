# AI Coding Agent Instructions for ACC Backup

## Project Overview

ACCBackup is a C# .NET 9 CLI application that backs up Autodesk Construction Cloud (ACC) projects via Autodesk Platform Services (APS). It features incremental backup via manifest tracking, concurrent folder enumeration (producer), sequential file downloads (consumer), and producer-consumer pipelines for efficient high-volume project backup (170+ projects @ 225GB).

## Architecture

```
ApiClient/                  → Autodesk APS API abstraction with retry/rate-limit handling
  Entities/                 → Domain models: Project, Folder, File, BackupManifest
  RestApiResponses/         → Deserialization contracts for APS JSON responses
  TwoLeggedApiClient.cs     → Fluent builder providing IApiClient implementation
  ApiClientConfiguration.cs → Retry policy, HTTP client, auth setup
  
Backup/                     → Orchestration, file I/O, backup workflow
  Program.cs                → DI setup, Serilog, CLI arg parsing
  Backup.cs                 → Producer-consumer pipeline, manifest management
  CommandLineArgs.cs        → CLI schema with [Option] attributes
  BackupConfiguration.cs    → Parsed/validated args with defaults
  Entities/                 → Backup-local entities (ProjectBackup, etc)

Library.Logger/             → Serilog abstraction (ILogger) enabling pluggable logging
Library.Testing/            → MockHttpMessageHandler, test helpers, FakeData
Library.Extensions/         → FlattenRecursive utility for folder hierarchies
Library.SecretsManager/     → Cross-platform secret/credential storage (optional)
```

**Data Flow (High-Level)**:
1. `Backup.Run()` → parse config + DI setup → `IApiClient.GetProjects()` → filter projects
2. **Producer** (concurrent): `SemaphoreSlim(3)` enumerates project folders recursively
3. **Channel**: Queues `ProjectBackup` with enumerated folder structure
4. **Consumer** (sequential): Downloads files respecting APS rate limits
5. **Manifest**: `BackupManifest.json` tracks file metadata (FileId, VersionNumber, LastModifiedTime, StorageSize)
6. **Storage**: Single sync dir (`--backupstorotate 1`) or timestamped versions (`2+`)

## Critical Patterns

### 1. **Fluent Builder for API Client Configuration**
```csharp
// ApiClient/TwoLeggedApiClient.cs - implements staged builder interfaces
var client = TwoLeggedApiClient
    .Configure()
    .WithClientId(id)
    .AndClientSecret(secret)
    .ForAccount(accountId)
    .WithOptions(opts => opts.RetryAttempts = 15)
    .Create();
```
**Why**: Type-safe fluent API enforces required configuration order. Each stage is an interface (e.g., `IClientIdSelectionStage`) constraining the next call.

### 2. **Producer-Consumer Pipeline with System.Threading.Channels**
Located in `Backup/Backup.cs` (lines ~100+):
- **Producer**: Concurrent folder enumeration (up to 3 projects) using `SemaphoreSlim` to prevent API overwhelm
- **Channel**: Unbounded, multiple writers (enumeration tasks), single reader (download task)
- **Consumer**: Sequential downloads to respect rate limits
- **Why**: Overlaps slow enumeration with fast download prep; avoids thread pool starvation vs explicit Task.Run loops

### 3. **Incremental Backup with Manifest Tracking**
- `BackupManifest.json` created per backup directory; stores FileId, VersionNumber, LastModifiedTime, StorageSize
- On next run, unchanged files copied from previous backup (verified by hash/metadata); only modified/new files re-downloaded
- **Default mode** (`--backupstorotate 1`): Single sync directory, auto-cleanup of obsolete files
- **Versioned mode** (`--backupstorotate 2+`): Timestamped folders (e.g., `yyyy-MM-dd_HH-mm`), no cleanup
- **Force option**: `--force-full-download` bypasses incremental logic

### 4. **Retry Policy with Exponential Backoff and 429 Handling**
`ApiClientConfiguration.GetRetryPolicy()` (Polly library):
- Exponential backoff: `retry * InitialRetryInSeconds` (default: 15 retries × 2s = 4 min total)
- Special: HTTP 429 (rate limit) extracts `Retry-After` header, capped to 10 minutes
- Skips retry on HTTP 403 (Forbidden) — permanent auth failure
- Logs all retries at Warn level with timing info

### 5. **Configuration via Command-Line Parser Library**
`Backup/CommandLineArgs.cs` + `BackupConfiguration.cs`:
- Uses `CommandLine.Parser` (NuGet) with `[Option]` attributes
- Maps CLI args → `BackupConfiguration` properties
- Defaults: `MaxDegreeOfParallelism=8`, `RetryAttempts=15`, `InitialRetryInSeconds=2`
- Secrets (ClientId, ClientSecret) sourced from CLI; no hardcoding

### 6. **Dependency Injection & Serilog Setup**
`Backup/Program.cs` (Microsoft.Extensions.Hosting):
- Parses args early to determine log level (`--trace-logging`, `--debug-logging`)
- Configures Serilog code-first (console + daily rolling file logs in `Logs/` directory)
- Registers `ILogger` (Library.Logger abstraction) as singleton wrapping Serilog
- No dependency on external config files — all code-driven

### 7. **Thread-Safe Directory Creation**
`ApiClient/ApiClient.cs`:
```csharp
private static readonly object _directoryCreationLock = new object();
// Double-check locking pattern when creating directories during parallel downloads
```
**Why**: Parallel file downloads may race on directory creation; static lock ensures atomicity.

### 8. **Recursive Hierarchy Flattening**
`Library.Extensions/Extensions.cs`:
```csharp
public IEnumerable<Folder> SubfoldersRecursive => 
    RootFolder?.Subfolders.FlattenRecursive(x => x.Subfolders);
```
Fold down nested folder tree to flat enumerable for LINQ operations. Used for recursive file listing.

## Domain Entity Relationships

### Project → Folder → File hierarchy
- **Project** (root): `RootFolder` property populated via `GetProject(projectId, getRootFolderContents:true)`
- **Folder**: Recursive structure via `Subfolders` collection; read-only properties like `IsRootFolder`, `SubfoldersRecursive`, `FilesRecursive`
- **File**: Domain model stores APS metadata (FileId, VersionNumber, LastModifiedTime, StorageSize, DownloadUrl)
- **Key Pattern**: Use `SubfoldersRecursive` and `FilesRecursive` properties (computed from recursive flattening) rather than manual recursion

### BackupManifest Structure
```json
{
  "backupDate": "2025-08-19T14:22:30.1234567",
  "backupDirectory": "C:\\backups\\",
  "files": {
    "ProjectName/FolderPath/FileName.ext": {
      "fileId": "urn:adsk.objects:os.object:fs.file:aHR0cHM...",
      "versionNumber": 5,
      "lastModifiedTime": "2025-08-19T12:00:00Z",
      "storageSize": 1048576
    }
  }
}
```
- Created per backup after first download
- Keys are relative file paths (case-insensitive dictionary for cross-platform)
- Compared on subsequent runs to skip unchanged files
- Enables 75% bandwidth reduction for mostly-static projects

## CLI Command Examples & Common Workflows

### Basic Backup (In-Place Sync Mode)
```bash
# Default mode: single directory, auto-deletes obsolete files
dotnet run --project Backup -- \
  --backupdirectory ./backups \
  --clientid <CLIENT_ID> \
  --clientsecret <CLIENT_SECRET> \
  --accountid <ACCOUNT_ID>
```

### Versioned Backups (Keep Multiple Snapshots)
```bash
# Creates yyyy-MM-dd_HH-mm subdirectories; no cleanup
dotnet run --project Backup -- \
  --backupdirectory ./backups \
  --clientid <CLIENT_ID> \
  --clientsecret <CLIENT_SECRET> \
  --accountid <ACCOUNT_ID> \
  --backupstorotate 3
```

### Selective Project Backup
```bash
# Backup only specific projects by name or ID
dotnet run --project Backup -- \
  --backupdirectory ./backups \
  --clientid <CLIENT_ID> \
  --clientsecret <CLIENT_SECRET> \
  --accountid <ACCOUNT_ID> \
  --projectstobackup "Project A,Project B,12345-abcd-uuid"
```

### Force Full Re-Download
```bash
# Bypass incremental logic; re-download all files
dotnet run --project Backup -- \
  --backupdirectory ./backups \
  --clientid <CLIENT_ID> \
  --clientsecret <CLIENT_SECRET> \
  --accountid <ACCOUNT_ID> \
  --force-full-download
```

### Dry-Run (Preview Structure)
```bash
# Create 0-byte placeholders; show what would be downloaded
dotnet run --project Backup -- \
  --backupdirectory ./backups \
  --clientid <CLIENT_ID> \
  --clientsecret <CLIENT_SECRET> \
  --accountid <ACCOUNT_ID> \
  --dryrun
```

### Debug Logging
```bash
# Show detailed trace logs in console + Logs/log-*.txt
dotnet run --project Backup -- \
  --backupdirectory ./backups \
  --clientid <CLIENT_ID> \
  --clientsecret <CLIENT_SECRET> \
  --accountid <ACCOUNT_ID> \
  --trace-logging
```

### Email Notifications (SMTP)
```bash
dotnet run --project Backup -- \
  --backupdirectory ./backups \
  --clientid <CLIENT_ID> \
  --clientsecret <CLIENT_SECRET> \
  --accountid <ACCOUNT_ID> \
  --smtphost smtp.gmail.com \
  --smtpport 587 \
  --smtpusername user@gmail.com \
  --smtppassword "<app-password>" \
  --smtpfromemail backup@company.com \
  --smtpfromname "ACC Backup Service" \
  --smtptoemail admin@company.com \
  --smtpenablessl
```

## Error Handling & Retry Behavior

### HTTP 429 (Rate Limit)
- **Detection**: `HttpRequestException` with `StatusCode = TooManyRequests`
- **Response**: Extracts `Retry-After` header from APS response; caps to 10-minute max
- **Behavior**: Waits specified seconds then retries immediately (no exponential backoff on 429)
- **Logged at**: Warn level with `Retry {n}/{max} in {seconds}s`

### HTTP 403 (Forbidden)
- **Detection**: `HttpRequestException` with `StatusCode = Forbidden`
- **Behavior**: **Does NOT retry** — permanent auth failure, logged and continues to next file/project
- **Cause**: Invalid credentials, insufficient project access, or revoked token

### HTTP 5xx (Server Errors)
- **Behavior**: Retries with exponential backoff (retry# × InitialRetryInSeconds)
- **Default**: 15 attempts × 2s = 30s, 60s, 90s, 120s... up to 4+ minutes total

### Transient Errors (Network, Timeouts)
- **Handled by**: Polly `Handle<Exception>` that catches anything not explicitly skipped
- **Logged at**: Warn level with full exception context

### File Download Verification
- After copy from previous backup, hash/metadata verified against manifest
- If mismatch: falls back to re-download from APS
- If re-download fails after retries: file skipped with error logged

## Threading & Concurrency Model

### Why SemaphoreSlim(3) for Enumeration?
- **3 concurrent projects** balance API rate limits vs latency overlap
- **SemaphoreSlim over Parallel.ForEach**: More explicit, easier to tune
- Located in `Backup.cs` line ~115: controls folder tree enumeration only
- Downloads remain sequential (`MaxDegreeOfParallelism` for file downloads, not enumeration)

### Why System.Threading.Channels over BlockingCollection?
- **Lower GC pressure**: Channels use more efficient memory pooling
- **Async-friendly**: `WriteAsync()` / `ReadAsync()` without spinning
- **Single reader, multiple writers**: Channel configuration enforces this pattern
- Producer tasks write enumerated projects; single download task reads and downloads sequentially

### Thread Safety Considerations
1. **Directory creation lock** (`_directoryCreationLock` in ApiClient.cs): Double-check pattern prevents race on `Directory.CreateDirectory()`
2. **ConcurrentBag for file collections**: Used during concurrent enumeration
3. **BackupManifest dictionary**: Case-insensitive after deserialization for cross-platform compatibility
4. **No shared state between producer tasks**: Each project enumeration is independent

## Testing Strategy & Patterns

### Unit Tests (Xunit + FluentAssertions)
Located in `ApiClient.UnitTests/`, `Backup.UnitTests/`:
```csharp
[Fact]
public void Building_WithOptions_Should_HaveAppropriateValues()
{
    // Arrange: Use fluent builder same as production
    var client = TwoLeggedApiClient
        .Configure()
        .WithClientId(clientId)
        .AndClientSecret(clientSecret)
        .ForAccount(accountId)
        .WithOptions(opts => opts.RetryAttempts = 15)
        .Create();
    
    // Act & Assert
    client.Config.RetryAttempts.Should().Be(15);
}
```

### Mocking HTTP Responses
From `Library.Testing/MockHttpMessageHandler.cs`:
```csharp
var handler = new MockHttpMessageHandler();
handler.AddResponse(
    HttpMethod.Get, 
    "https://developer.api.autodesk.com/project/v1/hubs/...",
    projectJsonResponse);

var config = new ApiClientConfiguration(...);
config.HttpClient = new HttpClient(handler);
```

### Test Data
- **FakeData.cs**: Realistic APS JSON responses for projects, folders, files
- **ExampleRestApiResponses/**: JSON fixture files (ProjectResponse.json, etc.)
- **Usage**: Deserialize and assert domain model mapping

### Integration Tests (Real APS Communication)
- Located in `ApiClient.IntegrationTests/`
- Requires valid APS credentials in environment or code
- Skipped in CI unless explicitly enabled
- Use for validating retry policy, actual API changes

### Testing the Producer-Consumer Pipeline
```csharp
// Mock IApiClient to control enumeration output
var mockApiClient = new Mock<IApiClient>();
mockApiClient
    .Setup(x => x.GetProjects())
    .ReturnsAsync(new List<Project> { fakeProject });

var backup = new Backup(config, mockApiClient.Object, logger);
await backup.Run();

// Assert channel wrote projects, downloads completed
```

## Code Style & Conventions

### Naming
- **Private fields**: Camel case with underscore prefix (`_accessToken`, `_projectsProcessed`)
- **Properties**: PascalCase (public auto-properties and explicit getters)
- **Methods**: PascalCase, async methods end with `Async` only if returning Task
- **Constants**: UPPER_CASE in rare cases; prefer `const` at class level

### Comments
- **API comments**: Include links to Autodesk documentation (e.g., `https://forge.autodesk.com/en/docs/data/v2/reference/...`)
- **Complex logic**: Explain "why" not "what"; code is self-documenting for algorithms
- **Section headers**: `/*\n * These properties are mapped from Autodesk Api\n */` pattern separates mapped vs computed properties

### Async / Await
- **Consistently async**: All I/O uses `async Task` or `async Task<T>`, never `.Result` or `.Wait()`
- **CancellationToken**: Passed through call chains where cancellation is possible (e.g., downloads)
- **ConfigureAwait**: Not used (console app, no sync context issues)

### Null Handling
- **Null-conditional**: `RootFolder?.Subfolders` pattern throughout
- **Null-coalescing**: `HubId ??= $"b.{AccountId}"` for defaults
- **Validation**: Early throws on misconfiguration (e.g., missing ClientId in fluent builder stages)

### Logging
- **Abstraction**: Always inject `ILogger`, never `Serilog.ILog`
- **Levels**: Trace (very verbose), Debug (method entry/exit), Info (key milestones), Warn (retries, unexpected), Error/Fatal (failure)
- **Null-safe logging**: `Logger?.Debug(message)` because logger is optional in ApiClient

## Extending & Customization Points

### Add New APS Endpoints
1. Extend `IApiClient` interface with new method
2. Implement in `ApiClient.cs`:
   - Use `Config.RetryPolicy.ExecuteAsync()` to wrap HTTP call
   - Call `EnsureAccessToken()` before HTTP request
   - Handle response status codes via `HandleUnsuccessfulStatusCode()`
   - Return domain model entities
3. Create corresponding `RestApiResponse.cs` DTO for JSON deserialization
4. Add unit test with `MockHttpMessageHandler`

### Customize Retry Policy
In `ApiClientConfiguration.GetRetryPolicy()`:
- Change exponential backoff multiplier: `TimeSpan.FromSeconds(retryAttempt * initialRetryDelayInSeconds)`
- Change max retry cap for 429: `Math.Min(..., 600)` → adjust upper bound
- Add retry-on-specific-exceptions: Modify `Handle<Exception>` predicate
- Example: `Handle<Exception>(ex => ex is HttpRequestException && ex.StatusCode != HttpStatusCode.Forbidden && ex.StatusCode != HttpStatusCode.NotFound)`

### Change Concurrency Limits
- **Enumeration**: `SemaphoreSlim(3, 3)` in `Backup.Run()` — increase for faster enumeration (watch API rate limits)
- **Downloads**: `--maxdegreeofparallelism` CLI option — default 8, increase for faster downloads on good connections
- **Channel buffer**: Currently `Channel.CreateUnbounded()` — change to bounded if memory is concern

### Add Custom Logging Output
In `Backup/Program.cs`, extend Serilog config:
- **Sinks**: Add `.WriteTo.Seq(...)` for centralized logging
- **Enrichers**: Add `.Enrich.WithProperty("Environment", "Production")`
- **Filters**: Add `.Filter.ByExcluding(le => le.Level < LogEventLevel.Warning)` for production filtering

## Debugging & Troubleshooting

### Enable Full Trace Logging
```bash
# Reveals every HTTP request/response, retry attempts, auth flow
dotnet run --project Backup -- ... --trace-logging

# Output in console + Logs/log-*.txt daily rolling files
```

### Common Issues

**401 Unauthorized**
- Cause: Invalid credentials or expired token
- Fix: Verify ClientId and ClientSecret in APS app dashboard
- Check: `ApiClient.EnsureAccessToken()` — token refresh happens automatically via retry

**429 Too Many Requests**
- Cause: Enumeration is too aggressive or concurrent downloads competing with enumeration
- Fix: Reduce `--maxdegreeofparallelism` from 8 to 4; or reduce `SemaphoreSlim(3)` to `SemaphoreSlim(1)` for enumeration
- Expected: Logs show "Rate limit (429) error... Retry N/15 in Xs"

**Files Not Syncing After Incremental Backup**
- Cause: File metadata changed but not detected (edge case: same size/date, different content)
- Fix: Use `--force-full-download` to re-download all files
- Prevention: APS API uses FileId/VersionNumber which is reliable; falls back to re-download on manifest mismatch

**OutOfMemoryException During Enumeration**
- Cause: Extremely large projects (1000s of nested folders) loaded into memory
- Fix: (Rare) Increase heap size: `dotnet run --generate-heap-dump MyApp.exe -- <args>` or in production script

### Performance Profiling
- Use `--debug-logging` to see enumeration & download timing per project
- Monitor `_totalWaitTime` and `_totalDownloadTime` in Backup.cs output
- Bottleneck typically: APS API rate limits (not network, not disk)

## Project Organization & File Locations

| Component | Location | Responsibility |
|-----------|----------|-----------------|
| API Contract | `ApiClient/IApiClient.cs` | Defines GetProject(s), GetFolder(s), etc. |
| API Implementation | `ApiClient/ApiClient.cs` | HTTP, auth, retries, streaming downloads |
| Fluent Builder | `ApiClient/TwoLeggedApiClient.cs`, `TwoLeggedApiClientStages.cs` | Enforces configuration order |
| Auth & Retry | `ApiClient/ApiClientConfiguration.cs`, `ApiClientOptions.cs` | Polly policy, token refresh, HTTP client |
| Domain Models | `ApiClient/Entities/*.cs` | Project, Folder, File, BackupManifest |
| API DTOs | `ApiClient/RestApiResponses/*.cs` | Deserialization targets from APS |
| CLI Entry | `Backup/Program.cs` | DI, Serilog setup, arg parsing |
| Orchestration | `Backup/Backup.cs` | Producer-consumer pipeline, manifest management |
| CLI Schema | `Backup/CommandLineArgs.cs`, `BackupConfiguration.cs` | Option attributes, parsing, defaults |
| Logger Abstraction | `Library.Logger/ILogger.cs`, `SerilogLogger.cs` | Pluggable logging interface |
| Test Utilities | `Library.Testing/*.cs` | MockHttpMessageHandler, FakeData |
| Recursive Utilities | `Library.Extensions/Extensions.cs` | FlattenRecursive LINQ helper |

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `Backup/Program.cs` | Main entry; DI setup, Serilog config |
| `Backup/Backup.cs` | Backup orchestration, producer-consumer pipeline |
| `ApiClient/IApiClient.cs` | Core contract: GetProject(s), GetFolder(s) |
| `ApiClient/TwoLeggedApiClient.cs` | Fluent builder stages |
| `ApiClient/ApiClientConfiguration.cs` | Retry policy + HTTP client + auth |
| `ApiClient/Entities/Project.cs` | Domain model + recursive props (SubfoldersRecursive, FilesRecursive) |
| `Backup/CommandLineArgs.cs` | CLI schema |
| `Backup/BackupConfiguration.cs` | Parsed args + defaults |
| `Library.Logger/ILogger.cs` | Logging abstraction (decouples from Serilog) |

## Solution-Wide Project Dependencies

| Project | Type | Purpose |
|---------|------|---------|
| `autodesk-construction-cloud-backup.sln` | Solution | Container for all projects |
| `ApiClient` | Class Library | Core APS API abstraction with entity models |
| `ApiClient.UnitTests` | Test | Unit tests for API client, builder, retry logic |
| `ApiClient.IntegrationTests` | Test | Integration tests with real APS (requires credentials) |
| `ApiClient.Benchmarks` | Benchmark | Performance testing (BenchmarkDotNet) |
| `ApiClient.UsageExample` | Console | Example of configuring and using API client |
| `Backup` | Console App | Main CLI entry point; orchestrates backup |
| `Backup.UnitTests` | Test | Tests for backup pipeline, manifest, filtering |
| `Library.Logger` | Class Library | Serilog abstraction (ILogger interface) |
| `Library.Logger.UnitTests` | Test | Logger configuration and output tests |
| `Library.Logger.UsageExample` | Console | Example of using logger abstraction |
| `Library.Extensions` | Class Library | LINQ utilities (FlattenRecursive) |
| `Library.Testing` | Class Library | Shared test utilities (MockHttpMessageHandler, FakeData) |
| `Library.SecretsManager` | Class Library | Cross-platform credential storage (Windows Credential Manager, Linux keyring, macOS Keychain) |

## Dependency Graph

```
Backup (main entry)
  ├─ ApiClient (core API)
  │   ├─ Library.Logger (logging)
  │   └─ Library.Extensions (utilities)
  ├─ Library.Logger (logging)
  └─ CommandLine.Parser NuGet (CLI argument parsing)

ApiClient
  ├─ Polly NuGet (retry policy)
  ├─ Newtonsoft.Json NuGet (REST response deserialization)
  ├─ System.IdentityModel.Tokens.Jwt NuGet (JWT token handling)
  └─ Library.Extensions (FlattenRecursive utility)

Tests
  ├─ Xunit NuGet (test framework)
  ├─ FluentAssertions NuGet (assertions)
  ├─ Moq or similar (mocking)
  └─ Library.Testing (MockHttpMessageHandler, FakeData)
```

## Build & Execution

```bash
# Restore & build (VS or CLI)
dotnet build autodesk-construction-cloud-backup.sln

# Run all tests
dotnet test

# Run specific test project
dotnet test ApiClient.UnitTests/ApiClient.UnitTests.csproj

# Run backup app with minimal args (in-place sync mode)
dotnet run --project Backup -- \
  --backupdirectory ./backups \
  --clientid YOUR_CLIENT_ID \
  --clientsecret YOUR_CLIENT_SECRET \
  --accountid YOUR_ACCOUNT_ID

# Run with custom concurrency and versioning
dotnet run --project Backup -- \
  --backupdirectory ./backups \
  --clientid YOUR_CLIENT_ID \
  --clientsecret YOUR_CLIENT_SECRET \
  --accountid YOUR_ACCOUNT_ID \
  --maxdegreeofparallelism 4 \
  --backupstorotate 3
```

## Common Modifications

1. **Add new retry logic**: Modify `ApiClientConfiguration.GetRetryPolicy()` 
2. **Change concurrency**: Adjust `SemaphoreSlim(3, 3)` in `Backup.Run()` for enumeration or `MaxDegreeOfParallelism` for downloads
3. **Extend logging**: All classes accept `ILogger` via constructor/property; use `Logger?.Trace/Debug/Info/Warn/Error`
4. **Add new Autodesk endpoints**: Extend `IApiClient` interface, implement in `ApiClient`, handle auth + retry automatically
5. **Modify backup structure**: Alter logic in `Backup.Run()` around `BackupsToRotate` modes or manifest loading

## Performance Considerations

- **Incremental backup** saves 75% bandwidth on unchanged files
- **Concurrent enumeration** (3 projects) exploits APS latency while respecting rate limits
- **SemaphoreSlim over Parallel.ForEach**: More explicit control; easier to tune for API constraints
- **System.Threading.Channels** over BlockingCollection: Lower GC pressure, async-friendly

## Important Notes & Edge Cases

- **HubId default**: If not provided, defaults to `b.{AccountId}` per APS conventions; stored in `BackupConfiguration`
- **In-place sync mode** (`--backupstorotate 1`): Automatically enabled as default; deletes files no longer in ACC. Opt-in to this by not specifying `--backupstorotate` or setting to 1.
- **Versioned backups** (`--backupstorotate 2+`): Creates `yyyy-MM-dd_HH-mm` subdirectories; must manually rotate old backups if storage is concern
- **No hardcoded config**: All settings via CLI args; use `Library.SecretsManager` to store credentials securely if needed (not default)
- **Cross-platform paths**: BackupManifest uses case-insensitive dictionary; file paths use `Path.Combine()` not hardcoded `\`
- **DryRun mode**: Creates 0-byte placeholder files and full directory structure without actual downloads; useful for testing manifests
- **Project filtering**: Both `--projectstobackup` and `--projectstoexclude` supported; comma-separated by name or UUID
- **SMTP optional**: All SMTP settings are null by default; omit to skip email notifications
- **Access token expiry**: Automatically refreshed via `EnsureAccessToken()` before each API call; transparent to caller

## Key Files by Responsibility

| Task | Primary File | Secondary Files |
|------|--------------|------------------|
| Make an APS API call | `ApiClient/ApiClient.cs` | `ApiClientConfiguration.cs` (retry), `Entities/*.cs` (models) |
| Configure the backup | `Backup/BackupConfiguration.cs` | `Backup/CommandLineArgs.cs` (CLI schema) |
| Understand retry logic | `ApiClient/ApiClientConfiguration.cs` | Test: `ApiClient.UnitTests/RetryPolicyTests.cs` |
| Fix threading issue | `Backup/Backup.cs` (enumeration) or `ApiClient/ApiClient.cs` (directory creation) | — |
| Add email feature | `Backup/BackupConfiguration.cs` (SMTP props) | `Backup/Backup.cs` (call sender) |
| Write a test | Corresponding `.UnitTests/` project | `Library.Testing/MockHttpMessageHandler.cs` |
| Understand domain model | `ApiClient/Entities/Project.cs`, `Folder.cs`, `File.cs` | `Library.Extensions/Extensions.cs` (FlattenRecursive) |

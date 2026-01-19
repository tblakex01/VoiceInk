# CLAUDE.md - VoiceInk Windows Migration Guide

## Project Overview

**VoiceInk** is a voice transcription application originally built for macOS using Swift/SwiftUI. This document guides the Windows migration effort to create a C# WinUI 3 version with full feature parity.

**Current Status:** ~5-10% complete (macOS codebase modernized, no Windows code exists yet)

**Migration Plan:** See [WINDOWS_MIGRATION_PLAN.md](./WINDOWS_MIGRATION_PLAN.md) for detailed phased implementation.

---

## Tech Stack

### Windows Application (Target)

#### Core Technologies
- **Language:** C# 11+ (.NET 7 or .NET 8)
- **UI Framework:** WinUI 3 (Windows App SDK 1.4+)
- **Architecture:** MVVM (Model-View-ViewModel)
- **Dependency Injection:** Microsoft.Extensions.DependencyInjection

#### Key Libraries
- **Audio Recording:** NAudio or CSCore (WASAPI wrapper)
- **Database:** Entity Framework Core 7+ with SQLite
- **HTTP Client:** System.Net.Http.HttpClient
- **JSON:** System.Text.Json
- **Testing:** xUnit + FluentAssertions + Moq
- **Logging:** Microsoft.Extensions.Logging with Serilog

#### Native Interop
- **Whisper.cpp:** P/Invoke wrapper for C API
- **Windows APIs:** User32.dll, Kernel32.dll for system integration
- **Hotkeys:** SetWindowsHookEx (WH_KEYBOARD_LL)
- **Clipboard:** System.Windows.Clipboard or Win32 APIs

### macOS Application (Reference)
- **Language:** Swift 5.9+
- **UI Framework:** SwiftUI + AppKit
- **Audio:** CoreAudio (AUHAL)
- **Database:** SwiftData
- **Key Dependencies:** KeyboardShortcuts, LaunchAtLogin, SelectedTextKit

---

## Project Structure

```
VoiceInk.Windows/
├── VoiceInk.Core/                      # Platform-agnostic business logic
│   ├── Models/                         # Data models, DTOs
│   ├── Services/                       # Business services
│   │   ├── CloudTranscription/         # Cloud provider implementations
│   │   ├── AIEnhancement/              # AI text enhancement
│   │   └── Interfaces/                 # Service contracts
│   ├── Data/                           # EF Core DbContext
│   └── Configuration/                  # App configuration models
│
├── VoiceInk.Audio/                     # Audio recording & device management
│   ├── WasapiRecorder.cs              # Core WASAPI recording implementation
│   ├── AudioDeviceManager.cs          # Device enumeration & notifications
│   ├── MediaController.cs             # System audio control
│   └── Interfaces/                     # Audio service contracts
│
├── VoiceInk.Whisper/                   # whisper.cpp integration
│   ├── WhisperInterop.cs              # P/Invoke wrapper
│   ├── WhisperContext.cs              # High-level API
│   └── Native/                         # Native DLL binaries
│       ├── x64/whisper.dll
│       └── ARM64/whisper.dll
│
├── VoiceInk.UI/                        # WinUI 3 presentation layer
│   ├── Views/                          # XAML views
│   │   ├── MainWindow.xaml
│   │   ├── SettingsWindow.xaml
│   │   └── HistoryWindow.xaml
│   ├── ViewModels/                     # View models (MVVM)
│   ├── Controls/                       # Custom controls
│   ├── Converters/                     # Value converters
│   ├── Services/                       # UI-specific services
│   └── App.xaml                        # Application entry point
│
├── VoiceInk.Tests/                     # Test projects
│   ├── VoiceInk.Core.Tests/           # Unit tests for Core
│   ├── VoiceInk.Audio.Tests/          # Unit tests for Audio
│   ├── VoiceInk.Whisper.Tests/        # Unit tests for Whisper
│   └── VoiceInk.Integration.Tests/    # Integration tests
│
├── VoiceInk.Installer/                 # MSIX packaging project
│   ├── Package.appxmanifest
│   └── Assets/                         # App icons, splash screens
│
└── VoiceInk.sln                        # Visual Studio solution
```

---

## Development Guidelines

### Code Quality Standards

#### 1. Test Coverage Requirements
- **Minimum Coverage:** 80% for all new code
- **Critical Paths:** 95%+ coverage for:
  - Audio recording pipeline
  - Transcription services
  - Hotkey management
  - Data persistence
  - AI enhancement

#### 2. Testing Strategy

**Unit Tests (xUnit + Moq)**
```csharp
// Example test structure
public class AudioDeviceManagerTests
{
    private readonly Mock<ILogger<AudioDeviceManager>> _loggerMock;
    private readonly AudioDeviceManager _sut;

    public AudioDeviceManagerTests()
    {
        _loggerMock = new Mock<ILogger<AudioDeviceManager>>();
        _sut = new AudioDeviceManager(_loggerMock.Object);
    }

    [Fact]
    public void GetAvailableInputDevices_ShouldReturnAtLeastOneDevice()
    {
        // Arrange & Act
        var devices = _sut.GetAvailableInputDevices();

        // Assert
        devices.Should().NotBeEmpty();
        devices.Should().OnlyContain(d => !string.IsNullOrEmpty(d.Id));
    }

    [Theory]
    [InlineData("device-id-1")]
    [InlineData("device-id-2")]
    public void GetDeviceById_WithValidId_ShouldReturnDevice(string deviceId)
    {
        // Test implementation
    }
}
```

**Integration Tests**
```csharp
public class RecordingWorkflowTests : IDisposable
{
    private readonly ServiceProvider _serviceProvider;

    public RecordingWorkflowTests()
    {
        // Set up real dependencies
        var services = new ServiceCollection();
        services.AddSingleton<IWasapiRecorder, WasapiRecorder>();
        services.AddSingleton<IWhisperContext, WhisperContext>();
        // ... configure all services
        _serviceProvider = services.BuildServiceProvider();
    }

    [Fact]
    public async Task RecordAndTranscribe_EndToEnd_ShouldProduceTranscription()
    {
        // End-to-end test of recording workflow
    }
}
```

**Test Organization Rules**
- One test class per production class
- Use AAA pattern (Arrange, Act, Assert)
- Test file naming: `{ClassName}Tests.cs`
- Use descriptive test names: `MethodName_Scenario_ExpectedBehavior`
- Mock external dependencies, use real objects for simple classes
- Use `FluentAssertions` for readable assertions

#### 3. Code Style

**Follow Microsoft C# Conventions**
- Use PascalCase for public members
- Use camelCase for private fields with `_` prefix
- Use explicit types (avoid `var` unless obvious)
- Async methods must end with `Async`
- Dispose resources properly (use `IDisposable`, `using` statements)

**Example Service Implementation**
```csharp
public class TranscriptionService : ITranscriptionService, IDisposable
{
    private readonly ILogger<TranscriptionService> _logger;
    private readonly IWhisperContext _whisperContext;
    private bool _disposed;

    public TranscriptionService(
        ILogger<TranscriptionService> logger,
        IWhisperContext whisperContext)
    {
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        _whisperContext = whisperContext ?? throw new ArgumentNullException(nameof(whisperContext));
    }

    public async Task<string> TranscribeAsync(string audioFilePath, CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("Starting transcription for file: {FilePath}", audioFilePath);

        try
        {
            // Implementation
            var result = await _whisperContext.TranscribeAsync(audioFilePath, cancellationToken);
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Transcription failed for file: {FilePath}", audioFilePath);
            throw;
        }
    }

    public void Dispose()
    {
        if (_disposed) return;

        _whisperContext?.Dispose();
        _disposed = true;
    }
}
```

#### 4. Error Handling

**Use Specific Exceptions**
```csharp
public class AudioRecordingException : Exception
{
    public AudioRecordingException(string message) : base(message) { }
    public AudioRecordingException(string message, Exception inner) : base(message, inner) { }
}

// Usage
throw new AudioRecordingException("Failed to initialize WASAPI recorder", ex);
```

**Log All Exceptions**
```csharp
catch (Exception ex)
{
    _logger.LogError(ex, "Failed to record audio from device {DeviceId}", deviceId);
    throw new AudioRecordingException($"Recording failed for device {deviceId}", ex);
}
```

#### 5. Async/Await Guidelines

- Always use `ConfigureAwait(false)` in library code (not UI code)
- Provide `CancellationToken` parameters for long-running operations
- Never use `.Result` or `.Wait()` - use `await`
- Return `Task` or `Task<T>`, not `void` (except for event handlers)

```csharp
public async Task<Transcription> ProcessAudioAsync(
    string audioPath,
    CancellationToken cancellationToken = default)
{
    var audioData = await ReadAudioFileAsync(audioPath, cancellationToken)
        .ConfigureAwait(false);

    var transcription = await TranscribeAsync(audioData, cancellationToken)
        .ConfigureAwait(false);

    return transcription;
}
```

---

## Implementation Workflow

### Phase Implementation Checklist

When implementing each phase from the migration plan:

- [ ] **1. Read Reference Files**
  - Review corresponding Swift files from macOS codebase
  - Understand existing patterns and logic
  - Identify platform-specific vs portable code

- [ ] **2. Design API**
  - Define interfaces first (`I{ServiceName}`)
  - Plan dependency injection setup
  - Design data models (DTOs, entities)

- [ ] **3. Write Tests First (TDD)**
  - Write unit tests for expected behavior
  - Aim for 80%+ coverage before implementation
  - Include edge cases and error scenarios

- [ ] **4. Implement Code**
  - Follow C# conventions
  - Add XML documentation comments for public APIs
  - Use dependency injection
  - Log important events and errors

- [ ] **5. Verify Tests Pass**
  - Run all tests: `dotnet test`
  - Check coverage: `dotnet test /p:CollectCoverage=true`
  - Ensure coverage meets 80% threshold

- [ ] **6. Integration Testing**
  - Test end-to-end scenarios
  - Verify interop with other components
  - Test on real hardware (audio devices, etc.)

- [ ] **7. Code Review Checklist**
  - [ ] All public APIs have XML documentation
  - [ ] No hardcoded values (use configuration)
  - [ ] Proper error handling and logging
  - [ ] Resources properly disposed
  - [ ] Async/await used correctly
  - [ ] No code smells (long methods, deep nesting)

- [ ] **8. Commit**
  - Write descriptive commit message
  - Reference phase number
  - Include testing notes

---

## Porting Reference Guide

### macOS → Windows API Mappings

| macOS Technology | Windows Equivalent | Notes |
|------------------|-------------------|-------|
| **CoreAudio (AUHAL)** | WASAPI (NAudio) | Use loopback capture for system audio |
| **AudioObjectGetPropertyData** | `IMMDeviceEnumerator` | Device enumeration |
| **NSEvent (global monitoring)** | `SetWindowsHookEx` | Low-level keyboard hooks |
| **Carbon (keyboard events)** | `RegisterHotKey` or hooks | Global hotkey registration |
| **CGEvent** | `SendInput` API | Keyboard/mouse simulation |
| **NSWorkspace.shared** | `GetForegroundWindow` | Active window detection |
| **ScreenCaptureKit** | `Windows.Graphics.Capture` | Screen capture (Windows 10 1803+) |
| **NSAppleScript** | Chrome DevTools Protocol | Browser automation (no direct equivalent) |
| **UserDefaults** | `ApplicationData.LocalSettings` | Settings storage |
| **Keychain** | `CredentialManager` | Secure credential storage |
| **SwiftData** | Entity Framework Core | Database ORM |
| **Combine** | `System.Reactive` (Rx.NET) | Reactive programming |

### Common Swift → C# Patterns

**Swift Optional → C# Nullable**
```swift
// Swift
var deviceId: String?
if let id = deviceId {
    print(id)
}
```
```csharp
// C#
string? deviceId = null;
if (deviceId != null)
{
    Console.WriteLine(deviceId);
}
```

**Swift Guard → C# Early Return**
```swift
// Swift
guard let device = audioDevice else { return }
```
```csharp
// C#
if (audioDevice == null) return;
var device = audioDevice;
```

**Swift Protocol → C# Interface**
```swift
// Swift
protocol TranscriptionService {
    func transcribe(audioPath: String) async throws -> String
}
```
```csharp
// C#
public interface ITranscriptionService
{
    Task<string> TranscribeAsync(string audioPath, CancellationToken cancellationToken = default);
}
```

**Swift Enum with Associated Values → C# Class Hierarchy**
```swift
// Swift
enum TranscriptionResult {
    case success(String)
    case failure(Error)
}
```
```csharp
// C#
public abstract class TranscriptionResult { }
public class SuccessResult : TranscriptionResult
{
    public string Text { get; init; }
}
public class FailureResult : TranscriptionResult
{
    public Exception Error { get; init; }
}
```

---

## Critical Implementation Notes

### Phase 1: Audio Recording (WASAPI)

**Key Challenge:** WASAPI is low-level and requires careful buffer management.

**Reference Implementation:**
- Study `VoiceInk/CoreAudioRecorder.swift` lines 1-862
- Pay attention to buffer sizes (whisper.cpp requires 16kHz mono PCM Int16)
- Handle device disconnection gracefully

**Sample Format Configuration:**
```csharp
var waveFormat = new WaveFormat(
    rate: 16000,        // 16kHz for whisper.cpp
    bits: 16,           // 16-bit PCM
    channels: 1         // Mono
);
```

**Testing Requirements:**
- Test with at least 3 different audio input devices
- Test device switching during recording
- Test with no audio devices available
- Test buffer overflow scenarios
- Verify WAV file output format matches whisper requirements

### Phase 2: Whisper.cpp P/Invoke

**Key Challenge:** Memory management across managed/unmanaged boundary.

**Critical Guidelines:**
- Always marshal strings as UTF-8
- Free unmanaged memory properly
- Use `SafeHandle` for native pointers
- Test for memory leaks with long-running sessions

**P/Invoke Pattern:**
```csharp
[DllImport("whisper.dll", CallingConvention = CallingConvention.Cdecl)]
private static extern IntPtr whisper_init_from_file(
    [MarshalAs(UnmanagedType.LPUTF8Str)] string path);

[DllImport("whisper.dll", CallingConvention = CallingConvention.Cdecl)]
private static extern void whisper_free(IntPtr ctx);
```

**Testing Requirements:**
- Test model loading (tiny, base, small models)
- Test memory cleanup (no leaks after 100 transcriptions)
- Test concurrent transcription requests
- Test with corrupted audio files
- Benchmark: < 1s for 5s audio with tiny model

### Phase 3: Global Hotkeys

**Key Challenge:** Windows 11 security restrictions on global hooks.

**Implementation Notes:**
- Require app to run with appropriate permissions
- Provide fallback activation methods (tray icon, window focus)
- Handle hook failures gracefully
- Run hook on dedicated thread with message pump

**Testing Requirements:**
- Test on Windows 10 and Windows 11
- Test with UAC-elevated applications
- Test hotkey conflicts (e.g., Ctrl+Alt+Del)
- Test multi-monitor setups
- Test rapid key presses (debouncing)

---

## Configuration Management

### appsettings.json Structure
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "VoiceInk": "Debug"
    }
  },
  "VoiceInk": {
    "Audio": {
      "DefaultSampleRate": 16000,
      "DefaultBitDepth": 16,
      "DefaultChannels": 1,
      "BufferDurationMs": 100
    },
    "Whisper": {
      "ModelDirectory": "%LOCALAPPDATA%/VoiceInk/Models",
      "DefaultModel": "tiny.en",
      "ThreadCount": 4
    },
    "Hotkeys": {
      "DefaultRecordingHotkey": "Ctrl+Alt+Space",
      "AllowGlobalHooks": true
    },
    "UI": {
      "Theme": "System",
      "MinimizeToTray": true,
      "ShowNotifications": true
    },
    "CloudProviders": {
      "Groq": {
        "BaseUrl": "https://api.groq.com/openai/v1",
        "Model": "whisper-large-v3"
      },
      "OpenAI": {
        "BaseUrl": "https://api.openai.com/v1",
        "Model": "whisper-1"
      }
    }
  }
}
```

### User Settings Storage
- Use `ApplicationData.Current.LocalSettings` for simple key-value pairs
- Use SQLite database for structured data (transcriptions, custom vocabulary)
- Use Windows Credential Manager for API keys (never store in config)

---

## Performance Requirements

### Benchmarks (Target)

| Operation | Target | Measurement |
|-----------|--------|-------------|
| App startup | < 2s | Cold start to main window |
| Audio recording start | < 200ms | From hotkey to recording |
| Transcription (tiny, 5s audio) | < 1s | File to text |
| Transcription (base, 5s audio) | < 3s | File to text |
| UI responsiveness | 60 FPS | During recording |
| Memory usage (idle) | < 100 MB | With no recordings |
| Memory usage (recording) | < 150 MB | Active recording |

### Performance Testing
- Profile with Visual Studio Diagnostic Tools
- Test on minimum spec hardware (4GB RAM, dual-core CPU)
- Monitor for memory leaks (use PerfView)
- Test with large model files (base, small, medium)

---

## Debugging and Troubleshooting

### Common Issues

**WASAPI Recording Issues**
- **Problem:** No audio captured
  - Check: Device permissions, sample rate mismatch, wrong device ID
  - Debug: Enable verbose logging, test with Windows Sound Recorder

**Whisper.cpp Issues**
- **Problem:** DLL not found
  - Check: DLL in correct output directory (x64/ARM64)
  - Debug: Use Dependency Walker to check DLL dependencies

**Hotkey Issues**
- **Problem:** Hotkeys not registering
  - Check: UAC elevation, Windows 11 restrictions, conflict with other apps
  - Debug: Test with simple message box on key press

### Logging Strategy

**Use Structured Logging:**
```csharp
_logger.LogInformation(
    "Recording started: Device={DeviceId}, SampleRate={SampleRate}, Duration={Duration}",
    deviceId, sampleRate, duration);

_logger.LogError(
    ex,
    "Transcription failed: Model={Model}, AudioFile={AudioFile}",
    model, audioFile);
```

**Log Levels:**
- `Trace`: Very detailed, performance-sensitive paths
- `Debug`: Diagnostic information, useful during development
- `Information`: Key events (recording started, transcription completed)
- `Warning`: Recoverable errors, degraded functionality
- `Error`: Failures requiring user attention
- `Critical`: Application-level failures

---

## Dependency Injection Setup

### Registering Services (Program.cs or App.xaml.cs)

```csharp
public static IServiceProvider ConfigureServices()
{
    var services = new ServiceCollection();

    // Logging
    services.AddLogging(builder =>
    {
        builder.AddConsole();
        builder.AddDebug();
        builder.AddSerilog();
    });

    // Configuration
    var configuration = new ConfigurationBuilder()
        .AddJsonFile("appsettings.json")
        .Build();
    services.AddSingleton<IConfiguration>(configuration);

    // Audio services
    services.AddSingleton<IAudioDeviceManager, AudioDeviceManager>();
    services.AddTransient<IWasapiRecorder, WasapiRecorder>();
    services.AddSingleton<IMediaController, MediaController>();

    // Whisper
    services.AddSingleton<IWhisperContext, WhisperContext>();

    // Transcription services
    services.AddSingleton<ITranscriptionServiceFactory, TranscriptionServiceFactory>();
    services.AddTransient<ILocalTranscriptionService, LocalTranscriptionService>();
    services.AddTransient<IGroqTranscriptionService, GroqTranscriptionService>();
    services.AddTransient<IOpenAITranscriptionService, OpenAITranscriptionService>();

    // Core services
    services.AddSingleton<IHotkeyManager, HotkeyManager>();
    services.AddSingleton<IClipboardService, ClipboardService>();
    services.AddSingleton<ITextInjector, TextInjector>();
    services.AddTransient<IRecordingOrchestrator, RecordingOrchestrator>();

    // Data access
    services.AddDbContext<VoiceInkDbContext>(options =>
        options.UseSqlite("Data Source=voiceink.db"));
    services.AddScoped<IHistoryService, HistoryService>();
    services.AddScoped<ISettingsService, SettingsService>();

    // ViewModels
    services.AddTransient<MainViewModel>();
    services.AddTransient<SettingsViewModel>();
    services.AddTransient<HistoryViewModel>();

    return services.BuildServiceProvider();
}
```

---

## Security Considerations

### API Key Storage
- **Never** commit API keys to git
- Use Windows Credential Manager for production
- Use User Secrets for development (`dotnet user-secrets`)

```csharp
// Reading from Credential Manager
var credential = CredentialManager.ReadCredential("VoiceInk_OpenAI_API_Key");
var apiKey = credential?.Password;
```

### Audio Privacy
- Never upload audio without user consent
- Provide clear indicators when recording
- Allow users to delete recordings
- Support local-only mode (no cloud)

### Permissions
- Request microphone permission on first use
- Handle permission denials gracefully
- Provide clear UI for managing permissions

---

## Documentation Requirements

### XML Documentation (Required for All Public APIs)

```csharp
/// <summary>
/// Manages audio input devices and provides device change notifications.
/// </summary>
public class AudioDeviceManager : IAudioDeviceManager
{
    /// <summary>
    /// Gets all available audio input devices.
    /// </summary>
    /// <returns>A list of available audio input devices.</returns>
    /// <exception cref="AudioException">Thrown when device enumeration fails.</exception>
    public List<AudioDevice> GetAvailableInputDevices()
    {
        // Implementation
    }

    /// <summary>
    /// Occurs when the list of available audio devices changes.
    /// </summary>
    public event EventHandler<DeviceChangedEventArgs>? DeviceListChanged;
}
```

### README Updates
- Update main README.md with Windows build instructions
- Document prerequisites (Visual Studio, Windows SDK, .NET SDK)
- Provide troubleshooting guide

---

## Git Workflow

### Branch Strategy
- Main branch: `main` (stable releases)
- Development branch: `develop`
- Feature branches: `feature/phase-{N}-{description}`
- Bug fixes: `fix/{issue-description}`

### Commit Message Format
```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types:** feat, fix, test, refactor, docs, style, perf
**Scope:** audio, whisper, ui, hotkeys, etc.

**Example:**
```
feat(audio): Implement WASAPI recorder with device hot-plug support

- Add WasapiRecorder class with 16kHz mono PCM support
- Implement device enumeration and change notifications
- Add audio level metering
- Include comprehensive unit tests (85% coverage)

Closes #123
```

### Pre-Commit Checklist
- [ ] All tests pass (`dotnet test`)
- [ ] Code coverage ≥ 80%
- [ ] No compiler warnings
- [ ] XML documentation for public APIs
- [ ] Code formatted (run `dotnet format`)

---

## AI Assistant Instructions

### When Working on This Project:

1. **Always Read First:**
   - Read the migration plan phase you're implementing
   - Read corresponding macOS Swift files
   - Understand the existing patterns

2. **Test-Driven Development:**
   - Write tests BEFORE implementation
   - Ensure 80%+ coverage for all new code
   - Run tests frequently during development

3. **Follow the Architecture:**
   - Use dependency injection
   - Follow MVVM for UI code
   - Keep platform-specific code isolated
   - Use interfaces for all services

4. **Code Quality:**
   - Follow Microsoft C# conventions
   - Add XML documentation
   - Handle errors properly
   - Log important events

5. **Ask Questions:**
   - If requirements are unclear, ask before implementing
   - If you find a better approach, propose it
   - If you encounter blockers, document them

6. **Incremental Progress:**
   - Implement one component at a time
   - Test each component before moving on
   - Commit working code frequently

7. **Performance Awareness:**
   - Profile code for performance issues
   - Optimize hot paths (audio recording, transcription)
   - Monitor memory usage

8. **Reference the macOS Code:**
   - The macOS Swift implementation is the source of truth for behavior
   - Port logic, not just syntax
   - Understand WHY the code does something, not just WHAT

---

## Getting Help

### Resources
- **Migration Plan:** [WINDOWS_MIGRATION_PLAN.md](./WINDOWS_MIGRATION_PLAN.md)
- **WinUI 3 Docs:** https://learn.microsoft.com/en-us/windows/apps/winui/
- **WASAPI Guide:** https://learn.microsoft.com/en-us/windows/win32/coreaudio/
- **whisper.cpp:** https://github.com/ggerganov/whisper.cpp
- **NAudio:** https://github.com/naudio/NAudio

### Key Reference Files (macOS)
- `VoiceInk/CoreAudioRecorder.swift` - Audio recording patterns
- `VoiceInk/Recorder.swift` - Recording orchestration
- `VoiceInk/HotkeyManager.swift` - Hotkey management
- `VoiceInk/Services/LocalTranscriptionService.swift` - Transcription service
- `VoiceInk/Whisper/LibWhisper.swift` - Whisper C API usage

---

## Success Metrics

### Phase Completion Criteria
- ✅ All planned components implemented
- ✅ Test coverage ≥ 80%
- ✅ All tests passing
- ✅ Integration tests successful
- ✅ Manual testing on real hardware completed
- ✅ Documentation updated
- ✅ Code reviewed
- ✅ Committed with descriptive message

### MVP Success (Week 5)
- ✅ Can record audio from any input device
- ✅ Can transcribe using whisper.cpp locally
- ✅ Global hotkeys work reliably
- ✅ Basic UI shows transcription results
- ✅ Performance: < 2s app startup, < 1s transcription (tiny model, 5s audio)
- ✅ Test coverage ≥ 80% across all components

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-01-19 | Initial CLAUDE.md created with migration guidelines |

---

**Remember:** This is a complex, long-term migration. Focus on quality over speed, test thoroughly, and maintain the high standards set by the macOS version. The goal is feature parity with excellent Windows integration.

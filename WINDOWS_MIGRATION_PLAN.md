# VoiceInk Windows Migration Plan

## Executive Summary

**Current Status:** VoiceInk is 100% macOS-native (181 Swift files, SwiftUI/AppKit)
**Migration Progress:** ~5-10% (recent CoreAudio modernization, but no Windows-compatible code exists)
**Target:** Windows application with WinUI 3 (C#/XAML) achieving feature parity incrementally

### User Requirements
- **Approach:** MVP first (core features), then iterate to full parity
- **UI Framework:** WinUI 3 (C#/XAML)
- **Timeline:** Flexible/incremental
- **Priority Features:**
  1. Audio recording & transcription (highest)
  2. Global hotkeys (high)
  3. Browser integration (medium)
  4. Screen capture (lower)

### Architecture Decision

**Strategy:** Separate Windows project in C# with shared core logic
- **VoiceInk.macOS:** Existing Swift codebase (maintained)
- **VoiceInk.Windows:** New C# WinUI 3 application
- **Shared:** whisper.cpp (cross-platform C++), cloud API patterns

**Why C# over Swift:**
- WinUI 3 native integration
- Superior Windows API support
- Mature WASAPI audio libraries
- Better tooling and ecosystem

---

## Project Structure

```
VoiceInk/
├── VoiceInk.macOS/              # Existing Swift code
└── VoiceInk.Windows/            # New Windows implementation
    ├── VoiceInk.Core/           # Business logic (platform-agnostic)
    ├── VoiceInk.Audio/          # WASAPI audio recording
    ├── VoiceInk.Whisper/        # whisper.cpp P/Invoke wrapper
    ├── VoiceInk.UI/             # WinUI 3 views & ViewModels
    ├── VoiceInk.Installer/      # MSIX packaging
    └── VoiceInk.Tests/          # Unit & integration tests
```

---

## Phased Implementation Plan

### Phase 0: Project Setup & Infrastructure (Week 1)
**Goal:** Establish development environment and build whisper.cpp

**Deliverables:**
- Create Visual Studio solution structure (5 projects)
- Install WinUI 3 SDK and dependencies
- Build whisper.cpp as Windows DLL using CMake
- Verify basic WinUI 3 app launches

**Testing:** WinUI app runs, whisper.cpp DLL loads, simple transcription works with pre-recorded audio

**Complexity:** Simple | **Dependencies:** None

---

### Phase 1: Core Audio Recording (Weeks 2-3)
**Goal:** Implement WASAPI-based audio recording with device management

**Components to Create:**
1. **`VoiceInk.Audio/WasapiRecorder.cs`** (500-800 lines)
   - Port `CoreAudioRecorder.swift` logic
   - 16kHz mono PCM Int16 format (whisper.cpp requirement)
   - Real-time audio level metering
   - Device hot-plug detection
   - Use NAudio or CSCore library

2. **`VoiceInk.Audio/AudioDeviceManager.cs`** (300-400 lines)
   - Port `AudioDeviceManager.swift`
   - Enumerate input devices
   - Handle device change notifications

3. **`VoiceInk.Audio/MediaController.cs`** (150-200 lines)
   - Port `MediaController.swift`
   - System audio mute/unmute during recording
   - Windows Audio Session API

**Testing:**
- Record 5-second clips from different devices
- Verify WAV format (16kHz mono PCM)
- Test device switching during recording
- Test audio level meter accuracy

**Complexity:** Complex (WASAPI is low-level) | **Dependencies:** Phase 0

**Reference Files:**
- `/home/user/VoiceInk/VoiceInk/CoreAudioRecorder.swift` (862 lines - main recording logic)
- `/home/user/VoiceInk/VoiceInk/Services/AudioDeviceManager.swift`
- `/home/user/VoiceInk/VoiceInk/MediaController.swift`

---

### Phase 2: Whisper Integration (Week 4)
**Goal:** Connect audio recording to whisper.cpp transcription

**Components to Create:**
1. **`VoiceInk.Whisper/WhisperInterop.cs`** (400-500 lines)
   - P/Invoke wrapper for whisper.cpp C API
   - Context management and memory safety

2. **`VoiceInk.Core/Services/LocalTranscriptionService.cs`** (200-300 lines)
   - Port `LocalTranscriptionService.swift`
   - WAV file parsing to float samples
   - Async transcription

3. **`VoiceInk.Core/Models/`** - Port data models:
   - `TranscriptionModel.cs`
   - `Transcription.cs`
   - `ModelProvider.cs`

**Testing:**
- Load whisper models (test with tiny.en)
- Transcribe pre-recorded audio files
- Verify accuracy matches macOS version
- Test multi-threaded safety
- Benchmark transcription speed

**Complexity:** Medium (P/Invoke complexity) | **Dependencies:** Phase 0, 1

**Reference Files:**
- `/home/user/VoiceInk/VoiceInk/Services/LocalTranscriptionService.swift`
- `/home/user/VoiceInk/VoiceInk/Whisper/LibWhisper.swift` (C API patterns)
- `/home/user/VoiceInk/VoiceInk/Whisper/WhisperState.swift`

---

### Phase 3: Global Hotkeys (Week 5)
**Goal:** System-wide keyboard shortcuts for recording

**Components to Create:**
1. **`VoiceInk.Core/Hotkeys/HotkeyManager.cs`** (400-600 lines)
   - Port `HotkeyManager.swift`
   - Use `SetWindowsHookEx` with WH_KEYBOARD_LL
   - Push-to-talk detection (hold key)
   - Support modifier combinations (Ctrl, Alt, Shift, Win)

2. **`VoiceInk.Core/Hotkeys/HotkeyDefinition.cs`**
   - Key definition model
   - Toggle vs PushToTalk modes

**Testing:**
- Test various key combinations (Ctrl+Alt+Space, etc.)
- Test push-to-talk (hold/release)
- Verify no interference with other apps
- Test multiple simultaneous hotkeys

**Complexity:** Medium | **Dependencies:** Phase 0

**Reference Files:**
- `/home/user/VoiceInk/VoiceInk/HotkeyManager.swift` (300+ lines)

---

### 🎯 MVP CHECKPOINT (Week 5)
**Success Criteria:**
- ✅ Record audio from any input device
- ✅ Transcribe with whisper.cpp (locally)
- ✅ Activate via global hotkey
- ✅ Display transcription in console/basic UI
- ✅ Basic device/model selection

---

### Phase 4: MVP UI & Recording Workflow (Weeks 6-7)
**Goal:** Build functional UI for recording and transcription

**Components to Create:**
1. **`VoiceInk.UI/Views/MainWindow.xaml`** (WinUI 3)
   - System tray icon
   - Recording button with visual feedback
   - Audio level meter visualization
   - Transcription result display
   - Settings button

2. **`VoiceInk.UI/ViewModels/MainViewModel.cs`** (400-500 lines)
   - MVVM architecture
   - Recording orchestration
   - Commands: StartRecording, StopRecording
   - Observable transcription results

3. **`VoiceInk.UI/Views/SettingsWindow.xaml`**
   - Audio device selection
   - Hotkey configuration
   - Model selection
   - Output preferences

4. **`VoiceInk.Core/Services/RecordingOrchestrator.cs`** (300-400 lines)
   - Port `Recorder.swift` logic
   - Coordinate audio → transcription → display pipeline

**Testing:**
- End-to-end: Press hotkey → Record → Transcribe → Display
- UI responsiveness during transcription
- Real-time audio meter updates
- Device switching from settings
- Error handling (no mic, model not found)

**Complexity:** Medium | **Dependencies:** Phases 1, 2, 3

**Reference Files:**
- `/home/user/VoiceInk/VoiceInk/Recorder.swift`
- `/home/user/VoiceInk/VoiceInk/Views/Recorder/` (UI inspiration)

---

### Phase 5: Clipboard & Text Injection (Week 8)
**Goal:** Auto-paste transcribed text at cursor

**Components to Create:**
1. **`VoiceInk.Core/Services/ClipboardService.cs`** (150-200 lines)
   - Port `ClipboardManager.swift`
   - Save/restore clipboard
   - Configurable restoration delay

2. **`VoiceInk.Core/Services/TextInjector.cs`** (200-300 lines)
   - Port `CursorPaster.swift`
   - Option 1: Clipboard + Ctrl+V simulation (simple)
   - Option 2: SendInput API (robust)

**Testing:**
- Test paste in various apps (Notepad, Word, Chrome)
- Test clipboard restoration
- Test special characters and Unicode
- Test with clipboard managers

**Complexity:** Simple | **Dependencies:** Phase 4

**Reference Files:**
- `/home/user/VoiceInk/VoiceInk/CursorPaster.swift`
- `/home/user/VoiceInk/VoiceInk/ClipboardManager.swift`

---

### Phase 6: Cloud Transcription Services (Week 9)
**Goal:** Add cloud transcription providers

**Components to Create:**
1. **`VoiceInk.Core/Services/ITranscriptionService.cs`** (interface)

2. **`VoiceInk.Core/Services/CloudTranscription/`** - Port all services:
   - `GroqTranscriptionService.cs`
   - `OpenAITranscriptionService.cs`
   - `DeepgramTranscriptionService.cs`
   - `GeminiTranscriptionService.cs`
   - All are HTTP-based (use HttpClient)

3. **`VoiceInk.Core/Services/TranscriptionServiceFactory.cs`**
   - Service selection logic
   - API key management (Windows Credential Manager)

**Testing:**
- Test each provider with real API calls
- Test error handling (network errors, rate limits)
- Verify audio format compatibility

**Complexity:** Simple (HTTP APIs) | **Dependencies:** Phase 2

**Reference Files:**
- `/home/user/VoiceInk/VoiceInk/Services/CloudTranscription/GroqTranscriptionService.swift`
- `/home/user/VoiceInk/VoiceInk/Services/CloudTranscription/DeepgramTranscriptionService.swift`

---

### Phase 7: AI Enhancement (Week 10)
**Goal:** AI-powered text enhancement (grammar, formatting, style)

**Components to Create:**
1. **`VoiceInk.Core/Services/AIEnhancementService.cs`** (400-500 lines)
   - Port `AIEnhancementService.swift`
   - Context-aware enhancement

2. **`VoiceInk.Core/Models/AIPrompt.cs`**
   - Port prompt templates

3. **`VoiceInk.Core/Services/AI/`**
   - `OpenAIService.cs` (GPT-4, GPT-3.5)
   - `AnthropicService.cs` (Claude)
   - `OllamaService.cs` (local models)

**Testing:**
- Test enhancement quality
- Test different prompts and styles
- Test streaming responses

**Complexity:** Simple | **Dependencies:** Phase 5

**Reference Files:**
- `/home/user/VoiceInk/VoiceInk/Services/AIEnhancement/AIEnhancementService.swift`
- `/home/user/VoiceInk/VoiceInk/Models/AIPrompts.swift`

---

### Phase 8: Data Persistence (Week 11)
**Goal:** Save transcription history and preferences

**Components to Create:**
1. **`VoiceInk.Core/Data/VoiceInkDbContext.cs`** (Entity Framework Core)
   - Transcriptions, WordReplacements, VocabularyWords

2. **`VoiceInk.Core/Services/HistoryService.cs`**
   - CRUD operations for transcriptions

3. **`VoiceInk.Core/Services/SettingsService.cs`**
   - Port UserDefaults to `ApplicationData.Current.LocalSettings`

4. **`VoiceInk.UI/Views/HistoryWindow.xaml`**
   - List, search, filter, export transcriptions

**Testing:**
- Database migrations
- CRUD operations
- Search and filtering
- Settings persistence across restarts

**Complexity:** Medium | **Dependencies:** Phase 4

**Reference Files:**
- `/home/user/VoiceInk/VoiceInk/Models/Transcription.swift`
- `/home/user/VoiceInk/VoiceInk/Views/History/`

---

### 🎯 FEATURE COMPLETE CHECKPOINT (Week 11)
**Success Criteria:**
- ✅ All MVP features
- ✅ Cloud transcription working
- ✅ AI text enhancement
- ✅ Auto-paste at cursor
- ✅ Transcription history saved
- ✅ All core features from macOS version functional

---

### Phase 9: System Integration (Weeks 12-13)
**Goal:** Active window detection, browser URL extraction

**Components to Create:**
1. **`VoiceInk.Core/Services/ActiveWindowService.cs`** (200-300 lines)
   - Port `ActiveWindowService.swift`
   - Use GetForegroundWindow, GetWindowText
   - Window change notifications

2. **`VoiceInk.Core/Services/BrowserAutomationService.cs`** (300-400 lines)
   - Port `BrowserURLService.swift`
   - Chrome/Edge: Chrome DevTools Protocol
   - Firefox: WebExtension native messaging
   - Fallback: Parse window title

3. **`VoiceInk.Core/Services/ScreenCaptureService.cs`** (200-300 lines)
   - Port `ScreenCaptureService.swift`
   - Use Windows.Graphics.Capture API or GDI+

**Testing:**
- Window detection across different apps
- Browser URL extraction (Chrome, Edge, Firefox)
- Screen capture quality and performance
- Multi-monitor support

**Complexity:** Complex | **Dependencies:** Phase 4

**Reference Files:**
- `/home/user/VoiceInk/VoiceInk/PowerMode/ActiveWindowService.swift`
- `/home/user/VoiceInk/VoiceInk/PowerMode/BrowserURLService.swift`
- `/home/user/VoiceInk/VoiceInk/Services/ScreenCaptureService.swift`

---

### Phase 10: Power Mode (Week 14)
**Goal:** Context-aware AI with automatic prompt switching

**Components to Create:**
1. **`VoiceInk.Core/Services/PowerModeManager.cs`** (400-500 lines)
   - Configuration matching (app + URL patterns)
   - Automatic prompt switching

2. **`VoiceInk.Core/Models/PowerModeConfig.cs`**
   - App matchers, URL matchers
   - AI prompts per context

3. **`VoiceInk.UI/Views/PowerModeSettingsWindow.xaml`**
   - Create/edit configurations
   - Test matchers

**Testing:**
- App detection accuracy
- URL matching with wildcards
- Real-world scenarios (email, code, browser)

**Complexity:** Medium | **Dependencies:** Phase 7, 9

**Reference Files:**
- `/home/user/VoiceInk/VoiceInk/PowerMode/` (all files)

---

### Phase 11: Dictionary & Word Replacements (Week 15)
**Goal:** Custom vocabulary and text replacement

**Components to Create:**
1. **`VoiceInk.Core/Services/WordReplacementService.cs`**
   - Port `WordReplacementService.swift`
   - Pattern matching and replacement

2. **`VoiceInk.Core/Services/CustomVocabularyService.cs`**
   - Custom vocabulary for whisper prompts

3. **`VoiceInk.UI/Views/DictionarySettingsWindow.xaml`**
   - Manage replacements and vocabulary

**Testing:**
- Various replacement patterns
- Custom vocabulary impact on accuracy
- Import/export

**Complexity:** Simple | **Dependencies:** Phase 2, 8

**Reference Files:**
- `/home/user/VoiceInk/VoiceInk/Services/WordReplacementService.swift`
- `/home/user/VoiceInk/VoiceInk/Views/Dictionary/`

---

### Phase 12: Advanced UI & Polish (Weeks 16-17)
**Goal:** Complete UI feature parity

**Components:**
1. **Enhanced Main Window**
   - Mini-window mode (always-on-top)
   - System tray with quick actions
   - Notification system
   - Audio playback

2. **Complete Settings Window**
   - All settings in organized tabs
   - Model management (download, delete)
   - API key management
   - Theme support (light/dark)

3. **Additional Windows**
   - Onboarding wizard
   - About window
   - Metrics/usage statistics
   - Log viewer

4. **`VoiceInk.UI/Services/NotificationService.cs`**
   - Windows toast notifications

**Testing:**
- UI/UX with real users
- Accessibility (keyboard navigation, screen readers)
- Multi-monitor support
- Performance (memory, CPU)

**Complexity:** Medium | **Dependencies:** All previous

---

### Phase 13: Installation & Distribution (Week 18)
**Goal:** Packaging, auto-updates, deployment

**Deliverables:**
1. **MSIX Package**
   - App manifest configuration
   - Code signing
   - Installer creation

2. **Auto-Update System**
   - Integrate Squirrel.Windows
   - Version checking
   - Update flow

3. **Documentation**
   - User guide
   - Installation instructions
   - Troubleshooting
   - Developer documentation

4. **CI/CD Pipeline**
   - GitHub Actions for builds
   - Automated testing
   - Release pipeline

**Testing:**
- Installation on clean machines
- Update flow from previous versions
- Windows 10/11 compatibility
- x64 and ARM64 architectures

**Complexity:** Medium | **Dependencies:** All previous

---

### 🎯 PRODUCTION READY (Week 18)
**Success Criteria:**
- ✅ Full feature parity with macOS
- ✅ Installer package working
- ✅ Auto-updates functional
- ✅ Documentation complete
- ✅ Beta testing with 10+ users
- ✅ Performance acceptable (< 100MB RAM idle)

---

## Timeline Summary

| Milestone | Duration | Cumulative | Features |
|-----------|----------|------------|----------|
| **MVP** | 5 weeks | Week 5 | Audio, transcription, hotkeys, basic UI |
| **Feature Complete** | 11 weeks | Week 11 | + Cloud, AI, clipboard, history, all core features |
| **Full Parity** | 18 weeks | Week 18 | + System integration, power mode, polish, distribution |

---

## Critical Files Reference

**Must-read for implementation:**

### Phase 1 (Audio) - Highest Priority
- `VoiceInk/CoreAudioRecorder.swift` (862 lines) - WASAPI equivalent patterns
- `VoiceInk/Services/AudioDeviceManager.swift` - Device management
- `VoiceInk/Recorder.swift` - Recording orchestration

### Phase 2 (Transcription)
- `VoiceInk/Whisper/LibWhisper.swift` - C API usage for P/Invoke
- `VoiceInk/Services/LocalTranscriptionService.swift` - Service patterns
- `VoiceInk/Models/TranscriptionModel.swift` - Data architecture

### Phase 3+ (Features)
- `VoiceInk/HotkeyManager.swift` - Hotkey patterns
- `VoiceInk/Services/AIEnhancement/AIEnhancementService.swift` - AI integration
- `VoiceInk/PowerMode/` - Context-aware system

---

## Risk Mitigation

### High-Risk Areas

1. **WASAPI Audio Recording (Phase 1)**
   - **Risk:** Complex low-level API, device compatibility
   - **Mitigation:** Use NAudio library initially, extensive device testing, fallback to simpler APIs

2. **whisper.cpp Integration (Phase 2)**
   - **Risk:** P/Invoke issues, memory management
   - **Mitigation:** Reference existing .NET whisper wrappers, thorough testing, memory profiling

3. **Global Hotkeys (Phase 3)**
   - **Risk:** Conflicts with other apps, Windows 11 permissions
   - **Mitigation:** Extensive testing, provide alternative activation methods

4. **Browser Automation (Phase 9)**
   - **Risk:** Browser updates breaking integration
   - **Mitigation:** Multiple fallback methods, graceful degradation

---

## Verification Strategy

### After Each Phase:
1. **Unit Tests:** All new services have >= 80% code coverage
2. **Integration Tests:** End-to-end scenarios work
3. **Manual Testing:** Real-world usage scenarios
4. **Performance Testing:** No memory leaks, acceptable CPU usage
5. **Git Commit:** Clean commit with descriptive message

### MVP Checkpoint (Week 5):
- Record 10 audio samples from different devices
- Transcribe with 3 different whisper models
- Activate via 5 different hotkey combinations
- Verify transcription accuracy >= 90% for clean audio
- Test with 3 different users

### Feature Complete (Week 11):
- All core features tested end-to-end
- Cloud providers tested with real API keys
- AI enhancement tested with 10+ text samples
- History tested with 100+ transcriptions
- Performance: < 100MB RAM idle, < 1s transcription (5s audio, tiny model)

### Production Ready (Week 18):
- Beta testing with 10+ users for 1 week
- Zero P0/P1 bugs remaining
- Installer tested on 5+ different Windows machines
- Documentation reviewed and complete
- Performance benchmarks met

---

## Next Steps

1. **Immediate:**
   - Set up development environment (Visual Studio 2022 + WinUI 3)
   - Create GitHub branch for Windows development
   - Begin Phase 0: Project setup

2. **Week 1:**
   - Complete project structure
   - Build whisper.cpp for Windows
   - Verify basic toolchain works

3. **Weeks 2-5:**
   - Focus intensely on MVP (Phases 1-3)
   - Get to working recorder as fast as possible
   - Test early and often

4. **Post-MVP:**
   - Iterate based on user feedback
   - Prioritize features based on actual usage
   - Maintain quality over speed

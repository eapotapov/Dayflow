# Developer Setup Guide

## Prerequisites

### System Requirements
- macOS 13.0 (Ventura) or later
- Xcode 15.0 or later
- 8GB RAM minimum (16GB recommended for local AI)
- 5GB free disk space
- Apple Developer account (free or paid)

### Required Tools
```bash
# Check Xcode version
xcodebuild -version

# Install Xcode Command Line Tools if needed
xcode-select --install

# Install Homebrew (optional, for tools)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

## Setting Up Your Fork

### Option 1: Fork on GitHub (Recommended)
```bash
# 1. Fork on GitHub UI first
# Go to https://github.com/JerryZLiu/Dayflow and click "Fork"

# 2. Clone your fork
git clone git@github.com:YOUR_USERNAME/Dayflow.git
cd Dayflow

# 3. Set up remotes
git remote add upstream git@github.com:JerryZLiu/Dayflow.git
git remote -v  # Verify remotes
```

### Option 2: Private Repository
```bash
# 1. Create new private repo on GitHub

# 2. Change remote
git remote set-url origin git@github.com:YOUR_USERNAME/YOUR_REPO.git

# 3. Keep original as upstream
git remote add upstream git@github.com:JerryZLiu/Dayflow.git

# 4. Push to your repo
git push -u origin main
```

### Option 3: Local Development Only
```bash
# Remove remote for local-only development
git remote remove origin

# You can always add a remote later
```

## Xcode Configuration

### 1. Open Project
```bash
cd Dayflow
open Dayflow/Dayflow.xcodeproj
```

### 2. Configure Signing
1. Select the project in navigator
2. Select "Dayflow" target
3. Go to "Signing & Capabilities" tab
4. Change settings:
   ```
   Team: Your Apple Developer Team
   Bundle Identifier: com.yourname.Dayflow (must be unique)
   Signing Certificate: Development
   ```

### 3. Fix Swift Package Dependencies
If packages show errors:
1. File → Packages → Reset Package Caches
2. File → Packages → Update to Latest Package Versions

## API Keys Setup

### Gemini API Key (for AI features)

#### Get API Key
1. Visit https://aistudio.google.com/app/apikey
2. Click "Create API key"
3. Copy the key (starts with `AIza...`)

#### Add to Xcode
1. Edit Scheme (⌘+<)
2. Select "Run" → "Arguments"
3. Add Environment Variable:
   ```
   GEMINI_API_KEY = your-api-key-here
   ```

#### Alternative: Keychain Storage
```swift
// For production, store in Keychain
KeychainManager.shared.store(apiKey, for: "gemini")
```

### Local AI Setup (Optional)

#### Ollama
```bash
# Install
brew install ollama

# Download models
ollama pull llava  # Vision model
ollama pull llama3  # Text model

# Start server
ollama serve  # Runs on http://localhost:11434
```

#### LM Studio
1. Download from https://lmstudio.ai
2. Install and launch
3. Download a vision model (e.g., BakLLaVA)
4. Enable "Local Server" in settings

## Building and Running

### First Build
1. Clean build folder: ⇧⌘K
2. Build: ⌘B
3. Run: ⌘R

### Grant Permissions
On first run, grant required permissions:
1. Screen & System Audio Recording
2. Automation (if needed)

### Common Build Issues

#### "No account for team"
**Solution**: Add your Apple ID in Xcode → Settings → Accounts

#### "Bundle identifier already exists"
**Solution**: Change to unique identifier like `com.yourname.Dayflow`

#### "Provisioning profile doesn't include entitlement"
**Solution**: Ensure entitlements match your developer account capabilities

## Development Workflow

### Code Style
```swift
// Follow existing patterns
// 4-space indentation
// SwiftUI declarative style
// Async/await for concurrency
```

### Testing Changes

#### Unit Tests
```bash
# Run tests in Xcode
⌘U

# Or from command line
xcodebuild test -scheme Dayflow -destination "platform=macOS"
```

#### Manual Testing
1. Reset onboarding: ⇧⌘R (in app)
2. Clear data: Delete `~/Library/Application Support/Dayflow/`
3. Test with different providers (Gemini, Ollama)

### Debugging

#### Enable Debug Logging
```swift
UserDefaults.standard.set(true, forKey: "llmDebugLogging")
UserDefaults.standard.set(true, forKey: "networkDebugLogging")
```

#### View Logs
```bash
# Console.app
# Filter by "Dayflow" or "[LLMService]"

# Or use log command
log show --predicate 'process == "Dayflow"' --last 1h
```

#### Database Inspection
```bash
# Open database
sqlite3 ~/Library/Application\ Support/Dayflow/chunks.sqlite

# Common queries
.tables
SELECT * FROM timeline_cards ORDER BY created_at DESC LIMIT 10;
SELECT * FROM llm_calls WHERE status = 'failure';
```

## Making Changes

### Adding a New Feature

1. **Plan the implementation**
```markdown
## Feature: Custom Export
- [ ] Add export menu item
- [ ] Create export service
- [ ] Implement JSON/CSV formats
- [ ] Add UI feedback
```

2. **Create feature branch**
```bash
git checkout -b feature/custom-export
```

3. **Implement changes**
- Follow existing patterns
- Add appropriate error handling
- Update relevant documentation

4. **Test thoroughly**
- Unit tests for logic
- Manual testing for UI
- Edge cases and error conditions

5. **Commit with clear messages**
```bash
git add .
git commit -m "feat: add custom export functionality

- Add JSON and CSV export formats
- Implement through File menu
- Include date range selection"
```

### Modifying LLM Prompts

1. **Locate prompt** in `GeminiDirectProvider.swift`
2. **Test with curl** first:
```bash
# Test prompt changes directly
curl -X POST 'https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-pro:generateContent?key=YOUR_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"contents": [{"parts": [{"text": "YOUR_PROMPT"}]}]}'
```
3. **Update in code**
4. **Test with real videos**

### Adding a New Provider

1. **Create provider file**
```swift
// Core/AI/MyProvider.swift
class MyProvider: LLMProvider {
    func transcribeVideo(...) async throws -> (observations: [Observation], log: LLMCall)
    func generateActivityCards(...) async throws -> (cards: [ActivityCardData], log: LLMCall)
}
```

2. **Add to provider types**
```swift
// Core/AI/LLMProvider.swift
enum LLMProviderType {
    case myProvider(config: String)
}
```

3. **Update LLMService**
```swift
// Core/AI/LLMService.swift
case .myProvider(let config):
    return MyProvider(config: config)
```

4. **Add UI configuration**
- Update `LLMProviderSetupView.swift`
- Add to `SettingsView.swift`

## Release Process

### Local Testing Build
```bash
# Build Release configuration
xcodebuild -project Dayflow/Dayflow.xcodeproj \
  -scheme Dayflow \
  -configuration Release \
  build
```

### Create DMG
```bash
# Use the release script
./scripts/release_dmg.sh

# Or manually
# 1. Archive in Xcode
# 2. Export as Developer ID signed app
# 3. Create DMG with create-dmg tool
```

### Distribution
For your fork:
1. Create GitHub Release
2. Upload DMG
3. Share with users

## Syncing with Upstream

### Get Latest Changes
```bash
# Fetch upstream changes
git fetch upstream

# Merge into your branch
git checkout main
git merge upstream/main

# Or rebase your changes
git rebase upstream/main
```

### Resolve Conflicts
```bash
# If conflicts occur
git status  # See conflicted files
# Edit files to resolve
git add .
git rebase --continue
```

## Troubleshooting

### App Crashes on Launch
1. Check Console.app for crash logs
2. Delete `~/Library/Application Support/Dayflow/`
3. Reset UserDefaults: `defaults delete com.dayflow.app`

### Recording Not Working
1. Check System Settings → Privacy → Screen Recording
2. Verify with test: `ScreenRecorder.shared.canRecord`
3. Look for errors in Console.app

### LLM Provider Errors
1. Verify API key is correct
2. Check network connectivity
3. Test with curl command
4. Review `llm_calls` table in database

### Build Failures
1. Clean build folder: ⇧⌘K
2. Delete DerivedData: `~/Library/Developer/Xcode/DerivedData/`
3. Reset packages: File → Packages → Reset Package Caches

## Resources

### Documentation
- [Architecture Overview](architecture.md)
- [Database Schema](database.md)
- [API Providers](api-providers.md)
- [Prompting Guide](prompting.md)

### External Links
- [Swift Documentation](https://www.swift.org/documentation/)
- [SwiftUI Tutorials](https://developer.apple.com/tutorials/swiftui)
- [ScreenCaptureKit](https://developer.apple.com/documentation/screencapturekit)
- [GRDB](https://github.com/groue/GRDB.swift)

### Community
- Original repo: https://github.com/JerryZLiu/Dayflow
- Issues: https://github.com/JerryZLiu/Dayflow/issues
- Discussions: https://github.com/JerryZLiu/Dayflow/discussions

## Getting Help

### Debug Information to Provide
```swift
// App version
Bundle.main.infoDictionary?["CFBundleShortVersionString"]

// macOS version
ProcessInfo.processInfo.operatingSystemVersionString

// Provider type
UserDefaults.standard.data(forKey: "llmProviderType")

// Recent errors
"SELECT * FROM llm_calls WHERE status = 'failure' ORDER BY created_at DESC LIMIT 5"
```

### Creating Bug Reports
Include:
1. Steps to reproduce
2. Expected behavior
3. Actual behavior
4. Screenshots/videos
5. Console logs
6. Database queries if relevant
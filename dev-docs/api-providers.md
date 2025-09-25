# API Providers Documentation

## Overview

Dayflow supports multiple LLM providers for video analysis, allowing users to choose between cloud-based and local processing options. Each provider implements the `LLMProvider` protocol to ensure consistent behavior.

## Provider Protocol

### Core Interface
```swift
protocol LLMProvider {
    func transcribeVideo(
        videoData: Data,
        mimeType: String,
        prompt: String,
        batchStartTime: Date,
        videoDuration: TimeInterval,
        batchId: Int64?
    ) async throws -> (observations: [Observation], log: LLMCall)

    func generateActivityCards(
        observations: [Observation],
        context: ActivityGenerationContext,
        batchId: Int64?
    ) async throws -> (cards: [ActivityCardData], log: LLMCall)
}
```

## Available Providers

### 1. Gemini Direct Provider
**File**: `Core/AI/GeminiDirectProvider.swift`
**Endpoint**: `https://generativelanguage.googleapis.com`

#### Configuration
```swift
// API Key stored in Keychain
KeychainManager.shared.store(apiKey, for: "gemini")

// Provider initialization
let provider = GeminiDirectProvider(apiKey: apiKey)
```

#### Processing Flow
1. **Upload video** to Google's servers
2. **Single API call** for transcription with video URI
3. **Single API call** for card generation
4. **Total**: 2 LLM calls per batch

#### Models Used
- **Primary**: `gemini-2.5-pro` (video transcription)
- **Fallback**: `gemini-2.5-flash` (text processing)

#### API Endpoints
```swift
// File upload
POST https://generativelanguage.googleapis.com/upload/v1beta/files

// Content generation
POST https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent
```

#### Rate Limiting
- Free tier: 2 RPM (requests per minute)
- Paid tier: Higher limits based on quota
- Built-in retry logic with exponential backoff

### 2. Ollama Provider (Local)
**File**: `Core/AI/OllamaProvider.swift`
**Default Endpoint**: `http://localhost:11434`

#### Configuration
```swift
// No API key needed
let provider = OllamaProvider(endpoint: "http://localhost:11434")
```

#### Processing Flow
1. **Extract 30 frames** from video
2. **Generate descriptions** for each frame (30 calls)
3. **Merge descriptions** into observations (1 call)
4. **Generate title** (1 call)
5. **Create activity cards** (1+ calls)
6. **Total**: 33+ LLM calls per batch

#### Models Required
- Vision model for frame analysis (e.g., `llava`, `bakllava`)
- Text model for merging/generation (e.g., `llama3`, `mistral`)

#### Setup Commands
```bash
# Install Ollama
brew install ollama

# Pull required models
ollama pull llava
ollama pull llama3

# Start server
ollama serve
```

### 3. LM Studio Provider
**File**: Uses `OllamaProvider` with different endpoint
**Default Endpoint**: `http://localhost:1234`

#### Configuration
```swift
let provider = OllamaProvider(endpoint: "http://localhost:1234")
```

#### Setup
1. Download LM Studio from https://lmstudio.ai
2. Load a vision-capable model
3. Enable "Local Server" in settings
4. Configure endpoint if not using default port

### 4. Dayflow Backend (Future)
**File**: `Core/AI/DayflowBackendProvider.swift`
**Endpoint**: `https://api.dayflow.app`

Currently placeholder for future hosted service.

## Provider Selection

### Storage
Provider type stored in UserDefaults:
```swift
enum LLMProviderType: Codable {
    case geminiDirect
    case dayflowBackend(endpoint: String)
    case ollamaLocal(endpoint: String)
}
```

### Runtime Selection
```swift
// In LLMService.swift
private var provider: LLMProvider? {
    switch providerType {
    case .geminiDirect:
        return GeminiDirectProvider(apiKey: apiKey)
    case .ollamaLocal(let endpoint):
        return OllamaProvider(endpoint: endpoint)
    // ...
    }
}
```

## API Response Handling

### Gemini Response Format
```json
{
  "candidates": [{
    "content": {
      "parts": [{
        "text": "Generated content here"
      }]
    }
  }]
}
```

### Ollama Response Format
```json
{
  "model": "llava",
  "response": "Generated content here",
  "done": true
}
```

## Error Handling

### Common Error Codes

#### Gemini Errors
- `400`: Invalid API key
- `401`: Unauthorized/expired key
- `403`: Access forbidden
- `429`: Rate limit exceeded
- `500-599`: Server errors

#### Ollama Errors
- Connection refused: Server not running
- Timeout: Model loading or processing too slow
- Invalid response: Model compatibility issues

### Error Recovery
```swift
// Automatic retry with backoff
func retryWithBackoff<T>(
    maxAttempts: Int = 3,
    initialDelay: TimeInterval = 1.0,
    operation: () async throws -> T
) async throws -> T
```

## Performance Comparison

| Provider | API Calls | Latency | Quality | Cost |
|----------|-----------|---------|---------|------|
| Gemini | 2 | 5-10s | High | $0.001/min video |
| Ollama | 33+ | 30-60s | Medium | Free (local compute) |
| LM Studio | 33+ | 30-60s | Medium | Free (local compute) |

## Testing Providers

### Test Connection
```swift
// Built into onboarding flow
func testConnection() async throws -> Bool {
    let testVideo = generateTestVideo()
    let (observations, _) = try await provider.transcribeVideo(
        videoData: testVideo,
        mimeType: "video/mp4",
        prompt: "Test",
        batchStartTime: Date(),
        videoDuration: 15.0,
        batchId: nil
    )
    return !observations.isEmpty
}
```

### Debug Logging
```swift
// Enable detailed logging
UserDefaults.standard.set(true, forKey: "llmDebugLogging")

// Logs include:
// - Request/response bodies
// - Timing information
// - Curl commands for reproduction
```

## Adding New Providers

### Step 1: Implement Protocol
```swift
class MyProvider: LLMProvider {
    func transcribeVideo(...) async throws -> (observations: [Observation], log: LLMCall) {
        // Your implementation
    }

    func generateActivityCards(...) async throws -> (cards: [ActivityCardData], log: LLMCall) {
        // Your implementation
    }
}
```

### Step 2: Add to Provider Type
```swift
enum LLMProviderType: Codable {
    // ...existing cases
    case myProvider(config: MyConfig)
}
```

### Step 3: Update LLMService
```swift
private var provider: LLMProvider? {
    switch providerType {
    // ...existing cases
    case .myProvider(let config):
        return MyProvider(config: config)
    }
}
```

### Step 4: Add UI Configuration
Update `LLMProviderSetupView.swift` to include setup UI for your provider.

## Security Considerations

### API Key Storage
- **Never** hardcode API keys
- Store in Keychain using `KeychainManager`
- Keys are encrypted at rest
- Access controlled by macOS security

### Network Security
- HTTPS for all cloud providers
- Certificate validation enabled
- No custom certificate acceptance

### Local Provider Security
- Localhost-only by default
- No authentication for local models
- User must explicitly enable network access

## Monitoring and Analytics

### Tracked Metrics
```swift
// Per batch processing
AnalyticsService.capture("analysis_batch_completed", [
    "provider": providerName,
    "cards_generated": cardCount,
    "processing_duration_seconds": duration,
    "llm_calls": callCount
])
```

### Performance Monitoring
- Latency per LLM call
- Token usage (when available)
- Error rates by provider
- Success/failure ratios

## Best Practices

### For Cloud Providers
1. Implement rate limiting
2. Cache responses when possible
3. Use appropriate model sizes
4. Monitor API usage/costs

### For Local Providers
1. Ensure adequate RAM (8GB+ for vision models)
2. Use GPU acceleration when available
3. Pre-load models to reduce latency
4. Monitor CPU/memory usage

### General
1. Always validate video duration before processing
2. Implement timeout handling
3. Provide user-friendly error messages
4. Log enough detail for debugging